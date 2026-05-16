## Purpose

定义 NFS 协议连接器的接口规范，用于在 Kubernetes 节点上挂载和卸载 NFS 文件系统共享，支持块设备和文件系统挂载模式。

## Requirements

### Requirement: NFS 连接器应挂载和卸载 NFS 共享
NFS 连接器 SHALL 实现 VolumeConnector 接口，用于在 Kubernetes 节点上挂载/卸载 NFS 文件系统共享。它支持块设备挂载（srcType=block）和文件系统挂载（srcType=fs）。

#### Scenario:挂载 NFS 共享（文件系统类型）
- **WHEN** 连接器收到带有 srcType=fs、sourcePath、targetPath 和 protocol 的 ConnectVolume 请求时
- **THEN** 连接器解析连接信息（如果未设置则将 fsType 默认为 ext4），根据协议确定挂载类型（DPC 使用 dpc，DTFS 使用 dtfs），并调用 mountFS，该函数使用 MountToDir 挂载 NFS 共享

#### Scenario:挂载块设备时的格式化详情
- **WHEN** 连接器挂载块设备（srcType=block）时
- **THEN** 它执行以下操作：
  - 读取设备并使用 blkid 检查是否有文件系统；如果未格式化，则检查是否有其他进程正在格式化（等待 10 秒，如果是则返回错误）
  - 确定磁盘大小类型（default/big/huge/large/veryLarge，基于大小阈值：0.5TiB/1TiB/10TiB/100TiB/512TiB）；如果大小超过 512TiB 则返回错误
  - 使用适当的 mkfs 选项格式化磁盘并挂载
  - 如果 fsType=xfs 且源是 /dev/* 设备，则自动在挂载选项中添加 "nouuid" 以允许挂载具有相同 UUID 的克隆卷

#### Scenario:挂载已有文件系统的块设备
- **WHEN** 连接器收到带有 srcType=block 的 ConnectVolume 请求且设备已有文件系统时
- **THEN** 连接器挂载设备，并且如果 accessMode 不是 MULTI_NODE_MULTI_WRITER 或 MULTI_NODE_READER_ONLY，则调用 ResizeMountPath 扩展文件系统

#### Scenario:卸载 NFS 共享
- **WHEN** 连接器收到带有 targetPath 的 DisConnectVolume 请求时
- **THEN** 连接器使用 Unmount 卸载目标路径并删除目标目录

#### Scenario:处理 NFS 挂载失败
- **WHEN** NFS 服务器不可达或共享不存在时
- **THEN** 连接器返回带有挂载失败原因的错误

#### Scenario:使用只读选项挂载 NFS
- **WHEN** mountFlags 包含 "ro" 时
- **THEN** 连接器以只读方式挂载 NFS 共享

#### Scenario:使用协议特定选项挂载 NFS 共享
- **WHEN** 连接器挂载 NFS 共享（srcType=fs）时
- **THEN** 它解析连接信息（如果未设置则将 fsType 默认为 ext4），根据协议确定挂载类型（DPC 使用 dpc，DTFS 使用 dtfs），并将其作为 -t 选项传递给挂载命令

#### Scenario:拒绝不支持的源类型
- **WHEN** 连接器收到 srcType 不是 "block" 或 "fs" 的 ConnectVolume 请求时
- **THEN** 连接器返回错误 "not support source type"

#### Scenario:拒绝缺少源路径的挂载
- **WHEN** 连接器收到不带 sourcePath 的 ConnectVolume 请求时
- **THEN** 连接器返回错误 "there are no source path in the connection info"

#### Scenario:拒绝缺少目标路径的挂载
- **WHEN** 连接器收到不带 targetPath 的 ConnectVolume 请求时
- **THEN** 连接器返回错误 "there are no target path in the connection info"
