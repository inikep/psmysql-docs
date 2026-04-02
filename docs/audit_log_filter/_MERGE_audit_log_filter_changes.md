# Audit Log Filter Changes — Percona Server 8.4

This document describes all user-visible and behavioral changes to the
`audit_log_filter` component, organized by commit. Each section explains what
changed from a user / DBA / documentation perspective.

---

## PS-9836: Performance regression with audit_log_filter vs audit_log plugins

**Commit:** `0d5f3c3b561c`

### Summary

Reduces the CPU overhead of the `audit_log_filter` component by optimizing
internal timestamp generation and file-size queries. No configuration changes
are required; the improvement is automatic.

### User-visible impact

- **Throughput increase of ~20–40%** for JSON-format audit logging under
  concurrent workloads, depending on the filter complexity.
- Benchmark results (8 sysbench threads, JSON format):

  | Filter complexity | Before (qps) | After (qps) | Improvement |
  |---|---|---|---|
  | Log everything (`{"filter":{"log":true}}`) | 18 763 | 26 226 | +40% |
  | Two classes (general + query) | 26 810 | 34 538 | +29% |
  | One class (general) | 37 559 | 45 510 | +21% |
  | Single event subclass | 53 889 | 57 117 | +6% |

### Upgrade notes

No action required. The optimization is transparent.

---

## PS-10131: Fix Audit Log file rotation stall

**Commit:** `af58fb97e199`

### Summary

Fixes a bug where requesting a log rotation in **ASYNCHRONOUS** or
**PERFORMANCE** write strategy could stall for up to one second because the
background flush thread was not signaled promptly.

### User-visible impact

- Log rotation requests (`audit_log_filter_rotate()`) now complete
  immediately instead of waiting up to 1 second.
- **Rotated file names** may now include a sequence number suffix when
  multiple rotations happen within the same second. For example:

  ```
  audit_filter.20250401T120000.log      -- first rotation at 12:00:00
  audit_filter.20250401T120000-1.log    -- second rotation at 12:00:00
  ```

  This prevents overwriting of previously rotated files.

### Upgrade notes

- If you have scripts that parse rotated log file names, update them to
  accept the optional `-N` sequence suffix.

---

## PS-9834: Fixed Audit Log file size counter overflow

**Commit:** `aca9a4693c9d`

### Summary

Fixes an integer overflow in the code that calculates the combined size of all
rotated audit log files. The accumulator was declared as `int`, causing an
overflow at 2 GB.

### User-visible impact

- **Before the fix:** once the combined size of rotated log files exceeded
  2 GB, the size calculation overflowed and Percona Server deleted *all*
  rotated log files prematurely.
- **After the fix:** size-based log pruning (`audit_log_filter_max_size`)
  works correctly for log sets of any size.

### Upgrade notes

No configuration changes needed. Users who experienced unexpected deletion of
rotated log files should verify their `audit_log_filter_max_size` setting is
correct.

---

## PS-10324: Add missing fields, update filtering and formatting

**Commit:** `c78154d269fa`

### Summary

Adds fields that were missing from audit event records, making them available
for both **filtering** and **log output**.

### User-visible impact

- Additional fields are now populated in `general` and `table_access` event
  records. These fields can be used in:
  - **Filter conditions** — e.g., filtering on fields that were previously
    absent.
  - **Log output** — more complete records in JSON, XML, and NEW XML formats.

### Upgrade notes

- Existing filters continue to work unchanged.
- If you rely on exact-match comparison of audit log output (e.g., in
  compliance tooling), expect additional fields in the output.

---

## PS-10347: Ensure strict JSON compliance in audit_log_read()

**Commit:** `c0efd304cc37`

### Summary

Fixes the JSON serialization in `audit_log_read()` UDF output to eliminate
trailing commas and ensure the returned JSON is always strictly valid.

### User-visible impact

- **Before the fix:** `audit_log_read()` could return JSON objects with
  trailing commas (e.g., `{"field": "value",}`), which is not valid JSON
  and could break downstream parsers.
