---
name: cqa-init
description: Use FIRST in any ContextQA session — confirms the MCP is connected and authenticated, names the org, user and active workspace, and reports a ready/not-ready verdict with the exact next step. Detects the agent (Claude Code / Codex / Cursor / Antigravity / Claude Desktop) and points at the right config file when nothing is wired. Triggers on "set up contextqa", "is the contextqa mcp working", "verify my mcp", "first time with contextqa", "/cqa-init", or proactively when another `/cqa-*` skill is invoked and connection state is unknown.
---

# ContextQA Init / Health Gate

Run once at the start of a session, or whenever tool calls start failing. Output
is one ready/not-ready verdict plus the next move. Don't re-run if a successful
verdict already exists this session.

## Step 0 — Is the MCP already there?

If `list_contextqa_skills` or `get_current_user` is callable, the MCP is wired —
skip to Step 2. Only go hunting through config files when the tools are absent.

## Step 1 — Not wired: point at the right config

Detect the agent (cheapest check first) and print the snippet it needs. The
hosted server is **`https://mcp.contextqa.com/mcp`**; setup docs are at
[`mcp.contextqa.com/docs`](https://mcp.contextqa.com/docs).

| Agent | Where it goes |
|---|---|
| Claude Code | `claude mcp add --transport http contextqa https://mcp.contextqa.com/mcp` |
| Codex CLI | `~/.codex/mcp_config.json` → `mcpServers.contextqa` |
| Cursor | `~/.cursor/mcp.json` (or **Settings → Tools & MCP → New MCP Server**) |
| Antigravity | `~/.gemini/antigravity/mcp_config.json` |
| Claude Desktop | **Settings → Connectors → Add custom connector** |

```json
{ "mcpServers": { "contextqa": { "url": "https://mcp.contextqa.com/mcp" } } }
```

Then stop: the agent must restart before the tools appear. Re-run `/cqa-init`
after the restart.

**Never edit the user's MCP config yourself.** Print the snippet and let them
paste it.

## Step 2 — Orient (this is the real check)

```
get_current_user          # org (tenant) + signed-in user + active workspace, in one call
get_current_workspace
```

`get_current_user` is the right probe: it is read-only, cheap, and it answers
the question that actually matters — *which tenant and which workspace am I
about to write into?* Never infer either from the API host or from data that
came back.

Interpret the failure modes:

| What you see | What to do |
|---|---|
| tool not found | the MCP isn't loaded in this session — add config, restart |
| OAuth / auth error | complete the browser login and **keep the redirect tab open until it finishes** — closing it early is the most common failure |
| `invalid session` | the session expired; sign in again from the MCP client |
| workspace missing | `list_workspaces` → `switch_workspace(workspace_version_id=N)` |
| network / 5xx | nothing local to fix; say so |

**A missing session must fail loudly.** Never work around it — a fallback to
service-account credentials would run as a different identity in a different
workspace.

## Step 3 — Read the live skill catalogue

```
list_contextqa_skills
```

The MCP serves its own skills and they are newer than anything installed
locally. Report what it offers, and read `contextqa-platform` before doing
anything substantive if the session does not already know the platform.

## Step 4 — Tenant snapshot

Run these together and synthesise one paragraph:

```
get_test_plans(size=5)        # each carries a last_run summary
get_test_cases(size=5)
list_environments()
list_knowledge_bases
```

Note three things while you are here, because they all change what other skills
should do:

- **Is this a Ship org?** Ship (`ship.contextqa.com`) is the self-serve product:
  its onboarding runs a mandatory website crawl, so a fresh Ship org **already
  has generated test cases** before anyone authors one. Read them before
  creating anything. Ship also enrols exactly one repository and excludes mobile
  testing at launch.

- **Does a default test plan exist?** A workspace version without one cannot
  execute anything — `execute_test_case` answers `400` with an empty body. Say
  so now rather than after fifteen cases have been authored into it.
- **Is there a usable environment with a base URL?** If every case carries a
  literal address instead, flag it and point at `/cqa-environments`.

## Step 5 — Verdict

> ✓ ContextQA ready — Org `<tenant>` · User `<name>` · Workspace `<name>` (`v<id>`)
> · Cases `<n>` · Plans `<n>` (last run: `<result>`, `<when>`) · Environments `<n>`
>
> Next: `/cqa-environments` · `/cqa-author` · `/cqa-suites-and-plans` ·
> `/cqa-regression` · `/cqa-debug` · `/cqa-impact` · `/cqa-bug-hunter` ·
> `/cqa-tunnel`

Add a warning line for anything found in Step 4 — no default plan, no
environment, or a tenant where the `/requirements/upload` pipeline is disabled
(which gates `/cqa-author`'s swagger / figma / video / excel / requirements /
code-diff paths and makes it fall back to manual authoring).

## Rules

- This is the **only** skill that talks about agent config files. The others
  assume the MCP is wired and call tools directly.
- The probe must be read-only. Never use `execute_test_case`, `execute_test_plan`,
  any `create_*`, or `bug_fix_from_ticket` as a health check.
- One-shot, not a heartbeat. Don't re-probe after a successful verdict.
- If the user says "skip init", respect it and don't gate anything.
