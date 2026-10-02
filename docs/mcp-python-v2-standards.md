# MCP 2.0 best practices and standards for Python

Condensed from the documentation bundled with the `mcp` 2.2.0 source distribution from PyPI (`docs/`, `VERSIONING.md`, `DEPENDENCY_POLICY.md`), retrieved 2026-10-02. This is a working summary for grocy-mcp, not a copy of the upstream docs. Where it says "the docs say", the page is named so you can check it. Upstream: https://py.sdk.modelcontextprotocol.io/

"MCP 2.0" means two things that shipped together:

- **Python SDK v2** (`mcp` 2.x on PyPI; 2.2.0 is the newest, 2.0.0 is the first stable). Requires Python 3.10+. `pip install mcp` now installs 2.x.
- **Protocol revision 2026-07-28**, which removes the handshake, the session, and every server-initiated request. v2 serves both this revision and 2025-era clients at once, with nothing to configure.

## 1. What breaks for grocy-mcp (verified)

grocy-mcp declares `mcp>=1.2.0` and imports `from mcp.server.fastmcp import FastMCP`. I installed `mcp==2.2.0` in a scratch virtualenv and ran `import grocy_mcp`. It fails:

```
ModuleNotFoundError: No module named 'mcp.server.fastmcp'. This is mcp 2.x, where FastMCP was renamed
to MCPServer (from mcp.server.mcpserver import MCPServer) ... or pin 'mcp<2' to keep running v1 code.
```

So a fresh install of this package today resolves to 2.x and does not start. The fix is either a `<2` upper bound (fast, safe) or porting to `MCPServer` (the real fix). See P0-1 and P1-6 in `ISSUES.md`.

## 2. Versioning and support policy (`VERSIONING.md`)

- Semantic versioning. Breaking changes only in a new major; minors add features and deprecations; patches are fixes only.
- Two supported lines: **2.x** (bug fixes, security fixes, features) and **1.x** (critical and security fixes only).
- APIs marked **provisional** (for example the middleware chain) may change in a minor release. **Experimental** APIs are opt-in previews.
- A library that is not ready to migrate should keep an upper bound, for example `mcp>=1.28,<2`.
- `mcp` and `mcp-types` release in lockstep, each `mcp` requiring the exact matching `mcp-types`.

## 3. Python SDK v2 changes that matter (`whats-new.md`, `migration.md`)

| Area | v1 | v2 |
|---|---|---|
| High-level server | `from mcp.server.fastmcp import FastMCP` | `from mcp.server import MCPServer` |
| Submodules | `mcp.server.fastmcp.*` | `mcp.server.mcpserver.*` |
| Context | `get_context()`, `ctx.fastmcp` | declare a `ctx: Context` parameter; `ctx.mcp_server` |
| Base exception | `FastMCPError` | `MCPServerError` |
| Protocol error | `McpError` | `MCPError(code, message, data)` |
| Wire types | camelCase attributes (`inputSchema`, `isError`) | snake_case attributes (`input_schema`, `is_error`); JSON on the wire is unchanged |
| Transport options | constructor (`host`, `port`, `stateless_http`, `json_response`) | `run()` and the app builders. `MCPServer("x", port=9000)` is a `TypeError` |
| Client | `ClientSession` plus transport context managers | one `Client` class (URL, stdio params, or the server object in memory) |
| HTTP client dependency | `httpx` | `httpx2` (TLS now verified against the OS trust store via `truststore`) |
| Sync tool functions | run on the event loop | run on a worker thread |
| Streamable HTTP lifespan | once per session (once per request when stateless) | once, at startup, shared by all sessions |
| Removed | WebSocket transport, experimental Tasks API, `mount_path`, `get_context()`, `mcp.shared.session` and others | |

Unchanged for decorator-built servers: `@mcp.tool()`, `@mcp.resource()`, `@mcp.prompt()` and type-hint-derived schemas. For a server like grocy-mcp the port is mostly the import, the class name, the error handling and the tests.

Note that grocy-mcp's own use of `httpx` is separate: it is a direct dependency of this package and keeps working. Only the SDK's client transports moved to `httpx2`.

## 4. Tools (`servers/tools.md`)

