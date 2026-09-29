---
name: cqa-debug
description: Use when a ContextQA test case or execution has failed and needs diagnosing and fixing. Inputs can be a `result_id`, a `test_case_id` whose latest run failed, a plan `execution_id` with failures, or a bug ticket that needs reproducing first. Gathers telemetry in parallel, separates a product bug from a broken test, fixes, verifies, reports. Capped at 3 attempts. Triggers on "debug this failure", "fix this failing test", "why did result <id> fail", "reproduce and fix this bug", "/cqa-debug <ref>".
---

# ContextQA Debug & Fix

The first question is always **"is this a product bug or a broken test?"**
Answering it wrong wastes the whole session — you either patch working code or
file a ticket against a stale selector.

Cap fix attempts at 3. Never loop.

## Step 0 — Resolve the failure

| You have | Do |
|---|---|
| `result_id` | → Step 2 |
| `test_case_id` | `get_test_case(id)` → `last_run`, or `get_test_case_results(...)` for the latest `result_id` |
| plan `execution_id` | `get_test_plan_execution_status(execution_id)`; pick the failed `result_id`s. Many failures → hand off to `/cqa-regression` |
| a ticket, no test yet | → Step 1 |

Also capture `app_url` and how the fix reaches it. **Default assumption: a local
file save is immediately live at `app_url`, no build step.** Override only if
the user says otherwise — and if they do, a `FAILURE` after a fix is still a
real failure, not deployment lag.

## Step 1 — Reproduce (only when the input is a ticket)

1. Fetch the ticket body — matching MCP → `gh issue view` / `glab issue view` →
   `curl`. If nothing works, ask once and accept a paste. **Never pass a raw
   URL forward.**
2. `bug_fix_from_ticket(ticket_text=<body>, url=<app_url>, repo_url=..., deployment_info=...)`
   returns `test_case_id`, `session_rules[]`, `fix_guide[]` and a note. The run
   is queued; `execution_url` stays null until you poll.
3. **Treat the returned `session_rules` as authoritative** — where they conflict
   with this skill, they win.
4. Follow `fix_guide` — it is the canonical loop for that session.
5. Continue at Step 2.

## Step 2 — Gather evidence (parallel, one message)

- **A — `investigate_failure(result_id)`.** Failing element, error, immediate
  failing step, suggested cause.
- **B — `get_execution_step_details(result_id)`** plus
  `get_step_children_details(step_result_id, depth)` for any loop, step group or
  IF/ELSE. Return an ordered breadcrumb of the last 3–5 steps with inputs and
  outputs, and the verbatim `failure_reason`.
- **C — `get_network_logs`, `get_console_logs`, `get_trace_url`** on the same
  `result_id`. Return the 3 most relevant network entries, console errors, and
  the trace link.
- **D — codebase search** for the failing element, endpoint or error string from
  A. Return the 3 most likely source files with line numbers. Skip if the
  failure is purely UI with no identifier to search on.

**If the case is data-driven, B returns `steps: []`.** That is expected — read
the verdict from `get_test_case(id).last_run` (`total_steps` / `passed_steps` /
`result`) instead.

## Step 3 — Classify before you hypothesise

Decide which of these it is, and say which:

| Class | Tell |
|---|---|
| **Product bug** | the app returned the wrong thing; network/console show a real error; the step targeted the right element |
| **Broken test** | selector no longer matches, a step lost its locators, an assertion encodes old copy |
| **Environment** | `*\|var\|` missing from the bound environment, wrong base URL, expired credential, the app under test is not running |
| **Infrastructure** | `FAILED_TO_START`, a run that `COMPLETED` with 0 passed and 0 failed, no default test plan in the workspace |

Two failures that masquerade as product bugs:

- **`" is not clickable on the page…"` with a leading space** is an *empty
  element name* — the step has no locators. That is a broken test, not a
  missing button.
- **A step that fails in ~188ms reporting "network idle"** is a `navigateToUrl`
  with no URL — missing `event.href` / `requestList.url`.

Environment and infrastructure classes do not get a code fix. Say so and route:
`/cqa-environments` or `/cqa-suites-and-plans`.

## Step 4 — Hypothesis

1. Failing step (from B)
2. Symptom — expected vs actual
3. Evidence — pointers into A/B/C
4. Class (Step 3) and hypothesis, one sentence
5. Proposed fix — file path or step id, and the change in 1–3 lines
6. Confidence — `high` / `medium` / `low`

If `low`, stop and ask.

## Step 5 — Confirmation gate

*"Apply the fix to `<file:line>` / step `<id>`? (y / edit / no)"* Wait. If
edited, re-confirm.

## Step 6 — Apply and verify

**Broken test** — `manage_test_step(action="update", test_case_id=N, step_id=S,
step={...})`. Use `dry_run=True` first to see the before/after field diff. Then
`execute_test_case(test_case_id=N)`.

**Product bug** — make the code change, then `verify_bug_fix(test_case_id=N)`.

Either way: share the `live_url` with the user before your next tool call, then
poll `get_execution_status(session_id=..., wait=True, timeout=60)`, posting a
line each time it returns `timed_out`.

Read `result` — `SUCCESS` / `FAILURE`. It is `null` while `is_completed` is
false, and **a new result row appearing is not a pass.**

If the source was a ticket: `post_run_comment(...)` and post the body.

## Step 7 — Loop or land

- `SUCCESS` → Step 8.
- `FAILURE`, attempts < 3 → back to Step 2 with the new `result_id`. The fix was
  wrong or incomplete; do not blame "not deployed yet".
- `FAILURE`, attempts == 3 → stop. Surface all three attempts and ask.

A well-formed fix moves the failure **forward** — the old step goes green and
the next one becomes the new failure. A failure that stays on the same step
means the fix missed.

## Step 8 — Report

If a ticket was the source, `report_fix_to_ticket(...)` and post it. Close with:
attempts, final verdict, the portal `url` from `get_test_case(id)` (never a
hand-built link — they are tenant-specific), the execution link, and the files
or steps changed.

## Rules

- Every code change is followed by `verify_bug_fix`. No exceptions.
- Never call `report_fix_to_ticket` without a real verification result.
- Never pass a raw ticket URL into `bug_fix_from_ticket`.
- Never "fix" a case by deleting it and dropping in one AI step.
- Cap at 3 attempts, then report honestly. Do not delete a failing case to tidy
  the numbers.