- **After the fix:** all JSON returned by `audit_log_read()` is strictly
  compliant.

### Upgrade notes

No action required. This is a correctness fix for the UDF output format.

---

## PS-10387: Fix `audit_log_read()` ignoring `max_array_length` and pagination

**Commit:** `fca9873c2f97`

### Summary

Fixes two bugs in the `audit_log_read()` UDF:

1. The `max_array_length` parameter was ignored — all matching records were
   returned regardless of the limit.
2. Attempting to fetch the next page of results caused an infinite loop or
   parsing error because internal parser state was lost between calls.

### User-visible impact

- `audit_log_read()` now correctly limits the number of returned records to
  the value specified in `max_array_length`.
- Subsequent paginated calls correctly return the next batch of records.
- Malformed audit data no longer causes infinite loops.

### Upgrade notes

- If your application relied on `max_array_length` being silently ignored
  (i.e., returning all records), update the parameter or remove it to get
  all records explicitly.

---

## PS-10229: Change default index name to `filtername`

**Commit:** `65fd975fa617`

### Summary

The foreign key index on `mysql.audit_log_user` may be named `filtername`
(the default when created by system scripts) or `filter_name` (the legacy
explicit name from older install scripts). The component now supports both
names transparently.

### User-visible impact

- No user action required. The component tries `filtername` first, then
  falls back to `filter_name`.
- Fresh installations use the standard `filtername` index name, consistent
  with other MySQL system table conventions.

### Upgrade notes

- Existing installations with the `filter_name` index continue to work
  without manual changes.

---

## PS-10345: Fixed crashes when creating a filter with invalid definition

**Commit:** `ae5aac5b8f1f`

### Summary

Fixes server crashes triggered by calling `audit_log_filter_set_filter()` with
a filter definition containing event data replacement fields with incorrect
data types.

### User-visible impact

- **Before the fix:** supplying a filter definition with wrong field types in
  replacement rules caused a server crash.
- **After the fix:** the UDF returns an error message and the filter is
  rejected. The server remains running.

### Upgrade notes

No action required. This is a crash-safety fix.

---

## PS-9828: Fixed crash for non-existent `audit_log_filter_file` directory

**Commit:** `d72098c6955d`

### Summary

Fixes a server crash during Audit Log Filter initialization when
`audit_log_filter_file` pointed to a file in a directory that does not exist.

### User-visible impact

- **Before the fix:** setting `audit_log_filter_file` to a path with a
  non-existent parent directory caused the server to crash on startup.
- **After the fix:** the component reports an error and the server starts
  without the audit log filter component active.

### Upgrade notes

Ensure the directory specified in `audit_log_filter_file` exists before
starting the server.

---

## PS-10332: Fixed crash caused by incorrect config settings

**Commit:** `e29567f6ad7c`

### Summary

Fixes a server crash caused by passing an invalid configuration setting or
command-line parameter to the Audit Log Filter component. A double-free during
deinitialization was the root cause.

### User-visible impact

- **Before the fix:** invalid audit_log_filter configuration options caused
  a server crash (double-free in `Mysql_thd_store_service_imp::unregister_slot`).
- **After the fix:** the component gracefully reports validation failures
  and the server continues to start.

### Upgrade notes

No action required. This is a crash-safety fix.

---

## PS-10398: Improve stability of `udf_audit_log_read_validate_output` test

**Commit:** `9572962d5558`

### Summary

Internal test improvement. Updates the `udf_audit_log_read_validate_output`
MTR test to use a dedicated user and connection, ensuring deterministic audit
events and stable test results.

### User-visible impact

None. This is a test-only change.

---

## PS-10338: Propagate detailed parse errors to UDF result

**Commit:** `00f461886a55`

### Summary

The `audit_log_filter_set_filter()` UDF now returns the specific parse error
in its result string instead of a generic message.

### User-visible impact

