## ADDED Requirements

### Requirement: Promise wrapper for all public methods
Every public method on `airstage.lan.Client` that currently accepts a callback MUST also be available as a method that returns a Promise.

#### Scenario: Promise resolves on success
- **WHEN** `getPowerState(deviceId)` is called without a callback and the operation succeeds
- **THEN** the returned Promise resolves with the result value

#### Scenario: Promise rejects on error
- **WHEN** `getPowerState(deviceId)` is called without a callback and the operation fails
- **THEN** the returned Promise rejects with the error string

#### Scenario: Callback usage remains unchanged
- **WHEN** `getPowerState(deviceId, callback)` is called with a callback
- **THEN** the callback is invoked with `(error, result)` as before
- **AND** the returned Promise (if any) is ignored by the caller

### Requirement: Promise wrapper preserves callback semantics
The Promise wrapper MUST not change the behavior of existing callback-based callers.

#### Scenario: Callback is invoked before Promise resolves
- **WHEN** both a callback and no-callback usage exist for the same method
- **THEN** the callback is invoked with `(error, result)` exactly as before
- **AND** the Promise resolves/rejects with the same values

#### Scenario: Multiple callbacks on same call are supported
- **WHEN** the internal implementation calls the callback multiple times (if any method does this)
- **THEN** each callback is invoked as before
- **AND** the Promise resolves with the last successful result

### Requirement: Private internal methods use async/await
Internal methods within `lan/client.js` that compose multiple public-method calls MUST use `async/await` or `Promise.all()` instead of nested callbacks.

#### Scenario: Internal composition uses Promises
- **WHEN** an internal method like `_getDeviceFromApi` calls multiple API methods sequentially
- **THEN** it uses `async/await` or `Promise.all()` for control flow
- **AND** the public callback-based API still works correctly (wraps the Promise internally)

### Requirement: No breaking changes to external API
The existing callback-based API surface of `airstage.lan.Client` MUST remain unchanged.

#### Scenario: All existing method signatures work
- **WHEN** existing code calls `client.getDevice(id, cb)`, `client.setParameter(id, name, value, cb)`, etc.
- **THEN** the calls work identically to before this change
- **AND** no existing tests fail
