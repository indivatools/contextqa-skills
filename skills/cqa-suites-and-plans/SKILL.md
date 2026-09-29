---
name: cqa-suites-and-plans
description: Use when organising ContextQA test cases into suites and building the test plans that actually run them — picking browsers or devices, binding an environment, setting timeouts and parallelism, wiring email notifications, and executing or re-running. Also covers folders, tags and modules for keeping a growing suite navigable. Triggers on "group these tests", "create a test plan", "run these on Chrome and Edge", "set up a smoke suite", "schedule the regression", "add these cases to the nightly plan", "/cqa-suites-and-plans".
---

# ContextQA Suites and Test Plans

Two objects, and confusing them costs an afternoon:

- A **test suite** is a named set of test cases. It **cannot be executed.**
- A **test plan** maps suites to browsers or devices, binds an environment, and
  **is the runnable unit** — the thing you schedule, re-run and point CI at.

So the shape is always: cases → suite(s) → plan → run.

## Step 0 — See what exists

```
get_current_workspace
get_test_suites(size=50)
get_test_plans(size=50)          # each plan carries a last_run summary
list_folders(entity_type="test_suite")
```

`get_test_plans` returns `last_run` (`result`, `status`, `duration_ms`,
`total`/`passed`/`failed`) per plan — that is how you tell a maintained plan
from an abandoned one before adding to it. Prefer extending a plan over creating
a near-duplicate; two plans that drift apart is the usual end state.

## Step 1 — Design the suites

A suite should answer one question. Good axes:

| Suite shape | Why it works |
|---|---|
| `Smoke` | the 5–15 cases that must pass before anything else runs |
| by feature — `Checkout`, `Auth`, `Admin` | matches how failures get assigned |
| by risk — `Payments critical` | lets a plan run the expensive subset alone |
| by type — `API contract` | `API_TESTCASE` cases run without a browser |

**A suite has one `test_type`** (`BROWSER`, `MOBILE`, `API_TESTCASE`) and adding
a case of a different type is rejected. Split by type before splitting by
feature.

```
create_test_suite(name="Smoke", test_case_ids=[101,102,103],
                  test_type="BROWSER", tags=["smoke"],
                  folder_id=<from list_folders>)
```

- **`update_test_suite` is append-only and idempotent.** Ids already in the
  suite are reported as `already_present`, new ids are appended; existing
  members are never wiped. Use `remove_test_case_ids` to take cases out.
- **`prerequisite_suite_id`** makes another suite pass first — the portal's
  "Prerequisite cases" setting. This is suite-to-suite; a *case*-level
  prerequisite is `manage_test_case(action="set_prerequisites")` and is what you
  want for a login fixture.
- **`tags` on update replaces the whole list.** Read first if you are adding.
- Deleting a suite still used by a plan is a **409**, and so is permanently
  deleting a case still linked to a suite. That is the platform protecting a
  plan, not an error to route around.

## Step 2 — Keep it navigable while it is still small

- **Folders** for cases and suites: `create_folder`, `move_to_folder`,
  `update_folder` (re-parent with `parent_id`; `0` is root, cycles rejected).
  Names must be unique among siblings. Colour and description are accepted but
  **not shown in the portal** — do not rely on them to communicate anything.
- **Tags** for cross-cutting selection: `manage_test_case(action="add_tags",
  test_case_ids=[…], tags=["release-1"])` works in bulk and is non-destructive.
- **`module_name`** groups a case under a product area and is filterable.
  Writes merge; **a blank string is the only way to clear it** —
  `module_name=""`. Listing `"moduleName"` in `null_fields` does nothing at all.

## Step 3 — Build the plan

```
get_available_devices(device_type="browser")     # device_key values
create_test_plan(
  name="Nightly regression — Staging",
  devices=[
    {"browser": "chrome", "suite_ids": [10, 11], "title": "Chrome all"},
    {"browser": "edge",   "suite_ids": [10],     "title": "Edge smoke"}
  ],
  environment_id=<staging env>,
  parallel_nodes=2,
  element_timeout=30, page_timeout=30,
  notify_emails="qa@acme.com,lead@acme.com"
)
```

What each decision buys you:

