# MCP specification 2026-07-28: overview (recorded)

Source: the "Specification" landing page of modelcontextprotocol.io, pasted by the project owner on 2026-10-02. Only this one page has been provided so far. It is an overview; the normative detail is in the pages it links to (see "Not yet recorded" at the end).

Authoritative schema, per the page: `schema/2026-07-28/schema.ts` in the `modelcontextprotocol/specification` repository.

## Requirement language

"MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY" and "OPTIONAL" are interpreted as in BCP 14 (RFC 2119, RFC 8174) only when written in capitals. Lowercase uses are ordinary English and not requirements. This matters when reading the security principles below: the page itself mixes "must" (lowercase, principle) and "SHOULD" (capitals, requirement).

## What MCP is

An open protocol for integrating LLM applications with external data sources and tools. It uses JSON-RPC 2.0 messages between three roles:

| Role | Meaning |
|---|---|
| **Host** | The LLM application that initiates connections |
| **Client** | A connector inside the host application |
| **Server** | A service that provides context and capabilities |

It is modelled on the Language Server Protocol: one standard so an ecosystem of tools and applications can interoperate.

## Base protocol (2026-07-28)

- JSON-RPC message format.
- **Stateless, self-contained requests.**
- **Per-request capability negotiation.**

These three lines are the headline change from earlier revisions, where a connection was opened with an `initialize` handshake and a session. This agrees with what the Python SDK v2 docs say (every request carries its protocol version and client capabilities in `_meta`; no session). See `docs/mcp-python-v2-standards.md`.

## Features

Servers can offer:

| Feature | Purpose | Controlled by |
|---|---|---|
| **Resources** | Context and data, for the user or the AI model | application or user |
| **Prompts** | Templated messages and workflows | the user |
| **Tools** | Functions for the AI model to execute | the model |

(The "controlled by" column is standard MCP terminology and is not stated on this page; it will be confirmed from the Server Features pages.)

Clients can offer:

- **Elicitation**: server-initiated requests for additional information from users.

The page lists **no other client features**. Roots and sampling are absent here, which is consistent with the SDK docs, where they are deprecated (SEP-2577). The SDK docs add that, on 2026-07-28 connections, the server cannot call the client directly and elicitation travels as a multi-round-trip result instead. That mechanism is not described on this page.

## Additional utilities

Configuration, progress tracking, cancellation, error reporting. (`ping` and MCP-level logging are not listed; the SDK docs say `ping` is removed and logging is deprecated.)

## Extensions

Optional, modular functionality beyond the core protocol. Always **opt-in**, requiring explicit support from both client and server, negotiated during initialization.

| Extension | Purpose |
|---|---|
| **Tasks** | Asynchronous execution of long-running operations, with polling, mid-flight input and durable handles |
| **Skills over MCP** | Structured instructions for agent workflows, discovered and consumed through MCP |
| **MCP Apps** | Interactive UI (charts, forms, video players) rendered inline in conversations |

Tasks is an extension, not core. This agrees with the SDK docs, which say the experimental Tasks API was removed from the SDK because 2026-07-28 moves tasks into an official extension (SEP-2663) that the SDK does not implement yet.

## Security and trust and safety

The page states these principles. They shape server design even though the protocol cannot enforce them.

1. **User consent and control**
   - Users must explicitly consent to and understand all data access and operations.
   - Users must retain control over what data is shared and what actions are taken.
   - Implementors should provide clear UIs for reviewing and authorizing activities.
2. **Data privacy**
   - Hosts must obtain explicit user consent before exposing user data to servers.
   - Hosts must not transmit resource data elsewhere without user consent.
   - User data should be protected with appropriate access controls.
3. **Tool safety**
   - Tools represent arbitrary code execution and must be treated with appropriate caution.
   - **Descriptions of tool behaviour, such as annotations, are untrusted unless obtained from a trusted server.**
   - Hosts must obtain explicit user consent before invoking any tool.
   - Users should understand what each tool does before authorizing its use.

Implementors **SHOULD**: build robust consent and authorization flows; document security implications clearly; implement access controls and data protections; follow security best practice; consider privacy implications in feature design.

### What this means for grocy-mcp

- Annotations (`readOnlyHint` and the others) are advisory and, by the spec's own wording, untrusted. Adding them (P1-3) helps hosts present and gate tools, but they are not a substitute for server-side protection. The server must still enforce its own limits: an entity allowlist (P1-5), path-segment validation (P1-1), and so on.
- Tools are described as arbitrary code execution that needs user consent. Write tools that mutate a household database (stock, shopping list, deletes, merges) are the ones a host should confirm. Accurate `destructive_hint` and `idempotent_hint` values are how the server tells the host which ones those are.
- Do not return more user data than the tool needs (relevant to P3-5 over-fetching, and to the public iCal sharing link in COV-P3-6).

## Not yet recorded

The page links to these. None has been provided, so nothing here should be treated as known from them. The questions in the project owner's reply list which are most needed.

- Architecture (`/specification/2026-07-28/architecture`)
- Base Protocol (`/specification/2026-07-28/basic`), including lifecycle and discovery, transports, authorization, cancellation, progress
- Server Features (`/specification/2026-07-28/server`): tools, resources, prompts, completion, pagination
- Client Features (`/specification/2026-07-28/client`): elicitation
- Extensions overview, Tasks, Skills over MCP, MCP Apps
- `schema.ts` for 2026-07-28
- The documentation index at `/llms.txt`
