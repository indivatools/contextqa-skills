---
name: cqa-author
description: Use when the goal is to author new ContextQA test cases from a source — a Linear/Jira/GitHub ticket, a Swagger/OpenAPI spec, a Figma file, a video, an Excel sheet, an n8n workflow, a code diff, or a free-text requirements doc. Investigate the source, plan the suite shape, then create cases in parallel via subagents. Triggers on phrases like "create tests from this ticket", "generate tests from swagger/figma/video", "/cqa-author <source>", "build a test suite for X".
---

# ContextQA Test Authoring

Pick the right path for the source, fetch before generating, review before creating manual extras.

## Step 0 — Identify

Capture (ask once if missing):
- `source_type` — `linear` | `jira` | `github_issue` | `swagger` | `figma` | `video` | `excel` | `n8n` | `code_diff` | `requirements_text`
- `source_ref` — URL, file path, ticket id, branch, or raw text
- `app_url` — required for browser/mobile cases (skip for `swagger` / `API_TESTCASE`)

## Step 1 — Investigate (subagent if external)

If the source needs fetching (Linear/Jira/GH/GitLab ticket, remote file): dispatch ONE subagent. Order of preference: Linear MCP / Jira MCP / GitLab MCP → `gh issue view` / `glab issue view` → `curl`. Return raw content — never just a URL.

If no fetcher is available, ask once: *"I couldn't fetch <ref>. Install the matching MCP for best fidelity, OR paste the body and I'll continue."* Plain-text input is supported — for tickets it routes through `reproduce_from_ticket`; for free-form requirements it uses the two-phase `generate_tests_from_requirements` → `start_requirements_generation` flow.

Skip this step if the source is already in-hand text, a local file path, or a URL the matching MCP tool accepts directly.

## Step 2 — Pick the generation path

| Source | Tool | Notes |
|---|---|---|
| `linear` | `generate_tests_from_linear_ticket` | Pass fetched title/description/repro/expected/actual |
| `jira` / `github_issue` | `reproduce_from_ticket` (or `generate_tests_from_requirements`) | Treat body as ticket text |
| `swagger` | `generate_tests_from_swagger(file_path_or_url=...)` | API tests |
| `figma` | `generate_tests_from_figma(figma_url=...)` | |
| `video` | `generate_tests_from_video(video_url=...)` | |
| `excel` | `generate_tests_from_excel(file_path, sheet_name)` | |
| `n8n` | `generate_contextqa_tests_from_n8n(file_path_or_url, app_url)` | |
| `code_diff` | `generate_tests_from_code_change(diff_text, app_url, name_prefix)` | |
| `requirements_text` | `generate_tests_from_requirements` → answer questions → `start_requirements_generation` | Always run both phases |

If multiple sources are given, run sequentially — the requirements pipeline shares session state.

**Tenant feature gating:** `swagger`, `figma`, `video`, `excel`, `requirements_text`, **and `code_diff`** all use the `/requirements/upload` backend. On tenants where this isn't enabled they fail with HTTP 404 (`/requirements/upload`) or "Failed to fetch latest requirement ID: no response". When that happens, fall back to manual case authoring in Step 4 — extract scenarios from the source yourself and create cases via `create_test_case` + `create_complex_test_step`. Tell the user the fallback is happening; don't fail silently. (`generate_tests_from_linear_ticket`, `reproduce_from_ticket`, and `generate_contextqa_tests_from_n8n` use different backends and are not affected by this gating.)

**Public-URL constraint:** `generate_tests_from_swagger` / `_from_figma` / `_from_video` fetch the URL server-side. Private GitHub raw URLs return 404 — host the file publicly first, or pass a local file path if the server supports it.

## Step 3 — Plan and confirm

After generation returns (or before, for slow paths), print:

1. Source summary (1–3 lines)
2. Proposed cases — name + `Happy` / `Error` / `Edge` / `Branch` / `API` / `Visual`
3. Coverage gaps the generator missed
4. `app_url` and `test_type` per case (`BROWSER` / `MOBILE` / `API_TESTCASE`)

Ask: *"Approve [N] generated cases? Add the [K] manual gaps? (y / edit / no)"*. Wait.

## Step 4 — Execute manual augmentations (parallel subagents, ≤5)

One subagent per gap. Brief:
- Case name, scenario, `test_type`, `app_url`, ordered steps
- Tools: `create_test_case` (case + first natural-text step), `create_complex_test_step` (REST/loop/conditional), `update_test_case_step` (additional natural-text steps)
- For API tests, pass `api_url`, `api_method`, payload fields directly to `create_test_case`
- Reports: `test_case_id`, step count, case URL

For fixes to generator output: same pattern with `get_test_case_steps` + `update_test_case_step`.

## Step 5 — Optional suite

If asked: `create_test_suite(name, test_case_ids, test_type)`, then `get_available_devices` + `create_test_plan` for a runnable plan.

## Step 6 — Final report

Per case: id, name, step count, `https://contextqatest.contextqa.com/td/cases/<id>/steps`. Include suite id if created. End with a next-step suggestion: `execute_test_plan(<id>)` or `/cqa-regression`.

## Rules

- Linear/Jira/GH tickets: always fetch the body before any `generate_tests_from_*`.
- `requirements_text` MUST go through both phases.
- No browser/mobile case without `app_url`.
- Step 3 confirmation gate runs even if generation succeeded.
- Don't parallelize generator calls — only investigate-fetch and manual augment.
- All ContextQA tool responses are wrapped as `{"result": "..."}`. Tools like `reproduce_from_ticket` return an embedded `next_step` field — follow it when present.
