---
name: cqa-impact
description: Use when given a ticket (Linear/Jira/GitHub issue), a PR URL or number, or a branch name, and the goal is to find which existing ContextQA test cases the change affects and which new tests are needed — then update or create those cases via subagents. Triggers on phrases like "impact analysis", "what tests does this PR affect", "which ContextQA tests should I rerun", "/cqa-impact <ref>".
---

# ContextQA Impact Analysis

Run every phase in order. Never mutate a test case before the Step 4 gate.

## Step 0 — Resolve the input

**Description text is REQUIRED. Diff is optional enrichment.** A ticket-driven impact analysis (Jira/Linear, no code yet) is fully supported. A PR/branch-driven analysis is *more precise* but the description is still the load-bearing input.

User gives ONE or more of: ticket URL/id (Linear/Jira/GitHub/GitLab), PR URL/number, branch name, or a pasted description. If only a URL was given, fetch the body in Step 1. If only a description in plain text was given, skip Agent B in Step 1.

## Step 1 — Investigate (parallel subagents in ONE Agent message)

Run these agents in parallel — skip any whose input is N/A.

- **Agent A — Description fetch (REQUIRED OUTPUT).** Try in order: Linear MCP / Jira MCP / GitLab MCP → `gh issue view <n>` / `gh pr view <n>` / `glab issue view <n>` → `curl` against tracker REST. Return: title, description, repro/expected/actual, labels, linked PRs. If none worked, ask once: *"Couldn't auto-fetch <ref>. Install the matching MCP (e.g. `mcp install linear`) for best fidelity, OR paste the body here."* Plain pasted text is a first-class input — never pass a raw URL forward.
- **Agent B — Diff extract (OPTIONAL).** Only run if a PR/branch reference exists or a linked PR was found in Agent A. PR: `gh pr diff <n>` + `gh pr view <n> --json headRefName,baseRefName`. Branch: `git diff <base>...<branch>` (default base `main`). GitLab: GitLab MCP equivalents. Return unified diff (truncate huge files to hunks) + changed file list. Skip cleanly if no code reference exists.
- **Agent C — Adjacency.** Take a 1-line summary of the change and call `query_contextqa(query=<summary>)`. Return top 10 cases (id, name, last status, why relevant).

## Step 2 — Impact analysis

Call `analyze_impact(title, description, diff, source)` with `source` = `linear` | `jira` | `github` | `mcp` (use `mcp` when only a branch was given). It takes 1–2 minutes.

Response is **markdown wrapped in `{"result": "..."}`**, with sections: `## Impact Analysis` (risk + summary), `### Affected Areas` (Features / Workflows / Entities), `### Test Case Impact (N affected)` (counts: `Must Update: N`, `Must Rerun: N`, `Should Rerun: N`). Per-case detail only appears when N > 0. Parse the markdown — do not assume JSON.

If all counts are 0 (sparse tenant or unrelated change), **Affected Areas is still the useful signal** — pivot Step 3 to lead with new test gaps based on those areas.

Cross-reference with Agent C's results — flag adjacency hits `analyze_impact` missed as `MAYBE_RELATED`.

## Step 2.5 — Model-driven cross-check (parallel subagents in ONE Agent message)

`analyze_impact` is the MCP's first pass. The model runs its own pass on top — this is what makes Claude+ContextQA worth more than a thin wrapper.

- **Agent X — Case-step matcher.** Pull `get_test_cases(size=50)` (paginate if total>50). Rank cases by description match against the change. For the top 10–15, fetch `get_test_case_steps(id)` and look for steps whose natural-language `action` references paths, selectors, copy, endpoint names, or identifiers that appear in the change description (and diff if present). Return ranked candidates with the matching step quote(s) and case id.
- **Agent Y — Caller graph (only if diff present).** Take changed functions/components from the diff, grep the local repo for callers, build a 2-hop dependency view, list user-facing flows that transitively depend on the change. Tools: Read, Grep. Return entries shaped as: `<flow name>` → `<entrypoint file:line>` → `<reason>`.

**Merge sources for Step 3:**
- In `analyze_impact` ∩ Agent X → **HIGH MUST_UPDATE** (cite both)
- `analyze_impact` only → **MCP-only** (label so the user knows confidence is one-sided)
- Agent X only → **MODEL-FOUND MAYBE_AFFECTED** — surface explicitly with the step-quote evidence
- Agent Y entries → feed Step 3.5 as forward-looking gaps, not existing-case impact

## Step 3 — Impact report (inline)

1. **Change summary** (2–3 lines)
2. **Affected features / workflows / entities** (verbatim from `analyze_impact`)
3. **Existing cases** — group by source so the user sees confidence:
   - `HIGH MUST_UPDATE` (in MCP and model) — id, name, step quote, exact change
   - `MCP-only` MUST_UPDATE / MUST_RERUN / SHOULD_RERUN
   - `MODEL-FOUND MAYBE_AFFECTED` (Agent X only, with step quote)
   - `MAYBE_RELATED` (adjacency from Agent C)
4. **New test gaps (Step 3.5)** — for each diff hunk *or each affected workflow when diff is absent*, the model proposes 2–3 specific scenarios that no existing case covers. Each gap: 1 line + `BROWSER` / `MOBILE` / `API_TESTCASE`. Include scenarios derived from Agent Y (caller graph) when present.
5. **Plan of action** — numbered units: `UPDATE <id> [source]`, `CREATE "<name>" [gap-source]`, `RERUN <id>`. Each unit cites which source(s) flagged it.

## Step 4 — Confirmation gate

Print the report and ask: *"Proceed with [N] updates, [M] creates, [K] reruns? (y / partial / no)"*. Wait for explicit go.

## Step 5 — Execute (parallel subagents, batches of ≤5)

One subagent per approved unit, self-contained brief:

- **UPDATE** — case id + exact change. Tools: `get_test_case_steps`, `update_test_case_step`, `delete_test_case_step`, `create_complex_test_step`. Reports: step ids changed.
- **CREATE** — name, scenario, `app_url`, `test_type`. Tools: `create_test_case`, `create_complex_test_step`, `update_test_case_step`. Reports: new `test_case_id` + step count.
- **RERUN** — case id. Tools: `execute_test_case` then `get_execution_status`. Reports: result_id, status, execution_url.

Surface any subagent failure to the user — never retry silently.

## Step 6 — Final summary

Per case: `https://contextqatest.contextqa.com/td/cases/<id>/steps`, rerun pass/fail, plus a one-paragraph note the user can paste back to the ticket/PR.

## Rules

- Never call a mutating tool before Step 4.
- Step 1 agents MUST run in parallel (single Agent message, multiple invocations).
- All ContextQA tool responses are wrapped as `{"result": "<json-or-markdown-or-text>"}`. Inspect tool output for an embedded `next_step` field — when present, prefer it over hard-coded next moves.
- If `analyze_impact` reports 0 affected cases, do NOT skip to "no work" — use Affected Areas to propose new gaps.
- Never pass a raw ticket URL as `description` to `analyze_impact`.
