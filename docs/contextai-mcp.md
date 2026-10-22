![ContextQA](../assets/contextqa-logo.svg)

# ContextAI MCP

ContextAI MCP gives compatible AI assistants access to an agent-evaluation workspace. Use it to prepare evaluation coverage, connect an agent, start a test run and inspect the resulting evidence. The assistant chooses from structured tools; ContextAI performs the configured evaluations.

ContextAI evaluates text and voice agents. ContextQA's separate MCP server at `https://mcp.contextqa.com/mcp` handles enterprise application test automation. Connect to the server that matches your task.

## Connect

| Setting | Value |
|---|---|
| Server | `https://ai.contextqa.com/api/mcp` |
| Transport | Streamable HTTP |
| Authentication | OAuth 2.1 with PKCE through the browser |
| Reporting scope | `contextai:read` |
| Authoring and execution | `contextai:read` and `contextai:write` |
| OAuth discovery | `https://ai.contextqa.com/.well-known/oauth-authorization-server` |

You need an active ContextAI account, access to the intended workspace and a client that supports Streamable HTTP and browser OAuth. Setting up a new evaluation target also requires the appropriate agent endpoint and provider configuration. Add secrets through the product's designated configuration fields; do not paste them into a conversation.

For a client with a remote-server configuration screen, add the server URL above, choose HTTP/OAuth when asked, and connect. During browser sign-in, select the workspace, review access and allow the callback to finish. A static ContextAI bearer token is not required.

For Claude Code, register the remote server:

```sh
claude mcp add --transport http contextai https://ai.contextqa.com/api/mcp
```

Then use `/mcp` to inspect the connection and finish sign-in.

For Codex CLI, register the URL:

```sh
codex mcp add contextai --url https://ai.contextqa.com/api/mcp
```

Use `codex mcp login contextai` to start authentication. Client versions and organization policies can affect connection options; consult the installed client's help for scope-selection flags and supported settings.

Begin with a read request:

> List my active ContextAI projects. Do not create resources, test connections or start an evaluation.

Confirm that the returned projects belong to the workspace selected during sign-in. Request the read-only scope when the client only needs reports or configuration inspection.

## Tools

The table describes the documented tool surface. The authenticated server's tool discovery is authoritative for the exact schema, available actions and permissions on your connection.

| Tool | Role |
|---|---|
| `manage_project` | Inspect and maintain evaluation projects, including archive/restore operations. |
| `manage_connection` | Configure agent connections and, when requested, test reachability. |
| `manage_agent` | Maintain evaluation targets and their associated connections. |
| `manage_metric` | Define scoring criteria using deterministic rules or an LLM rubric. |
| `manage_behavior` | Organize behavioral coverage and its evaluation settings. |
| `manage_use_case` | Define the workflows and goals represented by the test suite. |
| `manage_persona` | Configure synthetic user characteristics and conversational styles. |
| `manage_test_case` | Find and maintain static or dynamic conversational tests. |
| `add_document` | Add reference text used in evaluation design. |
| `manage_variable` | Maintain ordinary, secret or generated project values. |
| `manage_knowledge_base` | Configure supported project knowledge sources and test their connectivity. |
| `generate_test_cases` | Produce cases from the project's configured context and coverage. |
| `list_agents` | Retrieve available evaluation targets. |
| `start_run` | Launch one evaluation for the selected project and target. |
| `get_run` | Inspect progress for an existing run identifier. |
| `get_run_results` | Read completed evaluation results and diagnostic breakdowns. |

Read-only permission does not authorize editing or evaluation execution. Connection checks, generation and evaluation runs can make external requests and consume provider or product credits.

## Evaluate an agent

1. Identify the intended project and agent. Inspect the connection, modality, metrics, personas and test cases.
2. Review a small scenario set and the expected outcomes before starting an evaluation.
3. When appropriate, check the target connection with `manage_connection`. A successful check establishes reachability at that moment; it does not guarantee every conversation will succeed.
4. Start one evaluation with `start_run` and retain its full returned UUID.
5. Poll that same UUID with `get_run`, then use `get_run_results` after completion.

Results can include pass rate, metric scores, behavioral and use-case breakdowns, latency and explanations for failed cases. Keep observed results separate from an assistant's suggested explanation or fix.

## Voice-agent workflow

The documented voice connection types include Amazon Connect WebRTC and generic phone. Other documented agent adapters include Amazon Connect, Amazon Bedrock, Salesforce Agentforce, Snowflake Cortex and custom HTTP; confirm the adapter and modality required by the target.

Create or select a voice evaluation project, configure the target connection and define representative synthetic callers. Static cases express planned conversational turns; dynamic cases give the simulated caller goals and let the conversation adapt to the agent's responses. Review the scoring criteria and expected business outcome before execution.

Start with a small, permitted set of scenarios such as a straightforward request, ambiguous intent, interruption, silence or escalation. After a run, compare task completion, configured metrics and latency, then inspect the failed-case explanations. Access to raw recordings or transcripts may require the ContextAI web application.

Business-outcome verification and changes to the tested agent are separate concerns. Platform verifier endpoints can supply ground-truth evidence during grading where configured. That does not mean the MCP server exposes general management of the agent's own tools or automatically changes its implementation.

## Boundaries and troubleshooting

The reviewed tool surface does not establish standalone agent-tool management, document listing/editing/deletion, historical run discovery, or raw audio/transcript download through MCP. Keep the full run UUID from the start response. If tools change, use the current discovered manifest rather than guessing a replacement operation.

If tools are missing, check that the server is enabled and authenticated, then start a new client session. If reads succeed but writes fail, inspect the approved scopes and workspace access. When authorization finishes, wait for the callback before leaving the browser page.

After a timeout, check whether the original operation completed before retrying. Avoid starting a second evaluation merely because the client stopped waiting. To investigate a voice failure, compare the connection result, evaluation evidence and the relevant provider or browser diagnostics; one error message alone may not identify the failing component.

## Product links

- [ContextQA](https://contextqa.com/)
- [AI-agent testing](https://contextqa.com/platform/ai-agents-testing/)
- [Voice-agent testing](https://contextqa.com/solutions/voice-agent-testing/)
- [ContextQA application-testing MCP documentation](https://mcp.contextqa.com/docs)

## Distribution note

This repository provides integration documentation and client-side skills. The hosted ContextAI service is a separate commercial product; the repository license does not license the hosted server implementation. Available tools and plan capabilities depend on the signed-in workspace.
