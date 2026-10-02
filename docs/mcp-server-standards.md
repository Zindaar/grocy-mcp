# How to build an MCP server correctly (spec 2026-07-28, Python SDK v2)

This is the working standard for grocy-mcp. It is synthesised from the primary sources recorded in this repository, not from memory:

- the MCP specification, revision **2026-07-28** (`docs/mcp-spec/2026-07-28/spec/`, `schema.ts`)
- the official guides and SEPs next to it (`docs/mcp-spec/2026-07-28/guides/`, `seps/`)
- the Python SDK v2 documentation (summarised in `docs/mcp-python-v2-standards.md`)

Provenance and the list of what has not been read are in `docs/mcp-spec/README.md`. Where this document says MUST, SHOULD or MAY in capitals, it is quoting the spec. Where the spec and the SDK differ, both are stated.

Source commit: `modelcontextprotocol/modelcontextprotocol` @ `3098fe9` (2026-10-01).

## 1. The model in one page

- MCP is JSON-RPC 2.0 between a **host** (the LLM application), **clients** (one per server, inside the host) and **servers** (you).
- **Servers never initiate requests.** The client sends requests and notifications; the server answers with one response, optionally preceded by notifications scoped to that request. If a server needs input from the user, it returns an `InputRequiredResult` and the client retries (section 7).
- **Stateless.** Every request carries what the server needs: protocol version, client capabilities, optional client identity in `_meta`. No handshake, no session, no `Mcp-Session-Id`. The server MUST NOT rely on earlier requests on the same connection. State that must span requests is an explicit handle the model passes back as a tool argument (section 6).
- **Three server features**, each with a different controller: **tools** (the model decides), **resources** (the application decides), **prompts** (the user decides). Servers MUST declare the capability for each feature they offer.
- **Optional extensions** (Tasks, MCP Apps, Skills over MCP, auth extensions) are opt-in on both sides and negotiated through `capabilities.extensions`.
- Authorization (OAuth 2.1) applies to HTTP transports. **stdio servers SHOULD NOT use it** and take credentials from the environment.

## 2. Mandatory protocol behaviour

### 2.1 Messages

- Requests: `id` is a string or integer, never `null`, and unique among in-flight requests. Responses carry the same `id`. Notifications have no `id` and get no reply.
- Every result MUST carry `resultType`: `"complete"` for normal results, `"input_required"` for MRTR. (Clients treat an absent value as `"complete"` for old servers; a new server must still send it. The SDK does this for you.)
- Errors: integer `code`, string `message`, optional `data`.
- JSON-RPC messages MUST be UTF-8.

### 2.2 Per-request `_meta`

Requests carry these under `_meta`:

| Key | Required | Meaning |
|---|---|---|
| `io.modelcontextprotocol/protocolVersion` | Yes | e.g. `"2026-07-28"` |
| `io.modelcontextprotocol/clientCapabilities` | Yes | what the client supports for this request |
| `io.modelcontextprotocol/clientInfo` | No (SHOULD send) | client name and version |
| `io.modelcontextprotocol/logLevel` | No | deprecated logging opt-in |
| `progressToken` | No | opts in to progress notifications |

- A request missing a required field is malformed: reject with `-32602`, and HTTP `400`.
- A server MUST NOT rely on a capability the client did not declare. If a request needs one, return `MissingRequiredClientCapabilityError` (`-32021`) with `data.requiredCapabilities`.
- Servers SHOULD identify themselves in each result's `_meta["io.modelcontextprotocol/serverInfo"]`. `clientInfo` and `serverInfo` are self-reported: use them for display and logs, never for security or behaviour.
- `_meta` key names: optional reverse-DNS prefix plus name. Any prefix whose second label is `modelcontextprotocol` or `mcp` is reserved. Your own keys use your own prefix (for example `io.github.you/...`).

### 2.3 Versioning and discovery

- Servers MUST implement `server/discover`. It returns `supportedVersions`, `capabilities`, optional `instructions` (natural-language guidance for the model), and `serverInfo` in `_meta`. The result is cacheable (`ttlMs`, `cacheScope`).
- If a request names an unsupported version, respond with `UnsupportedProtocolVersionError` (`-32022`), `data: {supported: [...], requested: "..."}`; on HTTP also `400`.
- A server may be **dual-era**: answer `initialize` as a legacy server and per-request `_meta` as a modern one, on the same endpoint. A modern-only server SHOULD name its supported versions in the error it returns to `initialize`. The Python SDK v2 does the dual-era serving for you.

### 2.4 Error codes

