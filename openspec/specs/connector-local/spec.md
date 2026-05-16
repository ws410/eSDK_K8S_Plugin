## Purpose

定义 Local 协议连接器的接口规范，用于连接和断开 FusionStorage 直连本地 SCSI 卷，支持设备发现和轮询机制。

## Requirements

### Requirement: Local 连接器应连接和断开本地/SCSI 卷
Local 连接器 SHALL 为 FusionStorage 直连（本地 SCSI）卷实现 VolumeConnector 接口。

#### Scenario:连接本地卷
- **WHEN** 连接器收到带有本地参数（tgtLunWWN）的 ConnectVolume 请求时
- **THEN** 连接器从连接属性中提取 tgtLunWWN，验证其存在，使用 LocalDriver 类型和 tryConnectVolume 函数调用 ConnectVolumeCommon，该函数按 WWN 发现本地 SCSI 设备（例如 /dev/disk/by-id/wwn-0x*）并返回设备路径

#### Scenario:断开本地卷
- **WHEN** 连接器收到带有设备 WWN 的 DisConnectVolume 请求时
- **THEN** 连接器使用 LocalDriver 类型和 tryDisConnectVolume 函数调用 DisConnectVolumeCommon，该函数刷新并移除本地 SCSI 设备

#### Scenario:拒绝不带 tgtLunWWN 的 ConnectVolume
- **WHEN** 连接器收到连接属性中不包含 tgtLunWWN 的 ConnectVolume 请求时
- **THEN** 连接器返回错误 "key tgtLunWWN does not exist in connection properties"

#### Scenario:连接本地 SCSI 卷时的设备发现详情
- **WHEN** 连接器连接本地 SCSI 卷时
- **THEN** 它执行以下操作：
  - 通过 waitDevOnline 轮询 /dev/disk/by-id/wwn-0x<tgtLunWWN>，最多 30 次，间隔 2 秒（总计 60 秒）
  - 解析符号链接以获取实际设备路径（例如 /dev/sdX）用于设备验证
  - 如果超时则返回空字符串（而非错误）

#### Scenario:断开本地卷是有意的空操作
- **WHEN** 连接器收到本地卷的 DisConnectVolume 请求时
- **THEN** tryDisConnectVolume 函数有意设计为空操作（返回 nil 而不执行任何操作）；这是针对本地/直通场景的设计，主机拥有该设备，不应由 CSI 驱动断开连接
