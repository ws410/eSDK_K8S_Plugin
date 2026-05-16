## ADDED Requirements

### Requirement: update-backend command shall update backend account information
The `oceanctl update backend` command shall update a backend's Secret with new credentials (password and/or authentication mode), creating a new Secret with UUID, updating the SBC's secretMeta, and cleaning up the old Secret.

#### Scenario: Update backend password in default namespace
- **WHEN** the user runs `oceanctl update backend <name> --password`
- **THEN** the CLI queries the SBC, prompts for new password, creates a new Secret with UUID suffix, updates the SBC's secretMeta to point to the new Secret, deletes the old Secret, and prints "backend [<name>] updated"

#### Scenario: Update backend password in specified namespace
- **WHEN** the user runs `oceanctl update backend <name> -n <namespace> --password`
- **THEN** the CLI performs the update in the specified namespace

#### Scenario: Update backend authentication mode to LDAP
- **WHEN** the user runs `oceanctl update backend <name> --password --authenticationMode=ldap`
- **THEN** the CLI converts the authentication mode to scope "1" (LDAP), creates the new Secret with the updated authenticationMode, and updates the SBC

#### Scenario: Update backend authentication mode to local
- **WHEN** the user runs `oceanctl update backend <name> --password --authenticationMode=local`
- **THEN** the CLI converts the authentication mode to scope "0" (local), creates the new Secret, and updates the SBC

#### Scenario: Reject update-backend with non-existent backend
- **WHEN** the user runs `oceanctl update backend <name>` and the SBC doesn't exist
- **THEN** the CLI prints "Backend <name> not found" and returns without error

#### Scenario: Reject update-backend with multiple backend names
- **WHEN** the user runs `oceanctl update backend <name1> <name2>`
- **THEN** the CLI validator rejects the request (ValidateNameIsSingle)

#### Scenario: Reject update-backend with invalid authentication mode
- **WHEN** the user runs `oceanctl update backend <name> --password --authenticationMode=invalid`
- **THEN** the CLI validator rejects the request (ValidateAuthenticationMode)

#### Scenario: Update backend with rollback on failure
- **WHEN** the SBC update fails after the new Secret is created
- **THEN** the CLI restores the old SBC secretMeta by re-applying the old claim YAML, and deletes the newly created Secret
