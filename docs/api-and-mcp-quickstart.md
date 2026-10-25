# ContextQA REST API and MCP quickstart

Use REST from CI/CD and workflow tools for specific operations. Use hosted MCP from a compatible assistant to discover and call the tools available to your workspace.

- [Create a ContextQA account](https://ship.contextqa.com/)
- [Hosted MCP setup and tool documentation](https://mcp.contextqa.com/docs)
- [Agent skills and installation](../README.md)
- [Postman read-only quickstart](../examples/postman/contextqa-read-only.postman_collection.json)
- [Postman explicit single-run action](../examples/postman/contextqa-execute-once.postman_collection.json)

## Authentication

A superadmin creates a scoped REST API token in **Settings → API tokens**. Send `Authorization: Bearer <your-token>` to `https://server.contextqa.com`. The token identifies its tenant; no tenant header is required. Store it privately, never in a published collection or example response.

REST tokens and MCP OAuth are separate flows. Token management is outside this token-authenticated REST contract. Permissions and optional IP restrictions still apply. The limit is 120 requests per minute per token. Respect `Retry-After` on 429 responses. Do not retry a request that might have started work.

## Read-only first

Import `contextqa-read-only.postman_collection.json`. Set `contextqa_api_token` privately, keep the supplied HTTPS base URL and begin with **List test plans**. The collection also lists cases, suites and plan results. Set `runId` to an existing numeric test-plan-result ID to inspect its summary.

These examples contain no saved customer responses. They were prepared from the API specification; authenticated REST validation has not been recorded for this package. Required permissions are stated on each request.

## Start one run explicitly

Import the separate execution collection. Inspect the selected plan and its default environment and obtain approval for the intended target. Set collection variables `planId` and `execution_approval=RUN_ONCE`. The pre-request guard clears approval before sending.

```text
GET /test_plans/{id}/execution
```

Despite using GET, this starts a billed, non-idempotent run. Do not run the execution collection through a runner, monitor or schedule. Do not enable automatic retries. Retain the returned `id`. After an ambiguous timeout, locate the original run before another attempt. The API-token contract does not expose a stop-run operation.

## Inspect the original result

Use **Read existing run summary** in the read-only collection:

```text
GET /test_plan_results/{id}/results
```

Keep the same run ID. Wait for `status=STATUS_COMPLETED`, then inspect `result` and counts. Queued or in-progress work is not a pass; report aborted, stopped and empty runs accurately.

## Hosted MCP

Connect to `https://mcp.contextqa.com/mcp` using Streamable HTTP and browser OAuth. Start with `get_current_workspace` and confirm the intended workspace before making changes. Follow the [current client-specific setup guide](https://mcp.contextqa.com/docs).

Capabilities depend on the account, workspace and product. These REST examples do not imply that every MCP capability exists in REST. ContextAI voice/agent evaluation uses a separate service and account flow.

## n8n

Use n8n's [HTTP Request node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/) with a private Header Auth credential: name `Authorization`, value `Bearer <your-token>`. Restrict reuse to the ContextQA API host. Begin with a manual plan read, explicitly approve one run, persist its ID and poll that result. Leave execution retries off.

These examples do not establish a verified native n8n integration, public template acceptance or a live paid marketplace offering.