- The function name is the tool name, the docstring is the description, and the type hints are the input schema. Hints are the contract: bad arguments are rejected before your function runs, with an error the model can read and retry.
- Use `Annotated[..., Field(description=..., ge=1, le=50)]` to describe and bound parameters, `Literal[...]` for enums, and a Pydantic model for structured bodies. Constraints in the schema replace hand-written validation.
- Use `async def` for I/O. A plain `def` also works because the SDK runs it on a thread.
- Override with `@mcp.tool(name=..., title=..., annotations=...)`. `title` is for UIs.
- **Annotations** are behaviour hints: `read_only_hint`, `destructive_hint`, `idempotent_hint`, `open_world_hint`. `destructive_hint` and `idempotent_hint` are only defined for tools that write. The docs are explicit that these are hints, never security; do not rely on a client honouring them.

## 5. Errors (`servers/handling-errors.md`)

Three outcomes, chosen by one question: could a smarter model have avoided this?

| Raise | Model sees | Use when |
|---|---|---|
| `ToolError` (from `mcp.server.mcpserver.exceptions`) | the message, with `is_error=True` | execution failed and the model can recover: a wrong id, an upstream timeout, a missing row |
| `MCPError` (from `mcp`) | nothing; the host gets a JSON-RPC error | the request itself should be rejected: missing capability, server not ready |
| any other exception | only `Error executing tool <name>`; traceback goes to the server log at ERROR | genuine crashes. Internals are deliberately not leaked |

Key rule from the docs: **never return an error message from a tool.** A returned string has `is_error=False`, so the model and every client UI treat it as a successful answer. Raise instead. Resources use `ResourceNotFoundError` (protocol error `-32602` with the URI in `data`) and `ResourceError`.

## 6. Structured output (`servers/structured-output.md`)

- The return type annotation is the output schema. Return a Pydantic model, `TypedDict`, or dataclass and the SDK publishes an `output_schema` and fills both `content` (text for the model) and `structured_content` (typed data for the app).
- Scalars and lists are wrapped as `{"result": ...}`; `dict[str, ...]` is not wrapped.
- `jsonschema` validates structured output against the declared schema before it leaves the server.

## 7. Pagination (`advanced/pagination.md`)

- `MCPServer` answers every `list_*` request (tools, resources, prompts, templates) in one page. That is the right answer for a few dozen items.
- Protocol-level pagination is cursor-based and opaque (`next_cursor`, `None` means last page). It needs the low-level `Server`. The protocol has no `limit`, `total` or `has_more`; the server owns page size.
- This applies to MCP list methods. Data-returning *tools* such as grocy-mcp's list tools can still define their own `limit`/`offset` parameters, which is a tool-design choice, not a protocol feature.

## 8. Running and deploying (`run/`)

- `MCPServer(...)` describes what the server is (name, instructions, lifespan, auth). How it is served belongs to `run()` and the ASGI app builders.
- stdio for local servers; Streamable HTTP for remote. SSE is legacy. WebSocket is removed.
- Streamable HTTP is session-less on the 2026-07-28 path, so any replica behind a round-robin load balancer can answer. 2025-era clients still open sessions and need whatever stickiness they needed before.
- Multi-round-trip retries carry a sealed `request_state` whose default key is minted per process. A scaled-out deployment passes `RequestStateSecurity(keys=[...])`.
- Deploy checklist (see `run/deploy.md`): Host allowlist, the `request_state` key, notifications across replicas (shared `SubscriptionBus`).
- Mount the ASGI app to serve under a prefix (`mount_path` is gone).

## 9. Authorization and security (`run/authorization.md`, `SECURITY.md`)

- Remote servers use OAuth. The client validates the `iss` returned with the authorization code (RFC 9207), sends `application_type` on registration, and never replays credentials against a different authorization server.
- For local stdio servers like grocy-mcp, the practical rules are: keep keys in environment variables, validate inputs, and do not leak internals in errors (v2 now enforces the last one by sanitising unexpected exceptions).
- URI templates are real RFC 6570 now; path traversal in extracted values is rejected by default. grocy-mcp has no resource templates, but its tools interpolate user-controlled values into URL paths and need their own validation (P1-1).

## 10. Interaction with the user (`handlers/`)

