## Purpose

定义 CSI NodeStageVolume RPC 的接口规范，用于在节点上暂存卷以供使用，支持 SAN 和 NAS 存储类型的连接和挂载操作。

## Requirements

### Requirement: NodeStageVolume RPC 必须在节点上暂存卷
NodeStageVolume RPC SHALL 通过将卷暂存到暂存目标路径来准备卷以供节点使用。驱动创建 manage.Manager（根据后端协议为 SanManager 或 NasManager）并将暂存操作委托给它。SAN 暂存涉及通过适当的协议连接器（iSCSI、FC、FC-NVMe、RoCE、NVMe、SCSI）将卷连接到主机，而 NAS 暂存涉及挂载 NFS/NFS+/DPC/DTFS 共享。

#### Scenario:暂存 SAN 卷（iSCSI/FC/FC-NVMe/RoCE/NVMe/SCSI）
- **WHEN** CO 发送带有 VolumeId、StagingTargetPath、VolumeCapability、PublishContext（包含 publishInfo）和 VolumeContext 的 NodeStageVolumeRequest 时
- **THEN** 驱动分割 VolumeId 获取 backendName，为后端协议创建 SanManager，使用 WithProtocol、WithConnector、WithVolumeCapability、WithControllerPublishInfo 和 WithMultiPathType 构建参数，执行任务流：clearResidualPathWithWwn → clearResidualPathWithLunId → connectVolume →（stageForBlock 或 stageForMount，基于 volumeMode）→ saveWwnToDisk，成功后返回空的 NodeStageVolumeResponse

#### Scenario:暂存 SAN 块卷
- **WHEN** CO 发送 VolumeCapability.GetBlock() != nil 的 NodeStageVolumeRequest 时
- **THEN** 驱动在参数中设置 volumeMode="Block" 和 stagingPath=StagingTargetPath + "/" + VolumeId，执行 stageForBlock 任务在暂存路径创建块设备，并将 WWN 保存到磁盘

#### Scenario:暂存 SAN 文件系统卷
- **WHEN** CO 发送 VolumeCapability.GetMount() != nil 的 NodeStageVolumeRequest 时
- **THEN** 驱动从挂载能力中提取 fsType，验证 fsType（如果指定必须是 ext2、ext3、ext4 或 xfs），提取 mountFlags，确定 accessMode（ReadOnly 添加 "ro" 标志），在参数中设置 targetPath=StagingTargetPath、fsType、mountFlags 和 accessMode，执行 stageForMount 任务创建文件系统并挂载，并将 WWN 保存到磁盘

#### Scenario:暂存 NAS 卷（NFS/NFS+/DPC/DTFS）
- **WHEN** CO 发送 NAS 后端（协议：nfs、nfs+、dpc、dtfs）上卷的 NodeStageVolumeRequest 时
- **THEN** 驱动创建 NasManager，使用 WithProtocol、WithPortals、WithVolumeCapability 和 WithDeviceWWN 构建参数；对于 DTree 存储，跳过暂存（立即返回）；对于其他 NAS 类型，从协议和门户生成 sourcePath（例如 NFS 为 "portal:/"，DPC/DTFS 为 "/"），与 volumeName 拼接，并将共享挂载到暂存目标路径

#### Scenario:使用双活文件系统模式暂存 NAS 卷
- **WHEN** CO 发送 NAS 卷的 NodeStageVolumeRequest，且 PublishContext 中 protocol=nfs+ 且 filesystemMode=HyperMetro 时
- **THEN** 驱动将本地门户和双活门户合并为单个门户列表用于挂载操作

#### Scenario:使用设备 WWN 暂存 DTFS 卷
- **WHEN** CO 发送 DTFS 协议上带有 deviceWWN 的卷的 NodeStageVolumeRequest 时
- **THEN** 驱动在挂载操作的 mountFlags 中添加 "cid=deviceWWN"

#### Scenario:拒绝创建管理器失败的 NodeStageVolume
- **WHEN** CO 发送 NodeStageVolumeRequest 且后端的 manage.NewManager 调用失败（例如不支持的协议）时
- **THEN** 驱动返回 codes.Internal 错误，消息指示后端和失败原因

#### Scenario:拒绝 publishInfo 缺失且自动附加失败的 NodeStageVolume
- **WHEN** CO 发送 PublishContext 中不带 publishInfo 的 NodeStageVolumeRequest 且自动附加操作失败（后端离线、构建后端失败、获取主机名失败、附加失败或编组失败）时
- **THEN** 驱动返回 codes.Internal 错误，包含具体的失败原因

#### Scenario:拒绝 fsType 无效的 NodeStageVolume
- **WHEN** CO 发送 fsType 不是 ext2、ext3、ext4 或 xfs 之一的 NodeStageVolumeRequest（对于挂载类型）时
- **THEN** 驱动返回 codes.Internal 错误，包含支持的文件系统类型列表

#### Scenario:拒绝卷能力无效的 NodeStageVolume
- **WHEN** CO 发送 VolumeCapability 既不是 Block 也不是 Mount 类型的 NodeStageVolumeRequest 时
- **THEN** 驱动返回 codes.Internal 错误，指示卷能力无效

#### Scenario:publishInfo 缺失时自动附加（后端离线）
- **WHEN** CO 发送不带 publishInfo 的 NodeStageVolumeRequest 且 StorageBackendContent 状态为 nil 或 Online=false 时
- **THEN** attachVolume 函数返回错误 "attach volume failed cause backend offline, backend name: <name>"，驱动返回 codes.Internal

