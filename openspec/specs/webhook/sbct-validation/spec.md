## ADDED Requirements

### Requirement: sbct-validation shall validate StorageBackendContent on create and update
The admission webhook shall validate StorageBackendContent resources to ensure they meet the required schema and business rules.

#### Scenario: Validate SBCT with required Provider field
- **WHEN** a user creates an SBCT without the Provider field
- **THEN** the webhook rejects the request with an error indicating Provider is required

#### Scenario: Validate SBCT BackendClaim format
- **WHEN** a user creates an SBCT with BackendClaim not in "<namespace>/<name>" format
- **THEN** the webhook rejects the request with a format error

#### Scenario: Validate SBCT ConfigmapMeta format
- **WHEN** a user creates an SBCT with ConfigmapMeta not in "<namespace>/<name>" format
- **THEN** the webhook rejects the request with a format error

#### Scenario: Validate SBCT with valid configuration
- **WHEN** a user creates an SBCT with all required fields and valid formats
- **THEN** the webhook allows the request

#### Scenario: Validate SBCT SecretMeta format
- **WHEN** a user creates an SBCT with SecretMeta not in "<namespace>/<name>" format
- **THEN** the webhook rejects the request with a format error

#### Scenario: Validate SBCT MaxClientThreads range
- **WHEN** a user creates or updates an SBCT with MaxClientThreads outside the valid range
- **THEN** the webhook rejects the request with a range validation error
