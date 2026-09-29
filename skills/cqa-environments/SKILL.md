---
name: cqa-environments
description: Use when setting up or fixing ContextQA environments, base URLs, secrets, global variables or test-data profiles — or when a case needs to run against Dev, Staging and Prod without being rewritten. Covers the `*|var|` / `${var}` / `{{var}}` / bare-column token grammar, the read-only vs read-write toggle, password handling, per-run environment overrides, and data-driven cases. Triggers on "set up environments", "put the base url in an environment", "add a secret", "parameterise this test", "run this against staging", "/cqa-environments".
---

# ContextQA Environments, Variables and Test Data

One rule holds the whole skill together: **nothing that changes between
deployments belongs in a step.** Hostnames, credentials, tenant ids, feature
flags and seed data all live outside the case, so the same case runs everywhere.

A hardcoded URL is the most expensive mistake here because it does not fail — it
passes against the old deployment. ContextQA's own PR-impact preflight ranks
`FIXED_URL` as **more** misleading than a missing variable for exactly this
reason.

## Step 0 — Look before you create

```
get_current_workspace
list_environments(include_parameters=True)   # pages to completion; secrets masked
list_environment_variables(environment_id=N) # one environment's variable names
manage_global_variable(action="list")
manage_test_data(action="list")
```

An environment named `" QA "` now collides with `"QA"` — names are trimmed
before the uniqueness check, and a blank name is rejected. Reuse before you
create.

## Step 1 — Choose the right token for each value

| Token in step text | Kind | Scope | What you pass with it |
|---|---|---|---|
| `*\|base_url\|` | environment variable | one environment | an `environment_id` that **defines** it |
| `${order_id}` | runtime / test-case variable | one case | a value in `variables` |
| `{{support_email}}` | global variable | workspace version | **nothing** |
| `login_email` (bare) | test-data profile column | the bound profile | the profile must have that column |
| `${result.body.id}` | API response chaining | one case | not a variable — an earlier REST step's response |

Three consequences worth memorising:

- **A profile column is bare.** Wrapping it — `*|login_email|` — produces
  `"login_email variable is not present in the … Environment"`.
- **`testDataType` is `"raw"` for an environment variable**, not
  `"environment"`. `*|url|` resolves against the bound `environmentId`
  regardless; `"environment"` produces a dead step.
- **A global variable takes no binding at all.** If you find yourself passing an
  id for a `{{name}}`, you have the wrong token.

## Step 2 — Create the environment set

Give every environment the same variable *names* and different values. That is
what makes one case portable.

```
manage_environment(action="create", name="Staging", parameters={
  "base_url":  {"type": "text",     "value": "https://staging.acme.com"},
  "username":  {"type": "text",     "value": "qa@acme.com", "permissionMode": "READ_ONLY"},
  "password":  {"type": "password", "value": "<supplied by the user>"},
  "tenant_id": {"type": "number",   "value": 42}
})
manage_environment(action="set_default", environment_id=N)
```

- **Types** are `text`, `number`, `boolean`, `password`, `vault`. Anything else
  is rejected — and a near-miss like `secret` would *not* be secret-handled, so
  the rejection is protecting you.
- **`permissionMode`** is the portal's RO/RW toggle: `READ_ONLY` (aliases
  `READ`, `RO`) or `READ_WRITE` (the default). Mark anything a test should never
  write — credentials, tenant ids — as `READ_ONLY`.
- **`vault` params are manual provisioning only.** Do not write them through
  `merge_params`.
- **Adding to an existing environment:** use
  `manage_environment(action="merge_params", ...)`, which never overwrites an
  existing key unless you pass `overwrite=True`. A full `update` is a PUT and
  replaces everything.
- **`manage_environment` reports HTTP 202 as an error** though the write
  succeeded. Re-read with `action="get"` before believing a failure.

**Every test case carries an environment, always** — even one with no tokens.
`create_test_case` binds the `isDefault` environment, else the first in the
workspace, else it creates one called `Default`. Setting a sensible default is
therefore a real decision, not housekeeping.

## Step 3 — Global variables for the things that do not vary

Values that are the same in every environment — a support address, a fixed SKU,
a copy string you assert on — belong in globals, not duplicated four times.