- **Before:**

  ```sql
  SELECT audit_log_filter_set_filter('bad', '...');
  -- ERROR: Incorrect rule definition
  ```

  The detailed error was only written to the server error log.

- **After:**

  ```sql
  SELECT audit_log_filter_set_filter('bad', '...');
  -- ERROR: Incorrect rule definition: Unknown field name "WRONG.str" for class "general"
  ```

  The specific reason is returned directly to the SQL client.

### Upgrade notes

- If you parse UDF result strings programmatically, the error message format
  has changed. The new format is:
  `ERROR: Incorrect rule definition: <specific reason>`

---

## PS-10338: Validate audit log filter field names at parse time

**Commit:** `bd176ecda5a4`

### Summary

Filter definitions with unknown field names (e.g., `"WRONG.str"`) are now
rejected at parse time instead of being silently stored.

### User-visible impact

- **Before:** filters with invalid field names were accepted and stored but
  never matched any events at runtime, silently producing no audit output.
- **After:** `audit_log_filter_set_filter()` rejects such filters with
  `"Incorrect rule definition: Unknown field name ..."`, matching MySQL
  Enterprise Audit behavior.
- Validation covers both field conditions (`"log": {"field": ...}`) and
  function arguments (`{"string": {"field": "..."}}`).

### Upgrade notes

- Existing filters stored with unknown field names will fail to load after
  upgrade. Review and correct any filter definitions that reference
  non-existent field names.

---

## PS-10348: Add typed field values to audit log filter

**Commit:** `4af1da30920f`

### Summary

Integer event fields (e.g., `error_code`, `connection_id`, `connection_type`)
are now stored with their native types instead of being converted to strings.
The filter parser accepts both integer and string values for these fields.

### User-visible impact

- Filter rules can now use integer values directly for numeric fields:

  ```json
  {"filter": {"class": [{"name": "general", "event": {"name": "status"},
    "log": {"field": {"name": "general_error_code.value", "value": 0}}}]}}
  ```

- `connection_type` accepts integer values 0–5 or symbolic constants:
  `::undefined`, `::tcp/ip`, `::socket`, `::named_pipe`, `::ssl`,
  `::shared_memory`.
- Invalid types are rejected: negative values for unsigned fields and integer
  values for string-only fields produce a parse error.

### Upgrade notes

- **Backward compatible.** Existing filters using string values for numeric
  fields continue to work.
- New filters can use native integer values for improved clarity.

---

## PS-10348: Validate class/event names, empty arrays, and unknown keys

**Commit:** `3a1b0f958760`

### Summary

Strengthens filter definition validation to reject definitions that were
previously accepted silently but produced no useful output.

### User-visible impact

- **Invalid class names** are now rejected (e.g., `"name": "nonexistent"`).
- **Invalid event subclass names** within a class are rejected.
- **Empty arrays** (e.g., `"class": []`) are rejected.
- **Unknown JSON keys** in filter definitions are rejected instead of
  silently ignored.

### Upgrade notes

- Existing filters with typos in class/event names or extraneous keys will
  fail to load. Review and fix any such filter definitions.

---

## PS-10312: Audit Log Filter refactoring

**Commit:** `68462d13fe1d`

### Summary

Internal code refactoring for stability. Fixes potential thread-safety issues
in the buffered file writer and simplifies internal initialization logic.

### User-visible impact

- Improved stability of the buffered write path (ASYNCHRONOUS / PERFORMANCE
  strategies). Fixes a potential issue where class members were accessed
  without proper mutex protection during shutdown.
- No configuration or behavioral changes.

### Upgrade notes

No action required.

---

## PS-10872: Fix remaining audit filter parser gaps

**Commit:** `01361aec0f27`

### Summary

Fixes edge cases in the filter parser where class-level print rules were not
validated against all classes in a class-name array, and nested replacement-rule
parse errors were not propagated to the UDF result.

### User-visible impact

- Filter definitions with `print` rules referencing invalid fields for one of
  the classes in a multi-class array are now correctly rejected.
