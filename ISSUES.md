# grocy-mcp issue tracker

Working list and record of findings from two static reviews run on 2026-10-02:

- **CR** = `code-review` skill (reviewed `HEAD~1..HEAD`, the userfields/chores/tasks commit)
- **MB** = `mcp-builder` skill, used as an evaluation of the existing server against MCP best practices
- **CR+** / **MB+** = found while reading the code after the skill ran (not in the skill's own output)
- **GA** = API coverage gap analysis against `docs/grocy.openapi.json` (see the coverage section below the P3 list)
- **V2** = MCP 2.0 (Python SDK v2 and protocol 2026-07-28) review, see `docs/mcp-python-v2-standards.md`
- **SPEC** = checked against the MCP specification 2026-07-28 itself, see `docs/mcp-server-standards.md` and `docs/mcp-spec/`

Neither review ran tests, `scripts/live_readonly_test.py` or the MCP Inspector, so every item is unverified by execution.

## Priority key

| Level | Meaning |
|-------|---------|
| P0 | Data loss, security breach or server unusable. Fix immediately. |
| P1 | Real bug or security weakness that can cause wrong behaviour. Fix next. |
| P2 | Best-practice gap that degrades reliability or agent experience. Schedule soon. |
| P3 | Polish, consistency, nice-to-have. |

## Decisions (project owner, 2026-10-02)

- **SDK target: v2 only** (`mcp>=2,<3`). v1 support is not a goal.
- **Transport: stdio only for now.** The HTTP transport and authorization sections of the spec do not apply yet.
- **Registry: publishing is wanted** ("would be nice"). See P2-8.
- **MCP Tasks extension: not needed** (Grocy calls are quick). Grocy's own tasks (to-do items) are covered by COV-P1-2.

Status: `[ ]` open, `[x]` done.

**Summary:** 1 x P0, 6 x P1, 8 x P2, 10 x P3 (25 items), plus the API coverage gaps (19 grouped items covering 67 operations, tracked in their own section).

---

## P0 - Critical

- [ ] **P0-1 A fresh install does not start: `mcp>=1.2.0` resolves to SDK 2.x** (V2, verified)
  - Where: `pyproject.toml` (`dependencies`), `src/grocy_mcp/__init__.py:8`, `src/grocy_mcp/tools.py:8`
  - Problem: Python SDK 2.0 is the stable line (2.2.0 is current) and removed `mcp.server.fastmcp`. I installed `mcp==2.2.0` in a scratch virtualenv and `import grocy_mcp` fails with `ModuleNotFoundError: No module named 'mcp.server.fastmcp'`. Anyone who runs `pipx install` or `pip install` today gets a server that crashes on launch. `tests/test_protocol.py` starts the server as a subprocess, so it fails against a 2.x environment too.
  - Decision: v2 only, so the fix is to port (P1-6) and require `mcp>=2,<3`. If the port will take a while, an interim `mcp>=1.28,<2` cap on a patch release stops new installs breaking in the meantime (the upstream docs recommend exactly this); it is throwaway work if the port lands first.
  - Done when: a clean virtualenv install resolves `mcp` 2.x and the server starts (or, for the interim cap, resolves 1.x and starts).
  - Reference: `docs/mcp-python-v2-standards.md` section 1.

Otherwise no confirmed data loss or exploitable breach. P1-1 would become P0 if the server is ever exposed beyond a trusted local stdio client.

---

## P1 - High

- [ ] **P1-1 Path injection via unvalidated path segments** (CR, extended by CR+)
  - Where: `src/grocy_mcp/tools.py:308` (`grocy_set_userfields`), and the same pattern in `client.py` for `list_objects`, `get_object`, `create_object`, `update_object`, `set_userfields` (`entity`) and `product_by_barcode` (`barcode`).
  - Spec: servers MUST validate all tool inputs (`server/tools`, Security Considerations).
  - Problem: the value is interpolated straight into the URL path. `entity="../chores"` sends the request to an unintended API path. Write tools make this worse.
  - Fix: validate `entity` against an allowlist or `^[a-z_]+$`, and URL-encode `barcode`.
  - Done when: a test shows `../`, `?` and `/` in `entity` or `barcode` are rejected or encoded.

- [ ] **P1-2 `set_userfields` hides the write result and can trigger bad retries** (CR)
  - Where: `src/grocy_mcp/client.py:162-167`
  - Problem: it replaces the PUT response with a follow-up GET and returns only the `userfields` key, or `{}`. A successful write can look like a no-op. If the GET fails, the tool reports an error after the write already succeeded, which invites a retry.
  - Fix: return the PUT result as the source of truth. Make the refresh GET best-effort and never turn a successful write into an error.
  - Related: `update_object` (`client.py:100-106`) has the same PUT-then-GET shape.

- [ ] **P1-3 No tool annotations on any of the 27 tools** (MB)
  - Where: all of `src/grocy_mcp/tools.py`
  - Spec: the defaults are `destructiveHint=true` and `openWorldHint=true`, so an unannotated tool is described as destructive and open-world. Hosts are told to confirm sensitive calls; annotations are how the server says which ones.
  - Problem: no `readOnlyHint`, `destructiveHint`, `idempotentHint` or `openWorldHint`. Reads and writes (`grocy_consume_stock`, `grocy_update_entity_object`) look the same to clients, so they cannot gate or auto-approve sensibly.
  - Fix: add annotations through the FastMCP tool decorator. Reads get `readOnlyHint=True`. Write tools get `destructiveHint` set deliberately.

- [ ] **P1-4 Failures are returned as ordinary text, not flagged as errors** (MB)
  - Where: every tool's `except Exception as exc: return f"Error ..."` in `tools.py`
  - Problem: clients and agents cannot tell a failure from a result, so they may treat an error message as data.
  - Spec: `CallToolResult.isError` is how failures reach the model; SEP-1303 (Final) makes input-validation failures tool execution errors too.
  - Fix: raise `ToolError` (from `mcp.server.mcpserver.exceptions` in v2) for failures the model can recover from, so `is_error=True` is set. The v2 docs state it plainly: never return an error message from a tool, because a returned string has `is_error=False` and reads as a success. Keep the messages actionable. Avoid leaking internals (the client currently includes up to 500 chars of the response body). On v1 the equivalent is raising from the tool (FastMCP v1 also converts exceptions to error results).

- [ ] **P1-6 Port to the MCP Python SDK v2** (V2)
  - Where: `src/grocy_mcp/__init__.py`, `src/grocy_mcp/tools.py`, `tests/test_protocol.py`, `pyproject.toml`
  - Work: `from mcp.server import MCPServer` replaces `FastMCP`; `@mcp.tool()` and the `Annotated`/`Field` signatures carry over unchanged. Move any transport options to `run()` (none are set today). Replace tool error strings with `ToolError` (P1-4). Move tests to the in-memory `Client(mcp, raise_exceptions=True)`. Then change the dependency to `mcp>=2,<3`.
  - Behaviour changes to watch: sync tool functions now run on a worker thread; unexpected exceptions are sanitised to `Error executing tool <name>`; the SDK validates results before they leave.
  - Depends on: P0-1 shipped first so users are protected while the port happens.
  - Reference: `docs/mcp-python-v2-standards.md` sections 3 to 5 and 12; upstream `docs/migration.md` (2,884 lines, not yet read in full).

- [ ] **P1-5 Unrestricted generic write tools** (MB)
  - Where: `grocy_create_entity_object`, `grocy_update_entity_object` (`tools.py:213-236`)
  - Problem: any entity plus a raw JSON string, with no schema or entity allowlist. Combined with P1-1 this is the widest write surface.
  - Fix: restrict to an explicit writable-entity allowlist (the `WritableEntity` type is documentation only today). Consider typed tools for the common entities.

---

## P2 - Medium

- [ ] **P2-1 Blocking I/O inside async tools** (MB+)
  - Where: `client.py:52` (`httpx.request`) called from `async def` tools
  - Problem: each call blocks the event loop, so concurrent requests stall.
  - Fix: use `httpx.AsyncClient`, or `asyncio.to_thread`.

- [ ] **P2-2 Pagination has no metadata** (MB)
  - Where: `_page()` in `tools.py:374`, used by most list tools
  - Problem: there is no `total_count`, `has_more` or `next_offset`, so an agent cannot tell whether more rows exist. Out-of-range `limit` values are silently clamped, not rejected. The full dataset is fetched and then sliced.
  - Fix: return the pagination envelope from the guide, and validate `limit` and `offset` in the schema (`Field(ge=1, le=500)`; the SDK rejects out-of-range values before the function runs and the model can self-correct).
  - Note: MCP protocol-level cursors only apply to list methods like `tools/list`. These are data-returning tools, so `limit`/`offset` plus a metadata envelope is the right design.

- [ ] **P2-3 Defaults to the public demo server** (MB+)
  - Where: `client.py:31`, `__init__.py:19`
  - Problem: if `GROCY_BASE_URL` is unset, writes go to `https://demo.grocy.info`.
  - Also: `server.json` lists `GROCY_BASE_URL` as `isRequired: false` with the demo URL as placeholder, so the registry entry encourages the same default. Change both.
  - Fix: require `GROCY_BASE_URL`, and fail at startup with a clear message.

- [ ] **P2-4 API key is not validated at startup** (MB+)
  - Where: `__init__.py:20`
  - Problem: a missing key only shows up later as an HTTP error on the first call.
  - Fix: warn or fail at startup. The guide asks for clear authentication errors.

- [ ] **P2-5 Inconsistent output formats** (MB)
  - Where: `_format_products` and `_format_stock` return text, everything else returns JSON.
  - Problem: there is no `response_format` option, and a client cannot rely on one shape.
  - Fix: add a `response_format` parameter (`markdown` or `json`) and use it consistently.

- [ ] **P2-6 Test coverage gaps for the new commit and the bugs above** (CR+)
  - Where: `tests/test_client.py`, `tests/test_protocol.py`
  - Problem: no test appears to cover path injection, `set_userfields` behaviour, or failure of the follow-up GET.
  - Fix: add tests alongside P1-1 and P1-2. Confirm what the current suite covers first.

- [ ] **P2-7 No rate limiting on tool invocations** (SPEC)
  - Where: all tools in `src/grocy_mcp/tools.py`
  - Problem: the spec says servers MUST rate limit tool invocations (`server/tools`, Security Considerations). A model in a loop can hammer the Grocy instance or issue many writes.
  - Fix: a small per-process limiter (for example a token bucket) in front of the Grocy client, with write tools limited more tightly. Return a `ToolError` that says how long to wait.

- [ ] **P2-8 Registry metadata does not match this repository** (SPEC)
  - Where: `server.json`, `README.md` (mcp-name comment), `pyproject.toml`
  - Problem: everything is under `io.github.rusty4444/hermes-grocy-mcp` and `rusty4444/grocy-mcp`, but the remote is `Zindaar/grocy-mcp`. Registry GitHub auth requires the name to start with the authenticated account's `io.github.<user>/`, and PyPI ownership verification needs `mcp-name` in the package README. Also `GROCY_BASE_URL` is declared optional with the demo URL as placeholder (see P2-3), and the file's `version` must stay equal to the package version and be unique per publication.
  - Question for the owner: is this a fork to be published under its own name, or are changes going back to `rusty4444`? That decides the namespace, the PyPI package name and the GitHub metadata.
  - Reference: `docs/mcp-server-standards.md` section 15.

---

## P3 - Low

- [ ] **P3-1 No `outputSchema` / structured output** (MB). Add typed results where practical.
- [ ] **P3-2 Server name `grocy-mcp`** (MB). The guide suggests `grocy_mcp` for Python. Changing it is cosmetic but may affect existing client configs, so weigh before changing.
- [ ] **P3-3 `grocy_common_entities` is a static list** (MB+, `tools.py:137`). It can drift from the real writable set. Derive it from the allowlist in P1-5.
- [ ] **P3-4 `db_changed_time` has no tool** (MB+, `client.py:73`). It is dead code today. Either expose it (useful for cache checks) or remove it.
- [ ] **P3-5 `grocy_list_shopping_list_items` fetches all products on every call** (MB+, `tools.py:179`). It only needs names for the page being returned.
- [ ] **P3-6 Over-fetching in list and search tools** (MB+). `grocy_list_products`, `grocy_search_products` and others pull the full table before slicing. Push filtering to Grocy query parameters where supported.
- [ ] **P3-7 Documentation gaps** (MB). The guide asks for at least three working examples per major feature, plus documented security considerations and permissions. Check `README.md` and `skill/SKILL.md` against that.

- [ ] **P3-8 No server `instructions`** (SPEC). `server/discover` can carry natural-language guidance for the model. Useful here: default list id is 1, amounts use the product's stock unit, which tools write.
- [ ] **P3-9 No cache hints** (SPEC). Results of `server/discover` and `tools/list` MUST carry `ttlMs` and `cacheScope`; the Python SDK does this through `cache_hints=` (verify on port). `grocy_common_entities` is static and could advertise a long TTL.
- [ ] **P3-10 Destructive tools do not confirm** (SPEC). After the port, use elicitation through a `Resolve(...)` parameter to confirm deletes, shopping-list clears and merges (see COV-P1-4, COV-P2-4, COV-P2-7). Never request secrets through form mode.

---

## API coverage gap analysis (GA)

Source: `docs/grocy.openapi.json` (grocy/grocy @ `41206cb`) compared against the endpoints `src/grocy_mcp/client.py` calls. Method: every operation in the spec (method + path) was matched against the client's `_request` calls, with path parameters normalised.

**Result:** the spec has 87 operations. The client calls 20 of them (23%); 67 are not covered. One of the 20 (`GET /system/db-changed-time`) has no tool (see P3-4), so 19 are usable by an agent. Every client call matches a real spec endpoint, so there are no calls to endpoints that don't exist.

Partial workaround: the generic entity tools (`grocy_list_entity`, `grocy_get_entity_object`, `grocy_create_entity_object`, `grocy_update_entity_object`) reach raw table rows for entities such as batteries, recipes and tasks. They cannot reach the computed endpoints below (fulfillment, next estimated charge, undo, label printing and so on).

Priority here is judged by how much it completes half-covered workflows or provides a safety net for existing write tools. Items marked (admin) are sensitive and should be added read-only first, if at all.

### Coverage gaps - P1 (10 operations)

- [ ] **COV-P1-1 Read userfields** (1): `GET /userfields/{entity}/{objectId}`. There is a setter tool but no getter.
- [ ] **COV-P1-2 Complete / undo tasks** (2): `POST /tasks/{taskId}/complete`, `POST /tasks/{taskId}/undo`. Tasks can be listed but not finished.
- [ ] **COV-P1-3 Stock transfer and open** (2): `POST /stock/products/{productId}/transfer`, `.../open`. Core inventory operations next to add, consume and inventory.
- [ ] **COV-P1-4 Delete objects** (1): `DELETE /objects/{entity}/{objectId}`. Create and update exist, delete does not. Add it with the entity allowlist from P1-5 and a destructive annotation (P1-3).
- [ ] **COV-P1-5 Booking and transaction undo** (4): `GET /stock/bookings/{bookingId}`, `POST /stock/bookings/{bookingId}/undo`, `GET /stock/transactions/{transactionId}`, `POST /stock/transactions/{transactionId}/undo`. This is the safety net for the existing stock write tools, so it is worth having before more write tools are added.

### Coverage gaps - P2 (26 operations)

- [ ] **COV-P2-1 By-barcode stock operations** (5): `POST /stock/products/by-barcode/{barcode}/` `add`, `consume`, `transfer`, `inventory`, `open`.
- [ ] **COV-P2-2 Batteries** (4): `GET /batteries`, `GET /batteries/{batteryId}`, `POST /batteries/{batteryId}/charge`, `POST /batteries/charge-cycles/{chargeCycleId}/undo`.
- [ ] **COV-P2-3 Recipes** (4): `GET /recipes/fulfillment`, `GET /recipes/{recipeId}/fulfillment`, `POST /recipes/{recipeId}/consume`, `POST /recipes/{recipeId}/add-not-fulfilled-products-to-shoppinglist`.
- [ ] **COV-P2-4 Shopping list bulk actions** (4): `POST /stock/shoppinglist/` `add-missing-products`, `add-overdue-products`, `add-expired-products`, `clear`. `clear` is destructive.
- [ ] **COV-P2-5 Stock entries, locations, price history** (5): `GET` and `PUT /stock/entry/{entryId}`, `GET /stock/products/{productId}/locations`, `GET /stock/products/{productId}/price-history`, `GET /stock/locations/{locationId}/entries`.
- [ ] **COV-P2-6 Chore detail and undo** (2): `GET /chores/{choreId}`, `POST /chores/executions/{executionId}/undo`.
- [ ] **COV-P2-7 Product copy and merge** (2): `POST /stock/products/{productId}/copy`, `POST /stock/products/{productIdToKeep}/merge/{productIdToRemove}`. Merge is destructive.

### Coverage gaps - P3 (31 operations)

- [ ] **COV-P3-1 System endpoints** (4): `GET /system/config`, `GET /system/time`, `GET /system/localization-strings`, `POST /system/log-missing-localization`. `config` and `time` are the useful ones.
- [ ] **COV-P3-2 Users and permissions (admin)** (8): `GET /user`, `GET` and `POST /users`, `PUT` and `DELETE /users/{userId}`, `GET`, `POST` and `PUT /users/{userId}/permissions`. Read-only (`GET /user`, `GET /users`) first, if at all.
- [ ] **COV-P3-3 User settings** (4): `GET /user/settings`, `GET`, `PUT` and `DELETE /user/settings/{settingKey}`.
- [ ] **COV-P3-4 Files** (3): `GET`, `PUT` and `DELETE /files/{group}/{fileName}`. Binary handling needs design for an MCP text channel.
- [ ] **COV-P3-5 Label and thermal printing** (6): `GET .../printlabel` for stock entry, product, recipe, chore and battery, plus `GET /print/shoppinglist/thermal`. Needs a configured printer, so low value for most users.
- [ ] **COV-P3-6 Calendar** (2): `GET /calendar/ical`, `GET /calendar/ical/sharing-link`. The sharing link is a public URL, so treat it as sensitive.
- [ ] **COV-P3-7 Other** (4): `GET /stock/barcodes/external-lookup/{barcode}`, `POST /chores/{choreIdToKeep}/merge/{choreIdToRemove}`, `POST /chores/executions/calculate-next-assignments`, `POST /recipes/{recipeId}/copy`.

Coverage totals: P1 10 + P2 26 + P3 31 = 67 uncovered operations.

---

## Suggested order of work

0. **P0-1 first**: cap `mcp<2` and release a patch, so new installs work.
1. P1-1 and P1-2 together (the two reviewed bugs, plus their tests from P2-6).
2. P1-3 and P1-4 (annotations and error signalling), which are mechanical across all tools. Do them as part of, or just before, the v2 port (P1-6).
3. P1-6 (port to SDK v2), then P1-5 and P3-3 (they share the entity allowlist).
4. COV-P1-1, COV-P1-2, COV-P1-3 (small additions that finish half-covered features), then COV-P1-4 and COV-P1-5 once annotations and the entity allowlist exist.
5. P2-1 to P2-5.
6. COV-P2 items, then P3 and COV-P3 items as convenient.

## Change log

| Date | Note |
|------|------|
| 2026-10-02 | File created from the `code-review` and `mcp-builder` reports. Nothing fixed yet. |
| 2026-10-02 | Added API coverage gap analysis against the Grocy OpenAPI spec (67 of 87 operations uncovered). |
| 2026-10-02 | Added MCP 2.0 review: new P0-1 (fresh install breaks on SDK 2.x) and P1-6 (port to v2); refined P1-4 and P2-2. Standards summary in `docs/mcp-python-v2-standards.md`. |
| 2026-10-02 | Recorded the MCP spec 2026-07-28 overview page (`docs/mcp-spec-2026-07-28-overview.md`). Remaining spec pages still to be provided. |
| 2026-10-02 | Recorded JSON-RPC 2.0, the Tasks extension and registry docs; recorded owner decisions (v2 only, stdio only, registry wanted); added P2-8; reworded P0-1. |
| 2026-10-02 | Cloned the MCP spec repository and recorded the full 2026-07-28 specification (`docs/mcp-spec/`), plus `docs/mcp-server-standards.md`. Added P2-7, P3-8, P3-9, P3-10 and spec notes on P1-1, P1-3, P1-4, P2-3. |
