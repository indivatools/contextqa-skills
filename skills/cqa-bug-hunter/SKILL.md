---
name: cqa-bug-hunter
description: Use when the goal is to find as many bugs as possible in a deployed UI by generating adversarial, negative and edge-case ContextQA tests at scale. Recons the surface, brainstorms hypotheses per element, generates cases against a shared login fixture, executes, triages and reports by severity. Built for breadth — most real bugs hide in negative paths and concurrency, not happy paths. Triggers on "find bugs in this app", "hunt for bugs at <url>", "build adversarial coverage", "stress-test this deployed UI", "/cqa-bug-hunter <url>".
---

# ContextQA Bug Hunter

Recon the surface, generate adversarial coverage, run it, triage what fails.

**This skill deliberately uses AI-agent steps.** Everywhere else in ContextQA
typed steps win, because you maintain them. Here the cases are disposable
probes: written once, run once, kept only if they find something. If a
hypothesis *does* find a real bug, convert that one case to typed steps
(`/cqa-locators`) so it becomes a regression test. The rest can be deleted.

## Step 0 — Resolve the target and the cost

Capture (ask once if missing):

- `app_url` — the deployed UI under test
- `credentials` — unlocks most of the surface (see the gate below)
- `budget` — soft cap on cases (default 50; offer 20 / 50 / 100)
- `existing_step_group_id` — an existing login fixture, if there is one

**This is the expensive skill.** Every case is an AI-driven run against a live
app. Before generating, say roughly how many runs this will be and confirm.

**Confirm the target is not production**, or that the user accepts adversarial
traffic against it. The hypothesis matrix below includes double-submits,
concurrency bursts and auth-boundary probes. Those create real records and
occasionally real charges. Get that agreed before Step 4, not after.

### The credential gate

For the authed surface the app password usually has to reach the automation:

1. **Warn** that it will be used to log in.
2. **Get explicit permission.**
3. **Confirm it is not production** and carries no billing consequence.
4. Store it as a `password`-typed environment parameter — never in step text,
   never in a file, never in the transcript. Filter `password` / `token` /
   `authorization` / `bearer` / `cookie` out of anything you print.
5. **Always offer the alternative:** the user signs in themselves and you take
   over the session.

## Step 1 — Recon

Don't assume an external browser. Use ContextQA itself.

1. `create_test_case(test_type="BROWSER", name="bug-hunter scout: <host>",
   environment_id=<env>, task_description="Open *|base_url|. If a login form
   appears, log in. Then explore the main navigation: list every distinct page
   title you can reach, every visible button label, every form field with its
   label or placeholder, and every API call observed. Output JSON with pages[],
   buttons[], forms[], endpoints[]. Stop after 60 seconds.")`
2. `execute_test_case(test_case_id=<scout>)` — share `live_url`, then
   `get_execution_status(session_id=..., wait=True, timeout=60)`.
3. `get_execution_step_details(result_id)` — the AI step's summary carries the
   recon JSON.
4. `get_ai_insights()` — surfaces the flows the tenant actually has usage data
   on. Weight those higher; a bug on a page nobody visits is worth less.
5. `query_contextqa(query="<host> coverage")` — skip anything already covered.

Produce a **surface map**: pages × buttons × forms × endpoints × auth states.
Cap at the 30 highest-traffic elements.

If your agent has a real browser tool, use it instead for recon — it is faster
and free. The ContextQA scout is the fallback that always works.

## Step 2 — Build the login fixture once

Skip if `existing_step_group_id` is set. Otherwise create a **step group**:

```
manage_test_case(action="create", name="bug-hunter fixture: login", is_step_group=True)
```

Author its steps (see `/cqa-locators` — this one is worth typing properly,
because every probe depends on it), then attach it to each generated case with
`manage_test_case(action="set_prerequisites", id=<case>, prerequisite_ids=[<fixture>])`.

**Never re-author login 50 times.** A flaky fixture turns into 50 false
positives.

## Step 3 — The hypothesis matrix

