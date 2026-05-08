---
name: cqa-init
description: Use FIRST in any ContextQA session — verifies the MCP is connected, authenticated, and the tenant is reachable. Detects the agent (Claude Code / Codex / Cursor / Antigravity / Claude Desktop), checks the right config file, runs a minimal probe, and reports a ready/not-ready verdict with the exact next step to fix it. Triggers on phrases like "set up contextqa", "is the contextqa mcp working", "/cqa-init", "verify my mcp", "first time with contextqa", or proactively when the user asks to use any other `/cqa-*` skill and connection state is unknown.
---

# ContextQA Init / Health Gate

Run this once at the start of a ContextQA session, or any time tool calls start failing. Output is a single ready/not-ready verdict + the exact next move. Don't re-run if a successful verdict already exists this session.

## Step 0 — Identify the agent

Detect which agent is running so the config guidance lands correctly. Check (cheapest first):

- **Claude Code** — env `CLAUDECODE=1` or `claude --version` works
- **Codex CLI** — `codex --version` works or `~/.codex/` exists
- **Cursor** — `~/.cursor/` and tools listed via Cursor's MCP UI
- **Antigravity** — `~/.gemini/antigravity/` exists
- **Claude Desktop** — `~/Library/Application Support/Claude/` exists (mac)

Record the detected agent. If multiple, ask once which the user is currently driving.

## Step 1 — Verify MCP is configured

For the detected agent, point at the right config file and check whether `contextqa` is listed:

| Agent | Config |
|---|---|
| Claude Code | `claude mcp list` (CLI) — look for `contextqa` |
| Codex CLI | `~/.codex/mcp_config.json` — look for `mcpServers.contextqa` |
| Cursor | `~/.cursor/mcp.json` — look for `mcpServers.contextqa` |
| Antigravity | `~/.gemini/antigravity/mcp_config.json` — look for `mcpServers.contextqa` |
| Claude Desktop | `~/Library/Application Support/Claude/claude_desktop_config.json` — look for `mcpServers.contextqa` (uses `mcp-remote` bridge) |

If missing, print the right snippet for that agent (hosted URL: `https://mcp.contextqa.com/mcp`) and instruct the user to add it + restart the agent. Then stop — re-run `/cqa-init` after restart.

## Step 2 — Probe connection (read-only, no side effects)

Call ONE lightweight read tool to confirm the session is authenticated and the tenant is reachable: `mcp__contextqa__list_knowledge_bases` or `mcp__contextqa__get_test_plans(size=1)`.

Interpret the response:
- **Successful JSON-shaped response** → connection healthy. Continue to Step 3.
- **Tool not found / not loaded** → MCP isn't wired into this agent session. Instruct: restart the agent after adding config, or run the agent's MCP refresh command.
- **OAuth / auth error** → tell the user to complete browser login (don't close the redirect tab early — that's the most common failure per the README). Wait, then re-probe.
- **`invalid session`** → expired session; instruct the user to sign in again from their MCP client.
- **Network / HTTP 5xx** → hit `/health` on the configured endpoint; if down, surface to the user — there's nothing to fix locally.

## Step 3 — Tenant orientation (one read pass)

When healthy, run these in parallel via ONE Agent message and synthesize a one-paragraph orientation:

- `get_test_plans(size=5)` — confirms workspace + shows recent plans
- `get_test_cases(size=5)` — confirms case access
- `list_knowledge_bases` — shows available custom prompts
- `get_environments(size=5)` — shows configured environments

Output: tenant has `<N>` plans, `<M>` test cases visible, `<K>` knowledge bases, `<E>` environments. Highlight the most-recently-run plan with its `last_run.result`.

## Step 4 — Ready verdict + next-skill suggestion

Print a single block:

> ✓ ContextQA MCP ready — Agent: `<agent>` · Tenant: `<workspace_name>` · Cases: `<count>` · Plans: `<count>` · Last plan run: `<result>` (`<when>`).
>
> Next steps you can run now:
> - **Find bugs in a deployed UI** → `/cqa-bug-hunter <url>`
> - **Impact-analyze a ticket / PR** → `/cqa-impact <ref>`
> - **Author tests from a source** → `/cqa-author <source>`
> - **Debug a failing run** → `/cqa-debug <result_id>`
> - **Run a regression** → `/cqa-regression <plan>`

If Step 2 or 3 surfaced a feature limitation (e.g. an earlier `/cqa-author` returned a `/requirements/upload` 404), include a line: *"⚠ Tenant gating detected: the requirements pipeline (swagger / figma / video / excel / requirements / code-diff) is disabled. `/cqa-author` will fall back to manual creation for those sources."*

## Rules

- This skill is the **only** one that talks about agent config files. Other `cqa-*` skills assume the MCP is already wired and call tools directly.
- Don't re-probe if a successful verdict was issued this session — `/cqa-init` is a one-shot check, not a heartbeat.
- Never write to the user's MCP config files automatically — print the snippet, let the user paste it. Config edits are user-confirmable, not auto-applied.
- Step 2 must use a read-only tool. Don't use `execute_test_case`, `execute_test_plan`, or any `create_*` / `bug_fix_from_ticket` tool as a probe — those mutate or consume time.
- If the user explicitly skips init (`"skip init"` / `"already done"`), respect that and don't gate other skills.
