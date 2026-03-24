- ## 1. 系统逻辑架构 (System Architecture)

  CodeArk 采用**分布式边缘计算**架构，将繁重的加解密与打包任务下放到客户端，服务端仅充当“大脑”与“信令中心”。

  ### 1.1 客户端设计 (The Ark-Client)

  客户端基于 **Go (Golang)** 开发，利用其天然的并发优势处理大文件 IO。

  - **FS-Watcher 模块：** 使用递归监听技术监控文件系统变更。
  - **Executor 模块：** 负责 ZIP 压缩与 AES 加密。加密采用流式处理，确保在处理数 GB 大小的项目时，内存占用控制在 256MB 以内。
  - **Uploader 模块：** 支持分片上传（Multipart Upload），针对大文件备份提供断点续传能力。

  ### 1.2 服务端设计 (The Ark-Server)

  服务端提供 RESTful API 与 WebSocket 实时双向通信。

  - **存储驱动抽象层 (Storage Driver Interface)：** * `S3 Adapter`: 适配阿里云 OSS、AWS S3、MinIO。
    - `FileSystem Adapter`: 适配本地磁盘、局域网 SMB/NFS 挂载点。
    - `SFTP Adapter`: 适配远程 VPS 存储。
  - **元数据管理器：** 使用 PostgreSQL 存储备份历史、哈希指纹及任务配置，确保数据的一致性。

  ------

  ## 2. 核心工作流深度拆解 (Workflow Deep Dive)

  ### 2.1 封存工作流 (The Archiving Flow)

  1. **扫描器触发：** 检测到静默期满或收到手动指令。
  2. **哈希比对：** 扫描文件树，计算当前状态快照，若与上次备份无差异则跳过。
  3. **管道打包：** * Step A: 将筛选后的文件流送入 ZIP 压缩器。
     - Step B: 为 ZIP 设置文件头密码。
     - Step C: 将输出流通过管道 (Pipe) 直接送入 AES-256-GCM 加密器，避免产生中间临时明文文件。
  4. **云端引渡：** 加密流直接推送至配置的存储节点。

  ### 2.2 恢复工作流 (The Recovery Flow)

  - **用户侧触发：** 在 Web 管理界面选择版本，点击“引渡回本地”。
  - **多路径恢复：** 客户端接收指令，下载密文流。
  - **本地解密：** 客户端调用本地存储的私钥解开 AES 层，还原为加密 ZIP，随后用户输入 ZIP 密码完成最终还原。

  ------

  ## 3. 安全性声明 (Security Specification)

  - **零知识原则：** 服务端不存储用户的 ZIP 密码。即使服务端数据库泄露，攻击者也无法通过备份文件获取代码内容。
  - **防篡改：** 每个备份包均附带 SHA-256 签名，下载恢复前会自动进行完整性校验。