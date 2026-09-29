# ContextQA Agent Skills

Twelve agent skills that drive the [ContextQA](https://contextqa.com) MCP from
any compatible coding agent — Claude Code, Codex CLI, Cursor, Antigravity,
Claude Desktop, and the rest.

## Install

One paste-able prompt for any agent is in [`INSTALL.md`](INSTALL.md). The short
forms:

```bash
# Claude Code — skills + the hosted MCP server, in one install
claude plugin marketplace add indivatools/contextqa-skills
claude plugin install contextqa@contextqa

# Any other agent — skills via the open Agent Skills format (skills.sh)
npx skills add indivatools/contextqa-skills
```

Then point your MCP client at **`https://mcp.contextqa.com/mcp`** if the plugin
didn't do it for you ([per-client setup](https://mcp.contextqa.com/docs)). The
first tool call opens a browser to sign in — keep the redirect tab open until it
completes.

Run `/cqa-init` once to confirm everything is wired.

## What's in the box

**Start here.**

| Skill | Use when |
|---|---|
| **`contextqa`** | The master skill. How the platform fits together — tenancy, the object graph, the variable token grammar, the execution path — plus the practices that make a suite last, and routing to everything below. Reads the MCP's own live skill catalogue rather than going stale. |

**Setup and authoring.**

| Skill | Use when |
|---|---|
| **`cqa-init`** | First run, or a connection check. Names the org, user and workspace; reports ready/not-ready with the next step. |
| **`cqa-environments`** | Environments, base URLs, secrets, global variables, test-data profiles. The `*\|env\|` / `${case}` / `{{global}}` / bare-column grammar, and why a literal URL in a step is the most expensive kind of green test. |
| **`cqa-author`** | Author cases from a source — ticket, Swagger/OpenAPI, Figma, video, Excel, n8n workflow, code diff, or free-text requirements. |
| **`cqa-locators`** | Write typed steps against known locators. The wire format that actually runs, the five locator sources, elements and screens, and repairing an AI-heavy case you inherited. |
| **`cqa-suites-and-plans`** | Group cases into suites and build the plans that run them — browsers and devices, environment binding, parallelism, notifications, folders and tags. |

**Running and fixing.**

| Skill | Use when |
|---|---|
| **`cqa-regression`** | Run a plan, poll it out loud, cluster the failures, dispatch read-only triage, hand confirmed bugs onward. |
| **`cqa-debug`** | One failing case or bug ticket: gather telemetry in parallel, separate a product bug from a broken test, fix, verify, report. Capped at 3 attempts. |
| **`cqa-impact`** | A ticket, PR or branch: which existing cases are affected, which coverage is missing. Layers the model's own case-step matcher and caller graph on top of `analyze_impact`. |
| **`cqa-bug-hunter`** | Find as many bugs as possible in a deployed UI — recon, adversarial hypothesis matrix, generate at scale, execute, triage by severity. Built for breadth. |

**Connecting things.**

| Skill | Use when |
|---|---|
| **`cqa-integrations`** | Connect GitHub, GitLab, Slack, Linear or Jira; enrol a repo for PR impact; work out why a connection says "connected" and does nothing. |
| **`cqa-tunnel`** | The app under test only runs locally. Publish `localhost` as an HTTPS URL the runners can reach, wire it into an environment, and optionally host a browser inside the network. |

## How to invoke

Each skill auto-triggers on prompts matching its description, or name it:

```
> /cqa-init
> /cqa-environments set up Dev, Staging and Prod with a base url
> /cqa-author from this Linear ticket: <paste body>
> /cqa-locators write the login steps, the locators are in case 1204
> /cqa-suites-and-plans build a smoke plan on chrome and edge
> /cqa-regression run the nightly plan
> /cqa-debug result_id 1406
> /cqa-impact PR #142
> /cqa-bug-hunter https://staging.myapp.com
> /cqa-tunnel expose localhost:3000
```

For `/cqa-impact` and `/cqa-debug`, **plain pasted text is a first-class
input** — no Linear/Jira MCP required. The skills opportunistically use Linear
MCP, Jira MCP, GitLab MCP, `gh` or `glab` when present, and ask for a paste when
absent.

## Design principles

- **The server's skills are the source of truth.** The MCP ships its own
  catalogue (`list_contextqa_skills` / `get_contextqa_skill`) and serves
  whatever version it is running. These skills are workflows that *read* that
  catalogue for platform ground truth, which is why they don't go stale when the
  platform moves.
- **Investigate → plan → confirm gate → execute.** Every skill that mutates
  ContextQA state, spends credits, or writes to a live app stops at a
  confirmation gate first.
- **Parallel for breadth, sequential for depth.** Investigation dispatches
  parallel subagents in one message; execution runs one subagent per unit of
  work in batches of ≤5. Triage subagents are always read-only.
- **Name what reads as success and is not.** A `COMPLETED` run with 0 passed and
  0 failed executed nothing; `preflight: null` means "not checked", not "clean";
  `FAILED_TO_START` is infrastructure, not a red build. The skills say so rather
  than reporting a green tick.
- **Typed steps are the goal; AI steps are a bootstrap.** Except in
  `cqa-bug-hunter`, where the cases are deliberately disposable probes.
- **Secrets are gated, never incidental.** The app password usually has to reach
  the automation for QA to work — so it gets an explicit warn → permission →
  not-production check, and the alternative where the user signs in themselves
  is always offered.

## Prerequisites

These skills *call* ContextQA tools — they don't ship the tools themselves.

1. **A ContextQA account** — [contextqa.com](https://contextqa.com).
2. **The ContextQA MCP configured** in your agent (the Claude Code plugin does
   this for you). Hosted at `https://mcp.contextqa.com/mcp`.
3. **A first-run sign-in** — the first tool call opens a browser for OAuth.

`cqa-init` handles every step of this in-conversation if anything is
misconfigured.

## Issues, contributions, ideas

- Issues and feature requests:
  [`indivatools/contextqa-skills/issues`](https://github.com/indivatools/contextqa-skills/issues)
- Source MCP server:
  [`indivatools/cqa-mcp`](https://github.com/indivatools/cqa-mcp)
- ContextQA platform: [contextqa.com](https://contextqa.com)

## License

MIT. See [`LICENSE`](LICENSE).
