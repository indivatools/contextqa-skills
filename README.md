# ContextQA Agent Skills

Six agent skills that drive the [ContextQA](https://contextqa.com) MCP from any compatible coding agent — Claude Code, Codex CLI, Cursor, Antigravity, Claude Desktop, and the rest.

## What's in the box

| Skill | Use when |
|---|---|
| **`cqa-init`** | First-time setup or a connection check — verifies the ContextQA MCP is wired in your agent and reports a ready/not-ready verdict per agent. Run this first. |
| **`cqa-impact`** | A Jira/Linear/GitHub ticket, a PR, or a branch needs impact analysis — which existing tests are affected, which gaps need new coverage. Layers Claude's own analysis on top of `analyze_impact`. |
| **`cqa-author`** | Author tests from a source — ticket, Swagger/OpenAPI, Figma, video, Excel, n8n workflow, code diff, or free-form requirements. |
| **`cqa-debug`** | A test case is failing or a bug ticket needs to be reproduced and fixed. Investigates telemetry in parallel, hypothesises a root cause, fixes via subagent, verifies, reports. Cap of 3 attempts. |
| **`cqa-regression`** | Run a test plan as a regression suite, poll to completion, cluster failures, dispatch parallel read-only triage subagents, hand confirmed bugs to `cqa-debug`. |
| **`cqa-bug-hunter`** | Find as many bugs as possible in a deployed UI — recon via a ContextQA scout test case, generate dozens of adversarial / negative / edge-case test cases (idempotency, concurrency, validation, auth boundaries…), execute, triage, report. Built for breadth. |

## Prerequisites

These skills *call* ContextQA tools — they don't ship the tools themselves. You need:

1. **A ContextQA account** — sign up at [contextqa.com](https://contextqa.com).
2. **The ContextQA MCP configured** in your agent. The hosted server is at `https://mcp.contextqa.com/mcp` — point your agent's MCP config at it. Setup details: [`mcp.contextqa.com/docs`](https://mcp.contextqa.com/docs).
3. **A first-run sign-in** — the first MCP tool call opens a browser for OAuth. Keep the redirect tab open until it completes.

`cqa-init` handles every step of this in-conversation if anything is misconfigured.

## Install

These skills are published in the open Agent Skills format ([skills.sh](https://skills.sh)). Install with the [skills CLI](https://github.com/vercel-labs/skills):

```bash
# All six skills, into your default agent's skills directory
npx skills add indivatools/contextqa-skills

# Just one
npx skills add indivatools/contextqa-skills --skill cqa-init
npx skills add indivatools/contextqa-skills --skill cqa-bug-hunter

# Globally for all your agents
npx skills add indivatools/contextqa-skills -g

# To a specific agent
npx skills add indivatools/contextqa-skills -a claude-code
npx skills add indivatools/contextqa-skills -a codex
npx skills add indivatools/contextqa-skills -a cursor

# List before installing
npx skills add indivatools/contextqa-skills --list
```

Supported agents include Claude Code, Codex, Cursor, Antigravity, Claude Desktop, OpenCode, Goose, Kilo Code, and ~40 others — see the [skills CLI README](https://github.com/vercel-labs/skills) for the full list.

## How to invoke

Each skill auto-triggers on natural-language prompts that match its description, or you can name it explicitly:

```
> /cqa-init
> /cqa-impact PR #142 in our repo
> /cqa-author from this Linear ticket: <paste body>
> /cqa-debug result_id 1406
> /cqa-regression run the smoke plan
> /cqa-bug-hunter https://staging.myapp.com
```

For `/cqa-impact` and `/cqa-debug`, **plain pasted text is a first-class input** — no Linear/Jira MCP required. The skills opportunistically use Linear MCP, Jira MCP, GitLab MCP, `gh` CLI, or `glab` CLI when present, and ask for a paste when absent.

## Design principles

- **Investigate → plan → confirm gate → execute.** Every skill that mutates ContextQA state stops at a confirmation gate before authoring or running anything.
- **Parallel subagents for breadth, sequential for depth.** Investigation phases dispatch parallel agents (one Agent message, multiple invocations); execution phases use one subagent per unit of work in batches of ≤5.
- **Model agency on top of the MCP.** `cqa-impact` runs its own case-step matcher and caller-graph analysis on top of `analyze_impact` — surfacing what the MCP misses with explicit `MODEL-FOUND` labels.
- **Lean by default.** Each skill is 500–1200 words. Rules sections capture must-not-violate constraints; everything else is the phased flow.
- **Tenant feature gating is handled gracefully.** When the `/requirements/upload` pipeline isn't enabled (gates `cqa-author`'s swagger / figma / video / excel / requirements / code-diff paths), the skill falls back to manual case authoring instead of failing silently.

## Trigger reference

| Skill | Sample triggers |
|---|---|
| `cqa-init` | "set up contextqa", "is the contextqa mcp working", "verify my mcp", "first time with contextqa" |
| `cqa-impact` | "impact analysis", "what tests does this PR affect", "which contextqa tests should I rerun" |
| `cqa-author` | "create tests from this ticket", "generate tests from swagger/figma/video", "build a test suite for X" |
| `cqa-debug` | "debug this failure", "fix this failing test", "why did test result <id> fail", "reproduce and fix this bug" |
| `cqa-regression` | "run the regression", "execute test plan X", "run my smoke suite and triage", "kick off nightly tests" |
| `cqa-bug-hunter` | "find bugs in this app", "hunt for bugs at <url>", "build adversarial coverage", "stress-test this deployed UI" |

## Issues, contributions, ideas

- Issues, feature requests: [`indivatools/contextqa-skills/issues`](https://github.com/indivatools/contextqa-skills/issues)
- Source MCP server: [`indivatools/cqa-mcp`](https://github.com/indivatools/cqa-mcp)
- ContextQA platform: [contextqa.com](https://contextqa.com)

## License

MIT. See [`LICENSE`](LICENSE).
