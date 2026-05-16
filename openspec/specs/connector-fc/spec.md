## Purpose

定义 FC（Fibre Channel）协议连接器的接口规范，用于在 Kubernetes 节点上连接和断开光纤通道卷，支持多路径配置。

## Requirements

### Requirement: FC 连接器应连接和断开光纤通道卷
FC 连接器 SHALL 实现 VolumeConnector 接口，用于在 Kubernetes 节点上连接/断开光纤通道卷，支持多路径。

#### Scenario:连接 FC 卷
- **WHEN** 连接器收到带有 FC 参数（tgtWWNs, tgtLunWWN）的 ConnectVolume 请求时
- **THEN** 连接器触发 FC HBA 重新扫描，按 WWN 发现 FC LUN，配置多路径，并返回设备路径

#### Scenario:断开 FC 卷
- **WHEN** 连接器收到带有设备 WWN 的 DisConnectVolume 请求时
- **THEN** 连接器移除多路径设备并刷新 FC HBA 缓存

#### Scenario:处理未找到 FC HBA
- **WHEN** 节点未安装 FC HBA 时
- **THEN** 连接器返回错误，指示不存在 FC 启动器

#### Scenario:连接 FC 卷时的发现和多路径详情
- **WHEN** 连接器连接 FC 卷时
- **THEN** 它执行以下操作：
  - 通过向所有在线 FC HBA 的 sysfs issue_lip 和 scan 文件写入来触发 FC HBA 重新扫描
  - 执行最多 3 次扫描尝试，间隔 2 秒，等待设备发现总计最多 60 秒
  - 通过 /dev/disk/by-path/ 发现 FC LUN 设备路径，由 FC HBA 端口 WWN、目标端 WWN 和 LUN 号构造
  - 如果启用了 HWUltraPath 多路径，则轮询 UltraPath 设备管理器直到其接管设备，然后返回虚拟设备路径
  - 在建立新连接之前调用 CleanDeviceByLunId 移除与 LUN ID 关联的过时设备路径
