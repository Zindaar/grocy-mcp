# JSON-RPC 2.0 Specification (recorded)

Source: https://www.jsonrpc.org/specification, pasted by the project owner on 2026-10-02. The site is blocked from this environment, so this is the owner's paste, reformatted as Markdown. Origin date 2010-03-26 (based on the 2009-05-24 version); updated 2013-01-04. Author: JSON-RPC Working Group.

Copyright (C) 2007-2010 by the JSON-RPC Working Group. The document and translations of it may be used to implement JSON-RPC, copied and furnished to others, and derivative works that comment on or explain it may be prepared and distributed, provided the copyright notice and this paragraph are included on all copies. The document itself may not be modified. It is provided "AS IS" with all warranties disclaimed. The limited permissions are perpetual and will not be revoked.

## 1 Overview

JSON-RPC is a stateless, light-weight remote procedure call (RPC) protocol. This specification defines several data structures and the rules around their processing. It is transport agnostic: the concepts can be used in the same process, over sockets, over HTTP, or in many message-passing environments. It uses JSON (RFC 4627) as its data format.

## 2 Conventions

MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY and OPTIONAL are as in RFC 2119.

JSON has four primitive types (String, Number, Boolean, Null) and two structured types (Object, Array). "Primitive" means any of the four; "Structured" means Object or Array. Type names are capitalised in the spec.

Member names exchanged between Client and Server that are considered for matching are **case-sensitive**. The terms function, method and procedure are interchangeable.

The **Client** is the origin of Request objects and the handler of Response objects. The **Server** is the origin of Response objects and the handler of Request objects. One implementation may fill both roles; the spec does not address that.

## 3 Compatibility

2.0 messages may not work with 1.0 peers. They are easy to tell apart: 2.0 always has a member `jsonrpc` with the String value `"2.0"`; 1.0 does not. Most 2.0 implementations should consider handling 1.0 objects.

## 4 Request object

A call is a Request object sent to a Server, with these members:

| Member | Rule |
|---|---|
| `jsonrpc` | String, MUST be exactly `"2.0"` |
| `method` | String naming the method. Names beginning with `rpc.` are reserved for rpc-internal methods and extensions and MUST NOT be used for anything else |
| `params` | Structured value (Array or Object). MAY be omitted |
| `id` | String, Number or Null if included. If absent, the request is a notification. SHOULD normally not be Null; Numbers SHOULD NOT contain fractional parts |

The Server MUST reply with the same `id` value. Notes from the spec: Null is discouraged as an id because a Response with an unknown id uses Null and because JSON-RPC 1.0 used Null for notifications; fractional parts are problematic because many decimal fractions cannot be represented exactly in binary.

### 4.1 Notification

A Request without an `id` member. It signals the Client's lack of interest in a response. The Server **MUST NOT** reply to a notification, including inside a batch. Notifications are not confirmable, so the Client cannot learn of errors such as "Invalid params" or "Internal error".

### 4.2 Parameter structures

If present, `params` MUST be Structured:

- **by-position**: an Array holding the values in the order the Server expects.
- **by-name**: an Object whose member names match the Server's expected parameter names exactly, including case. A missing expected name MAY cause an error.

## 5 Response object

The Server MUST reply to every call except notifications. The Response is a single JSON Object:

| Member | Rule |
|---|---|
| `jsonrpc` | String, MUST be exactly `"2.0"` |
| `result` | REQUIRED on success. MUST NOT exist if there was an error. Value is determined by the method |
| `error` | REQUIRED on error. MUST NOT exist if there was no error. MUST be an Object as in 5.1 |
| `id` | REQUIRED. MUST equal the Request's `id`. If the id could not be detected (parse error, invalid request) it MUST be Null |

Exactly one of `result` or `error` MUST be present, never both.

### 5.1 Error object

| Member | Rule |
|---|---|
| `code` | Number, MUST be an integer |
| `message` | String, short description. SHOULD be a concise single sentence |
| `data` | Primitive or Structured, additional information. MAY be omitted. Defined by the Server |

Codes from -32768 to -32000 are reserved for pre-defined errors. Any code in that range not defined below is reserved for future use.

| Code | Message | Meaning |
|---|---|---|
| -32700 | Parse error | Invalid JSON was received. An error occurred while parsing the JSON text |
| -32600 | Invalid Request | The JSON sent is not a valid Request object |
| -32601 | Method not found | The method does not exist or is not available |
| -32602 | Invalid params | Invalid method parameter(s) |
| -32603 | Internal error | Internal JSON-RPC error |
| -32000 to -32099 | Server error | Reserved for implementation-defined server errors |

The remainder of the space is available for application-defined errors.

## 6 Batch

A Client MAY send an Array of Request objects. The Server should respond with an Array of the corresponding Responses after all have been processed.

- A Response SHOULD exist for each Request, except none for notifications.
- The Server MAY process a batch as concurrent tasks, in any order and with any parallelism.
- Responses MAY come back in any order; the Client SHOULD match them by `id`.
- If the batch itself is not valid JSON, or is not an Array with at least one value, the Server MUST reply with a single Response object.
- If there are no Response objects to send, the Server MUST NOT return an empty Array and should return nothing at all.

