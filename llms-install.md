# ContextQA remote MCP setup

Requires a ContextQA account with workspace access and an MCP client supporting Streamable HTTP and OAuth. The hosted service is https://mcp.contextqa.com/mcp; no local server process is needed.

In Cline, open MCP Servers → Remote Servers and add ContextQA using that URL and Streamable HTTP. Complete the OAuth sign-in with your own ContextQA account. If your installed Cline version does not support OAuth remote servers, stop and update the client instead of placing passwords in config.

For clients accepting mcpServers configuration, merge this entry without replacing existing servers:

```json
{"mcpServers":{"contextqa":{"type":"streamableHttp","url":"https://mcp.contextqa.com/mcp"}}}
```

Transport spelling varies by client; use its UI for Cline. Do not add reviewer credentials, access tokens, or tenant-specific URLs to shared files.

Verify using get_current_user, get_current_workspace, and get_test_cases with page 0 and size 1. Check the identity and workspace before changes. Test execution can consume existing ContextQA credits; do not run tests during installation validation.

Public documentation: https://mcp.contextqa.com/docs
