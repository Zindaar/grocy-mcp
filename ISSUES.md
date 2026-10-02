# grocy-mcp issue tracker

Working list and record of findings from two static reviews run on 2026-10-02:

- **CR** = `code-review` skill (reviewed `HEAD~1..HEAD`, the userfields/chores/tasks commit)
- **MB** = `mcp-builder` skill, used as an evaluation of the existing server against MCP best practices
- **CR+** / **MB+** = found while reading the code after the skill ran (not in the skill's own output)

Neither review ran tests, `scripts/live_readonly_test.py` or the MCP Inspector, so every item is unverified by execution.

## Priority key

| Level | Meaning |
|-------|---------|
| P0 | Data loss, security breach or server unusable. Fix immediately. |
| P1 | Real bug or security weakness that can cause wrong behaviour. Fix next. |
| P2 | Best-practice gap that degrades reliability or agent experience. Schedule soon. |
| P3 | Polish, consistency, nice-to-have. |

Status: `[ ]` open, `[x]` done.

**Summary:** 0 x P0, 5 x P1, 6 x P2, 7 x P3 (18 items)

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

## Suggested order of work

1. P1-1 and P1-2 together (the two reviewed bugs, plus their tests from P2-6).
2. P1-3 and P1-4 (annotations and error signalling), which are mechanical across all tools.
3. P1-5, then P3-3 (they share the entity allowlist).
4. P2-1 to P2-5.
5. P3 items as convenient.

## Change log

| Date | Note |
|------|------|
| 2026-10-02 | File created from the `code-review` and `mcp-builder` reports. Nothing fixed yet. |
