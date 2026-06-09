## ADDED Requirements

### Requirement: Deduplicate in-flight device requests
When multiple callers request data for the same device while a request is already in-flight, new callers MUST wait for the existing request to complete instead of spawning a duplicate network call.

#### Scenario: First request spawns network call
- **WHEN** `getDevice(deviceId)` is called and no request for that device is in-flight
- **THEN** the client initiates a network request to the device

#### Scenario: Concurrent requests are deduplicated
- **WHEN** `getDevice(deviceId)` is called a second time while the first request is still in-flight
- **THEN** the second caller receives the result of the first request (no duplicate network call)
- **AND** both callers receive the same result object

#### Scenario: Deduplication applies to all device fetch operations
- **WHEN** `getModel`, `getPowerState`, `getOperationMode`, or any other method that triggers `getDevice` is called for a device with an in-flight request
- **THEN** the caller waits for the existing in-flight request to complete

#### Scenario: Failed requests are not deduplicated
- **WHEN** an in-flight request for a device fails
- **THEN** subsequent calls for that device initiate a new network request (the failed request is not cached)

#### Scenario: Successful set operations are not deduplicated
- **WHEN** `setParameter` is called, it MUST always be sent to the device (set operations are never deduplicated)