- Parse errors from nested replacement rules are now included in the
  `audit_log_filter_set_filter()` UDF result.

### Upgrade notes

No action required.

---

## PS-10872: Match upstream audit startup/shutdown JSON event format

**Commit:** `4103ebcdf432`

### Summary

Aligns the JSON format of audit startup and shutdown lifecycle events with
the upstream MySQL Enterprise Audit format.

### User-visible impact

- Startup events now include `connection_id`, `account`/`login` data, and a
  `startup_data` object containing `server_id`, OS version, server version,
  and command-line arguments (`argv`).
- Shutdown events include the real `connection_id` when available.
- `audit_log_read()` can now correctly parse and return `startup_data.args`
  arrays.
- Lifecycle events use `startup`/`shutdown` terminology instead of internal
  `audit`/`noaudit` names.

### Upgrade notes

- If you parse audit log JSON lifecycle events, update your parsers to expect
  the new field layout.

---

## PS-10872: Add MTR test for all audit log filter JSON event classes

**Commit:** `a6f295a7ece3`

### Summary

Adds comprehensive test coverage for all audit event classes and subclasses in
JSON format (audit, general, connection, table_access, global_variable,
command, query, stored_program, authentication, message, parse).

### User-visible impact

None. This is a test-only change.

---

## PS-10872: Fix parse event subclass names in audit log filter

**Commit:** `f8e654a5ff4b`

### Summary

Corrects parse event subclass names from the misleading rewrite-based names to
the actual API names (`preparse` and `postparse`).

### User-visible impact

- Parse event records in the audit log now use subclass names `preparse` and
  `postparse` instead of the previous rewrite-based names.
- Filter definitions must use `preparse` / `postparse`; the obsolete
  rewrite-based names are now rejected.

### Upgrade notes

- If you have filters referencing parse event subclass names, update them to
  use `preparse` or `postparse`.

---

## PS-10853: Drain async buffer before direct writes in audit log

**Commit:** `1b2ec46755b7`

### Summary

Fixes a data ordering issue when an audit record exceeds the async write
buffer size and must be written directly to the file.

### User-visible impact

- **Before the fix:** in ASYNCHRONOUS / PERFORMANCE mode, when an oversized
  record bypassed the buffer, previously buffered data (e.g., the XML header)
  could appear *after* the direct write, producing malformed output.
- **After the fix:** all pending buffered data is flushed before any direct
  write, preserving correct record ordering.

### Upgrade notes

No action required.

---

## PS-10853: Ignore audit lifecycle startup and shutdown events

**Commit:** `10c7206ffa6d`

### Summary

Removes the internal handling of `server_startup` and `server_shutdown`
lifecycle events since these event classes are not supported in filter
definitions.

### User-visible impact

- Startup and shutdown lifecycle callbacks from the server are no longer
  emitted as audit records. The component's own internal audit-start and
  audit-stop events continue to be logged.
- Simplifies log output by removing lifecycle records that could not be
  filtered or customized.

### Upgrade notes

- If you relied on `server_startup` / `server_shutdown` records in audit log
  output, note that they are no longer emitted. The component's own
  `audit`-class start/stop events remain.

---

## PS-10853: Harden noexcept functions against throwing std::filesystem APIs

**Commit:** `761181a26a56`

### Summary

Replaces throwing `std::filesystem` calls in `noexcept` functions with
non-throwing overloads and try/catch guards. Previously, a filesystem error
(e.g., permission denied, disk full) in these code paths would call
`std::terminate` and crash the server.

### User-visible impact

- The server no longer crashes due to filesystem errors during audit log
  directory scans, file size checks, or file renaming operations.
- Filesystem errors are now logged as warnings and the affected operation
  gracefully degrades (e.g., skipping an unreadable file).

### Upgrade notes

No action required. This is a robustness improvement.

---

## PS-10853: Return errors for incomplete directory scan in audit log filter

**Commit:** `1ef83ac2d4fa`

### Summary

Directory-scanning functions now report success/failure so callers can
distinguish "no files found" from "scan failed due to an I/O error."

