## ADDED Requirements

### Requirement: secret-management shall create and update backend Secrets
The secret-management process shall create Kubernetes Secret resources containing sensitive storage backend credentials (username, password) from StorageBackendClaim specifications.

#### Scenario: Create Secret from SBC
- **WHEN** a StorageBackendClaim is processed
- **THEN** the controller creates a Secret with the backend credentials (user, password) and authentication mode (local/ldap scope)

#### Scenario: Update Secret when SBC credentials change
- **WHEN** a StorageBackendClaim's SecretMeta is updated (e.g., via oceanctl update backend)
- **THEN** the controller creates a new Secret with a UUID suffix, updates the SBC's SecretMeta to point to the new Secret, and deletes the old Secret

#### Scenario: Create Secret with authentication mode
- **WHEN** a backend is configured with authenticationMode=ldap
- **THEN** the controller sets the authentication mode to scope "1" in the Secret
- **WHEN** a backend is configured with authenticationMode=local
- **THEN** the controller sets the authentication mode to scope "0" in the Secret
