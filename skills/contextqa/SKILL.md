---
name: contextqa
license: MIT
description: >
  Drive ContextQA from any coding agent: author test cases that actually run,
  manage environments and test data, group cases into suites, execute them as
  test plans, read real verdicts and evidence, and wire the results back into
  pull requests and tickets. Use when the task involves end-to-end or API
  testing, QA coverage for a change, reproducing or verifying a bug, running a
  regression, or exposing a locally running app or browser to the test runners.
  This is the entry-point skill: it explains how the platform fits together and
  routes to the specialist skill for the job.
---

# Build and run tests with ContextQA

ContextQA is an AI-native test automation platform. A test case is an ordered
list of **natural-language steps** that a runner turns into real browser, mobile
or API actions. You drive all of it through the **ContextQA MCP server** — one
connection, ~87 tools, live against a real tenant.

Every call in this skill mutates or reads a real customer workspace. There is no
sandbox mode. Orient before you write.

## Read the live skills — they are the source of truth

**The MCP server ships its own skill catalogue and serves whatever version it is
running.** That catalogue is newer than anything installed on your side, and it
carries the wire formats, the tenant-specific ids and the current list of known
defects. Read from it as part of the task; do not work from memory.

```
list_contextqa_skills                      # the live catalogue + rough token cost
get_contextqa_skill(name="contextqa-platform")
```

| Before you… | Read |
|---|---|
| touch anything, if you don't know the platform | `contextqa-platform` — tenancy, scoping, object graph, token grammar, the real execution path |
| write or edit a step | `cqa-authoring-tests` — the wire format that runs, locator sources, the execute-and-fix loop |
| move a case between workspaces or tenants | `cqa-copy-test-case` |
| go from a ticket to a reproduced, fixed bug | `cqa-bug-fix` |
| turn cases into a Playwright project | `cqa-export-playwright` |
| drive the PR-impact pipeline | `cqa-pr-impact` |
| generate cases from an n8n workflow | `n8n-testing` |

If the catalogue is unreachable, say so, work from the sections below, and avoid
inventing tenant-specific values — especially `naturalTextActionId` numbers,
which differ per tenant.

## Orient before you act

Three calls, always, at the start of a session:

```
get_current_user          # org (tenant) + signed-in user + active workspace
get_current_workspace     # the workspace version nearly every API wants
list_workspaces           # → switch_workspace(workspace_version_id=N)
```

Never infer the org from the API host or from data that came back. A test case's
workspace is baked into its record and cannot be moved later, so author in the
workspace that will run it. `GET /workspaces` lists workspaces you cannot open,
so a workspace can appear in the list and then refuse to be switched into.

## The object graph

```
Organization (tenant)  →  Workspace  →  Workspace version   ← workspace_version_id
    ├── Environment        named variables; every case binds one
    ├── Global variable    workspace-scoped, resolved by name
    ├── Element / Screen   named locators, grouped by page
    ├── Test-data profile  rows for data-driven cases
    ├── Test case          BROWSER | MOBILE | API_TESTCASE
    │     └── Steps        typed NLP · AI agent · step group · REST
    ├── Test suite         a set of cases — groups, never runs
    └── Test plan          suites × devices × environment — the runnable unit
          └── Run → test_case_result → test_step_result (+ logs, video, trace)
```

## Four things that look like variables

Getting one wrong is the classic "saved fine, failed at run time".

| Token | Kind | What you pass alongside it |
|---|---|---|
| `*\|name\|` | Environment variable | an `environment_id` that **defines** `name` |
| `${name}` | Runtime / test-case variable | a value in `variables` (rejected before create if undeclared) |
| `{{name}}` | Global variable | **nothing** — workspace-scoped, resolved by name |
| `name` (bare) | Test-data profile column | the bound profile must have that column |
| `${result.body.id}` | API response chaining | not a variable — an earlier REST step's response |

## How to build something that lasts

These are the habits that separate a suite people keep from one they abandon.

**Put the base URL in an environment, never in a step.** A literal address in a
step is the worst kind of green test: it passes, having exercised the old
deployment. Create `*|base_url|` per environment (Dev, Staging, Prod) and let
the same case run everywhere by binding a different environment. The PR-impact
preflight flags a hardcoded URL (`FIXED_URL`) as more dangerous than a missing
variable, for exactly this reason — see `cqa-environments`.

