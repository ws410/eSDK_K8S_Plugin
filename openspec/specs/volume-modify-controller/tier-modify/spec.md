## ADDED Requirements

### Requirement: tier-modify shall modify volume SmartTier settings
The volume-modify-controller shall support SmartTier (automated data tiering) modification for volumes through the DR-CSI ModifyVolume service.

#### Scenario: Modify volume SmartTier policy
- **WHEN** a VMC is created with Parameters containing SmartTier settings (tiering policy, migration schedule)
- **THEN** the controller calls DR-CSI ModifyVolume with ModifyVolumeType=SmartTier and the tiering parameters, and the storage plugin applies the SmartTier policy to the volume
