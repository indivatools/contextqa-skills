# Connect ContextQA to coding agents

Endpoint: `https://mcp.contextqa.com/mcp`. Transport: Streamable HTTP. Authentication: OAuth with your own ContextQA account. Never put account passwords in repository configuration.

## Devin

Open Customize → MCPs → New MCP. Name it ContextQA, select HTTP, enter the endpoint above and choose OAuth. Save, then Connect and Test tools. Organization-wide setup requires the corresponding admin permission. Public marketplace submission is tracked at https://github.com/CognitionAI/devin-marketplace/pull/238. A personal installation does not make a plugin publicly listed.

## Cursor

This repository includes `.cursor-plugin/plugin.json`, thirteen skills and the hosted MCP declaration in `.mcp.json`. For direct MCP setup, add the following to `.cursor/mcp.json`, preserving any existing servers:

```json
{"mcpServers":{"contextqa":{"url":"https://mcp.contextqa.com/mcp"}}}
```

Complete OAuth in Cursor. Marketplace submission is separate from local installation.

## Factory / Droid

Factory supports vendor-owned catalogs and does not accept outside submissions to its first-party catalog. ContextQA is available from this public repository:

```sh
droid plugin marketplace add https://github.com/indivatools/contextqa-skills
droid plugin install contextqa@contextqa-skills --scope user
droid plugin list --scope user
```

Factory CLI 1.14.0 successfully installed and enabled the existing Claude-compatible package at commit `6526335be0b9`. OAuth is required before using the tools. This branch also provides native Factory manifests. For an MCP-only installation:

```sh
droid mcp add contextqa https://mcp.contextqa.com/mcp --type http
droid mcp auth
droid mcp list
```

The authentication command may also prompt for other configured servers. Use `/mcp` to select ContextQA interactively instead.

## Verify before running tests

Ask: “Confirm my ContextQA account and workspace, then list five test cases without running or changing anything.” Verify the returned identity and workspace. For an authorized run, retain the returned session ID, poll execution status, and report the actual verdict. Never equate completed status with a passing test.

## Registry publishing

`server.json` describes the hosted server for the official MCP Registry. Publication requires authentication that can prove control of the `indivatools` GitHub namespace. A checked-in descriptor is not proof of registry publication.

## References

- https://docs.devin.ai/work-with-devin/mcp
- https://cursor.com/docs/plugins
- https://docs.factory.com/harness/mcp
- https://github.com/modelcontextprotocol/registry

## 8090 Software Factory

Its documented MCP catalog is under Settings → Connections → MCP Servers. A public third-party submission route has not been verified. Do not confuse connecting an account with inclusion in that catalog.

## Cline

See `llms-install.md` for the remote MCP configuration. Cline marketplace submission requires a successful Cline-specific installation test; this has not yet been completed.
