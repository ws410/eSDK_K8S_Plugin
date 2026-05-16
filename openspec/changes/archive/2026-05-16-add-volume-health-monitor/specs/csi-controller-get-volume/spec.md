## ADDED Requirements

### Requirement: ControllerGetVolume RPC 必须返回卷健康状态
ControllerGetVolume RPC SHALL 查询存储后端并返回卷的健康状态。返回的 VolumeCondition 中 `healthy` 字段表示卷健康状态，`abnormal` 字段不返回。驱动 SHALL NOT 返回 `published` 字段。

#### Scenario: SAN 存储卷健康状态正常
- **WHEN** CO 发送 ControllerGetVolumeRequest，请求 SAN 类型存储卷且存储返回 HEALTHSTATUS 为 "1"
- **THEN** 驱动返回 ControllerGetVolumeResponse，VolumeCondition 中 `healthy` 为 true

#### Scenario: SAN 存储卷健康状态异常
- **WHEN** CO 发送 ControllerGetVolumeRequest，请求 SAN 类型存储卷且存储返回 HEALTHSTATUS 不为 "1"
- **THEN** 驱动返回 ControllerGetVolumeResponse，VolumeCondition 中 `healthy` 为 false

#### Scenario: NAS 存储卷存在
- **WHEN** CO 发送 ControllerGetVolumeRequest，请求 NAS 类型存储卷且卷存在
- **THEN** 驱动返回 ControllerGetVolumeResponse，VolumeCondition 中 `healthy` 为 true

#### Scenario: NAS 存储卷不存在
- **WHEN** CO 发送 ControllerGetVolumeRequest，请求 NAS 类型存储卷且卷不存在
- **THEN** 驱动返回 ControllerGetVolumeResponse，VolumeCondition 中 `healthy` 为 false

#### Scenario: Dtree 存储卷存在
- **WHEN** CO 发送 ControllerGetVolumeRequest，请求 Dtree 类型存储卷且卷存在
- **THEN** 驱动返回 ControllerGetVolumeResponse，VolumeCondition 中 `healthy` 为 true

#### Scenario: Dtree 存储卷不存在
- **WHEN** CO 发送 ControllerGetVolumeRequest，请求 Dtree 类型存储卷且卷不存在
- **THEN** 驱动返回 ControllerGetVolumeResponse，VolumeCondition 中 `healthy` 为 false

#### Scenario: 请求的卷 ID 无效
- **WHEN** CO 发送 ControllerGetVolumeRequest，卷 ID 为空
- **THEN** 驱动返回 codes.InvalidArgument 错误

#### Scenario: 存储后端不可用
- **WHEN** CO 发送 ControllerGetVolumeRequest，存储后端不可达
- **THEN** 驱动返回 codes.Internal 错误

---

### Requirement: ControllerGetVolume 必须校验卷 ID 参数
ControllerGetVolume RPC SHALL 校验请求中的 volume_id 字段，为空时返回 codes.InvalidArgument 错误。

#### Scenario: 卷 ID 为空字符串
- **WHEN** CO 发送 ControllerGetVolumeRequest，volume_id 为空字符串
- **THEN** 驱动返回 codes.InvalidArgument 错误

#### Scenario: 卷 ID 有效
- **WHEN** CO 发送 ControllerGetVolumeRequest，volume_id 为有效值
- **THEN** 驱动继续执行卷健康状态查询
