---
name: cqa-debug
description: Use when there is a failing ContextQA test case or execution to diagnose and fix. Inputs can be a `result_id`, a `test_case_id` whose latest run failed, a plan `execution_id` with failures, or a bug ticket that needs to be reproduced first. Investigate telemetry in parallel → root-cause hypothesis → subagent fix → verify → report. Triggers on phrases like "debug this failure", "fix this failing test", "why did test result <id> fail", "reproduce and fix this bug", "/cqa-debug <ref>".
---

# ContextQA Debug & Fix

Cap fix attempts at 3. Never loop indefinitely.

## Step 0 — Resolve the failure

Capture or ask once:
- `result_id` → go to Step 2
- `test_case_id` → `get_test_case_results(test_case_id=...)` to find latest `result_id`
- plan `execution_id` → `get_test_plan_execution_status(...)`, pick failed `result_id`s; if many, hand off to `/cqa-regression`
- `ticket` (no test yet) → Step 1

Also capture `app_url` and `deployment_info`. **Default: local file saves are immediately live at `app_url` — no build step.** Override only if the user says so.

## Step 1 — Reproduce (only if input is a ticket)

1. Fetch ticket body. Order of preference: Linear MCP / Jira MCP / GitLab MCP → `gh issue view <n>` / `glab issue view <n>` → `curl` against the tracker REST API. If none available, ask once: *"I couldn't fetch <ref>. Install the matching MCP (e.g. `mcp install linear`) for best fidelity, OR paste the ticket body and I'll continue."* Plain-text bodies are fully supported — `bug_fix_from_ticket` accepts them natively. Never pass a raw URL forward.
2. `bug_fix_from_ticket(ticket_text=<body>, url=<app_url>, deployment_info=...)` returns a bundle with `test_case_id`, `task_description`, `session_rules[]`, `fix_guide[]`, and `note`. The execution is queued; `execution_url` will be `null` until you poll.
3. **Treat the returned `session_rules` as authoritative** — when they're present they override the default rules in this skill (e.g. deployment assumption, FAILED ≠ deployment lag, every change must be followed by `verify_bug_fix`).
4. Follow the steps in `fix_guide` — they are the canonical loop for that session.
5. Continue from Step 2.

## Step 2 — Investigate (parallel subagents, ONE Agent message)

- **Agent A** — `investigate_failure(result_id)`. Return: failing element, error, immediate failing step, auto-suggested cause.
- **Agent B** — `get_execution_step_details(result_id)` + `get_step_children_details` for any loop/conditional. Return: ordered breadcrumb of last 3–5 steps with inputs/outputs.
- **Agent C** — `get_network_logs`, `get_console_logs`, `get_trace_url` (all on the same `result_id`). Return: 3 most relevant network entries, console errors, trace URL.
- **Agent D** — Codebase grep for the failing element/endpoint/error from A. Return: 3 most likely source files with line numbers. Skip if the failure is purely UI with no clear identifier.

## Step 3 — Root-cause hypothesis (inline)

1. Failing step (from B)
2. Symptom — expected vs actual
3. Evidence — pointers to A/B/C lines
4. Hypothesis — one sentence
5. Proposed fix — file path, function, plain-English change (1–3 lines)
6. Confidence — `high` / `medium` / `low`

If `low`, stop and ask the user how to proceed.

## Step 4 — Confirmation gate

Ask: *"Apply the fix to <file:line>? (y / edit / no)"*. Wait. If edited, re-confirm.

## Step 5 — Execute fix (subagent) and verify

Dispatch ONE subagent: file path + change description, tools Read/Edit/Write, constraint: change only what the fix requires. Reports edits with file:line.

Then `verify_bug_fix(test_case_id=<id>)` and poll `get_execution_status(test_case_id, number_of_executions=1)` until done. The poll returns **plain text wrapped in `{"result": "..."}`** like `"Execution in progress\nNumber of executions: 1"` or terminal status — match on substrings (`in progress` / `completed` / `failed`), do not assume JSON.

If a ticket was the source: `post_run_comment(...)` then post the body to the ticket.

## Step 6 — Loop or land

- PASSED → Step 7.
- FAILED, attempts < 3 → return to Step 2 with the new `result_id`. The fix is wrong/incomplete; never blame "not deployed yet".
- FAILED, attempts == 3 → stop. Surface all attempts and ask the user.

## Step 7 — Report

If a ticket was the source: `report_fix_to_ticket(...)` and post the body. End with: total attempts, final status, case URL `https://contextqatest.contextqa.com/td/cases/<id>/steps`, execution URL, files changed.

## Rules

- Every code change MUST be followed by `verify_bug_fix`.
- Step 2 agents MUST run in parallel (single Agent message, ≥3 invocations).
- Never call `report_fix_to_ticket` without a real `verify_bug_fix` result.
- Never pass a raw ticket URL into `bug_fix_from_ticket`.
- Cap fix attempts at 3.
- Tool responses are wrapped as `{"result": "..."}` — inspect for an embedded `next_step` field and follow it when present. `get_execution_status` is plain text; match on `in progress` / `completed` / `failed` substrings.
