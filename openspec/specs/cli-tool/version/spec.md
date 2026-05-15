## ADDED Requirements

### Requirement: version command shall display the CLI tool version
The `oceanctl version` command shall display the current version of the oceanctl CLI tool.

#### Scenario: Display CLI version
- **WHEN** the user runs `oceanctl version`
- **THEN** the CLI outputs the version string (from build-time constants)