## 7 Examples

(`-->` sent to Server, `<--` sent to Client.)

Positional parameters:

```
--> {"jsonrpc": "2.0", "method": "subtract", "params": [42, 23], "id": 1}
<-- {"jsonrpc": "2.0", "result": 19, "id": 1}
```

Named parameters:

```
--> {"jsonrpc": "2.0", "method": "subtract", "params": {"subtrahend": 23, "minuend": 42}, "id": 3}
<-- {"jsonrpc": "2.0", "result": 19, "id": 3}
```

Notification (no response):

```
--> {"jsonrpc": "2.0", "method": "update", "params": [1,2,3,4,5]}
--> {"jsonrpc": "2.0", "method": "foobar"}
```

Non-existent method:

```
--> {"jsonrpc": "2.0", "method": "foobar", "id": "1"}
<-- {"jsonrpc": "2.0", "error": {"code": -32601, "message": "Method not found"}, "id": "1"}
```

Invalid JSON:

```
--> {"jsonrpc": "2.0", "method": "foobar, "params": "bar", "baz]
<-- {"jsonrpc": "2.0", "error": {"code": -32700, "message": "Parse error"}, "id": null}
```

Invalid Request object:

```
--> {"jsonrpc": "2.0", "method": 1, "params": "bar"}
<-- {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null}
```

Batch, invalid JSON, empty Array, invalid batches:

```
--> [ {"jsonrpc": "2.0", "method": "sum", "params": [1,2,4], "id": "1"}, {"jsonrpc": "2.0", "method" ]
<-- {"jsonrpc": "2.0", "error": {"code": -32700, "message": "Parse error"}, "id": null}

--> []
<-- {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null}

--> [1]
<-- [ {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null} ]

--> [1,2,3]
<-- [ three Invalid Request errors, each with "id": null ]
```

Mixed batch:

```
--> [
  {"jsonrpc": "2.0", "method": "sum", "params": [1,2,4], "id": "1"},
  {"jsonrpc": "2.0", "method": "notify_hello", "params": [7]},
  {"jsonrpc": "2.0", "method": "subtract", "params": [42,23], "id": "2"},
  {"foo": "boo"},
  {"jsonrpc": "2.0", "method": "foo.get", "params": {"name": "myself"}, "id": "5"},
  {"jsonrpc": "2.0", "method": "get_data", "id": "9"}
]
<-- [
  {"jsonrpc": "2.0", "result": 7, "id": "1"},
  {"jsonrpc": "2.0", "result": 19, "id": "2"},
  {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null},
  {"jsonrpc": "2.0", "error": {"code": -32601, "message": "Method not found"}, "id": "5"},
  {"jsonrpc": "2.0", "result": ["hello", 5], "id": "9"}
]
```

A batch of only notifications returns nothing.

## 8 Extensions

Method names beginning with `rpc.` are reserved for system extensions and MUST NOT be used for anything else. Each system extension is defined in a related specification. All are OPTIONAL.

---

## How MCP uses and departs from JSON-RPC (not part of the source)

Compiled from the MCP 2026-07-28 spec in `docs/mcp-spec/` and marked as this repository's notes.

| Topic | JSON-RPC 2.0 | MCP 2026-07-28 |
|---|---|---|
| Version member | `"jsonrpc": "2.0"` | same; MCP messages MUST follow JSON-RPC 2.0 |
| Request `id` | String, Number or Null (Null discouraged) | String or integer; **MUST NOT be null**; MUST NOT equal another in-flight request's id |
| Result members | `result` only | `result` MUST include `resultType` (`"complete"` or `"input_required"`) |
| Error codes | -32768 to -32000 reserved; -32000 to -32099 server errors | same standard codes; MCP reserves **-32020 to -32099** for itself (`-32020` HeaderMismatch, `-32021` MissingRequiredClientCapability, `-32022` UnsupportedProtocolVersion); -32000 to -32019 legacy |
| Params | by-position or by-name | MCP uses by-name Objects (with `_meta`) |
| Direction | either side may be Client or Server | the MCP client sends requests; **servers MUST NOT initiate requests** |
| Batching | Array of Requests allowed | the Streamable HTTP transport requires the POST body to be a **single** request or notification; batching is not part of the core patterns |
| Method names | `rpc.` prefix reserved | MCP methods use `/` (`tools/call`, `server/discover`); no `rpc.` methods |
| Notifications | no response | same; the HTTP transport answers a notification POST with `202 Accepted` and no body |
| Unknown method | -32601 | -32601; HTTP `404` |
| Bad params | -32602 | -32602 (also unknown tool, missing resource, bad cursor); HTTP `400` when `_meta` is malformed |

Practical consequence for grocy-mcp: all of this is handled by the Python SDK. The relevant rule for tool code is that failures the model can fix are tool results with `isError: true` (raise `ToolError`), not JSON-RPC errors.
