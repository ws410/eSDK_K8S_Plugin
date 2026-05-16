## Purpose

定义 CSI ControllerExpandVolume RPC 的接口规范，用于在华为存储后端上扩容卷容量，支持扇区对齐和存储类型验证。

## Requirements

### Requirement: ControllerExpandVolume RPC 必须扩容存储卷
ControllerExpandVolume RPC SHALL 在华为存储后端上扩容卷的容量。驱动必须验证容量范围，根据存储类型和访问模式验证扩容支持，将大小调整到扇区对齐，并指示是否需要节点侧扩容。

#### Scenario:扩容标准 LUN/FileSystem 卷
- **WHEN** CO 发送带有有效 VolumeId 和 CapacityRange（RequiredBytes，可选 LimitBytes）的 ControllerExpandVolumeRequest 时
- **THEN** 驱动验证容量范围（LimitBytes > 0 时必须 >= RequiredBytes），从 VolumeId 获取后端和卷名称，如果卷属性中未设置 disableVerifyCapacity 则验证扇区大小容量可用性，使用 TransVolumeCapacity 将大小调整到扇区对齐，使用 volName 和调整后的 size 调用 backend.Plugin.ExpandVolume，并返回 ControllerExpandVolumeResponse，包含 CapacityBytes（对齐后）和来自插件结果的 NodeExpansionRequired 标志

#### Scenario:扩容 DTree 卷
- **WHEN** CO 发送 DTree 存储后端（由 constants.IsDtreeStorage 标识）上卷的 ControllerExpandVolumeRequest 时
- **THEN** 驱动通过 GetDTreeParentNameByVolumeId 从卷 ID 映射中获取 DTree parentName，并使用 volName、parentName 和调整后的 size 调用 backend.Plugin.ExpandDTreeVolume，返回响应中包含来自插件结果的 NodeExpansionRequired

#### Scenario:拒绝缺少容量范围的扩容
- **WHEN** CO 发送不带 CapacityRange 的 ControllerExpandVolumeRequest 时
- **THEN** 驱动返回 codes.InvalidArgument 错误，消息为 "no capacity range provided"

#### Scenario:拒绝容量范围无效的扩容
- **WHEN** CO 发送 LimitBytes 已设置且小于 RequiredBytes 的 ControllerExpandVolumeRequest 时
- **THEN** 驱动返回 codes.InvalidArgument 错误，消息为 "limitBytes is smaller than requiredBytes"

#### Scenario:拒绝 RWX LUN 文件系统卷的扩容
- **WHEN** CO 发送 accessMode=MULTI_NODE_MULTI_WRITER、volumeMode=FileSystem 且 volumeType=lun 的卷的 ControllerExpandVolumeRequest 时
- **THEN** 驱动返回 codes.InvalidArgument 错误，指示此组合不支持扩容

#### Scenario:拒绝只读卷的扩容
- **WHEN** CO 发送 accessMode=MULTI_NODE_READER_ONLY 的卷的 ControllerExpandVolumeRequest 时
- **THEN** 驱动返回 codes.InvalidArgument 错误，指示只读卷不能扩容

#### Scenario:后端不存在时扩容卷
- **WHEN** CO 发送 VolumeId 引用不存在后端的 ControllerExpandVolumeRequest 时
- **THEN** 驱动返回 codes.Internal 错误，指示后端不存在

#### Scenario:拒绝卷 ID 缺失的扩容
- **WHEN** CO 发送 VolumeId 为空的 ControllerExpandVolumeRequest 时
- **THEN** 驱动返回 codes.InvalidArgument 错误，消息为 "no volume ID provided"

#### Scenario:扩容 NAS 卷（不需要节点扩容）
- **WHEN** CO 发送 NAS 存储类型的 ControllerExpandVolumeRequest 时
- **THEN** 驱动正常处理扩容，但 NAS 管理器的 ExpandVolume 返回 nil（不需要节点侧扩容），控制器响应中 NodeExpansionRequired 设置为 false

#### Scenario:扩容卷时进行扇区大小对齐
- **WHEN** CO 发送 RequiredBytes 不是扇区大小整数倍的 ControllerExpandVolumeRequest 时
- **THEN** 驱动向上舍入到下一个扇区大小倍数，验证对齐容量可用（除非设置了 disableVerifyCapacity），并在响应中返回对齐的 CapacityBytes

#### Scenario:拒绝缺少 VolumeCapability 的扩容
- **WHEN** CO 发送非 NAS 存储类型且请求中不带 VolumeCapability 的 ControllerExpandVolumeRequest 时
- **THEN** 驱动返回 codes.InvalidArgument 错误，指示 VolumeCapability 为空

#### Scenario:卷属性检索失败时跳过扇区大小验证
- **WHEN** CO 发送 ControllerExpandVolumeRequest 且 GetVolumeAttrsByVolumeId 检索卷属性失败时
- **THEN** verifySectorSize 函数记录警告并跳过容量验证（返回 nil），允许扩容继续进行

#### Scenario:卷属性为空时跳过扇区大小验证
- **WHEN** CO 发送 ControllerExpandVolumeRequest 且卷属性列表为空时
- **THEN** verifySectorSize 函数跳过验证（返回 nil），因为没有属性可检查

#### Scenario:PV 间 disableVerifyCapacity 冲突时跳过扇区大小验证
- **WHEN** CO 发送 ControllerExpandVolumeRequest 且多个具有相同 volumeId 的 PV 具有冲突的 disableVerifyCapacity 值时
- **THEN** verifySectorSize 函数记录关于冲突的警告并跳过验证（返回 nil）

#### Scenario:处理 SelectBackend 返回 nil 和错误的扩容
- **WHEN** CO 发送 ControllerExpandVolumeRequest 且 SelectBackend 返回 (nil, error) 时
- **THEN** 驱动条件 `backend == nil || err != nil` 评估为 true，并返回 codes.Internal 错误 "Backend <name> doesn't exist"

#### Scenario:拒绝 RequiredBytes 为零的扩容
- **WHEN** CO 发送 CapacityRange.RequiredBytes 设置为 0 的 ControllerExpandVolumeRequest 时
- **THEN** 驱动通过验证（CapacityRange 不为 nil），但后端插件收到大小为 0 扇区；后端可能根据实现拒绝或接受此请求

#### Scenario:拒绝请求大小小于当前大小（缩容）的扩容
- **WHEN** CO 发送 RequiredBytes 小于当前卷容量的 ControllerExpandVolumeRequest 时
- **THEN** 驱动不在 CSI 层检测缩容，而是委托给后端插件；后端的 Expand 方法可能返回错误拒绝请求

#### Scenario:扩容期间处理 panic 恢复
- **WHEN** ControllerExpandVolume 处理程序在执行期间遇到 panic 时
- **THEN** defer utils.RecoverPanic(ctx) 恢复 panic，记录堆栈跟踪，并返回适当的错误响应

#### Scenario:使用默认 disableVerifyCapacity 扩容卷（跳过验证）
- **WHEN** CO 发送 ControllerExpandVolumeRequest 且卷属性不包含 disableVerifyCapacity 时
- **THEN** verifySectorSize 函数将 disableVerifyCapacity 默认为 "true"，跳过扇区大小验证，并允许扩容在不进行对齐检查的情况下进行
