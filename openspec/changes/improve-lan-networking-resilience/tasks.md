## 1. Promise API Layer

- [ ] 1.1 Create `_promisify` helper in `lan/client.js` that wraps callback methods to return Promises when no callback is provided
- [ ] 1.2 Wrap `_makeHttpRequest` (api/client.js) with Promise-based `_makeHttpRequestAsync`
- [ ] 1.3 Wrap `postGetParam` and `postSetParam` with Promise-based versions
- [ ] 1.4 Wrap `getDevice` with Promise-based `_getDeviceAsync`
- [ ] 1.5 Verify all existing callback-based callers still work (run test suite)

## 2. Retry Logic

- [ ] 2.1 Define transient vs non-transient error classification in `lan/api/client.js` (timeout, ECONNRESET, ENOTFOUND = transient; parse errors, 4xx = non-transient)
- [ ] 2.2 Implement retry logic in `_makeHttpRequest` — retry once after 500ms for transient errors
- [ ] 2.3 Write tests for retry: timeout retry succeeds, timeout retry fails, non-transient errors not retried
- [ ] 2.4 Ensure retry is transparent to callers (retried success looks identical to first-attempt success)

## 3. Error Context Enrichment

- [ ] 3.1 Create `makeError(hostname, deviceId, operation, param, baseError)` helper in `lan/api/client.js`
- [ ] 3.2 Update `_makeHttpRequest` to include hostname in error context
- [ ] 3.3 Update `lan/client.js` methods to include deviceId and parameter name in error context for `getDevice`, `setParameter`, and all derived getters/setters
- [ ] 3.4 Write tests for error enrichment: timeout includes hostname+deviceId, set error includes param name, wrapped error preserves original message
- [ ] 3.5 Verify existing error strings are replaced (grep for old patterns like `"Parameter not available: "` and `"No hostname for device ID: "`)

## 4. In-Flight Request Deduplication

- [ ] 4.1 Create `inFlightRequests: Map<deviceId, Promise>` in `lan/client.js` constructor
- [ ] 4.2 Implement deduplication in `_getDeviceAsync` — check map before initiating network request
- [ ] 4.3 Implement subscriber pattern — concurrent callers for same device wait on existing Promise
- [ ] 4.4 Clean up map entry with `Promise.finally()` after settlement (success or failure)
- [ ] 4.5 Ensure `setParameter` is never deduplicated (always sent to device)
- [ ] 4.6 Write tests: first request spawns network call, concurrent requests are deduplicated, failed requests are not deduplicated

## 5. Single Round-Trip Parameter Fetch

- [ ] 5.1 Update `PARAMETER_NAMES` constant to include all 12 parameters in a single array
- [ ] 5.2 Implement `_getDeviceFromApiAsync` to attempt single `postGetParam` call with all parameters
- [ ] 5.3 Detect truncation: if `iu_model` is missing from response, fall back to two-call pattern
- [ ] 5.4 Write tests: single-call success with model, single-call missing model triggers fallback, fallback works correctly
- [ ] 5.5 Remove or deprecate `PARAMETER_NAMES_BESIDES_MODEL` and `PARAMETER_NAMES_ONLY_MODEL` if no longer needed (keep if fallback uses them)

## 6. Integration and Testing

- [ ] 6.1 Run full test suite — all existing tests must pass
- [ ] 6.2 Write integration-style test: device with cold cache, verify deduplication reduces network calls
- [ ] 6.3 Write integration-style test: device times out, verify retry happens and final error is enriched
- [ ] 6.4 Write integration-style test: multiple concurrent reads of same device, verify only one network request
- [ ] 6.5 Verify platform.js still works with the updated LAN client (no code changes needed, but confirm)

## 7. Documentation and Cleanup

- [ ] 7.1 Update JSDoc comments on public methods to note Promise return when no callback provided
- [ ] 7.2 Add inline comments explaining the retry behavior and deduplication logic
- [ ] 7.3 Review and remove any dead code from the two-round-trip pattern if fallback is unused
- [ ] 7.4 Confirm no new dependencies were added (package.json unchanged)
