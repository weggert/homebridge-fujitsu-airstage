## ADDED Requirements

### Requirement: Retry on transient failures
The LAN client MUST automatically retry a failed HTTP request once after a 500ms delay before surfacing the error to the caller.

#### Scenario: Successful first attempt
- **WHEN** a `postGetParam` or `postSetParam` request receives a valid JSON response
- **THEN** the result is returned to the caller immediately with no retry

#### Scenario: Timeout triggers retry
- **WHEN** an HTTP request times out (30s elapsed with no response)
- **THEN** the client waits 500ms and retries the same request once
- **AND** if the retry succeeds, the result is returned to the caller
- **AND** if the retry also times out, the error `"Request timeout"` is returned

#### Scenario: Connection error triggers retry
- **WHEN** an HTTP request fails with a connection error (ECONNREFUSED, ENOTFOUND, ECONNRESET)
- **THEN** the client waits 500ms and retries the same request once
- **AND** if the retry succeeds, the result is returned to the caller
- **AND** if the retry also fails, the original error is returned

#### Scenario: Retry does not apply to parse errors
- **WHEN** the server responds with valid JSON that indicates a logical error (e.g., `result: "error"` in the response body)
- **THEN** no retry is performed and the error is returned immediately

#### Scenario: Retry does not apply to authorization failures
- **WHEN** the server returns HTTP status 401 or 403
- **THEN** no retry is performed and the error is returned immediately

### Requirement: Retry limit is one
- **WHEN** a request has been retried once
- **THEN** no further retries are attempted regardless of the error type

### Requirement: Retry is transparent to callers
- **WHEN** a retried request succeeds
- **THEN** the caller receives the same result shape as a first-attempt success
- **AND** the caller has no way to distinguish a retried success from a first-attempt success
