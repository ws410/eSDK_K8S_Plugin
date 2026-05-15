## ADDED Requirements

### Requirement: tier-modify shall modify volume SmartTier settings
The volume-modify-controller shall support SmartTier (automated data tiering) modification for volumes through the DR-CSI ModifyVolume service.

#### Scenario: Modify volume SmartTier policy
- **WHEN** a VMC is created with Parameters containing SmartTier settings (tiering policy, migration schedule)
- **THEN** the controller calls DR-CSI ModifyVolume with ModifyVolumeType=SmartTier and the tiering parameters, and the storage plugin applies the SmartTier policy to the volume

#### Scenario: Reject SmartTier modify for unsupported volume
- **WHEN** the target volume's storage type doesn't support SmartTier (e.g., OceanDisk, DME A-Series, A-Series NAS)
- **THEN** the DR-CSI ModifyVolume call fails with "does not support volume modify feature" and the Content is marked as "Failed"

#### Scenario: SmartTier modify on OceanStor SAN
- **WHEN** a VMC targets a volume on oceanstor-san with SmartTier license
- **THEN** the OceanStor plugin's ModifyVolume method applies the SmartTier policy via the storage array API
