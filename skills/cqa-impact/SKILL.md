---
name: cqa-impact
description: Use when given a ticket, a PR, or a branch and the goal is to find which existing ContextQA test cases the change affects and which coverage is missing — then update, create or rerun those cases. Combines the platform's own analysis with the model's own pass over case steps and the caller graph, and can drive the real PR-impact pipeline when a repository is enrolled. Triggers on "impact analysis", "what tests does this PR affect", "which ContextQA tests should I rerun", "/cqa-impact <ref>".
---

# ContextQA Impact Analysis

Run the phases in order. **Never mutate a test case before the Step 5 gate.**

There are two different mechanisms, and picking the wrong one wastes the
session:

| | `analyze_impact` | `manage_pr_impact` |
|---|---|---|
| What it is | an ad-hoc semantic query over the case repository | the real PR pipeline |
| Needs a repo enrolled | no | yes |
| Stores anything | no | yes — one analysis per `(repo_id, pr_number)` |
| Runs anything | no | yes, per `testRunTiming` |
| Good for | "what would this change break?" | a PR that should get a check and a comment |

Default to `analyze_impact`. Reach for `manage_pr_impact` when the repository is
already enrolled and the user wants the verdict on the PR itself — see
`/cqa-integrations`.

## Step 0 — Resolve the input

**Description text is required; a diff is optional enrichment.** A ticket-driven
analysis with no code yet is fully supported. A PR or branch makes it more
precise, but the description carries the weight.

Accept any of: ticket URL/id, PR URL/number, branch name, or pasted text.

## Step 1 — Investigate (parallel, one message)

- **A — fetch the description (required output).** Matching MCP → `gh issue
  view` / `gh pr view` / `glab issue view` → `curl`. Return title, description,
  repro/expected/actual, labels, linked PRs. If nothing can fetch it, ask once
  and accept a paste. **Never pass a raw URL forward as the description.**
- **B — extract the diff (optional).** Only if a PR or branch exists. `gh pr
  diff <n>` plus `gh pr view <n> --json headRefName,baseRefName`; or `git diff
  <base>...<branch>`. Truncate huge files to hunks. Skip cleanly when there is
  no code reference.
- **C — adjacency.** `query_contextqa(query=<one-line summary>)`. Return the top
  10 cases with id, name, last status and why they look relevant.

## Step 2 — Platform analysis

```
analyze_impact(title=..., description=..., diff=..., source="github"|"jira"|"linear"|"mcp")
```

Takes 1–2 minutes. It returns **individual test cases only** — never suites or
plans — with a risk level, affected features/workflows/entities, and a per-case
action: `MUST_UPDATE` (steps need changing), `MUST_RERUN` (execute to verify no
regression), `SHOULD_RERUN` (related, run if there is time).

The response is markdown. Parse it; don't assume JSON.

**If every count is 0, do not conclude "no work".** On a sparse tenant that is
the normal answer. The *Affected Areas* section is still the useful signal —
pivot Step 4 to lead with coverage gaps in those areas.

Cross-reference with C and flag anything adjacency found that the analysis
missed as `MAYBE_RELATED`.

## Step 3 — The model's own pass (parallel, one message)

This is what makes an agent plus ContextQA worth more than a thin wrapper.

- **X — case-step matcher.** `get_test_cases(size=50)` (paginate on total).
  Rank by description match against the change, then for the top 10–15 call
  `get_test_case(id)` and read the steps: look for step text referencing paths,
  selectors, copy, endpoint names or identifiers that appear in the change.
  Faster shortcut when you know the string:
  `search_test_steps("element:*Checkout*,")` returns matching steps **and** the
  deduplicated `test_case_ids`. Never repeat a key in that grammar — it ANDs and
  silently matches nothing; use `@` for in-list.
- **Y — caller graph (only with a diff).** Take the changed functions and
  components, grep for callers, build a 2-hop view, and list the user-facing
  flows that transitively depend on the change. Return
  `<flow>` → `<entrypoint file:line>` → `<reason>`.

Merge:

| Source | Label |
|---|---|
| in `analyze_impact` **and** X | `HIGH MUST_UPDATE` — cite both |
| `analyze_impact` only | `MCP-only` — confidence is one-sided, say so |
| X only | `MODEL-FOUND MAYBE_AFFECTED` — surface with the step quote as evidence |
| Y | forward-looking coverage gaps, not existing-case impact |

## Step 4 — Report

1. Change summary (2–3 lines)
2. Affected features / workflows / entities, verbatim
3. Existing cases, grouped by the labels above, each with the step quote and the
   exact change needed
4. **Coverage gaps** — for each hunk, or each affected workflow when there is no
   diff, 2–3 specific scenarios nothing covers. One line each plus
   `BROWSER` / `MOBILE` / `API_TESTCASE`
5. **Plan of action** — numbered: `UPDATE <id> [source]`,
   `CREATE "<name>" [source]`, `RERUN <id>`

Flag one thing while you are here: any affected case whose steps contain a
**literal URL**. Those pass against the old deployment after a change — the
platform's own preflight ranks `FIXED_URL` as more misleading than a missing
variable. Route them to `/cqa-environments`.

## Step 5 — Confirmation gate

*"Proceed with [N] updates, [M] creates, [K] reruns? (y / partial / no)"* Wait
for an explicit go.

## Step 6 — Execute (parallel, batches of ≤5)

One subagent per approved unit, self-contained:

