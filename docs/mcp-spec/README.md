# MCP specification record

Verbatim copies of primary sources, kept so work on grocy-mcp can be checked against them. The distilled, project-specific standard is `docs/mcp-server-standards.md`.

## Source

- Repository: https://github.com/modelcontextprotocol/modelcontextprotocol (shallow clone, public)
- Commit: `3098fe94caa1b9e0afaaa6d30e040b61d5802471` (2026-10-01)
- Retrieved: 2026-10-02
- Why not the website: `modelcontextprotocol.io` is blocked by the session's egress proxy. The `docs/` folder of that repository is the source of the website pages, so the text is the same; rendering details (cards, tabs, images) are not.

## Contents

| Path | What it is |
|---|---|
| `2026-07-28/spec/` | The full 2026-07-28 specification as `.mdx` (architecture, base protocol, versioning, transports, patterns, authorization, server and client features, utilities, changelog, deprecated registry). Images and the generated `schema.mdx` page are omitted. |
| `2026-07-28/schema.ts` | The authoritative TypeScript schema. (`schema.json`, generated from it, is not copied.) |
| `2026-07-28/guides/` | Security best practices, server concepts, design principles, extensions overview, Tasks extension overview |
| `2026-07-28/seps/` | SEPs behind the headline changes: 1303 (validation errors are tool errors), 2243 (HTTP headers), 2322 (MRTR), 2549 (TTL), 2567 (sessionless), 2575 (stateless), 2577 (deprecate roots, sampling, logging) |

Related files elsewhere in `docs/`:

- `mcp-spec-2026-07-28-overview.md`: the overview page as pasted by the project owner.
- `mcp-python-v2-standards.md`: Python SDK v2 summary (from the SDK's own docs).
- `grocy.openapi.json`: the Grocy API.

## Read in full

Spec: `index`, `architecture`, `changelog`, `basic/index`, `basic/versioning`, `basic/transports/*` (index, stdio, streamable-http), `basic/patterns/*` (index, mrtr, cancellation, progress, subscriptions), `server/discover`, `server/tools`, `server/resources`, `server/prompts`, `server/utilities/*` (caching, pagination, completion, logging), `client/elicitation`, `basic/authorization/index`, `basic/authorization/security-considerations`.

Other: security best practices, server concepts, design principles, extensions overview, Tasks overview, the Python build-server quickstart, SEP-1303, the Tool Annotations interest-group charter, and the parts of `schema.ts` covering capabilities, tools and results.

## Copied but not yet read closely

`client/roots`, `client/sampling` (both deprecated), `basic/authorization/client-registration`, `basic/authorization/authorization-server-discovery`, `schema.ts` outside the sections above, and the SEPs other than 1303.

## Added later

| Path | Source |
|---|---|
| `extensions/tasks-2026-07-28.md` | `modelcontextprotocol/ext-tasks` @ `5246bc3`, `specification/2026-07-28/tasks.md`, read in full |
| `registry/` | `docs/registry/` of the spec repository: quickstart, package types, versioning, authentication (read in full) |
| `../jsonrpc-2.0-specification.md` | JSON-RPC 2.0, pasted by the project owner |

## Not available here

- `ext-apps`, `ext-skills`, `ext-auth` extension specifications (separate public repositories; not read, not relevant to a stdio Grocy server).
- Registry pages not copied: about, FAQ, GitHub Actions, moderation policy, remote servers, aggregators, terms of service.
