# Install prompt

Paste this into any coding agent — Claude Code, Codex, Cursor, Antigravity,
Claude Desktop, and the rest. It is the whole install.

---

> Install the ContextQA skills. If you're in Claude Code, run
> `claude plugin marketplace add indivatools/contextqa-skills`, then
> `claude plugin install contextqa@contextqa` — that installs all thirteen skills
> and wires up the hosted ContextQA MCP server in one step. If you're in another
> agent, run `npx skills add indivatools/contextqa-skills` and select your agent,
> then point your MCP client at `https://mcp.contextqa.com/mcp` (setup per client:
> https://mcp.contextqa.com/docs). Use one installation method. The first MCP tool
> call opens a browser to sign in — keep the redirect tab open until it finishes.
> You can read the master skill directly at
> https://github.com/indivatools/contextqa-skills/blob/main/skills/contextqa/SKILL.md
> (raw:
> https://raw.githubusercontent.com/indivatools/contextqa-skills/main/skills/contextqa/SKILL.md).
> Then use the ContextQA skill when working on tests for this project, and run
> `/cqa-init` once to confirm the connection and see what the tenant holds.

---

## Shorter variants

**Claude Code, one line each:**

```bash
claude plugin marketplace add indivatools/contextqa-skills
claude plugin install contextqa@contextqa
```

**Any other agent:**

```bash
npx skills add indivatools/contextqa-skills          # all thirteen
npx skills add indivatools/contextqa-skills -g       # globally, all agents
npx skills add indivatools/contextqa-skills -a codex # to one agent
npx skills add indivatools/contextqa-skills --skill contextqa   # master only
npx skills add indivatools/contextqa-skills --list   # look before installing
```

The skills CLI supports Claude Code, Codex, Cursor, Antigravity, Claude Desktop,
OpenCode, Goose, Kilo Code and ~40 others — see
[the skills CLI README](https://github.com/vercel-labs/skills).

**MCP only, no skills** (the server ships its own skill catalogue, so this is a
viable minimum):

```json
{ "mcpServers": { "contextqa": { "url": "https://mcp.contextqa.com/mcp" } } }
```

```bash
claude mcp add --transport http contextqa https://mcp.contextqa.com/mcp
```

## Verifying

Ask the agent to run `/cqa-init`. It confirms the MCP is connected and
authenticated, names the org, user and active workspace, lists the live skill
catalogue the server is serving, and reports a ready/not-ready verdict with the
next step.
