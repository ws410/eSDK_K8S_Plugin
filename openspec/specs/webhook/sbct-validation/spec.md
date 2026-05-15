## ADDED Requirements

### Requirement: sbct-validation is NOT IMPLEMENTED
The SBCT (StorageBackendContent) validation webhook is NOT implemented in the current codebase. There is no webhook handler registered for storagebackendcontents resources. No admitStorageBackendContent function exists. Validation of SBCT fields occurs indirectly through the controller-side logic (SplitMetaNamespaceKey on BackendClaim in backend_register.go), but this is not a webhook admission validation.

#### Scenario: Validate SBCT with required Provider field (NOT IMPLEMENTED)
- **WHEN** a user creates an SBCT without the Provider field
- **THEN** no webhook validation exists; the field is not validated at admission time

#### Scenario: Validate SBCT BackendClaim format (NOT IMPLEMENTED)
- **WHEN** a user creates an SBCT with BackendClaim not in "<namespace>/<name>" format
- **THEN** no webhook validation exists; the format is validated at controller runtime via SplitMetaNamespaceKey

#### Scenario: Validate SBCT ConfigmapMeta format (NOT IMPLEMENTED)
- **WHEN** a user creates an SBCT with ConfigmapMeta not in "<namespace>/<name>" format
- **THEN** no webhook validation exists; the format is validated at controller runtime

#### Scenario: Validate SBCT with valid configuration (NOT IMPLEMENTED)
- **WHEN** a user creates an SBCT with all required fields and valid formats
- **THEN** no webhook validation exists; the request is allowed without webhook checks

#### Scenario: Validate SBCT SecretMeta format (NOT IMPLEMENTED)
- **WHEN** a user creates an SBCT with SecretMeta not in "<namespace>/<name>" format
- **THEN** no webhook validation exists; the format is validated at controller runtime

#### Scenario: Validate SBCT MaxClientThreads range (NOT IMPLEMENTED)
- **WHEN** a user creates or updates an SBCT with MaxClientThreads outside the valid range
- **THEN** no webhook validation exists; this validation is NOT implemented
