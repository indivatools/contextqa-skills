---
name: cqa-author
description: Use when the goal is to author new ContextQA test cases from a source — a Linear/Jira/GitHub ticket, a Swagger/OpenAPI spec, a Figma file, a video, an Excel sheet, an n8n workflow, a code diff, or free-text requirements. Fetch the source, pick the generation path, review the plan at a gate, then create and harden the cases. Triggers on "create tests from this ticket", "generate tests from swagger/figma/video", "build a test suite for X", "/cqa-author <source>".
---

# ContextQA Test Authoring

Fetch before you generate. Gate before you create. Harden before you call it
done — a generated case is a draft, not a deliverable.

## Step 0 — Identify

Capture (ask once if missing):

- `source_type` — `linear` | `jira` | `github_issue` | `swagger` | `figma` |
  `video` | `excel` | `n8n` | `code_diff` | `requirements_text`
- `source_ref` — URL, file path, ticket id, branch, or raw text
- `app_url` — required for browser/mobile cases (not for `swagger` /
  `API_TESTCASE`)

Then orient: `get_current_user` names the org and workspace you are about to
write into. **A case's workspace is baked into its record and cannot be moved
later** — author in the workspace that will run it.

Before generating anything, check what exists: `query_contextqa(query=<one-line
summary>)` finds semantically adjacent cases. Duplicating coverage is worse than
adding none.

## Step 1 — Fetch the source

If it needs fetching (a ticket, a remote file), do that first and pass the
**content**, never a URL. Order of preference: the matching MCP (Linear / Jira /
GitLab) → `gh issue view` / `gh pr view` / `glab issue view` → `curl`.

If nothing can fetch it, ask once: *"I couldn't fetch `<ref>`. Connect the
matching MCP for best fidelity, or paste the body and I'll continue."* **Plain
pasted text is a first-class input** — no tracker integration required.

Skip this step when the source is already in-hand text, a local path, or a URL
the generation tool accepts directly.

## Step 2 — Pick the generation path

| Source | Tool |
|---|---|
| `linear` | `generate_tests_from_linear_ticket(ticket_id, title, description, app_url, steps_to_reproduce, expected_behavior, actual_behavior)` |
| `jira` / `github_issue` | `reproduce_from_ticket(ticket_text, url, name)` — or `bug_fix_from_ticket` for the full reproduce→fix→verify loop |
| `swagger` | `generate_tests_from_swagger(file_path_or_url)` |
| `figma` | `generate_tests_from_figma(figma_url)` |
| `video` | `generate_tests_from_video(video_url)` |
| `excel` | `generate_tests_from_excel(file_path, sheet_name)` |
| `n8n` | `generate_contextqa_tests_from_n8n(file_path_or_url, app_url)` — read `get_contextqa_skill(name="n8n-testing")` first |
| `code_diff` | `generate_tests_from_code_change(diff_text, app_url, name_prefix)` |
| `requirements_text` | `generate_tests_from_requirements(requirements_text)` → answer the returned questions → `start_requirements_generation(session_id, questions_json_str)` |

Run generators **sequentially** — the requirements pipeline shares session
state.

**`requirements_text` MUST go through both phases.** Phase 1 returns a
`session_id` and questions; stopping there creates nothing.

**Tenant feature gating.** `swagger`, `figma`, `video`, `excel`,
`requirements_text` **and `code_diff`** all ride the `/requirements/upload`
backend. Where it isn't enabled they fail with a 404 on `/requirements/upload`
or "Failed to fetch latest requirement ID". When that happens, **say so**, then
fall back to authoring by hand (Step 4). `generate_tests_from_linear_ticket`,
`reproduce_from_ticket` and the n8n path use different backends and are not
affected.

**Public-URL constraint.** The swagger / figma / video tools fetch the URL
server-side. A private GitHub raw URL returns 404 — host it publicly or pass a
local path.

## Step 3 — Plan and confirm (gate)

Print, and wait:

1. Source summary (1–3 lines)
2. Proposed cases — name + `Happy` / `Error` / `Edge` / `Branch` / `API` / `Visual`
3. Coverage gaps the generator missed
4. `test_type` (`BROWSER` / `MOBILE` / `API_TESTCASE`) and target environment
5. Where they will live — folder, tags, `module_name`, suite

*"Approve [N] generated cases? Add the [K] manual gaps? (y / edit / no)"*

**Decide the environment here, not later.** Every case binds one, and if you
don't choose, the workspace default is bound for you. Cases that will run
against more than one deployment need `*|base_url|` rather than a literal
address — see `/cqa-environments`.

## Step 4 — Author the gaps

Per gap:

```
manage_test_case(action="create", name=..., test_type="BROWSER",
                 tags=[...], module_name=..., folder_id=...)
manage_test_step(action="create", test_case_id=N, step={...}, dry_run=True)
```

- `manage_test_case(action="create")` makes a **bare** case; steps come from
  `manage_test_step`. `create_test_case(task_description=...)` instead makes one
  **AI step** — useful as a bootstrap, not as the finished article.
- For API cases, pass `api_steps` to `create_test_case` with `url`, `method`,
  `payload`, `expected_status`, `variable_name` — and chain responses with
  `${result.body.id}`.
- A repeated prefix (log in, seed data) belongs in a **step group**
  (`is_step_group=True`) or a **prerequisite**
  (`manage_test_case(action="set_prerequisites")`), not copy-pasted into every
  case.

Writing steps that actually run is its own job — read `/cqa-locators` and
`get_contextqa_skill(name="cqa-authoring-tests")` before hand-authoring. The
short version: a typed step needs either an `element_id` or
`event.pwLocator` + `event.selector`, or it saves cleanly and times out at run
time.

Fixing generator output is the same pattern: `manage_test_step(action="list")`
to see what is there, `action="update"` to change it. **Never delete a case and
replace it with one AI step.**

## Step 5 — Harden: run at least one

A case that has never run is a guess. Execute one representative case, share its
`live_url` immediately, then poll
`get_execution_status(session_id=..., wait=True, timeout=60)`.

Read `result` — the executor's own verdict. On `FAILURE`,
`get_execution_step_details(result_id)` names the failing step, and
`failure_reason` says what broke. Fix, re-run, advance one step per fix. Cap at
~3 attempts, then report the real reason rather than thrashing.

If the generator produced a wall of consecutive AI steps,
`manage_test_step(action="coalesce_ai_steps", test_case_id=N)` merges each run
into one (defaults to `dry_run=True` — review, then apply). N separate AI steps
means N independent agent invocations, each blind to the ones around it.

## Step 6 — Make them runnable

```
create_test_suite(name=..., test_case_ids=[...], test_type="BROWSER")
get_available_devices(device_type="browser")
create_test_plan(name=..., devices=[{"browser": "chrome", "suite_ids": [S]}],
                 environment_id=E, notify_emails="qa@acme.com")
```

Suites group; **plans run**. See `/cqa-suites-and-plans`.

## Step 7 — Report

Per case: id, name, step count, and the portal `url` returned by
`get_test_case` — **do not construct portal links by hand**, they are
tenant-specific. Include the suite and plan ids if created, plus one next step:
`execute_test_plan(<id>)` or `/cqa-regression`.

## Rules

- Always fetch a ticket's body before any `generate_tests_from_*`; never pass a
  raw URL as the description.
- `requirements_text` goes through **both** phases.
- No browser/mobile case without an `app_url`.
- The Step 3 gate runs even when generation succeeded.
- Don't parallelise generator calls — only the fetch and the manual authoring.
- Some tools return an embedded `next_step` field. Follow it when present.
