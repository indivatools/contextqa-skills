---
name: cqa-regression
description: Use when the goal is to run a ContextQA test plan as a regression, wait for it, and triage the failures in parallel. Inputs are a test plan id, a plan name to look up, or a freeform "run the regression" intent. Clusters failures by shared cause, dispatches read-only triage, and hands confirmed bugs to /cqa-debug. Triggers on "run the regression", "execute test plan X", "run my smoke suite and triage", "kick off nightly tests", "/cqa-regression <plan>".
---

# ContextQA Regression Run & Triage

Launch a plan, watch it, triage the failures. **Don't debug inline** — the value
of this skill is separating twelve failures into two causes, not fixing one.

## Step 0 — Resolve the plan

One of:

- a `test_plan_id` — use it
- a name — `get_test_plans(query=<name>)`, pick the best match; list and ask if
  several are plausible
- nothing — `get_test_plans(size=10)` and offer the top few

`get_test_plans` returns a `last_run` summary per plan (`result`, `status`,
`duration_ms`, `total`/`passed`/`failed`). Use it to tell a maintained plan from
an abandoned one before running something expensive.

Check the plan's `environment_id` before launching, and say which deployment
this run will hit. A regression against the wrong environment produces a page of
real-looking failures that mean nothing.

## Step 1 — Confirm, then launch

A regression consumes shared infrastructure — browsers, devices, AI step
interpretation, credits. **Get explicit consent even when a plan id was
supplied.** Print the plan (id, name, devices, environment, last run) and ask:
*"Kick off plan `<id>` '`<name>`' against `<environment>`? (y / no)"*

```
execute_test_plan(test_plan_id=N, knowledge_id=<optional>, environment_id=<optional override>)
```

Record the `execution_id`. To repeat a previous run:
`rerun_test_plan(execution_id=<prev>)`.

## Step 2 — Poll out loud

Loop `get_test_plan_execution_status(execution_id)` — every 30s, backing off to
60s after five minutes. Print one line per poll:
`running 12/30 cases — 4 passed, 1 failed`. Stop on `COMPLETED`, `STOPPED` or
`FAILED`.

**Never go silent for more than a minute** while a team is watching.

## Step 3 — Collect, and check it actually ran

Extract the failed `(result_id, test_case_id, case_name, failing_step_preview)`.

Before reporting anything: **a run that `COMPLETED` with 0 passed and 0 failed
executed nothing.** That is not a green regression — it is a plan with no
runnable cases, or an infrastructure failure. `FAILED_TO_START` likewise is
infrastructure, not a red build. Say which one it is.

Zero *real* failures → Step 6.

## Step 4 — Cluster

Group by, in order:

- identical failing-step text or selector
- same network host or endpoint
- same console error signature
- otherwise, one cluster per case

Cap at 5 clusters; the rest go in `MISC`.

Before dispatching, sanity-check for a single environmental cause. If every
failure is on the first step, or every one is a timeout, the answer is usually
one thing — a missing environment variable, an expired credential, the app under
test being down — not five independent bugs. Check
`list_environment_variables(environment_id=<plan env>)` first; it is one call
and it saves five subagents.

## Step 5 — Triage in parallel (read-only)

One subagent per cluster, in a single message. Brief each with the cluster name,
its `result_id`s, and the failing-step preview.

Tools, all **read-only**: `investigate_failure`, `get_execution_step_details`,
`get_step_children_details`, `get_network_logs`, `get_console_logs`,
`get_trace_url`, `fix_and_apply`.

Required output (~150 words): cluster verdict (`shared root cause` /
`independent issues`), a 1–3 sentence root cause, the file or component most
likely at fault, and a recommended action — `code-fix` / `update-test` /
`environment` / `flaky-rerun` / `unknown`.

**Triage subagents must not mutate code, edit cases or rerun the plan.** Fixes
go through `/cqa-debug`.

## Step 6 — Report

1. **Summary** — total, passed, failed, duration, plan id, `execution_id`,
   environment, and the portal links returned by the tools. Don't construct
   portal URLs by hand; they are tenant-specific.
2. **Per-cluster verdicts** — root cause → action → cases (id, name,
   `result_id`).
3. **MISC** — one line each.
4. **Next commands**:
   - `code-fix` → `/cqa-debug result_id=<id>`
   - `update-test` → `/cqa-locators`, or `/cqa-impact` with the change that
     caused it
   - `environment` → `/cqa-environments`
   - `flaky-rerun` → `rerun_test_plan(execution_id=<this>)`

If asked to file tickets, draft the bodies — one subagent per cluster — and
**do not post without explicit approval.**

## Rules

- Triage subagents are read-only.
- Always print poll progress; no silent waits over 60s.
- Don't auto-rerun on first failure. Flakiness is a verdict you reach, not a
  default you assume.
- Stop at triage. `/cqa-debug` does fixes.
- Cap at 5 parallel triage subagents; the rest go in `MISC`.
- Report the run honestly, including "this executed nothing".