```
manage_global_variable(action="upsert", name="support_email",
                       var_type="TEXT", value="help@acme.com")
```

`upsert` is keyed on `(name, workspace_version_id)`: a POST creates if the name
is new and updates if it is not. `PASSWORD`-typed values are never echoed back
and never logged.

## Step 4 — Test-data profiles for data-driven cases

A profile is a table; a data-driven case runs once per row.

```
manage_test_data(action="create", test_data_name="Logins", data=[
  {"name": "valid",   "expectedToFail": False, "data": {"login_email": "a@acme.com", "login_password": "..."}},
  {"name": "blocked", "expectedToFail": True,  "data": {"login_email": "b@acme.com", "login_password": "..."}}
])

manage_test_case(action="bind_test_data", id=<case>, is_data_driven=True,
                 test_data_id=<profile>, test_data_start_index=0, test_data_end_index=1)
```

- The platform's raw `POST /test_data` **persists only the profile shell and
  drops `rows`**, and its create must set `versionId` or the profile is
  invisible in the portal. `manage_test_data(action="create")` does the
  two-write dance and sets the id for you — which is why you should use the
  tool rather than hand-rolling the HTTP call.
- In step text a profile column is **bare** (`login_email`) and
  `testDataType` is `"test_data"`.
- **Password-typed columns come back masked on every read path.** There is no
  unmask parameter on authoring or export paths, deliberately.
- `test_data_id=0` clears the binding; `manage_test_case(action="bind_test_data",
  is_data_driven=False, ...)` unbinds atomically.
- **A data-driven case returns `steps: []` from `get_execution_step_details`.**
  Read its verdict from `get_test_case(id).last_run` instead
  (`total_steps` / `passed_steps` / `result`).

## Step 5 — Bind at the right level

| Bind an environment on… | How | When |
|---|---|---|
| the test case | `create_test_case(environment_id=N)` | the case's normal home |
| a single step | `manage_test_step` `environment_id` on an nlp step | one step reads a different environment |
| the test plan | `create_test_plan(environment_id=N)` / `manage_test_plan(action="patch")` | the whole plan targets one deployment |
| one run only | `execute_test_case(environment_id=N)` / `execute_test_plan(environment_id=N)` | a one-off check against another deployment |

**Test data can never be overridden at run time** — only the environment can.
If a run needs different rows, it needs a different binding on the case.

The usual shape for a team is: one plan per deployment, each binding its own
environment, all pointing at the same suites.

## Credentials — the consent gate

For QA the app password usually has to reach the automation. Handle it
explicitly, never silently:

1. **Warn** that the credential will be used to log into the app under test.
2. **Get explicit permission.**
3. **Confirm it is not production** and there are no billing consequences.
4. Then use it — storing it as a `password`-typed environment parameter or
   global variable, never in step text, never in a file, never in the
   transcript. Filter `password` / `token` / `authorization` / `bearer` /
   `apikey` / `cookie` out of anything you print.
5. **Always offer the alternative:** the user signs in themselves in a visible
   browser and you take over the session, so the password never reaches you.

## Verify before you move on

1. `list_environment_variables(environment_id=N)` — every name the steps use is
   present, in **every** environment you intend to run against. A name that
   exists in Staging and not in Dev is a case that passes in one and fails in
   the other for a reason unrelated to the product.
2. Run one throwaway case that navigates to `*|base_url|` and asserts something
   on the landing page, once per environment.
3. Grep your own steps for `http://` and `https://` literals. Every hit is a
   `FIXED_URL` waiting to pass against the wrong deployment.

## Known sharp edges

- **Scoping conventions differ per endpoint.** A hand-written workspace
  parameter can be ignored rather than rejected, leaving you with a listing that
  is not scoped the way you assumed. Let the MCP build the call rather than
  hitting `/environments`, `/test_data` or `/global_variables` yourself.
- `/test_data` rows carry **both** `versionId` and `workspaceVersionId`
  columns and the product writes `versionId`. Probe before trusting a
  test-data listing to be complete.
- Secret values arrive masked as `****` on reads. That is the platform
  masking them, not an empty variable.

Read `get_contextqa_skill(name="contextqa-platform")` for the scoping rules
behind all of this, and `cqa-locators` for how a bound variable reaches a
typed step.