**Prefer typed steps with real locators.** A typed step (`navigate`, `enter`,
`click`, `verify`) is deterministic, diffable and repairable one field at a
time. An AI-agent step re-derives its own actions on every run. Use AI to
*bootstrap* a page you have never seen — executing an AI step converts the case
in place into typed steps with captured locators — then hand-author from there.
**Never "fix" a case by deleting it and dropping in one AI step.** See
`cqa-locators`.

**Suites group; plans run.** A suite cannot be executed. Build suites around a
theme (smoke, checkout, auth), then a plan that maps suites to browsers or
devices and binds an environment. Plans are what you schedule, rerun, and point
at from CI. See `cqa-suites-and-plans`.

**Organise as you create, not later.** Folders for cases and suites, tags for
cross-cutting selection (`smoke`, `release-1`), `module_name` for the product
area. `move_to_folder` and `add_tags` are cheap; a flat list of 400 cases is
not.

**Close the loop with the integrations.** Connect GitHub/GitLab, Slack, Linear
or Jira so impact analysis runs on real pull requests and results land where the
team already is. See `cqa-integrations`.

**Test what is actually running.** If the app under test only exists on a
laptop or a build agent, publish it through the ContextQA tunnel and use the
tunnel URL as the environment's base URL — the same agent can also host a local
browser so runs execute inside the network. See `cqa-tunnel`.

**Treat the app's password as a decision, not a detail.** For QA the credential
usually has to reach the automation. Warn, get explicit permission, confirm it
is not production and carries no billing consequence, then use it — and never
write it to a file, a commit or the transcript. Always offer the alternative
where the user signs in themselves and you take over the session.

## Running, and reading the answer

```
execute_test_case(test_case_id=N)        # returns a handle in seconds
```

Reply to the user with `user_message` and `live_url` **before your next tool
call** — a link to a live run is worthless once the run ends. Then poll
`get_execution_status(session_id=..., wait=True, timeout=60)` and post a line
each time it comes back `timed_out`.

`result` is the executor's own verdict — `SUCCESS` / `FAILURE` — and it is
`null` while `is_completed` is false. **Never read "a new result row appeared"
as a pass.** On failure, `get_execution_step_details(result_id)` names the
failing step and `failure_reason` says what broke; `get_network_logs`,
`get_console_logs` and `get_trace_url` carry the rest of the evidence.

For a test plan: `execute_test_plan` → `get_test_plan_execution_status` →
`rerun_test_plan`.

## Things that read as success and are not

- A **`COMPLETED` run with 0 passed and 0 failed executed nothing.**
- **`preflight: null` means "not checked", not "clean."**
- **`FAILED_TO_START` is infrastructure, not a red build.**
- **`configured: false` on credits means there is no credit account** — never a
  balance of zero.
- **Scoping conventions differ per endpoint**, so a hand-written workspace
  parameter can be ignored rather than rejected — and you get a listing that
  isn't scoped the way you assumed. Let the MCP build the call.
- **`GET /test_cases` has no deleted-guard** — a count that disagrees with the
  portal is usually deletions, not missing rows.
- An integration row **can outlive the provider-side install**; only
  `manage_integrations(action="registry")` sees the truth.

Errors are worth reading rather than retrying: the platform's most common
rejection is `{"error": "One or more fields are invalid.", "fieldErrors": [...]}`
where only `fieldErrors` says anything. Do not branch on `code` — the numeric
enum is frozen and most new errors carry `code: null`. The MCP funnels all of
this through one describer, so a tool error is already a sentence.

## Where to go next

| You want to… | Skill |
|---|---|
| Check the MCP is wired up and see what the tenant holds | `cqa-init` |
| Set up environments, base URLs, secrets, test data | `cqa-environments` |
| Author cases from a ticket, spec, Figma, video or diff | `cqa-author` |
| Author precise typed steps against known locators | `cqa-locators` |
| Group cases and build a runnable plan | `cqa-suites-and-plans` |
| Run a plan and triage the failures | `cqa-regression` |
| Diagnose and fix one failing case or bug | `cqa-debug` |
| Work out what a PR or ticket breaks | `cqa-impact` |
| Hunt for bugs in a deployed UI at breadth | `cqa-bug-hunter` |
| Connect GitHub/GitLab/Slack/Linear/Jira | `cqa-integrations` |
| Test a local app, or run in the customer's network | `cqa-tunnel` |

Each of these is a workflow. The platform ground truth always comes from
`get_contextqa_skill` on the live server.