- Standard JSON-RPC: `-32700`, `-32600` to `-32603` (`-32601` method not found, `-32602` invalid params, `-32603` internal).
- MCP reserves `-32020` to `-32099` for the specification only. Defined: `-32020` `HeaderMismatch`, `-32021` `MissingRequiredClientCapability`, `-32022` `UnsupportedProtocolVersion`. Servers MUST NOT emit other codes in that range.
- `-32000` to `-32019` is legacy. New servers SHOULD NOT use it.
- Retired and MUST NOT be emitted: `-32002` (resource not found, now `-32602`) and `-32042`.
- Application-defined codes SHOULD sit outside `-32768` to `-32000`.

### 2.5 JSON Schema

- Default dialect is **JSON Schema 2020-12** when `$schema` is absent. Implementations MUST support it and SHOULD document any others.
- Do not auto-dereference a `$ref` that points at a network URI. Bound schema depth and size for composition keywords.
- Tool `inputSchema` MUST be an object schema (`type: "object"`), never `null`. `outputSchema` may be any 2020-12 schema.

## 3. Tools

### 3.1 Declaration and listing

- Declare `capabilities.tools` (with `listChanged: true` only if you will notify).
- `tools/list` MUST return the tools available to the requesting client. The set MUST NOT vary per connection or as a side effect of other requests. It MAY vary by the authorization presented on the request.
- Return tools in a **deterministic order** (SHOULD). This lets clients cache and improves model prompt-cache hits.
- The list result MUST carry `ttlMs` (>= 0) and `cacheScope` (`"public"` or `"private"`).
- `tools/list` supports cursor pagination (section 8).

### 3.2 Tool definition

| Field | Rule |
|---|---|
| `name` | SHOULD be 1 to 128 chars of `A-Z a-z 0-9 _ - .`, case-sensitive, unique within the server |
| `title` | optional human-readable name for UIs |
| `description` | what it does, written for the model |
| `inputSchema` | object schema. No parameters: `{"type":"object","additionalProperties":false}` (recommended) |
| `outputSchema` | optional; if present you MUST return conforming `structuredContent` |
| `annotations` | hints only (3.4) |
| `icons` | optional; clients must treat icon data as untrusted |

Aggregators prefix names to avoid collisions, so a server-name prefix (such as `grocy_`) is good practice.

`x-mcp-header` marks an input property that Streamable HTTP clients mirror into an `Mcp-Param-{Name}` header so gateways can route on it. Only primitives (string, integer, boolean), only statically reachable properties, and never secrets or PII.

### 3.3 Results

- `content`: text, image, audio, `resource_link` or embedded `resource` items. Each may carry annotations (`audience`, `priority`, `lastModified`).
- `structuredContent`: any JSON value. If you return it, SHOULD also return the serialised JSON in a text block for older clients. With an `outputSchema` it MUST conform, and clients SHOULD validate.
- `isError: true` marks a tool execution failure.

### 3.4 Annotations

`readOnlyHint` (default false), `destructiveHint` (default **true**, meaningful only when not read-only), `idempotentHint` (default false, meaningful only when not read-only), `openWorldHint` (default **true**).

- All are **hints**. Clients MUST treat annotations as untrusted unless the server is trusted. Never rely on them for security.
- Because the defaults are `destructive=true` and `openWorld=true`, a tool with no annotations is described as destructive and open-world. Set them deliberately on every tool.

### 3.5 Errors from tools (the most important rule)

Two mechanisms, chosen by whether the model could recover:

| Kind | Use for | Wire form |
|---|---|---|
| **Tool execution error** | API failures, **input validation errors**, business-logic errors | normal result with `isError: true` and an explanatory text block |
| **Protocol error** | unknown tool, malformed request, server fault | JSON-RPC `error` (for example `-32602`) |

- SEP-1303 (Final) moved all argument-validation failures into tool execution errors, so the model sees the reason and corrects itself. Example from the SEP: `"Dates must be in the future. Current date is 08/08/2025."`.
- Clients SHOULD show tool execution errors to the model; they MAY show protocol errors.
- **Never return an error as ordinary text with `isError` unset.** The model reads it as the answer.
- Do not leak internals (stack traces, SQL, upstream bodies) in error text.

### 3.6 Security duties on tools (MUST)

Servers MUST: validate all tool inputs; implement proper access controls; **rate limit tool invocations**; sanitise tool outputs.

Clients SHOULD: confirm sensitive operations, show inputs before calling, validate results, time out calls, and log usage. Hosts SHOULD keep a human in the loop with the ability to deny invocations. These client duties are why accurate annotations matter: they are the server's way of telling the host which calls to confirm.

