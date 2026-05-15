## ADDED Requirements

### Requirement: migration-modify shall modify volume SmartMigration settings
The volume-modify-controller shall support SmartMigration (online volume migration) modification for volumes through the DR-CSI ModifyVolume service.

#### Scenario: Modify volume SmartMigration
- **WHEN** a VMC is created with Parameters containing SmartMigration settings (target pool, migration speed)
- **THEN** the controller calls DR-CSI ModifyVolume with ModifyVolumeType=SmartMigration and the migration parameters, and the storage plugin initiates the online volume migration
