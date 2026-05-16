## ADDED Requirements

### Requirement: qos-modify is NOT IMPLEMENTED
The volume-modify-controller spec mentions QoS (Quality of Service) modification, but this feature is NOT implemented in the current codebase. The ModifyVolumeType enum only has Local2HyperMetro and HyperMetro2Local values. No QoS handler exists in the DR-CSI provider.

This spec file documents a planned but unimplemented feature. Do not rely on these scenarios for current behavior.