## 4. Resources and prompts

### 4.1 Resources (application-driven)

- Declare `capabilities.resources` (`listChanged`, `subscribe`). Methods: `resources/list`, `resources/templates/list` (RFC 6570 URI templates), `resources/read`. List and read results MUST carry `ttlMs` and `cacheScope`.
- Not found: error `-32602` with `data: {uri}`. Servers MUST NOT return an empty `contents` array for a missing resource.
- Servers MUST validate all URIs and sanitise file paths (directory traversal) for `file://`.
- Use `https://` only if the client can fetch it directly; prefer a custom scheme otherwise.
- Resource subscriptions go through `subscriptions/listen`, not the removed `resources/subscribe`.

### 4.2 Prompts (user-controlled)

- Declare `capabilities.prompts`. Methods: `prompts/list`, `prompts/get`. Invalid name or missing required arguments: `-32602`. Validate arguments; carefully validate inputs and outputs against injection.

### 4.3 Completion

- `completion/complete` with `ref/prompt` or `ref/resource`. At most 100 values per response, ranked by relevance. Rate limit it; do not disclose sensitive data through suggestions.

## 5. Transports

### 5.1 stdio (local servers; grocy-mcp uses this)

- Newline-delimited JSON-RPC, UTF-8, **no embedded newlines**.
- The server MUST NOT write anything but valid MCP messages to `stdout`. Log to `stderr` only. In Python: no `print()`, use `logging` (stderr by default).
- The server MUST NOT write JSON-RPC requests to stdout.
- Cancel: the client sends `notifications/cancelled`. Shutdown: close stdin; the server SHOULD exit promptly on EOF.
- Credentials come from the environment, not OAuth.
- A crashed server is simply restarted; because the protocol is stateless, in-flight requests are retried.

### 5.2 Streamable HTTP (remote servers)

- One MCP endpoint that accepts **POST**. Every client message is its own POST. The client sends `Accept: application/json, text/event-stream`. The server answers a request with either JSON or an SSE stream scoped to that request; a notification gets `202 Accepted`.
- **Security (MUST):** validate the `Origin` header (403 if present and invalid); when local, bind to `127.0.0.1`; authenticate connections.
- Required headers on every POST: `MCP-Protocol-Version` (must match `_meta`), `Mcp-Method`, and `Mcp-Name` for `tools/call`, `resources/read`, `prompts/get`. A mismatch with the body is `400` plus `-32020`. Non-ASCII values use `=?base64?...?=`.
- Unknown method: `404` plus `-32601`. Unsupported version: `400` plus `-32022`.
- No GET stream, no `Mcp-Session-Id`, no `Last-Event-ID` resumption. Respond `405` to GET and DELETE; ignore session and `Last-Event-ID` headers.
- Closing the SSE stream is cancellation. SSE responses SHOULD set `X-Accel-Buffering: no`.
- HTTP+SSE (2024-11-05) is **deprecated**.

## 6. Designing stateful tools