- **`devices`** is a list; one plan can run the same suites on several browsers,
  or different suites on each. `browser` takes a `device_key` from
  `get_available_devices` — never a display name you guessed.
- **`environment_id`** is the plan-level Environment in the portal. This is the
  clean way to run one set of suites against Dev, Staging and Prod: **one plan
  per deployment, same suites, different environment.** See `cqa-environments`.
- **`parallel_nodes`** is how many device configurations run at once. With 4
  devices and `parallel_nodes=2`, two run simultaneously.
- **`notify_emails`** is who gets the run report. Accepts one address, a list,
  or a comma-separated string; invalid addresses are rejected *before* any
  write.
- **`knowledge_id`** attaches a knowledge base to guide AI steps. Leave it
  unless a specific KB should apply.
- **Mobile app plans** need `platform: "MOBILE_APP"` **and** `app_upload_id` per
  device — the build is required and a device without one is rejected. Find
  builds with `manage_assets(action="list", query="uploadType:APK")` (or `IPA`)
  and **always ask which build** rather than picking one.

### Editing a plan instead of replacing it

```
manage_test_plan(action="patch", id=N, notify_emails=[...],
                 devices=[{"device_id": 5, "suite_ids": [10, 11, 12]}],
                 remove_device_ids=[7])
```

`patch` merges onto the current plan, so recipients, timeouts and device suite
bindings change without recreating (and orphaning) the plan. A `devices` entry
**with** `device_id` edits that device; **without** one it adds a new device.
Omitting `devices` leaves the list alone. `action="update"` is a full replace —
omitted fields reset to create defaults, which is almost never what you want.

## Step 4 — Run it

```
execute_test_plan(test_plan_id=N, environment_id=<optional override>)
get_test_plan_execution_status(execution_id=E)
rerun_test_plan(execution_id=E)
```

Poll on a widening interval — 30s, backing off to 60s after five minutes — and
print one progress line per poll. Never go silent for more than a minute while a
team is watching a run.

For a single case, `execute_test_case` returns a handle in seconds: share its
`user_message` and `live_url` **before your next tool call**, because a live
link is worthless once the run ends. Then poll
`get_execution_status(session_id=..., wait=True, timeout=60)`.

## Reading the result honestly

- `result` is the executor's verdict — `SUCCESS` / `FAILURE` — and is `null`
  while `is_completed` is false.
- **A `COMPLETED` run with 0 passed and 0 failed executed nothing.** It is not a
  pass.
- **`FAILED_TO_START` is infrastructure, not a red build.**
- Evidence lives on `result_id`: `get_execution_step_details`,
  `get_step_children_details` (loops, groups, IF/ELSE), `get_network_logs`,
  `get_console_logs`, `get_trace_url`, and the video.

Hand a failing run to `/cqa-debug`, or a whole failing plan to
`/cqa-regression`.

## Limits worth knowing before you promise something

- **There is no scheduling tool in the MCP.** `POST /schedule_test_plans` exists
  on the platform and the portal drives it, but the MCP does not expose it — set
  schedules in the portal, or trigger `execute_test_plan` from your own CI.
- **An environment can be overridden per run; test data never can.** A run that
  needs different rows needs a different binding on the case.
- **A workspace version with no default test plan cannot execute anything**;
  `execute_test_case` answers 400 with an empty body. Prove one throwaway run
  works before authoring fifteen cases into that workspace.
- **`MOBILE` on a plan without the mobile entitlement is rejected.** On **Ship**
  (`ship.contextqa.com`, the self-serve product) mobile testing is not included
  at launch at all — it is a Pro upgrade, so do not design a mobile plan for a
  Ship org.

## A shape that works

1. `Smoke` suite — 5–15 cases, one plan, chrome only, Staging environment,
   `parallel_nodes=1`. Runs on every merge.
2. `Regression` plan — every feature suite, chrome + edge, Staging,
   `parallel_nodes=2`, notifying the QA alias. Runs nightly.
3. `Prod verify` plan — the same `Smoke` suite, Production environment,
   read-only cases only. Runs after deploy.

Three plans, two suites shared across them, one environment binding each. That
is the whole pattern; everything else is more of the same.
