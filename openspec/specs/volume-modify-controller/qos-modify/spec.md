## ADDED Requirements

### Requirement: qos-modify shall modify volume QoS settings
The volume-modify-controller shall support QoS (Quality of Service) modification for volumes through the DR-CSI ModifyVolume service.

#### Scenario: Modify volume QoS
- **WHEN** a VMC is created with Parameters containing QoS settings (e.g., IOPS limits, bandwidth limits)
- **THEN** the controller calls DR-CSI ModifyVolume with ModifyVolumeType=QoS and the QoS parameters, and the storage plugin applies the QoS policy to the volume

#### Scenario: Reject QoS modify for unsupported volume
- **WHEN** the target volume's storage pool doesn't support QoS (SupportQoS=false)
- **THEN** the DR-CSI ModifyVolume call fails and the Content is marked as "Failed"
