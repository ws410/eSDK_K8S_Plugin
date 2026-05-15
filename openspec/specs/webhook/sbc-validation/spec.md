## ADDED Requirements

### Requirement: sbc-validation shall validate StorageBackendClaim on create and update
The admission webhook shall validate StorageBackendClaim resources to ensure they meet the required schema and business rules before being persisted.

#### Scenario: Validate SBC with required Provider field
- **WHEN** a user creates an SBC without the Provider field
- **THEN** the webhook rejects the request with an error indicating Provider is required

#### Scenario: Validate SBC ConfigMapMeta format
- **WHEN** a user creates an SBC with ConfigMapMeta not in "<namespace>/<name>" format
- **THEN** the webhook rejects the request with a format error

#### Scenario: Validate SBC SecretMeta format
- **WHEN** a user creates an SBC with SecretMeta not in "<namespace>/<name>" format
- **THEN** the webhook rejects the request with a format error

#### Scenario: Validate SBC update doesn't change immutable fields
- **WHEN** a user updates an SBC's Provider field
- **THEN** the webhook rejects the request (Provider is immutable after creation)

#### Scenario: Validate SBC with valid configuration
- **WHEN** a user creates an SBC with all required fields and valid formats
- **THEN** the webhook allows the request

#### Scenario: Validate SBC MaxClientThreads range
- **WHEN** a user creates or updates an SBC with MaxClientThreads outside the valid range
- **THEN** the webhook rejects the request with a range validation error

#### Scenario: Validate SBC Parameters format
- **WHEN** a user creates an SBC with Parameters that contain invalid key-value format
- **THEN** the webhook rejects the request with a parameter format error

#### Scenario: Validate SBC UseCert and CertSecret consistency
- **WHEN** a user creates an SBC with UseCert=true but CertSecret is empty
- **THEN** the webhook rejects the request indicating CertSecret is required when UseCert is enabled
