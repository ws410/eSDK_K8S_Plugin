## MODIFIED Requirements

### Requirement: ControllerGetCapabilities RPC 必须通告控制器能力
ControllerGetCapabilities RPC SHALL 通告驱动支持的控制器服务能力。驱动支持：CREATE_DELETE_VOLUME、PUBLISH_UNPUBLISH_VOLUME、EXPAND_VOLUME、CREATE_DELETE_SNAPSHOT、CLONE_VOLUME 和 GET_VOLUME。

#### Scenario:CO 查询控制器能力
- **WHEN** CO 发送 ControllerGetCapabilitiesRequest 时
- **THEN** 驱动返回 ControllerGetCapabilitiesResponse，包含能力：CREATE_DELETE_VOLUME、PUBLISH_UNPUBLISH_VOLUME、EXPAND_VOLUME、CREATE_DELETE_SNAPSHOT、CLONE_VOLUME 和 GET_VOLUME

---

### Requirement: 未实现的控制器 RPC
以下控制器 RPC SHALL NOT 由此驱动实现，对所有请求返回 codes.Unimplemented：ListVolumes、GetCapacity、ValidateVolumeCapabilities。

#### Scenario:CO 请求卷列表
- **WHEN** CO 发送 ListVolumesRequest 时
- **THEN** 驱动返回 codes.Unimplemented 错误，消息为 "Not implemented"

#### Scenario:CO 请求存储容量信息
- **WHEN** CO 发送 GetCapacityRequest 时
- **THEN** 驱动返回 codes.Unimplemented 错误，消息为 "Not implemented"

#### Scenario:CO 请求卷能力验证
- **WHEN** CO 发送 ValidateVolumeCapabilitiesRequest 时
- **THEN** 驱动返回 codes.Unimplemented 错误，消息为 "Not implemented"