- Server-initiated requests (push elicitation, sampling, `roots/list`) do not exist on 2026-07-28 connections. `ctx.elicit()` fails there with `NoBackChannelError`.
- Preferred replacement: annotate a tool parameter with `Resolve(fn)`. The SDK asks the user over whichever mechanism the connection supports, so one tool body serves both eras. Useful for confirmations before destructive tools (delete, clear shopping list, merge).
- Roots, sampling and MCP-level logging (`ctx.info()` and friends) are **deprecated** (SEP-2577). They still work against 2025-era sessions but emit `MCPDeprecationWarning`. `ping` is removed.

## 11. Other protocol additions

- Requests carry `Mcp-Method` / `Mcp-Name` headers so gateways can route without parsing bodies; `x-mcp-header` in an input schema mirrors a parameter into an `Mcp-Param-*` header (SEP-2243).
- Cache hints: list and read results can declare `ttlMs` and `cacheScope` (SEP-2549); set per method with `cache_hints=`. Useful for static lists such as `grocy_common_entities`.
- Extensions are first class (reverse-DNS identifiers, SEP-2133). Change notifications become one `subscriptions/listen` stream.
- Error codes were standardised: `-32020` header mismatch, `-32021` missing required capability, `-32022` unsupported protocol version; a missing resource is `-32602`.
- OpenTelemetry tracing is on by default as middleware and costs nothing until an exporter is configured.

## 12. Testing (`get-started/testing.md`)

- `Client(mcp, raise_exceptions=True)` connects in memory to the server object: no subprocess, no port. Use it as a pytest fixture with `pytest.mark.anyio`.
- Tool failures are always an `is_error=True` result, even with `raise_exceptions=True`. Assert on the result; a crash traceback is in the server log (pytest `caplog`).
- `Client(mcp)` negotiates 2026-07-28 by default, so a tool that calls `ctx.elicit()` fails in a test that passed on v1. Move the question into `Resolve(...)`, or pin `mode="legacy"`.
- The in-memory `Client` replaces v1's `create_connected_server_and_client_session()`.
- Run the MCP Inspector with `uv run mcp dev server.py` (`mcp[cli]` extra). `mcp dev` and `mcp install` now pin the spawned environment to your installed SDK version.

## 13. Checklist for a v2-conformant Python server

- [ ] `pyproject.toml` has a deliberate `mcp` range: `mcp>=2,<3` after porting, or `mcp>=1.28,<2` before.
- [ ] Imports use `from mcp.server import MCPServer`, not `fastmcp`.
- [ ] Every tool has `Field` descriptions and bounds (`ge`, `le`), and a `title`.
- [ ] Every tool has annotations; writes set `destructive_hint` and `idempotent_hint` deliberately.
- [ ] Failures are raised (`ToolError` for model-recoverable ones), never returned as strings.
- [ ] Return types are Pydantic models, `TypedDict`s or dataclasses so `output_schema` is published.
- [ ] I/O is `async` (the SDK's own client is async; blocking calls in `async def` stall the event loop).
- [ ] Transport options are passed to `run()`, not the constructor.
- [ ] Tests use the in-memory `Client`, and assert on `is_error` results.
- [ ] No use of deprecated features (`ctx.info()` logging, roots, sampling) unless legacy clients need them.
- [ ] Secrets come from the environment; user-controlled values are validated before they reach a URL, path or query.

## Sources

Both come from the `mcp-2.2.0.tar.gz` sdist downloaded from PyPI via pip on 2026-10-02:

- `docs/whats-new.md`, `docs/migration.md`, `docs/protocol-versions.md`, `docs/deprecated.md`, `docs/troubleshooting.md`
- `docs/servers/tools.md`, `handling-errors.md`, `structured-output.md`
- `docs/advanced/pagination.md`, `docs/run/*.md`, `docs/get-started/installation.md`, `testing.md`
- `VERSIONING.md`

Not read in full, so not summarised here beyond the headlines above: `migration.md` (2,884 lines), `run/deploy.md`, `run/authorization.md`, `handlers/*`, `client/*`, `advanced/low-level-server.md`, `advanced/middleware.md`. Read those before porting.

The public spec site (modelcontextprotocol.io) is blocked by the session's egress proxy, so the protocol specification itself was not fetched; protocol statements above come from the SDK docs.
