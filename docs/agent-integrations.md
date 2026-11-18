# Connect ContextQA to coding agents

Endpoint: `https://mcp.contextqa.com/mcp`. Transport: Streamable HTTP. Authentication: OAuth with your own ContextQA account. Never put account passwords in repository configuration.

## Devin

Open Customize → MCPs → New MCP. Name it ContextQA, select HTTP, enter the endpoint above and choose OAuth. Save, then Connect and Test tools. Organization-wide setup requires the corresponding admin permission. Marketplace placement is a separate application through Suggest MCP Integration.

## Cursor

This repository includes `.cursor-plugin/plugin.json`, thirteen skills and the hosted MCP declaration in `.mcp.json`. For direct MCP setup, add the following to `.cursor/mcp.json`, preserving any existing servers:

```json
{"mcpServers":{"contextqa":{"url":"https://mcp.contextqa.com/mcp"}}}
```

Complete OAuth in Cursor. Marketplace submission is separate from local installation.

## Factory / Droid

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
