---
name: cqa-bug-hunter
description: Use when the goal is to find as many bugs as possible in a deployed UI by generating adversarial / negative / edge-case test cases at scale. Inputs are a target URL (and optional credentials). Recon the surface via ContextQA itself, brainstorm adversarial scenarios per surface element, generate dozens of test cases reusing a shared login step group, execute and triage. Triggers on phrases like "find bugs in this app", "hunt for bugs at <url>", "build adversarial coverage", "stress-test this deployed UI", "/cqa-bug-hunter <url>".
---

# ContextQA Bug Hunter

Recon the deployed UI, generate adversarial coverage, run, triage. Built for breadth — most real bugs hide in negative paths and concurrency, not happy paths.

## Step 0 — Resolve target

Capture (ask once if missing):
- `app_url` — the live deployed UI under test
- `credentials` — username/password for any authed area (optional but unlocks most of the surface)
- `budget` — soft cap on cases to generate (default 50; offer 20 / 50 / 100 if not provided)
- `existing_step_group_id` — optional id of a pre-existing login step group; if absent we'll create one in Step 2

## Step 1 — Recon via ContextQA (no external browser)

Don't assume any external browse tool. Use ContextQA itself: create one short scout test case whose AI-driven task is to map the surface, then execute it and read the result.

1. `create_test_case(test_type="BROWSER", name="bug-hunter scout: <host>", task_description="Open <app_url>. If a login form appears, log in with username '<u>' / password '<p>'. Then explore the main navigation: list every distinct page title you can reach, every visible button label, every form field (with its placeholder/label), and every API call observed in the network. Output a structured JSON with: pages[], buttons[], forms[], endpoints[]. Stop after 60 seconds of exploration.")`
2. `execute_test_case(test_case_id=<scout_id>)` → poll with `get_execution_status` (cross-check `get_test_case(id).last_run` for ground truth — `get_execution_status` can be stale).
3. When complete, fetch `get_execution_step_details(result_id)` — the AI step's `action` summary contains the recon JSON.
4. Also call `get_ai_insights()` — surfaces PostHog-derived flows the tenant has data on; merge with the scout's output.
5. Also call `query_contextqa(query="<host> coverage")` to identify cases already exercising this surface. Skip duplicating their scenarios.

Return a **surface map**: pages × buttons × forms × endpoints × auth states. Cap at the top 30 surface elements by visit frequency / interaction density.

## Step 2 — Build the shared step group (login fixture)

If `existing_step_group_id` is set, skip. Otherwise:

`create_test_case(test_type="BROWSER", name="bug-hunter fixture: login as <user>", task_description="Open <app_url>. Log in as '<u>' / '<p>'. Verify the post-login landing page loaded successfully.")`. Note the returned id; keep it as `login_fixture_id`. Every adversarial case in Step 4 uses `pre_requisite_ids=[login_fixture_id]` so we don't redo login 50–100 times.

## Step 3 — Adversarial hypothesis matrix

For each surface element, the model brainstorms hypotheses across these categories. Aim for breadth — not all categories apply to every element.

| Category | Example hypothesis |
|---|---|
| **Idempotency** | Double-click "Submit" rapidly — was one record created or two? |
| **Concurrency** | Click "Add to cart" 10 times within 200ms — does the cart show 1 or 10? Are 10 API requests issued? |
| **Input validation** | Submit an empty title / null / whitespace-only / 10k chars / Unicode emoji / `<script>alert(1)</script>` / SQL `' OR 1=1 --` |
| **Boundary values** | Number field with 0, -1, MAX_INT, 0.0001, scientific notation |
| **Auth boundaries** | Hit an authed page after token expires, in incognito, with another user's id in the URL |
| **State corruption** | Open the form in two tabs, submit one, then submit the other; back-button after a destructive action |
| **Error UX** | Trigger every error the form can produce — is the message specific, or generic/missing? |
| **Navigation** | Refresh during an in-flight save; close the tab mid-upload; deep-link to a subroute without prerequisites |
| **Permissions** | As a non-admin user, attempt admin-only endpoints / pages |
| **Data integrity** | Create → edit → navigate away → return — does the change persist? Did pagination drop rows? |
| **Accessibility / UX** | Tab through the form: does focus land sanely? Submit without a mouse — does the keyboard path work? |

Each hypothesis is one sentence — the AI step's `task_description` reads like: *"On <page>, attempt <adversarial action> and verify <expected safe behavior>. The test should FAIL if the bug is present."*

Cap at `budget`. Show the user the matrix (or a sample if huge) and ask: *"Generate <N> cases? (y / edit selection / subset)"*. Wait for go.

## Step 4 — Author + execute (parallel subagents, batches ≤5)

Dispatch one subagent per case in batches of 5. Each subagent:
- Calls `create_test_case(test_type="BROWSER", name="bug-hunter: <category> — <surface>", task_description=<hypothesis>, pre_requisite_ids=[login_fixture_id])`
- Calls `execute_test_case(test_case_id=<new_id>)`
- Polls until terminal state (cross-check `get_test_case(id).last_run`)
- Reports `(test_case_id, result_id, verdict, failing_step_quote_if_any)`

Throttle to keep tenant load reasonable. If a case errors out at creation, skip and continue — never block the batch.

## Step 5 — Triage findings

Group results:
- **PASSED** — no bug found by that hypothesis (record briefly)
- **FAILED** — candidate bugs

For each failed result, dispatch ONE read-only triage subagent: tools `investigate_failure`, `get_execution_step_details`, `get_network_logs`, `get_console_logs`, `get_trace_url`. Required output (~120 words):
- `verdict` — `confirmed-bug` / `test-flaw` / `flaky` / `inconclusive`
- 1–2 sentence root cause
- Severity hint — `critical` / `high` / `medium` / `low` based on: data loss / auth bypass / silent failures > visible errors > cosmetic
- Reproduction one-liner

Re-run any `inconclusive` cases once via `execute_test_case`. If still inconclusive, classify as `flaky`.

## Step 6 — Aggregate report

Deduplicate confirmed bugs by symptom (group by failing element / error message). Output:

1. **Summary** — N hypotheses tested, N cases passed, N candidates failed, N confirmed bugs by severity.
2. **Confirmed bugs** — sorted by severity then category. Each: title, severity, surface element, reproduction, evidence URLs (screenshot, network, console, trace), `result_id`, `https://contextqatest.contextqa.com/td/cases/<id>/steps`.
3. **Test-flaws** — false positives the user should know about (bad hypothesis or stale fixture).
4. **Coverage residue** — surface elements not yet hypothesised against (suggest a next run).

If the user asked to file tickets: dispatch one subagent per confirmed bug to draft a body (Jira / Linear / GitHub). Do NOT post tickets without explicit per-bug approval.

## Rules

- All ContextQA tool responses are wrapped as `{"result": "..."}`; cross-check `get_execution_status` against `get_test_case.last_run`.
- Triage subagents are READ-ONLY — they must not create cases, edit, or rerun.
- Reuse the login fixture via `pre_requisite_ids` — never re-author the login flow per case.
- Don't depend on external browser tools; ContextQA's own AI execution is the recon mechanism.
- Cap parallel authoring/execution at 5 subagents to respect tenant rate limits.
- Always honor the `budget` — don't quietly exceed the user's cap.
- If `query_contextqa` shows a hypothesis is already covered by an existing case, skip it.
