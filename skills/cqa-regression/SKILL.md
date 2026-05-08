---
name: cqa-regression
description: Use when the goal is to run a ContextQA test plan as a regression suite, wait for results, and triage failures across multiple subagents in parallel. Inputs are a test plan id, a plan name to look up, or a freeform "run the regression suite" intent. Triggers on phrases like "run the regression", "execute test plan X", "run my smoke suite and triage", "kick off nightly tests", "/cqa-regression <plan>".
---

# ContextQA Regression Run & Triage

Launch a plan, wait for it, triage failures via parallel subagents. Don't debug failures inline — that's the subagents' job.

## Step 0 — Resolve the plan

ONE of:
- `test_plan_id` → use directly
- `plan_query` → `get_test_plans(query=<plan_query>)`, pick best match; if multiple are plausible, list and ask once
- nothing → `get_test_plans()`, offer top 5

Optional: `knowledge_id`.

`get_test_plans` returns each plan with a `last_run` summary (`result`, `status`, `duration_ms`, `total/passed/failed`) — use it to spot recently-passing plans vs. stale ones when picking.

## Step 1 — Confirm and kick off

A regression run consumes shared infrastructure (browsers, devices, AI step interpretation) — get explicit user consent before kickoff, even if a plan id was supplied. Print the resolved plan summary (id, name, last_run result + start_time) and ask: *"Kick off plan <id> '<name>'? (y / no)"*. Wait.

Then `execute_test_plan(test_plan_id=<id>, knowledge_id=<id-or-omit>)`. Record `execution_id`.

If re-running a previous execution: `rerun_test_plan(execution_id=<prev>)`.

## Step 2 — Poll

Loop on `get_test_plan_execution_status(execution_id)` — start at 30s, back off to 60s after 5 minutes. Print one progress line per poll: `running 12/30 cases — 4 passed, 1 failed`. Stop when status is `COMPLETED`, `STOPPED`, or `FAILED`.

Never go silent for >60s during this step.

## Step 3 — Collect failures

From the final status, extract failed `(result_id, test_case_id, case_name, failing_step_preview)`. If zero failures → Step 6.

## Step 4 — Cluster

Group failures by:
- Same failing-step text or selector
- Same network host / endpoint
- Same console error signature
- Otherwise one cluster per case

Cap at 5 clusters. Leftovers go in `MISC`.

## Step 5 — Triage in parallel (subagents, ONE Agent message)

One subagent per cluster. Brief:
- Cluster name, the `result_id`s, the failing-step preview
- Tools (READ-ONLY): `investigate_failure`, `get_execution_step_details`, `get_step_children_details`, `get_network_logs`, `get_console_logs`, `get_trace_url`, `fix_and_apply`
- Output (~150 words): cluster verdict (`shared root cause` / `independent issues`), 1–3 sentence root-cause hypothesis, the file/component most likely at fault, recommended next action — `code-fix` / `update-test` / `flaky-rerun` / `unknown`

Triage subagents MUST NOT mutate code, edit cases, or rerun the plan. Hand off to `/cqa-debug` for actual fixes.

## Step 6 — Final report

1. **Summary** — N total, P passed, F failed, duration, plan id, `execution_id`, link `https://contextqatest.contextqa.com/td/runs/<execution_id>`
2. **Per-cluster verdicts** — root cause → action → cases (id, name, result_id, execution_url)
3. **MISC** — one line each
4. **Suggested next commands**:
   - `code-fix` → `/cqa-debug result_id=<id>`
   - `flaky-rerun` → `rerun_test_plan(execution_id=<this>)`
   - `update-test` → `/cqa-impact` with the PR/ticket that introduced the change

If the user asked to file tickets: dispatch one subagent per cluster to draft a Jira/Linear/GH issue body. Do NOT post tickets without explicit approval.

## Rules

- Triage subagents are READ-ONLY.
- Always print poll progress; no silent waits >60s.
- Don't auto-rerun on first failure — flakiness is a verdict, not a default.
- Stop at triage. Hand off to `/cqa-debug` for fixes.
- Cap at 5 parallel triage subagents; rest go in `MISC`.
- All ContextQA tool responses are wrapped as `{"result": "..."}`; status polls may be plain text — match on `completed` / `failed` substrings.
