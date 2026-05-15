## ADDED Requirements

### Requirement: migration-modify shall modify volume SmartMigration settings
The volume-modify-controller shall support SmartMigration (online volume migration) modification for volumes through the DR-CSI ModifyVolume service.

#### Scenario: Modify volume SmartMigration
- **WHEN** a VMC is created with Parameters containing SmartMigration settings (target pool, migration speed)
- **THEN** the controller calls DR-CSI ModifyVolume with ModifyVolumeType=SmartMigration and the migration parameters, and the storage plugin initiates the online volume migration

#### Scenario: Reject SmartMigration for unsupported volume
- **WHEN** the target volume's storage type doesn't support SmartMigration (e.g., OceanDisk, DME A-Series, A-Series NAS, DTree)
- **THEN** the DR-CSI ModifyVolume call fails with "does not support volume modify feature" or "not implement" and the Content is marked as "Failed"

#### Scenario: SmartMigration on OceanStor SAN
- **WHEN** a VMC targets a volume on oceanstor-san with SmartMigration license
- **THEN** the OceanStor plugin's ModifyVolume method initiates the online volume migration to the target pool via the storage array API
