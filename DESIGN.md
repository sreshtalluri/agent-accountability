# Design: Agent Accountability Record (AAR) v0.1

Status: draft for review. Nothing is implemented yet.

## Position

[LANDSCAPE.md](LANDSCAPE.md) found that signed per-call receipts with MCP and LangGraph capture already exist (Tracekit, Signet and others). This design does not try to beat them at tamper evidence. It targets the three things the landscape found open:

1. **The join.** One record that carries identity lifted from verified tokens plus the keys needed to find the same event in the IdP log and the trace backend.
2. **Model version as a first-class field.**
3. **OpenTelemetry as the primary export**, so the record lands in tooling teams already run.

The record answers one sentence: *agent A (model M, version V) acted for user B, under delegation chain X, approved by C, through tool D, with these inputs and this outcome.*

## Schema

One JSON object per event. Two kinds: `tool_call` and `approval`. Fields marked * are required.

```jsonc
{
  "schema": "aar/0.1",                 // *
  "id": "01JC...",                     // * ULID, sortable by time
  "kind": "tool_call",                 // * tool_call | approval
  "ts_start": "2026-11-09T21:07:03.120Z", // *
  "ts_end":   "2026-11-09T21:07:03.480Z", // * equals ts_start for approval

  "correlation": {
    "trace_id": "4bf9...",             // W3C trace id, reused from OTel context if present
    "span_id": "00f0...",
    "run_id": "lg-run-7c1",            // * framework run or MCP session id
    "parent_id": null                  // record id of the calling agent's tool_call, for sub-agents
  },

  "actor": {                           // * who acted
    "agent_id": "support-agent",       // *
    "agent_version": "1.4.2",
    "model": "claude-sonnet-5-5",      // * "unknown" when captured at a proxy that cannot see it
    "model_version": "claude-sonnet-5-5-20261001", // exact id returned by the provider
    "provider": "anthropic",
    "source": "response_metadata"      // response_metadata | config | asserted | unknown
  },

  "principal": {                       // * on whose behalf
    "id": "user:alice@example.com",    // *
    "issuer": "https://idp.example.com",
    "source": "token",                 // * token | asserted
    "token_ref": "sha256:9a1c..."      // hash of jti, or of the token if no jti. Never the token.
  },

  "delegation": [                      // ordered root first; empty when principal runs the agent directly
    {"from": "user:alice@example.com", "to": "agent:support-agent",
     "scope": ["tickets:write", "refunds:create"],
     "source": "token", "ref": "sha256:9a1c..."}
  ],

  "tool": {                            // * for tool_call
    "name": "refund.create",           // *
    "server": "payments-mcp",          // MCP server name or "local"
    "transport": "mcp",                // mcp | langgraph | direct
    "call_id": "toolu_01..."
  },
  "inputs_hash": "sha256:5d41...",     // * SHA-256 of RFC 8785 canonical JSON of the arguments
  "outcome": {                         // *
    "status": "ok",                    // ok | error | denied | pending_approval
    "outputs_hash": "sha256:...",
    "error": null
  },
  "approval_id": "01JB...",            // id of the approval record that released this call, if any

  "external_refs": [                   // where to look this event up elsewhere
    {"system": "auth0", "type": "ciba_auth_req_id", "value": "..."},
    {"system": "okta", "type": "system_log_uuid", "value": "..."}
  ],

  "sig": {"alg": "ed25519", "kid": "recorder-1", "value": "base64..."}
}
```

An `approval` record uses the same envelope and replaces `tool`, `inputs_hash`, `outcome` with:

```jsonc
"approval": {
  "approver": {"id": "user:bob@example.com", "issuer": "...", "source": "token"}, // same shape as principal
  "decision": "approved",              // approved | denied | expired
  "subject": {"tool": "refund.create", "inputs_hash": "sha256:5d41..."}, // binds the approval to exact arguments
  "channel": "langgraph_interrupt"     // langgraph_interrupt | mcp_elicitation | ciba | other
}
```

Schema rules:

- **`source` is mandatory on every identity.** `token` means the recorder read it from a JWT it validated (`sub`, and the RFC 8693 `act` chain for delegation). `asserted` means the application passed a string. A viewer must show the difference. This is the main thing most existing formats skip.
- **Hashes, not payloads.** Raw arguments and results are never stored by default. An opt-in `capture_payloads` flag writes them to a separate file so the main log stays safe to ship.
- **Approval binds to `inputs_hash`.** An approval for a 20 dollar refund does not cover an 80 dollar one.
- **Signature** covers the RFC 8785 canonical form of the record without `sig`. It is optional and off unless a key is configured.

## Integration points

All three produce the same record and need a principal. The principal is resolved in this order: validated token on the session, then `acting_for(...)` context, then run metadata, else the record is written with `principal.id = "unknown"` and flagged. It never guesses.

### 1. LangGraph callback handler

```python
graph.invoke(inputs, config={"callbacks": [AccountabilityHandler(recorder)],
                             "metadata": {"principal": "user:alice@example.com"}})
```