### User-visible impact

- Log rotation and pruning operations correctly abort when a directory scan
  fails, instead of silently operating on incomplete data.
- Prevents incorrect file deletion when the directory listing is incomplete.

### Upgrade notes

No action required.

---

## PS-10853: Guard audit log filter entry points against exceptions

**Commit:** `f453634e54a9`

### Summary

Wraps all component entry points in function-try-blocks so that an unexpected
exception cannot crash the server.

### User-visible impact

- If an internal error occurs during audit event processing, the event is
  counted as "lost" (visible via `audit_log_filter_events_lost` status
  variable) but the server continues running.
- Initialization failures are handled via RAII, preventing resource leaks.

### Upgrade notes

No action required. This is a defense-in-depth improvement.

---

## PS-10436: Fix broken `ER_LOG_PRINTF_MSG` calls in audit log filter

**Commit:** `c19bb460228c`

### Summary

Fixes 71 call sites where `ER_LOG_PRINTF_MSG` was used with format specifiers
(`%s`, `%i`) that were silently ignored, resulting in raw `%s` / `%i` literals
appearing in the server error log.

### User-visible impact

- Server error log messages from the audit log filter component now show
  the intended variable values instead of raw format specifiers.
- Example — before: `"Failed to read %s"` ; after: `"Failed to read audit_log_filter.host"`.

### Upgrade notes

- If you have log monitoring rules that match the old raw `%s` / `%i`
  patterns in audit-log-related error messages, update them to match the
  new properly formatted messages.

---

## PS-10351: Add `audit_log_filter.event_mode` variable (REDUCED / FULL)

**Commit:** `eb852561d3ef`

### Summary

Introduces a new runtime-changeable system variable
`audit_log_filter.event_mode` that controls which event classes and subclasses
are processed.

### User-visible impact

- **`REDUCED`** (default) — limits logging to a curated subset of events:
  - `general/status`
  - `connection/connect`, `connection/disconnect`, `connection/change_user`
  - `table_access/*`
  - `message/*`

  Disabled classes: `global_variable`, `command`, `query`, `stored_program`,
  `authentication`, `parse`. Disabled subclasses: `general/log`,
  `general/error`, `general/result`, `connection/pre_authenticate`.

- **`FULL`** — all event classes and subclasses are processed (previous
  default behavior).

- The variable can be changed at runtime:

  ```sql
  SET GLOBAL audit_log_filter.event_mode = 'FULL';
  ```

- In REDUCED mode, `audit_log_filter_set_filter()` rejects filters that
  reference disabled event classes or subclasses with a descriptive error.

### Upgrade notes

- **The default is now REDUCED.** If your deployment requires logging of
  all event classes (e.g., `query`, `command`, `authentication`), set
  `audit_log_filter.event_mode = FULL` in your configuration.

---

## PS-10351: Skip disabled classes on filter reload instead of rejecting

**Commit:** `15627068ac6c`

### Summary

When loading persisted filters, disabled event classes/subclasses (per the
current `event_mode`) are silently skipped with a warning instead of causing
the entire filter to be rejected.

### User-visible impact

- Filters created under `event_mode=FULL` that reference classes disabled in
  `REDUCED` mode (e.g., `query`, `command`) will still load after a restart
  or `audit_log_filter_flush()` — the disabled classes are simply ignored.
- `audit_log_filter_set_filter()` (the UDF path) remains strict: referencing
  disabled classes in a new filter definition is still an error.
- Changing `event_mode` at runtime triggers an automatic filter reload.

### Upgrade notes

- No risk of losing filters on downgrade from FULL to REDUCED mode during
  normal operations or restarts.

---

## PS-10339: Make `audit_log_filter.format=NEW` follow upstream format

**Commit:** `4fd295c46687`

### Summary

Aligns the `NEW` XML format output with the upstream MySQL Enterprise Audit
log format in terms of both field contents and the amount of logged
information.

### User-visible impact