There is no session. If a workflow needs state (a basket, a transaction), return an **explicit handle** from a creating tool and accept it as an argument afterwards. Handle rules (from the spec's guidance and the security best-practice page):

- A handle is a name, not a credential. Validate the caller's authorization against it on every call; bind it server-side to the authenticated user.
- Make it opaque and unguessable (secure random, for example a UUIDv4) with a bounded lifetime, and say the lifetime in the creating tool's description.
- Unknown or expired handle: return a tool execution error that says so, so the model can start again.

grocy-mcp is stateless today (Grocy holds all state), so this applies only if it ever adds multi-step operations.

## 7. Asking the user for input (MRTR)

When a tool, `resources/read` or `prompts/get` needs information, **return** an `InputRequiredResult` (`resultType: "input_required"`) instead of sending a request:

- `inputRequests`: map of server-chosen keys to an `elicitation/create`, `sampling/createMessage` or `roots/list` request. Only request what the client declared support for.
- `requestState`: opaque string the client echoes back unchanged. At least one of the two fields MUST be present.
- The client retries the **same** request with a **new** JSON-RPC id, plus `inputResponses` and `requestState`. A client must not touch `requestState`.
- **`requestState` is attacker-controlled.** If it influences authorization, access or logic, protect its integrity (HMAC or AEAD) and reject failures. SHOULD include principal, a short TTL and a request fingerprint to limit replay. If something must be single-use, enforce that server-side.
- If the client omits requested information, send another `InputRequiredResult` rather than an error.
- Results of retries carrying `inputResponses` or `requestState` MUST NOT be cached.

### Elicitation specifics

- **Form mode**: flat object schema with primitive properties (string, number/integer, boolean, enums, multi-select enums). **MUST NOT request passwords, API keys, tokens or payment credentials.**
- **URL mode**: for sensitive or third-party flows. The URL MUST NOT contain user data or be pre-authenticated. Verify the user who opens the URL is the same user who started the request (phishing defence). `accept` means consent, not completion.
- Actions: `accept`, `decline`, `cancel`. Handle all three.
- Use elicitation for **confirmations before destructive tools** (delete, clear, merge). In the Python SDK, put it in a `Resolve(...)` parameter so the same tool works for legacy clients too.

## 8. Pagination, caching, notifications

- **Pagination**: opaque cursors on `tools/list`, `resources/list`, `resources/templates/list`, `prompts/list`. The server picks page size; no `limit` in the protocol. Missing `nextCursor` ends the list. An empty-string cursor is valid. An invalid cursor is `-32602`.
- **Caching**: MUST include `ttlMs` and `cacheScope` on complete results of `server/discover`, the four list methods and `resources/read`. `ttlMs` is `max-age` in ms. `"public"` means shareable across users and must not contain user-specific data; `"private"` is per authorization context. Never rely on `cacheScope` for access control.
- **Notifications** are opt-in: the client opens `subscriptions/listen` with a filter (`toolsListChanged`, `promptsListChanged`, `resourcesListChanged`, `resourceSubscriptions`). The server MUST acknowledge first (`notifications/subscriptions/acknowledged`) and tag every message with `io.modelcontextprotocol/subscriptionId`. Request-scoped notifications (`progress`) stay on that request's own stream.
- **Progress**: client sends `progressToken`; `progress` MUST increase; rate limit; stop at completion.
- **Cancellation**: HTTP closes the stream; stdio sends `notifications/cancelled`. Stop work and send nothing further. Implement timeouts and a hard maximum.

## 9. Deprecated and removed

| Item | Status | Do instead |
|---|---|---|
| Roots, Sampling, Logging (`notifications/message`, `logging` capability) | Deprecated (SEP-2577), at least 12 months before removal | pass paths in tool args or config; call LLM APIs directly; log to stderr or use OpenTelemetry |
| HTTP+SSE transport | Deprecated | Streamable HTTP |
| Dynamic Client Registration (RFC 7591) | Deprecated | Client ID Metadata Documents |
| `includeContext` `thisServer` / `allServers` | Deprecated | omit or `none` |
| `initialize` handshake, sessions, `Mcp-Session-Id`, GET stream, SSE resumability | Removed | per-request `_meta`, handles |
| `ping`, `logging/setLevel`, `roots/list_changed`, `resources/subscribe` | Removed | `subscriptions/listen` |
| Server-initiated requests | Removed | MRTR |
| Tasks in core | Moved to extension `io.modelcontextprotocol/tasks` | polling with `tasks/get` and `tasks/update` |

Deprecation policy: minimum 12 months in the Deprecated state before removal.

## 10. Authorization (HTTP servers only)

- Optional. If used: the MCP server is an **OAuth 2.1 resource server**; the client is the OAuth client; a separate authorization server issues tokens.
- Servers MUST implement RFC 9728 Protected Resource Metadata. 401 responses carry `WWW-Authenticate` with `resource_metadata` and SHOULD carry `scope`. Insufficient scope is `403` with `error="insufficient_scope"` and the scopes needed for the whole operation in one challenge.
- Servers MUST validate that a token was issued **for them** (audience, RFC 8707) and MUST NOT accept or pass through any other token. **Token passthrough is forbidden.** If the server calls an upstream API, it uses a separate token it obtained itself.
- Clients MUST use PKCE (`S256`), send the RFC 8707 `resource` parameter, validate `iss` (RFC 9207) and key stored credentials by issuer. Tokens go in the `Authorization` header, never the query string.
- Least privilege: advertise a minimal `scopes_supported`; elevate by challenge. No wildcard scopes.
- Proxy servers with a static upstream client ID MUST get per-client consent (confused deputy).
- Registration preference: Client ID Metadata Documents, pre-registration, then Dynamic Client Registration (deprecated).
- Stateful handles and `requestState` need user binding derived from the verified token, never from client-supplied identity.

## 11. Python SDK v2 mapping

How the spec rules above are met with `mcp` 2.x (details in `docs/mcp-python-v2-standards.md`):

| Spec rule | SDK v2 |
|---|---|
| Tool name, description, schema | `@mcp.tool()`: function name, docstring, type hints; `Annotated[..., Field(...)]` for descriptions and bounds |
| Validate inputs | schema validation before the function runs; failures become tool errors the model can read |
| `isError` for failures | raise `ToolError`; any other exception becomes `Error executing tool <name>` with the traceback logged; `MCPError` is a protocol error |
| Annotations | `@mcp.tool(annotations=...)` with `read_only_hint`, `destructive_hint`, `idempotent_hint`, `open_world_hint` |
| `outputSchema` / `structuredContent` | return a Pydantic model, `TypedDict` or dataclass; the return annotation is the output schema |
| `server/discover`, versions, dual-era | provided; `instructions=` on the server |
| `ttlMs` / `cacheScope` | `cache_hints=` per method |
| MRTR | `Resolve(fn)` parameters; hand-built `InputRequiredResult` for advanced cases |
| `requestState` integrity | SDK seals it; scaled-out deployments pass `RequestStateSecurity(keys=[...])` |
| stdout hygiene | `logging` (stderr); never `print()` |
| Deprecated logging | avoid `ctx.info()` (warns with `MCPDeprecationWarning`) |
| Streamable HTTP, Origin checks | `run()` and the app builders; `transport_security` and host allowlist; mount the ASGI app for prefixes |
| Testing | in-memory `Client(mcp, raise_exceptions=True)`; assert on `is_error` results |

Discrepancy to be aware of: the official Python quickstart (`build-server.mdx`) returns a plain error string when an API call fails. That conflicts with the spec and the SDK's own error-handling page. **Follow the spec and the SDK page: raise `ToolError`.**

## 12. What this means for grocy-mcp

Items below are tracked in `ISSUES.md`.

- **Install breaks on SDK 2.x** (P0-1) and needs porting to `MCPServer` (P1-6).
- **Path-segment injection**: the spec says servers MUST validate all tool inputs (P1-1).
- **Errors are returned as text** rather than raised, so `isError` is never set (P1-4). SEP-1303 and the SDK both say raise.
- **No annotations**: every tool currently reads as destructive and open-world by default (P1-3).
- **No rate limiting**, which the spec lists as a MUST for tool invocations (P2-7).
- **Write tools should confirm** through elicitation (section 7) once ported (P3-10).
- **No server `instructions`** (P3-8) and **no cache hints** on static lists (P3-9).
- **Public demo default** and optional `GROCY_BASE_URL` in `server.json` (P2-3).
- stdio, env-var credentials and stderr-only logging match the spec for a local server.
- If an HTTP transport is ever added: Origin validation, 127.0.0.1 binding, authentication, and the whole of section 10 apply, including the ban on token passthrough to Grocy.

## 13. Build checklist

Protocol
- [ ] Declares only the capabilities it implements (`tools`, optionally `resources`, `prompts`).
- [ ] `server/discover` works and carries `instructions`, `serverInfo`, `ttlMs`, `cacheScope`.
- [ ] Every result has `resultType`; list results have `ttlMs` and `cacheScope`.
- [ ] Only spec-defined codes in `-32020..-32099`; no `-32002`.

Tools
- [ ] Names match `[A-Za-z0-9_.-]{1,128}`, unique, service-prefixed; `tools/list` order is deterministic.
- [ ] Every parameter has a description and bounds; object schemas for all tools.
- [ ] Annotations set deliberately on every tool (writes set `destructive` and `idempotent`).
- [ ] Output types declared; `structuredContent` conforms to `outputSchema`.
- [ ] Failures raise `ToolError` with actionable text; no internals leaked.
- [ ] All inputs validated, including values placed in paths, URLs and queries.
- [ ] Invocations are rate limited; outputs are sanitised.

Interaction and state
- [ ] Destructive operations confirm via elicitation (`Resolve`) or are annotated so hosts can.
- [ ] No sensitive data requested through form-mode elicitation.
- [ ] No reliance on connection state; handles are opaque, bound to the user and expire.

Transport and security
- [ ] stdio: nothing but MCP messages on stdout; logging on stderr.
- [ ] Secrets from the environment, marked secret in `server.json`.
- [ ] HTTP (if any): Origin check, localhost bind, authentication, audience validation, no token passthrough.
- [ ] No deprecated features (logging, roots, sampling) unless needed for legacy clients.

Quality
- [ ] In-memory client tests assert `is_error` results and schemas.
- [ ] `mcp` dependency range is deliberate and tested.
- [ ] Verified with the MCP Inspector (`mcp dev`).
