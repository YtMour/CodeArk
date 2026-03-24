## 1. 系统架构图 (System Architecture)

CodeArk 采用 **Agent-Server-Storage** 模型：

- **Client (Agent):** 负责文件 IO 监听、任务切片、ZIP 封装、AES 加密。
- **Server (Control Plane):** 负责任务调度指令下发、存储节点路由、元数据持久化、用户权限校验。
- **Storage (Data Plane):** 物理载体，仅存储经过加密的 `.ark` 文件包。

## 2. 核心逻辑详解

### 2.1 静默备份触发算法 (Quiescence Algorithm)

为避免频繁 IO 导致性能下降，系统内置 `Watcher` 模块：

1. **事件截获：** 基于 `inotify` (Linux) 或 `ReadDirectoryChangesW` (Windows) 捕获文件写结束信号。
2. **计时器重置：** 每次 IO 事件发生，重置 $T_{idle}$ 计时器。
3. **阈值判定：** 当 $T_{now} - T_{last\_io} \ge T_{threshold}$（默认 600s）且 CPU 占用率 $< 20\%$ 时，启动备份流水线。

### 2.2 加密保障机制 (Encryption Pipeline)

系统对安全性有严格的序时要求：

- **Step 1:** 调用 `libzip` 将目标目录打包。
- **Step 2:** 在 ZIP 头信息中注入用户设置的 `Archive-Password`。
- **Step 3:** 生成 `Initialization Vector (IV)`，使用 `Master-Token` 通过 **AES-256-GCM** 对整个 ZIP 文件进行块加密。
- **Step 4:** 生成 `.sntl` 封装格式，包含：`[Header][IV][Encrypted Payload][HMAC Tag]`。

## 3. 技术栈选型理由

- **Go (Backend & Client):** 静态编译保证了 EXE 客户端在虚拟机中的零依赖运行，并发处理大文件压缩效率极高。
- **Vue 3 + Pinia (Frontend):** 响应式状态管理，支持多任务进度条实时刷新。
- **Wails:** 相比 Electron，体积更小，内存占用极低，适合作为后台驻留程序。