- Records written in `format=NEW` more closely match the upstream format.
- The `sql_command` attribute now uses the same command name mapping as
  upstream.
- Facilitates migration from upstream MySQL Enterprise Audit to Percona
  Server's audit_log_filter.

### Upgrade notes

- If you have parsers or compliance tools that match on exact NEW-format XML
  output, review the new field names and layout.

---

## PS-10339: Skip general/status Quit command events in REDUCED mode

**Commit:** `7de03969a892`

### Summary

Suppresses the `general/status` event generated by the internal `Quit` command
when `event_mode=REDUCED`, as it carries no useful audit information.

### User-visible impact

- In REDUCED mode, `Quit`-command `general/status` events are no longer
  written to the audit log, reducing log noise.
- In FULL mode, behavior is unchanged.

### Upgrade notes

No action required.

---

## PS-10228: Default empty filter object to log all events

**Commit:** `9222dd1cc47c`

### Summary

Fixes the behavior of an empty JSON filter object `{}` so it logs all events,
consistent with `{"filter": {"log": true}}`.

### User-visible impact

- **Before:** an empty filter `{}` logged nothing because the internal
  "log unmatched" flag was never set.
- **After:** an empty filter `{}` is equivalent to `{"filter": {"log": true}}`
  and logs all events.

### Upgrade notes

- If you intentionally used `{}` as a "log nothing" filter, replace it with
  `{"filter": {"log": false}}`.

---

## PS-10951: Cache per-session filter rule

**Commit:** `925869271a60`

### Summary

Introduces per-session caching of filter rules so that
`audit_log_filter_set_user()` only affects **new** connections, not sessions
already in progress.

### User-visible impact

- **Before:** calling `audit_log_filter_set_user()` immediately changed the
  filter for all active sessions of that user.
- **After:** existing sessions keep their original filter until they reconnect
  or issue `CHANGE_USER`. Only new connections pick up the new user-to-filter
  mapping.
- `audit_log_filter_flush()` and `audit_log_filter_remove_filter()` still
  invalidate all session caches (existing sessions re-resolve their filter on
  the next event).

### Upgrade notes

- This changes the documented behavior of `audit_log_filter_set_user()`.
  If your workflow relied on immediate filter changes for existing sessions,
  use `audit_log_filter_flush()` after `set_user()` to force all sessions to
  re-resolve.

---

## PS-10951: Close CONNECT/CHANGE_USER race with concurrent filter reload

**Commit:** `3c8679831cc6`

### Summary

Fixes a race condition where a `CONNECT` or `CHANGE_USER` event could cache a
stale or partially-reloaded filter rule when a concurrent
`audit_log_filter_flush()` or `audit_log_filter_remove_filter()` was in
progress.

### User-visible impact

- Sessions connecting during a filter reload no longer risk caching an
  inconsistent filter. If the registry is reloading, filter resolution is
  deferred to the next event.

### Upgrade notes

No action required.

---

## PS-10951: Detach current sessions after `audit_log_filter_flush()`

**Commit:** `66d9ddfd4183`

### Summary

Changes the behavior of `audit_log_filter_flush()` so existing sessions stop
logging until they reconnect or execute `CHANGE_USER`.

### User-visible impact

- **Before:** after `audit_log_filter_flush()`, existing sessions silently
  re-resolved a new filter rule on their next event.
- **After:** `audit_log_filter_flush()` detaches filters from existing
  sessions. They stop logging until the session reconnects or executes
  `CHANGE_USER`, at which point the filter is re-resolved from the reloaded
  registry.

### Upgrade notes

- If you rely on continuous audit logging across flush operations, instruct
  applications to reconnect after `audit_log_filter_flush()`.

---

## PS-10951: Detach only sessions whose cached filter was removed

**Commit:** `0e42b309c265`

### Summary

Refines `audit_log_filter_remove_filter()` so that only sessions using the
removed filter are detached; sessions using unrelated filters are unaffected.

### User-visible impact

