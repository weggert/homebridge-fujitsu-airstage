## Why

Users with multiple Fujitsu Airstage LAN devices experience slow startup (N×30s worst case), intermittent unresponsiveness due to transient WiFi failures, and opaque error messages that make debugging impossible. The current LAN networking layer makes three sequential HTTP round-trips per device read, has no retry logic, no request deduplication, and uses a callback pattern that makes every improvement harder to implement correctly.

## What Changes

- **Eliminate redundant round-trips**: Replace the two sequential `postGetParam` calls (one for all params except model, one for model) with a single `postGetParam` call that requests all parameters at once. Investigate whether the `iu_model` truncation issue still exists on current firmware and find a better workaround if needed.
- **Add automatic retry**: Retry failed LAN requests once after a 500ms delay before surfacing the error. Covers both connection timeouts and transient HTTP errors.
- **Deduplicate in-flight requests**: When multiple HomeKit characteristic reads target the same device while a request is already in-flight, queue the new requests behind the existing one instead of spawning duplicate network calls.
- **Enrich error messages**: Include hostname, device ID, and operation context (get/set, parameter name) in all error strings so users and logs can identify exactly which device and operation failed.
- **Wrap callbacks in Promises**: Add a Promise-based wrapper around the callback API so internal code can use `async/await` and `Promise.all()`. The external callback API remains unchanged for backward compatibility.

## Capabilities

### New Capabilities
- `lan-retry`: Automatic retry logic for transient LAN request failures
- `lan-request-dedup`: In-flight request deduplication to prevent duplicate network calls to the same device
- `lan-error-context`: Enriched error messages with device and operation context
- `lan-promise-api`: Promise-based wrapper around the LAN client callback API

## Impact

- `src/airstage/lan/api/client.js` — HTTP client layer: retry logic, error enrichment
- `src/airstage/lan/client.js` — 1024-line client: round-trip optimization, dedup, Promise wrapper
- `src/airstage/constants.js` — parameter lists may need adjustment
- Existing callback-based callers (platform.js, accessories) unaffected — the Promise wrapper is internal
- No new dependencies required (uses only Node.js builtins)