#### Scenario:publishInfo 缺失时自动附加（构建后端失败）
- **WHEN** CO 发送不带 publishInfo 的 NodeStageVolumeRequest 且 backend.BuildBackend 失败（配置无效、缺少凭据、插件初始化失败）时
- **THEN** attachVolume 函数返回错误 "attach volume failed while building backend, backend name: <name>, err: <error>"，驱动返回 codes.Internal

#### Scenario:publishInfo 缺失时自动附加（获取主机名失败）
- **WHEN** CO 发送不带 publishInfo 的 NodeStageVolumeRequest 且 utils.GetHostName 失败时
- **THEN** attachVolume 函数返回错误 "attach volume failed while getting hostname, err: <error>"，驱动返回 codes.Internal

#### Scenario:publishInfo 缺失时自动附加（附加到存储阵列失败）
- **WHEN** CO 发送不带 publishInfo 的 NodeStageVolumeRequest 且 buildBackend.Plugin.AttachVolume 失败时
- **THEN** attachVolume 函数返回错误 "attach volume failed while attaching volume, volume name: <name>, err: <error>"，驱动返回 codes.Internal

#### Scenario:仅对 UltraPath 使用 LUN ID 清理残留路径
- **WHEN** SanManager StageVolume 运行 clearResidualPathWithLunId 任务时
- **THEN** 函数检查是否 VolumeUseMultiPath=true 且 MultiPathType=HWUltraPath 且协议为 "iscsi" 或 "fc"；如果所有条件满足，则调用 connector.CleanDeviceByLunId 在连接之前按 LUN ID 清理过时设备

#### Scenario:暂存卷时检查 NVMe CLI 版本
- **WHEN** CO 发送 NVMe 协议后端上卷的 NodeStageVolumeRequest 时
- **THEN** NVMe 连接器验证已安装的 nvme-cli 版本 >= 1.9 后再继续连接；如果版本过低则返回错误

#### Scenario:暂存 SAN 文件系统卷时使用 XFS nouuid 挂载选项
- **WHEN** CO 发送 fsType=xfs 的 SAN 文件系统卷的 NodeStageVolumeRequest 时
- **THEN** mountDisk 函数自动在挂载选项中添加 "nouuid"，以允许在同一节点上挂载具有相同 UUID 的克隆卷

#### Scenario:暂存块卷时使用旧版符号链接处理
- **WHEN** CO 发送块卷的 NodeStageVolumeRequest 且暂存目标路径是符号链接（V4.6.0 之前的旧格式）时
- **THEN** BindMountRawBlockDevice 函数移除符号链接并将目标路径重新创建为常规文件，然后执行绑定挂载

#### Scenario:暂存 SAN 文件系统卷时使用基于磁盘大小的 mkfs 策略
- **WHEN** CO 发送未格式化块设备上 SAN 文件系统卷的 NodeStageVolumeRequest 时
- **THEN** getDiskSizeType 函数根据设备大小确定 mkfs 模板：<=0.5TiB="default"，0.5-1TiB="big"，1-10TiB="huge"，10-100TiB="large"，100-512TiB="veryLarge"，>512TiB 返回错误；formatDisk 函数应用相应的 mkfs 模板（例如 ext 文件系统使用 "-T big"）

#### Scenario:拒绝磁盘大小超出最大值的暂存卷
- **WHEN** CO 发送大于 512TiB 的未格式化块设备上 SAN 文件系统卷的 NodeStageVolumeRequest 时
- **THEN** getDiskSizeType 函数返回错误 "the disk size does not support"

#### Scenario:暂存 SAN 文件系统卷时检测并发格式化
- **WHEN** CO 发送 SAN 文件系统卷的 NodeStageVolumeRequest 且另一个进程已经在格式化该设备时
- **THEN** formatDisk 函数在 mkfs 输出中检测到 "in use by the system"，休眠 10 秒，并返回错误

#### Scenario:暂存 SAN 文件系统卷时跳过分区设备
- **WHEN** clearResidualPathWithWwn 任务扫描 /dev/disk/by-id/ 查找设备路径时
- **THEN** 分区设备（例如 sdc1、nvme0n1p1、dm-1 带尾随数字）在残留路径检测期间被显式跳过

#### Scenario:暂存 SAN 文件系统卷时处理 blkid 退出码 2
- **WHEN** CO 发送 SAN 文件系统卷的 NodeStageVolumeRequest 且 blkid 返回退出码 2 时
- **THEN** getFSType 函数调用 connector.IsDeviceFormatted 验证设备是否确实已格式化；如果已格式化但 blkid 失败，则返回模糊错误

#### Scenario:暂存 SAN 文件系统卷时挂载后扩容
- **WHEN** CO 发送已格式化设备上 SAN 文件系统卷的 NodeStageVolumeRequest 且 accessMode 不是 MULTI_NODE_MULTI_WRITER 或 MULTI_NODE_READER_ONLY 时
- **THEN** 挂载后调用 connector.ResizeMountPath 函数扩容文件系统（ext* 使用 resize2fs，xfs 使用 xfs_growfs）

#### Scenario:多节点访问时暂存 SAN 文件系统卷跳过扩容
- **WHEN** CO 发送 accessMode=MULTI_NODE_MULTI_WRITER 或 MULTI_NODE_READER_ONLY 的 SAN 文件系统卷的 NodeStageVolumeRequest 时
- **THEN** 挂载后跳过扩容步骤以防止与其他节点冲突

#### Scenario:使用 DPC/DTFS 协议挂载选项暂存卷
- **WHEN** CO 发送 protocol=dpc 或 protocol=dtfs 的 NAS 卷的 NodeStageVolumeRequest 时
- **THEN** parseNFSInfo 函数将 mntDashT 设置为挂载命令的适当协议类型