- **Before:** `audit_log_filter_remove_filter()` invalidated all session
  caches.
- **After:** only sessions whose cached filter matches the removed filter are
  detached. Other sessions continue logging normally.

### Upgrade notes

No action required. This is a precision improvement.

---

## PS-10987: Skip nested general/status events in reduced mode

**Commit:** `e2d2b51911a4`

### Summary

In REDUCED event mode, suppress internal `general/status` events generated by
statements inside stored programs, while preserving the outer `CALL` statement
event.

### User-visible impact

- In REDUCED mode, calling a stored procedure logs the `CALL` statement but
  not the individual SQL statements executed inside the procedure body.
- In FULL mode, behavior is unchanged.

### Upgrade notes

No action required.

---

## PS-10987: Include account metadata in audit message events

**Commit:** `695b457e044a`

### Summary

Audit API message records now carry account and login context (user, host, IP)
like other user-scoped events.

### User-visible impact

- JSON `message` event records now include `account` and `login` fields,
  allowing identification of who emitted each message.
- The message payload container is renamed from `message_attributes` to `map`
  in JSON output.

### Upgrade notes

- If you parse `message` events from JSON audit logs, update your parser to
  expect `map` instead of `message_attributes`, and handle the new `account`
  and `login` fields.

---

## PS-10987: Nest connection attributes under connection data

**Commit:** `baac23938a73`

### Summary

Moves `connection_attributes` inside the `connection_data` object in JSON
connection events for structural consistency.

### User-visible impact

- **Before:**

  ```json
  { "connection_data": {...}, "connection_attributes": {...} }
  ```

- **After:**

  ```json
  { "connection_data": { ..., "connection_attributes": {...} } }
  ```

### Upgrade notes

- Update JSON parsers that access `connection_attributes` at the top level
  of connection events.

---

## PS-10987: Align NEW XML audit records with legacy field order

**Commit:** `4e592ed2d253`

### Summary

Reorders XML fields in all NEW-format record types to match the legacy audit
plugin output order. Enriches message event records with account and connection
metadata.

### User-visible impact

- Field order in NEW-format XML now matches the legacy audit plugin, easing
  migration.
- Message events in XML now include `USER`, `OS_LOGIN`, `HOST`, `IP`,
  `STATUS`, `STATUS_CODE` fields.
- XML message elements are renamed: `MESSAGE_ATTRIBUTES` → `MAP`,
  `ATTRIBUTE` → `ELEMENT`, `NAME` → `KEY`.
- Record IDs are now 1-based.

### Upgrade notes

- If you parse NEW-format XML message events, update field name expectations.

---

## PS-10987: Reduce XML indentation from 2 spaces to 1 space

**Commit:** `34c529b01ad3`

### Summary

Halves the indentation depth for all NEW-format XML audit records.

### User-visible impact

- `AUDIT_RECORD` is indented by 1 space, child elements by 2, nested by 3,
  deepest by 4.
- Slightly reduces audit log file size for XML format.

### Upgrade notes

- If you have parsers or diff tools sensitive to whitespace, update them.

---

## PS-10987: Emit self-closing XML tags for empty field values

**Commit:** `c3e8e5897989`

### Summary

Empty field values in NEW-format XML now use self-closing tags (`<TAG/>`)
instead of empty tag pairs (`<TAG></TAG>`).

### User-visible impact

- Fields like `USER`, `OS_LOGIN`, `HOST`, `IP`, `COMMAND_CLASS`, `PRIV_USER`,
  `PROXY_USER`, and `DB` use `<TAG/>` when the value is empty.
- Slightly reduces audit log file size.

### Upgrade notes

- Update XML parsers if they distinguish between `<TAG/>` and `<TAG></TAG>`.

---

## PS-10989: Add JSONL (JSON Lines) output format

**Commit:** `0cdc996daf3e`

### Summary

Adds a new `JSONL` log format option. Each audit event is written as a single
compact JSON object on one line, conforming to the JSON Lines specification.

### User-visible impact

