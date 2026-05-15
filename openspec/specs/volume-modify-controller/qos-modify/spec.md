## ADDED Requirements

### Requirement: qos-modify shall modify volume QoS settings
The volume-modify-controller shall support QoS (Quality of Service) modification for volumes through the DR-CSI ModifyVolume service.

#### Scenario: Modify volume QoS
- **WHEN** a VMC is created with Parameters containing QoS settings (e.g., IOPS limits, bandwidth limits)
- **THEN** the controller calls DR-CSI ModifyVolume with ModifyVolumeType=QoS and the QoS parameters, and the storage plugin applies the QoS policy to the volume

#### Scenario: Reject QoS modify for unsupported volume
- **WHEN** the target volume's storage pool doesn't support QoS (SupportQoS=false)
- **THEN** the DR-CSI ModifyVolume call fails and the Content is marked as "Failed"

#### Scenario: QoS modify on OceanStor SAN
- **WHEN** a VMC targets a volume on oceanstor-san with SmartQoS license
- **THEN** the OceanStor plugin's ModifyVolume method applies the QoS policy via the storage array API

#### Scenario: QoS modify on A-Series
- **WHEN** a VMC targets a volume on oceanstor-a-series-nas with SmartQoS license
- **THEN** the A-Series plugin's SupportQoSParameters validates the QoS parameter value ranges before applying
