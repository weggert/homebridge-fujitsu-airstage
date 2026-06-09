## ADDED Requirements

### Requirement: Error messages include hostname
Every error string returned by the LAN client MUST include the hostname of the target device when the error originates from a network operation.

#### Scenario: Timeout error includes hostname
- **WHEN** an HTTP request to `192.168.1.100` times out
- **THEN** the error string contains `"192.168.1.100"`

#### Scenario: Connection error includes hostname
- **WHEN** a connection to `192.168.1.100` is refused
- **THEN** the error string contains `"192.168.1.100"`

### Requirement: Error messages include device ID
Every error string returned by the LAN client MUST include the device ID (MAC address, uppercase, no colons) when the error originates from an operation targeting a specific device.

#### Scenario: Timeout error includes device ID
- **WHEN** a request to device `39FFBB9D6EBA` times out
- **THEN** the error string contains `"39FFBB9D6EBA"`

### Requirement: Error messages include operation context
Every error string MUST indicate what operation was being performed (parameter name and whether it was a get or set).

#### Scenario: Get parameter error includes parameter name
- **WHEN** a `getDevice` call for device `39FFBB9D6EBA` at host `192.168.1.100` fails
- **THEN** the error string indicates that fetching parameters was attempted

#### Scenario: Set parameter error includes parameter name
- **WHEN** a `setParameter` call for `iu_onoff` on device `39FFBB9D6EBA` at host `192.168.1.100` fails
- **THEN** the error string contains `"iu_onoff"` and indicates a set operation

### Requirement: Error messages are human-readable
Error strings MUST be formatted as readable sentences, not concatenated raw values.

#### Scenario: Error message format
- **WHEN** a set operation for `iu_onoff` on device `39FFBB9D6EBA` at host `192.168.1.100` times out
- **THEN** the error string is formatted as: `"Timeout setting iu_onoff on device 39FFBB9D6EBA at 192.168.1.100"`

### Requirement: Internal errors preserve context
When a low-level error (from `lan/api/client.js`) is wrapped with context by `lan/client.js`, the original error message MUST be preserved in the enriched error string.

#### Scenario: Wrapped error preserves original message
- **WHEN** the API client returns error `"ECONNREFUSED"` for host `192.168.1.100`
- **THEN** the enriched error contains both the hostname and `"ECONNREFUSED"`