- New format value for `audit_log_filter.format`:

  ```sql
  SET GLOBAL audit_log_filter.format = 'JSONL';
  ```

- Each line in the audit log file is a self-contained JSON object, making it
  easy to process with line-oriented tools (`grep`, `jq`, `wc -l`), streaming
  pipelines, and log aggregation systems.
- `audit_log_read()` and `audit_log_read_bookmark()` support JSONL files.
- Encryption and compression work with JSONL just as they do with JSON.

### Upgrade notes

- JSONL is a new option; existing configurations are not affected.
- To use JSONL, set `audit_log_filter.format=JSONL` and restart or rotate the
  log.

---

## PS-10331: Drop PERFORMANCE events atomically

**Commit:** `1ab0abd81921`

### Summary

In PERFORMANCE mode, each audit event is now written as a single payload so
that when the buffer is full and an event must be dropped, the entire event is
dropped rather than just parts of it.

### User-visible impact

- **Before:** PERFORMANCE mode could drop just the separator or just the
  record body of an event, producing malformed JSON or XML in the audit log.
- **After:** events are either fully written or fully dropped, keeping the
  log output well-formed.

### Upgrade notes

No action required.

---

## PS-10331: Honor synchronous audit log writes

**Commit:** `ed47c070cef3`

### Summary

The `SYNCHRONOUS` write strategy now actually performs `fsync()` after every
write. Previously it was functionally identical to `SEMISYNCHRONOUS` because
the sync-on-write flag was stored but never read.

### User-visible impact

- **`audit_log_filter.strategy=SYNCHRONOUS`** now guarantees that every audit
  event is flushed to durable storage before the audited statement returns to
  the client.
- Expect higher write latency with SYNCHRONOUS strategy compared to before,
  as actual `fsync()` calls are now made.

### Upgrade notes

- If you are using `strategy=SYNCHRONOUS` and experience increased latency,
  this is now the correct (intended) behavior. Consider switching to
  `SEMISYNCHRONOUS` if per-event durability is not required.

---

## PS-10331: Serialize audit log prune and size reads with writes

**Commit:** `0cee73632ba2`

### Summary

Fixes a race condition where log pruning and size calculations could run
concurrently with writes, potentially causing data corruption or incorrect
pruning decisions.

### User-visible impact

- Log rotation, pruning, and size-based retention decisions are now properly
  serialized with write operations.
- Eliminates potential for incorrect file deletions during concurrent rotation
  and pruning.

### Upgrade notes

No action required.

---

## PS-10331: Bypass page cache for audit log writes via O_DIRECT

**Commit:** `9d3421852ab8`

### Summary

Adds a new read-only system variable `audit_log_filter.direct_io` that opens
the audit log file with `O_DIRECT` on Linux, bypassing the OS page cache.

### User-visible impact

- New system variable:

  ```ini
  [mysqld]
  audit_log_filter.direct_io = ON
  ```

  Default: `OFF`. Read-only (requires restart to change).

- When enabled, audit log writes bypass the Linux page cache via `O_DIRECT`,
  reducing memory pressure on busy servers with high audit log throughput.
- Writes use a 4 KB aligned staging buffer internally.
- If the filesystem does not support `O_DIRECT` or a direct write fails at
  runtime, the component gracefully falls back to buffered I/O with a warning.

### Upgrade notes

- `O_DIRECT` support depends on the filesystem. Verify compatibility
  (ext4, xfs are typically supported; tmpfs is not).
- Requires restart to enable/disable.

---

## PS-10331: Serialize FileHandle access against the async flush worker

**Commit:** `ab0f346c2e4e`

### Summary

Adds a per-FileHandle mutex to serialize file operations between the
foreground write path and the background flush thread in ASYNCHRONOUS /
PERFORMANCE mode.

### User-visible impact

- Eliminates potential data corruption or crashes caused by concurrent file
  access between the main thread and the background flush worker during
  rotation or shutdown.

### Upgrade notes

No action required. This is a correctness and stability fix.
