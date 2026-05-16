## Purpose

定义 FC-NVMe 协议连接器的接口规范，用于通过光纤通道传输连接和断开 NVMe 卷，支持线程安全操作和命名空间发现。

## Requirements

### Requirement: FC-NVMe 连接器应连接和断开 FC-NVMe 卷
FC-NVMe 连接器 SHALL 实现 VolumeConnector 接口，用于通过光纤通道连接/断开 NVMe 卷。它使用互斥锁实现线程安全操作。

#### Scenario:连接 FC-NVMe 卷
- **WHEN** 连接器收到带有 FC-NVMe 参数（PortWWNList, TgtLunGuid）的 ConnectVolume 请求时
- **THEN** 连接器从连接属性中提取 tgtLunGuid，验证其存在，使用 FCNVMe 驱动类型和 tryConnectVolume 函数调用 ConnectVolumeCommon，该函数发现 FC-NVMe 子系统，通过 FC-NVMe 传输连接，按 GUID 发现命名空间，并返回设备路径

#### Scenario:断开 FC-NVMe 卷
- **WHEN** 连接器收到带有设备 GUID 的 DisConnectVolume 请求时
- **THEN** 连接器使用 FCNVMe 驱动类型和 tryDisConnectVolume 函数调用 DisConnectVolumeCommon，该函数断开与 FC-NVMe 目标端的连接并清理

#### Scenario:拒绝不带 tgtLunGuid 的 ConnectVolume
- **WHEN** 连接器收到连接属性中不包含 tgtLunGuid 的 ConnectVolume 请求时
- **THEN** 连接器返回错误 "there is no Lun GUID in connect info"

#### Scenario:线程安全的 FC-NVMe 操作
- **WHEN** 多个 goroutine 同时调用 ConnectVolume 或 DisConnectVolume 时
- **THEN** 连接器的互斥锁确保对 FC-NVMe 操作的串行访问

#### Scenario:连接 FC-NVMe 卷时的发现详情
- **WHEN** 连接器连接 FC-NVMe 卷时
- **THEN** 它执行以下操作：
  - 运行 `nvme list-subsys -o json` 发现 FC-NVMe 通道，过滤 Transport="fc" 且 State="live" 的通道，按启动器和目标端 WWN 对进行匹配
  - 对每个通道运行 `nvme ns-rescan /dev/<channel>` 以发现新命名空间
  - 轮询最多 5 次，间隔 1 秒，用于 UltraPath-NVMe 设备发现，在返回之前通过 IsUpNVMeResidualPath 检查残留路径

#### Scenario:断开 FC-NVMe 卷时不进行会话清理
- **WHEN** 连接器断开 FC-NVMe 卷时
- **THEN** 与 NVMe over Fabrics 不同，FC-NVMe 连接器不执行显式的会话断开（FC 结构自动处理此操作）；它仅移除虚拟和物理设备，如果启用了多路径则刷新 DM 设备
