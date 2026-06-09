## Context

The LAN client (`src/airstage/lan/client.js`, 1024 lines) communicates with Fujitsu Airstage HVAC devices over HTTP on the local network. The API client (`src/airstage/lan/api/client.js`, 117 lines) is a thin wrapper around Node.js `http.request()`. The platform (`src/platform.js`) polls these devices on an interval and exposes them as HomeKit accessories.

Current pain points:
- Each device read requires 2 sequential HTTP round-trips (30s timeout each)
- Transient WiFi failures cause permanent errors with no retry
- Concurrent HomeKit reads spawn duplicate network requests
- Error messages like `"Parameter not available: iu_model"` don't identify which device failed
- Every method uses the callback pattern, making composition verbose and error-prone

## Goals / Non-Goals

**Goals:**
- Eliminate the two-round-trip pattern for device parameter fetches
- Add one automatic retry for transient network failures
- Deduplicate in-flight requests for the same device
- Enrich all error messages with hostname, device ID, and operation context
- Introduce a Promise-based API layer internally to enable `async/await` and `Promise.all()`

**Non-Goals:**
- Connection pooling / HTTP keep-alive (deferred to a follow-up change)
- Reaching the 50/60/90 second timeout debate (still 30s, unchanged)
- Push/websocket support for device state changes (device doesn't support it)
- Cloud client changes (out of scope)
- Changing the external callback API (backward compatibility required)

## Decisions

### Decision 1: Single round-trip for all parameters, with conditional fallback

**Current state:** Two sequential `postGetParam` calls — first for all params except `iu_model`, then for `iu_model` alone. The comment says the response gets truncated when `iu_model` is included.

**Decision:** Attempt a single `postGetParam` call with all 12 parameters. If the response indicates truncation or missing `iu_model`, fall back to the two-call pattern.

**Rationale:** 
- The truncation bug may be firmware-dependent. Newer devices may not have it.
- A single call path is simpler to reason about and test.
- The fallback preserves compatibility with older firmware.
- We can measure whether the fallback is ever triggered in production.

```
┌──────────────────────────────────────────────┐
│           getDevice(deviceId)                │
├──────────────────────────────────────────────┤
│                                              │
│  postGetParam(all 12 params)                 │
│      │                                       │
│      ├─ Success + iu_model present ──▶ done  │
│      │                                       │
│      └─ Missing iu_model ──▶ Retry with      │
│                            single-param call │
│                                              │
└──────────────────────────────────────────────┘
```

### Decision 2: Retry at the API client layer

**Decision:** Implement retry in `lan/api/client.js` (`_makeHttpRequest`), not in `lan/client.js`.

**Rationale:**
- The API client is the only place that knows what a "transient" error looks like (timeout, ECONNRESET, ENOTFOUND).
- `lan/client.js` doesn't need to know about retry logic — it just gets success or failure.
- One place to change if retry behavior needs tuning.

```
┌──────────────────────────────────────────┐
│       _makeHttpRequest()                 │
├──────────────────────────────────────────┤
│                                          │
│  attempt = 0                             │
│  maxAttempts = 2                         │
│  while attempt < maxAttempts:            │
│    try request                           │
│    if success → return result            │
│    if transient error (timeout,          │
│       ECONNRESET, ENOTFOUND):            │
│      attempt++                           │
│      if attempt < maxAttempts:           │
│        sleep(500ms)                      │
│        continue                          │
│      else:                               │
│        return error                      │
│    if non-transient (parse error,        │
│       401, 403):                         │
│      return error (no retry)             │
│                                          │
└──────────────────────────────────────────┘
```

### Decision 3: Deduplication map keyed by device ID

**Decision:** Maintain a `Map<deviceId, { promise, subscribers[] }>` of in-flight requests.

**Rationale:**
- A Map gives O(1) lookup by device ID.
- When a new request arrives for a device with an in-flight request, the new caller adds itself as a subscriber to the existing Promise.
- When the in-flight request completes (success or failure), all subscribers are notified.
- Failed requests are removed from the map immediately (subscribers get the error, next call starts fresh).

```
┌─────────────────────────────────────────────────┐
│  inFlightRequests: Map<deviceId, InFlight>      │
├─────────────────────────────────────────────────┤
│                                                 │
│  getDevice(deviceId):                           │
│    if deviceId in inFlightRequests:             │
│      return inFlightRequests.get(deviceId)      │
│    else:                                        │
│      promise = fetchDevice(deviceId)            │
│      inFlightRequests.set(deviceId, promise)    │
│      promise.finally(() => inFlightRequests.     │
│        delete(deviceId))                         │
│      return promise                             │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Decision 4: Error enrichment via a helper function

**Decision:** Create a `makeError(hostname, deviceId, operation, param, baseError)` helper that constructs enriched error strings.

**Rationale:**
- Centralizes error message formatting in one place.
- Each caller passes the context they have (hostname from the device map, deviceId, operation type, parameter name).
- The base error (from the API client) is embedded in the enriched string.

```
makeError('192.168.1.100', '39FFBB9D6EBA', 'set', 'iu_onoff', 'Request timeout')
→ "Timeout setting iu_onoff on device 39FFBB9D6EBA at 192.168.1.100"

makeError('192.168.1.100', '39FFBB9D6EBA', 'get', null, 'ECONNREFUSED')
→ "Connection refused fetching parameters for device 39FFBB9D6EBA at 192.168.1.100"
```

### Decision 5: Promise wrapper that preserves callbacks

**Decision:** Create an internal `_promisify(method)` helper that wraps a callback-based method. The wrapper:
- If a callback is provided: invokes the original method with the callback (existing behavior).
- If no callback: returns a Promise that resolves/rejects with the same values.

**Rationale:**
- Zero changes to existing callers — they all pass callbacks.
- Internal code can use `await client.getDevice(deviceId)` (no callback).
- The wrapper adds ~5 lines per method but eliminates callback nesting throughout the codebase.

```javascript
// Internal helper
function promisify(fn) {
  return function(...args) {
    const lastArg = args[args.length - 1];
    if (typeof lastArg === 'function') {
      // Callback provided — use original behavior
      return fn.apply(this, args);
    }
    // No callback — return Promise
    return new Promise((resolve, reject) => {
      fn.apply(this, [...args, (err, result) => {
        if (err) reject(err);
        else resolve(result);
      }]);
    });
  };
}
```

## Risks / Trade-offs

| Risk | Mitigation |
|------|-----------|
| Single round-trip fallback to two-call pattern adds complexity | The fallback is only triggered if `iu_model` is missing from the response — easy to detect and test |
| Retry adds 500ms latency on every failure, even non-transient ones | Retry only triggers on specific transient errors (timeout, ECONNRESET, ENOTFOUND). Parse errors and 4xx codes are not retried. |
| Deduplication map could leak if a Promise never settles | `Promise.finally()` removes entries after settlement. For truly hung requests, the 30s timeout ensures cleanup. |
| Promise wrapper adds a thin abstraction layer | The wrapper is ~10 lines total and is only used internally. External API is unchanged. |
| Enriched error messages change log output format | This is a net improvement — logs become more actionable. No downstream consumers parse these strings. |

## Migration Plan

No migration needed. This is an internal refactor:

1. Implement all changes in a single PR
2. Run existing test suite — all callback-based tests must pass unchanged
3. Add new tests for retry, dedup, and error enrichment
4. Deploy — no config changes, no breaking changes

## Open Questions

1. **Should the retry delay be configurable?** Currently hardcoding 500ms. Could add a `lanRetryDelay` config option, but that adds config surface area. 500ms is a reasonable default for LAN.

2. **Should we track per-device error counts?** If a device fails 5 times in a row, we might want to mark it as unreachable and skip polling it for a cooldown period. This is related to but distinct from the retry logic.

3. **Is the `iu_model` truncation issue still present?** The comment in the code suggests it is, but we haven't verified against current firmware versions. If it's fixed, the single-call path becomes the default and the fallback can be removed.
