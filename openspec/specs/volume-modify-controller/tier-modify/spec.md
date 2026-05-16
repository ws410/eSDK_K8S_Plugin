## ADDED Requirements

### Requirement: tier-modify (SmartTier) is NOT IMPLEMENTED
The volume-modify-controller spec mentions SmartTier (automated data tiering) modification, but this feature is NOT implemented in the current codebase. The ModifyVolumeType enum only has Local2HyperMetro and HyperMetro2Local values. No SmartTier handler exists in the DR-CSI provider or storage plugins.

This spec file documents a planned but unimplemented feature. Do not rely on these scenarios for current behavior.