For each surface element, brainstorm across these. Not every category applies to
every element — aim for breadth, not completeness.

| Category | Example hypothesis |
|---|---|
| **Idempotency** | double-click Submit rapidly — one record or two? |
| **Concurrency** | click Add to cart 10× in 200ms — does the cart show 1 or 10? |
| **Input validation** | empty / whitespace-only / 10k chars / emoji / `<script>alert(1)</script>` / `' OR 1=1 --` |
| **Boundary values** | 0, -1, MAX_INT, 0.0001, scientific notation |
| **Auth boundaries** | an authed page after the token expires; another user's id in the URL |
| **State corruption** | two tabs, submit both; back-button after a destructive action |
| **Error UX** | trigger every error the form can produce — is the message specific, or generic? |
| **Navigation** | refresh mid-save; close the tab mid-upload; deep-link past a prerequisite |
| **Permissions** | as a non-admin, reach admin-only pages and endpoints |
| **Data integrity** | create → edit → navigate away → return: did it persist? did pagination drop rows? |
| **Accessibility** | tab through the form — does focus land sanely? does the keyboard path work? |

Each hypothesis is one sentence, written so the case **fails when the bug is
present**: *"On `<page>`, attempt `<adversarial action>` and verify `<expected
safe behavior>`."*

Cap at `budget`. Show the matrix (or a sample) and ask: *"Generate `<N>` cases?
(y / edit / subset)"* Wait.

## Step 4 — Author and execute (batches of ≤5)

One subagent per case, five at a time:

- `create_test_case(test_type="BROWSER", name="bug-hunter: <category> — <surface>",
  task_description=<hypothesis>, environment_id=<env>, pre_requisite_ids=[<fixture>],
  tags=["bug-hunter"])`
- `execute_test_case(test_case_id=<new>)`, then poll to a terminal state
- Report `(test_case_id, result_id, verdict, failing_step_quote)`

Tag every generated case so the workspace can be cleaned up afterwards. Throttle
to keep tenant load reasonable. If one errors at creation, skip it — never block
the batch.

## Step 5 — Triage

`SUCCESS` means the hypothesis found nothing. `FAILURE` is a **candidate**, not
a bug — an adversarial AI step fails for bad reasons as often as good ones.

One read-only triage subagent per failure: `investigate_failure`,
`get_execution_step_details`, `get_network_logs`, `get_console_logs`,
`get_trace_url`. Required output (~120 words):

- `verdict` — `confirmed-bug` / `test-flaw` / `environment` / `flaky` / `inconclusive`
- 1–2 sentence root cause
- severity — `critical` / `high` / `medium` / `low`, weighted: data loss and
  auth bypass first, then silent failures, then visible errors, then cosmetic
- a one-line reproduction

Re-run `inconclusive` cases once. Still inconclusive → `flaky`.

Two things that are **never** bugs in the product: a step failing with
`" is not clickable"` (a leading space means the step had no element) and a
navigate step failing in ~188ms on "network idle" (the step had no URL). Both
are test flaws.

## Step 6 — Report

Deduplicate confirmed bugs by symptom — group by failing element and error
message, since one bug typically trips several hypotheses.

1. **Summary** — hypotheses tested, passed, candidates, confirmed by severity
2. **Confirmed bugs**, sorted by severity: title, surface, reproduction,
   evidence links (network, console, trace, video), `result_id`, and the portal
   `url` from `get_test_case` — never a hand-built link
3. **Test flaws** — false positives the user should know about
4. **Coverage residue** — surface elements not yet probed, as a next run
5. **Cleanup** — the `bug-hunter` tag, and which cases are worth keeping

If asked to file tickets, draft the bodies. **Do not post without explicit
per-bug approval.**

## Rules

- Triage subagents are read-only.
- Reuse the login fixture via prerequisites; never re-author it per case.
- Honour `budget`. Don't quietly exceed the cap.
- Skip any hypothesis `query_contextqa` shows is already covered.
- Cap parallel authoring and execution at 5.
- Confirm the target and the cost before Step 4. This skill spends real money
  and writes real records.
