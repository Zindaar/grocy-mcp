# grocy-mcp issue tracker

Working list and record of findings from two static reviews run on 2026-10-02:

- **CR** = `code-review` skill (reviewed `HEAD~1..HEAD`, the userfields/chores/tasks commit)
- **MB** = `mcp-builder` skill, used as an evaluation of the existing server against MCP best practices
- **CR+** / **MB+** = found while reading the code after the skill ran (not in the skill's own output)
- **GA** = API coverage gap analysis against `docs/grocy.openapi.json` (see the coverage section below the P3 list)

Neither review ran tests, `scripts/live_readonly_test.py` or the MCP Inspector, so every item is unverified by execution.

## Priority key

| Level | Meaning |
|-------|---------|
| P0 | Data loss, security breach or server unusable. Fix immediately. |
| P1 | Real bug or security weakness that can cause wrong behaviour. Fix next. |
| P2 | Best-practice gap that degrades reliability or agent experience. Schedule soon. |
| P3 | Polish, consistency, nice-to-have. |

Status: `[ ]` open, `[x]` done.

**Summary:** 0 x P0, 5 x P1, 6 x P2, 7 x P3 (18 items), plus the API coverage gaps (19 grouped items covering 67 operations, tracked in their own section).

---

## P0 - Critical

None identified. No confirmed data loss or exploitable breach. P1-1 would become P0 if the server is ever exposed beyond a trusted local stdio client.

---

## P1 - High

- [ ] **P1-1 Path injection via unvalidated path segments** (CR, extended by CR+)
  - Where: `src/grocy_mcp/tools.py:308` (`grocy_set_userfields`), and the same pattern in `client.py` for `list_objects`, `get_object`, `create_object`, `update_object`, `set_userfields` (`entity`) and `product_by_barcode` (`barcode`).
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
  - Problem: no `readOnlyHint`, `destructiveHint`, `idempotentHint` or `openWorldHint`. Reads and writes (`grocy_consume_stock`, `grocy_update_entity_object`) look the same to clients, so they cannot gate or auto-approve sensibly.
  - Fix: add annotations through the FastMCP tool decorator. Reads get `readOnlyHint=True`. Write tools get `destructiveHint` set deliberately.

- [ ] **P1-4 Failures are returned as ordinary text, not flagged as errors** (MB)
  - Where: every tool's `except Exception as exc: return f"Error ..."` in `tools.py`
  - Problem: clients and agents cannot tell a failure from a result, so they may treat an error message as data.
  - Fix: raise from the tool, or return an error result so `isError` is set. Keep the messages actionable. Avoid leaking internals (the client currently includes up to 500 chars of the response body).

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
  - Fix: return the pagination envelope from the guide, and validate `limit` and `offset` in the schema (`ge=` / `le=`).

- [ ] **P2-3 Defaults to the public demo server** (MB+)
  - Where: `client.py:31`, `__init__.py:19`
  - Problem: if `GROCY_BASE_URL` is unset, writes go to `https://demo.grocy.info`.
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

---

## P3 - Low

- [ ] **P3-1 No `outputSchema` / structured output** (MB). Add typed results where practical.
- [ ] **P3-2 Server name `grocy-mcp`** (MB). The guide suggests `grocy_mcp` for Python. Changing it is cosmetic but may affect existing client configs, so weigh before changing.
- [ ] **P3-3 `grocy_common_entities` is a static list** (MB+, `tools.py:137`). It can drift from the real writable set. Derive it from the allowlist in P1-5.
- [ ] **P3-4 `db_changed_time` has no tool** (MB+, `client.py:73`). It is dead code today. Either expose it (useful for cache checks) or remove it.
- [ ] **P3-5 `grocy_list_shopping_list_items` fetches all products on every call** (MB+, `tools.py:179`). It only needs names for the page being returned.
- [ ] **P3-6 Over-fetching in list and search tools** (MB+). `grocy_list_products`, `grocy_search_products` and others pull the full table before slicing. Push filtering to Grocy query parameters where supported.
- [ ] **P3-7 Documentation gaps** (MB). The guide asks for at least three working examples per major feature, plus documented security considerations and permissions. Check `README.md` and `skill/SKILL.md` against that.

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

1. P1-1 and P1-2 together (the two reviewed bugs, plus their tests from P2-6).
2. P1-3 and P1-4 (annotations and error signalling), which are mechanical across all tools.
3. P1-5, then P3-3 (they share the entity allowlist).
4. COV-P1-1, COV-P1-2, COV-P1-3 (small additions that finish half-covered features), then COV-P1-4 and COV-P1-5 once annotations and the entity allowlist exist.
5. P2-1 to P2-5.
6. COV-P2 items, then P3 and COV-P3 items as convenient.

## Change log

| Date | Note |
|------|------|
| 2026-10-02 | File created from the `code-review` and `mcp-builder` reports. Nothing fixed yet. |
| 2026-10-02 | Added API coverage gap analysis against the Grocy OpenAPI spec (67 of 87 operations uncovered). |