- `on_tool_start` / `on_tool_end` / `on_tool_error` emit one `tool_call` record.
- `on_chat_model_end` caches the model name and exact version from response metadata for the run, which fills `actor.model_version`.
- A human approval interrupt emits one `approval` record when the run resumes, with the approver taken from the resume config, and the next matching tool call gets its `approval_id`.

Zero changes to graph code. One line at the call site.

**Risk [Likely]:** how reliably a callback can see an interrupt and its resume differs across LangGraph versions. I have not verified this against the current release. If the callback cannot see it, the fallback is a two-line helper `record_approval(...)` called where the app resumes the graph. I will confirm in step 2c before writing the handler and tell you which path it took.

### 2. MCP proxy

```
aar-proxy --upstream https://payments.example.com/mcp --listen :8900 --log aar.jsonl
```

Point the client at the proxy instead of the server. It forwards everything and records each `tools/call` request and response pair.

- Principal and delegation come from the session's bearer token (`sub`, `act`, issuer) when it is a JWT the proxy can validate against a configured JWKS; otherwise from a configured header, marked `asserted`.
- `actor.model` is not visible at a proxy. It is read from `_meta` on the request if the client sets it, else `unknown`. This is an honest limit of proxy capture and the reason the handler and proxy records share a `trace_id`: the viewer joins them.
- Elicitation responses that approve a held call are recorded as `approval`.
- Supports stdio and streamable HTTP upstreams.

### 3. Server-side middleware (for MCP servers you own)

```python
server = FastMCP("payments")
server.add_middleware(AccountabilityMiddleware(recorder))
```

Same capture as the proxy without the extra hop, and it can record the outcome as the server saw it. Designed here, built after the first two (step 2c covers 1 and 2 only).

## Storage and export

| Option | What it is | Use when |
|---|---|---|
| **JSONL, append-only** (default) | One record per line, file opened `O_APPEND`, flushed per record. Optional `prev` field holding the previous record's hash when chaining is switched on. | Local dev, the demo, shipping to object storage. |
| **OpenTelemetry** | One span per record. `tool_call` spans are named `execute_tool {tool.name}` and reuse `gen_ai.tool.name`, `gen_ai.tool.call.id`, `gen_ai.agent.id`, `gen_ai.request.model`, `gen_ai.response.model` where those exist; everything else goes under `aar.*` (`aar.principal.id`, `aar.principal.source`, `aar.approval.approver.id`, `aar.inputs_hash`, ...). Approval is a span named `aar.approval` linked to the tool span. | You already run a collector. |

Both sinks can run at once. `gen_ai.*` names are still in Development upstream, so the mapping lives in one file and the JSONL schema is the source of truth.

## Public API (target: 8 names)

| Name | Purpose |
|---|---|
| `Recorder(sinks, agent_id, signing_key=None)` | Builds and writes records. |
| `JsonlSink(path)` | Append-only file sink. |
| `OtelSink(tracer_provider=None)` | OpenTelemetry sink. |
| `acting_for(principal, delegation=None)` | Context manager that sets the principal for code not carrying a token. |
| `record_approval(recorder, approver, decision, tool, inputs)` | Explicit approval record, for flows the handler cannot see. |
| `AccountabilityHandler(recorder)` | LangGraph / LangChain callback. |
| `run_proxy(upstream, listen, recorder)` | MCP proxy; also the `aar-proxy` CLI. |
| `read_records(path, principal=None, since=None, verify_key=None)` | Load, filter and optionally verify. Backs the viewer. |

## Deliberately left out

- **Policy enforcement and blocking.** This records; it does not decide. Gateways and Tracekit do that.
- **Tamper-evidence beyond a signature and optional hash link.** No witnesses, transparency logs, Merkle anchoring or receiver co-signing. If you need those, use Tracekit or Signet and treat this as the identity layer.
- **Completeness guarantees.** An in-process recorder can be bypassed by the process it runs in. The proxy is harder to bypass but only sees MCP traffic. The record proves what was captured, not that nothing else happened.
- **Issuing tokens or agent identities.** It consumes what Okta, Auth0, WorkOS, Descope or Entra issue.
- **Raw prompts, arguments and outputs** by default.
- **Key management.** One local Ed25519 key from a file or env var.
- **Languages other than Python**, and frameworks other than LangGraph, in v0.1.
- **A hosted service or database.** The viewer in step 2d reads JSONL.

## Decisions I need from you

1. **Build this, or contribute the fields to Tracekit?** My recommendation: build the small package, because you need a working demo for the discovery conversations in 2e and those conversations are the real test of whether the join matters. But describe it publicly as an identity join layer that sits beside Tracekit and friends, not as a new audit trail. Pitching it as "the missing accountability layer" will get corrected by the first engineer who has seen Tracekit. **[Likely]**
2. **Signature on or off by default?** I have it off. A self-signed record from the same process adds little, and it is the feature where existing projects are strongest.
3. **Name.** `aar` is a placeholder. The PyPI name needs checking before 2c.
4. **Is `unknown` model at the proxy acceptable**, given the demo question "which model version made the call" will be answered from the LangGraph side and joined by trace id?