- **UPDATE** — case id and the exact change. Tools:
  `manage_test_step(action="list")`, `manage_test_step(action="update")`,
  `manage_test_step(action="delete")`. Use `dry_run=True` first on anything
  structural. Reports the step ids changed.
- **CREATE** — name, scenario, `test_type`, environment. Tools:
  `manage_test_case(action="create")` then `manage_test_step(action="create")`.
  Read `/cqa-locators` first. Reports the new `test_case_id` and step count.
- **RERUN** — `execute_test_case(test_case_id=N)`, share `live_url`, then
  `get_execution_status(session_id=..., wait=True, timeout=60)`. Reports
  `result_id`, the verdict, and the execution link.

Surface any subagent failure. Never retry silently.

## Step 7 — Summary

Per case: the portal `url` from `get_test_case` (never a hand-built link), the
rerun verdict, and one paragraph the user can paste back onto the ticket or PR.

If the repository is enrolled and the user wants this verdict on the PR itself,
continue into the pipeline below.

## The PR pipeline — when a repository is enrolled

Everything above is ad-hoc. This is the standing pipeline that puts a verdict on
the PR itself: **enrol a repo → a PR triggers an analysis → review the per-case
items → run the impacted cases → a comment and a check are posted back.**

```
manage_pr_impact(action="register_repo", integration_id=N,
                 name="acme-web", repo_full_name="acme/acme-web",
                 default_branch="develop")
manage_pr_impact(action="list_repos")          # read syncStatus on every row
manage_pr_impact(action="get_settings")
manage_pr_impact(action="trigger", repo_id=R, pr_number="142")   # 202: dispatched
manage_pr_impact(action="find_by_pr", repo_id=R, pr_number="142")
manage_pr_impact(action="get_analysis", analysis_id=A)
manage_pr_impact(action="list_items", analysis_id=A, item_class="UPDATE")
manage_pr_impact(action="set_item_status", analysis_id=A, item_id=I, status="APPROVED")
manage_pr_impact(action="run", analysis_id=A, environment_id=E)
```

Connect the provider first — that is `/cqa-integrations`. Registration always
binds the current workspace, because a repo with no workspace blocks every
analysis.

### The setting that silently drops every PR

**`baseBranchFilter` defaults to the literal string `main`.** In an organisation
that merges into `develop` or `qa` it drops every pull request, and nothing
anywhere reports it. This is the first thing to check when a correctly-connected
repo analyses nothing.

`testRunTiming` decides whether anything runs at all:

| Value | Behaviour |
|---|---|
| `OFF` (default) | analyse and report, run nothing |
| `POST_MERGE` | run after merge, once `deploymentWaitMinutes` (0–720) elapses or a deployment webhook arrives early |
| `PRE_MERGE` | run against the PR's own build, matched on the head commit |

Settings are an **org-wide singleton**, not per workspace. `update_settings`
takes only the keys to change and merges them, because the underlying PUT is a
full replace that would otherwise blank the rest of the org's configuration.

**Stored but not enforced by the platform:** `preventSelfApproval`,
`requiredApproversQuorum`, `archiveApproversQuorum`, `autoDemoteOnRejections`,
`pathScope`, `suppressPaths`, `maxFilesPerAnalysis`. They come back in the
object and read like working controls. Never describe them to a user as active —
`pathScope` in particular looks like a working filter and matches nothing. And
setting `autonomyTier` to `AUTONOMOUS` **also forces `autoPromoteOnMerge` true
server-side**, so the object can come back changed in a field you did not send.

### Running and reviewing

- **A PR is identified by `(repo_id, pr_number)`.** The head sha plays no part:
  there is one analysis row per PR forever, and each new commit overwrites it in
  place. `analysis_id` is that row's `uploadRequirementId`; find it with
  `find_by_pr`.
- The PR number is the **entire** request body for `trigger` — title, author,
  base branch and SHAs are read server-side from the integration, so a PR cannot
  be analysed against metadata that is not its own.
- **Poll on `lifecycle`, never on `status`.**
- `RERUN` items imply **no test-case change** — branch on `analysisClass` before
  acting. `AUTO_APPROVED` is server-set and always rejected.
- `set_item_status` fails once the analysis is finalized.

### Four shapes that read as success and are not

1. **`preflight: null` means "not checked", not "clean."** A real report with
   `casesWithFindings == 0` is the earned all-clear.
2. **A `COMPLETED` run with 0 passed and 0 failed executed nothing.**
3. **`FAILED_TO_START` is infrastructure, not a red build.**
4. **`decisionCarry: null` means nothing carried**, not that everything survived.

And a naming trap: the run block is on the wire as `postMergeRun` **even for a
pre-merge run** — the class was renamed and the JSON key deliberately was not.
Read its `timing` field.

The preflight itself is a static advisory scan asking "would a run against this
environment actually exercise the PR's build?". Its findings are ordered by how
badly they mislead: **`FIXED_URL` — a literal address in a step — is worse than
`MISSING_ENV_KEY`**, because the case *passes*, having tested the old
deployment. That is the argument for `/cqa-environments` in one finding.

## Rules

- No mutating tool before the Step 5 gate.
- Steps 1 and 3 run their agents in parallel, one message each.
- Never pass a raw ticket URL as `description`.
- `analyze_impact` returning 0 affected cases is not "no work" — use Affected
  Areas for gaps.
- Label every finding with its source. One-sided confidence should look
  one-sided.
