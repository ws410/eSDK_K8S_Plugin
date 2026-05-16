## ADDED Requirements

### Requirement: migration-modify (SmartMigration) is NOT IMPLEMENTED
The volume-modify-controller spec mentions SmartMigration (online volume migration) modification, but this feature is NOT implemented in the current codebase. The ModifyVolumeType enum only has Local2HyperMetro and HyperMetro2Local values. No SmartMigration handler exists in the DR-CSI provider or storage plugins.

This spec file documents a planned but unimplemented feature. Do not rely on these scenarios for current behavior.
