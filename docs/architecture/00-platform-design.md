# Octopus Embedded 核心架构设计与工程决策规范 (Architecture Specification & RFCs)

**简体中文** | [English](./00-platform-design.en.md)

> **文档定位**：本文档是 Octopus Embedded 开源项目的**总体架构设计规范与核心决策论证（Architecture Specification & Decision Records）**。  
> **适用阶段**：指导 Phase 1（核心骨架）至 Phase 4（量产加固）的工程实现，作为全系统代码实现、跨芯片移植与开源社区协作的根本技术指引。

---

## 1. 业务痛点与核心使命

嵌入式 Linux 操作系统虽然在各类工业智能网关、边缘计算盒子、储能控制器、移动机器人及车载终端中被广泛采用，但长期以来，绝大多数嵌入式团队的研发过程都陷入了**“底层频繁推倒重来、中间件三套皮割裂、现场偶发故障排查无门、硬件换型脱一层皮”**的恶性泥潭。

深入审视当今嵌入式系统的开发与量产全流程，核心存在以下 8 大工程痛点：

### 1.1 芯片 SDK 强绑定与生命周期外壳机制碎片化
芯片原厂 SDK（如瑞芯微 Rockchip、全志 Allwinner、星宸 SigmaStar 等）的交付本质上是以“能跑通公版演示板（Demo Board）”为核心导向，而非面向“工业级高可靠产品全生命周期”：
- **镜像格式与烧录工装各异**：不同厂商各自发明芯片专用镜像打包格式与配套上位机刷机工具，格式互不兼容；
- **启动与分区命名随心所欲**：分区表参数格式（parameter/env/sys_config/DTS）五花八门，U-Boot 环境变量结构和加载脚本各行其是；
- **外壳机制深度绑死**：看门狗启动、进程守护、出厂恢复脚本散落在各种 `/etc/init.d/` 或原厂私有内核补丁中。

当硬件因缺货、降本或性能升级需要从 RK3568 切换到 SSD20X 或全志时，业务逻辑代码或许只要重新交叉编译，但**整套固件打包脚本、产线烧录工装、OTA 服务器升级协议、U-Boot 选槽与掉电回退逻辑必须全量推倒重写**。研发团队 70% 的精力都在被动应付原厂的外壳碎片化，无法沉淀真正的核心软件资产。

```text
 瑞芯微 SDK (RK3568)               全志 SDK (Allwinner)              星宸 SDK (SSD20X)
 ┌──────────────────────┐          ┌──────────────────────┐          ┌──────────────────────┐
 │ 私有镜像格式 / 分区表  │          │ 私有镜像格式 / 分区描述│          │ 私有 IPL / 烧录格式   │
 │ 厂商烧录与打包工具    │          │ 厂商烧录工具与脚本    │          │ 厂商升级与烧录路径    │
 │ SDK 内置 recovery 逻辑│          │ 厂商启动回退逻辑      │          │ 厂商启动链与看门狗    │
 └──────────┬───────────┘          └──────────┬───────────┘          └──────────┬───────────┘
            ▼                                 ▼                                 ▼
       三套无法复用的外壳工具 ───► 每换一款 SoC，升级/产线/售后机制全部重写一遍
```

### 1.2 存储芯片频繁变动与裸 Flash / 块设备物理断层
在长生命周期的工业硬件中，Flash 存储芯片的变动极其频繁：
- **单板降本换料**：从昂贵的 eMMC 改为 SPI-NAND，或为极致压缩成本换为单 SPI-NOR；
- **物理特性的天然鸿沟**：
  - **SPI-NAND / SLC NAND（裸 Flash）**：出厂即允许存在 1%~2% 的初始物理坏块，且在使用寿命中会动态产生新坏块。直接使用常规 `dd` 刷写分区必然导致数据损坏；若升级程序跳坏块的算法与 U-Boot 引导加载跳坏块的算法不完全对称，设备升级后必将变砖；
  - **NOR Flash**：线性可读但擦写块尺寸极小，容量受限，需要极高压缩比的只读文件系统；
  - **eMMC / SD**：带有内置 FTL 控制器的块设备，突发断电容易损坏文件系统元数据；
- **无抽象导致的上层灾难**：缺乏统一的存储抽象层（SAL），使得 OTA 刷写工具、开机挂载脚本和引导配置必须针对每块板子的具体介质硬编码 `/dev/mtdX` 或 `/dev/mmcblk0pX`，改一次版整套存储管线崩溃一次。

### 1.3 管理面割裂：CLI、Web、RPC、云端与本地屏各写各的
现场设备的配置入口不止一条。运维用串口或 SSH 命令行，局域网用 Web，云端和手机 App 走 JSON-RPC，带屏的网关、机器人、工控主机还在 HDMI 或 LCD 上跑一套本地界面（Qt、LVGL 等）：
- **多套解析器**：每增加一个参数，就要在命令行、网页、云端协议、本地屏和存盘模块里各写一遍。漏改一处，各入口看到的值就不一样；
- **本地屏私自存一份**：HDMI/LCD 界面把亮度、网络、业务参数写进自己的数据库或 JSON。串口改过的值屏上仍是旧的；在屏上点保存，又把 Web 和云端刚下发的配置盖掉；
- **并发脏写**：没有统一的内存配置和事务锁。现场工程师正在串口里改参数，另一个人在网页或本地屏上点保存，后写入的一方整份覆盖；
- **掉电截断**：配置直接覆盖写文件，写到一半断电，文件变成空文件或半截，下次启动业务起不来。

### 1.4 PID 1 进程编排两极困境：systemd 臃肿吃爆内存 vs BusyBox 单薄崩溃即瘫痪
嵌入式 Linux 在系统初始化与服务监护（PID 1）上长期处于两极分化的尴尬境地：
- **桌面派生方案（systemd）**：功能完整，但 PID 1 加上 journald、udevd 等配套组件后常驻内存在十余 MB 到数十 MB（视启用组件而定），强依赖 glibc（musl 需额外补丁），未经深度裁剪时启动耗时数秒到十余秒。在 64MB RAM 的紧凑型网关上，这部分开销挤占了业务可用内存，裁剪与维护成本很高；
- **原始方案（BusyBox init / SysVinit）**：依靠 `/etc/inittab` 与 rc 脚本串行启动。`respawn` 可以重拉单个进程，但服务之间没有依赖 DAG（有向无环图），开发者只能在脚本中写 `sleep 2` 猜时序；没有就绪通知、退避重启和健康检查，进程反复崩溃时只会被无限快速重拉，或者停在“进程还在、服务已死”的假死状态。

### 1.5 现场死机排查黑盒化与“无法复现”的售后泥潭
工业设备部署在野外高杆、变电站、工厂流水线或深井矿区，现场偶发死机排查极其痛苦：
- **看门狗盲目复位，现场零痕迹**：系统发生死锁或崩溃后，唯一的保底手段是硬件看门狗硬复位，复位后 RAM 易失性数据彻底蒸发，现场日志空空如也；
- **Flash 容量过小无法 Core Dump**：嵌入式设备可写 Flash 分区通常只有几 MB 至几十 MB，桌面 Linux 生成动辄上百兆的 Core Dump 文件根本无法保存，强行落盘还会迅速磨穿 Flash 寿命；
- **隐蔽的多线程死锁**：持锁超时、ABBA 交叉死锁、慢回调阻塞往往需要系统连续高负载运行数周才偶然复现一次，现场无法挂接 GDB，更无法连 JTAG；
- **慢速微量内存泄漏**：某个通信子模块每小时缓慢泄漏 10KB 内存，连续运行三个月后最终引发 OOM 崩溃，开发团队在成千上万行代码中盲猜排查，修复周期长达数月。

### 1.6 突发断电下的根文件系统损坏与系统变砖
在工农业与特种设备现场，直接拔电源、产线拉闸或母线瞬时欠压极其频繁（每周甚至每天都可能发生）：
- 如果设备采用读写混合的单一文件系统（如直接格式化为 ext4 或 ubifs 的单根分区），正在进行日志落盘或配置写入时突发掉电，极易损坏文件系统元数据（Superblock/Journal/Inode）；
- 再次上电时内核无法挂载根分区，屏幕或串口直接输出 `Kernel panic: VFS: Unable to mount root fs`，现场设备彻底变砖，必须返厂拆机重刷。

### 1.7 具身智能与 AI 边缘时代的“大小脑割裂与吞吐瓶颈”
面向带有 NPU 算力的现代智能边缘网关或移动机器人（AGV/AMR/巡检机械臂），架构面临全新挑战：
- **控制与推理混杂的单点崩溃风险**：厂商提供的 NPU 用户态运行时与第三方神经网络算法栈，往往比管理面进程更重、也更容易出现泄漏或偶发故障。如果将实时运动控制（小脑）与 AI 视觉（大脑）塞在同一个巨型进程，AI 侧崩溃将直接导致底层电机失控，引发严重的物理机械碰撞事故；
- **跨进程视频吞吐开销打满 CPU**：若将两者拆为独立进程，使用传统的本地 Socket 或 HTTP 传输 1080P/4K 高清视频帧，频繁的内存拷贝与内核上下文切换将把嵌入式 CPU 占用率推高至 100%，不仅产生高达上百毫秒的端到端时延，更会挤占核心控制线程的实时调度带宽。

### 1.8 通用升级框架的落地断层
SWUpdate、RAUC 等开源升级框架提供了用户态升级与双槽切换的通用逻辑，但在工业级嵌入式落地时存在天然断层：
1. **启动链真空**：通用框架假设 U-Boot 已经具备完善的选槽与坏块处理逻辑，对各类 SoC 的 Bootloader 差异无能为力；
2. **缺乏系统协同**：通用框架只管把镜像写入分区，不管理运行时的配置版本迁移、进程生命周期编排、锁监控与死机自愈；
3. **资源门槛偏高**：在 64MB RAM / 16MB Flash 的超紧凑物联网网关上，通用框架难以深度裁切。

### 1.9 Octopus 的核心使命与破局之道
Octopus Embedded 的立项初衷，就是要**将工业产品的“全生命周期基础设施”彻底固定并产品化**。
通过自研规范屏蔽底层芯片与硬件介质的天然断层，将芯片厂商的差异完全收敛于 Linux 内核设备树和极薄的板级接口（Board HAL），实现**“一套底座，多芯通用；一次编写，全线复用”**：

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │ 业务应用层（网关路由协议、具身机器人控制、AI 视觉推理进程）                 │ 换 SoC 时保持不动
 ├────────────────────────────────────────────────────────────────────────┤
 │ Octopus 通用产品基础设施（固定生命周期）                                  │
 │  · 统一固件包格式 (OFI)        · A/B 双槽引导与防变砖状态机               │ 跨芯片完全统一
 │  · 轻量级 PID 1 (supervisor)   · 单一真相配置中心 (octopus_config)       │ 零迁移成本
 │  · 统一层级命令行 (octopus_cli)· 硬件看门狗联动与崩溃黑匣子                │
 ├────────────────────────────────────────────────────────────────────────┤
 │ Linux 内核 + 设备树 + 薄板级 HAL (liboctopus_board)                      │ 仅该层随芯片适配
 └────────────────────────────────────────────────────────────────────────┘
```

---



## 2. 架构设计原则与非功能基线（NFR）



### 2.1 核心设计哲学

1. **换 SoC 保住业务**：芯片专属差异完全下沉至设备树与纯 C 编写的薄板级 HAL；
2. **掉电可自愈**：全系统支持任意时刻突发掉电，双固件槽自动仲裁回退，配置文件原子写盘；
3. **故障域物理隔离**：大内存消耗的 AI 视觉、重型动态路由协议放入独立进程，崩溃绝不波及主管理面；
4. **配置单一真相**：内存只维护一份运行配置。串口 CLI、Web、JSON-RPC、云端和本机 HDMI/LCD 界面都是这份配置的客户端，任一入口的修改对其余入口立即可见；落盘采用可读可审计的文本配置；
5. **克制演进，杜绝过度设计**：核心底座优先使用轻量 C/C++，杜绝庞大外部依赖，适配小内存硬件。

> **「零迁移成本」的边界（重要限定）**：第 1 条追求的是「换 SoC 时业务代码与 OTA 逻辑不动」，但它的成立有一个前提 —— **目标平台的 libc 与内核能力处在同一基线之上**。已验证的例外见 [reliability 4.2](reliability/00-critical-topics.md)：SSD20X 一类的老 BSP 停留在 glibc 2.24 / uClibc-ng + 内核 4.9，`_TIME_BITS=64` 在该组合下不起作用，`time_t` 仍为 32 位，2038 问题无法彻底消除。此类平台的降级是**不可消除**的，必须在产品文档中写明支持期限，不能对外笼统宣称「零迁移成本」。



### 2.2 系统资源与非功能基线（NFR）


| 维度          | 紧凑型网关（如 SSD20X）                     | 工业智能边缘（如 RK3568）          | 检验与约束方法                |
| ----------- | ----------------------------------- | ------------------------- | ---------------------- |
| **内存硬件规格**  | 64 MB ~ 128 MB DDR                  | 1 GB ~ 4 GB LPDDR4        | 硬件基准配置                 |
| **持久存储介质**  | 16 MB ~ 32 MB SPI-NOR / 128MB NAND  | 8 GB ~ 32 GB eMMC / NVMe  | 分区规划基准                 |
| **底座常驻内存**  | **≤ 12 MB**（含 supervisor, app, cli） | **≤ 35 MB**（含基础路由插件）      | `ps_mem` / cgroup 限制   |
| **固件根文件系统** | **≤ 8 MB**（SquashFS 压缩）             | **≤ 64 MB**（完整工具链与模型）     | OFI 打包体积门槛             |
| **冷启动时间**   | **≤ 3.5 秒**（上电至主控制面可用）              | **≤ 8.0 秒**（上电至 NPU/网络就绪） | 串口时间戳（grabserial）+ GPIO 打点示波器计时 |
| **突发掉电耐受**  | **10,000 次** 随机断电无变砖、无坏死            | **10,000 次** 随机断电配置无丢失    | 产线断电继电器老化测试            |
| **看门狗响应**   | 硬件看门狗硬超时 20 秒，心跳 5 秒                | 硬件看门狗硬超时 20 秒，心跳 5 秒      | `/dev/watchdog` 独占锁定   |
| **最低内核版本**  | 4.9（厂商 BSP 现状，无 PSI、无 64 位 time 系统调用） | 4.19 / 5.10                | 能力探测 + 降级路径，见 9.3 内存压力阶梯 |
| **工作温度范围**  | **-20 ~ +70 °C**（整机满载，含 NPU 机型）      | **-20 ~ +70 °C**              | 高低温箱 + 满载拷机，判据见 9.5 |
| **存储温度范围**  | **-40 ~ +85 °C**（非上电状态）              | **-40 ~ +85 °C**              | 高低温箱，非上电 |
| **`octopus_app` 单进程内存** | **≤ 8 MB**（RSS，不含 cli 与 supervisor） | **≤ 28 MB**              | `ps_mem` / cgroup，用于 9.3 的自检阈值 |

> **温度指标说明**：上表是**整机**承诺值，不是器件规格值。器件（DDR、eMMC、PHY、NPU）各自的允许范围更宽，整机指标由其中最窄的一环与散热设计共同决定。具体数字须在硬件选型阶段按散热实测确定；**软件侧必须提供温度采样通道与分级降级策略（见 9.5），否则该指标无法被验证也无法被守住**。

> **容量下限说明**：A/B 双槽要求 Flash 至少能放下两份“内核 + 根文件系统”。按上表 8MB 根文件系统加 3MB 左右内核估算，**16MB SPI-NOR 放不下完整 A/B**。16MB 机型只能二选一：把根文件系统压到 5MB 以内，或者退化为“主系统 + 精简救援系统（recovery）”的非对称双槽，此时救援系统只负责联网重新下载主系统。这一取舍必须在硬件选型阶段确定，不能留到量产。


---



## 3. 系统分层与运行时进程拓扑



### 3.1 四层系统架构

```text
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. 应用与扩展层 (Application & Ext Layer)                                │
│                                                                         │
│   静态 Web 页面    本机 HDMI/LCD 界面   第三方客户进程    可选：AI 视觉进程 │
│   (HTML/JS)        (Qt / LVGL，只做显示)  (C / Go / Python)  (独占 NPU)   │
│         │                 │                    │                 │        │
│         └──── JSON-RPC ───┴──── 本机 IPC ───────┴─────────────────┘        │
│              （Web / 本地屏 / CLI / 云端读写同一份运行配置）                │
├─────────────────────────────────────────────────────────────────────────┤
│ 3. 主管理面 octopus_app (Management Plane Core)                         │
│                                                                         │
│   ┌────────────┐ ┌────────────┐ ┌──────────────┐ ┌──────────────────┐   │
│   │ octopus_cli│ │ JSON-RPC   │ │ 本机 IPC     │ │ octopus_config   │   │
│   │ 命令行引擎 │ │ 服务端     │ │ 服务端 (UDS) │ │ 配置中心引擎     │   │
│   └────────────┘ └────────────┘ └──────────────┘ └──────────────────┘   │
│                          │ 共享内存数据 (Candidate-Running 事务保护)      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │ 内建模块：系统管理 / 网络配置 / 固件升级编排 / 诊断自省          │   │
│   │ 可选插件：switch_offload 交换芯片硬件转发表卸载                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────────┤
│ 2. 运行环境层 (Runtime Environment)                                     │
│                                                                         │
│   octopus_loader (早期启动合成)         octopus_supervisor (PID 1 监护)  │
│   组装只读 SquashFS + 内存 OverlayFS    按依赖拓扑拉起服务，异常自动拉起 │
├─────────────────────────────────────────────────────────────────────────┤
│ 1. 硬件与内核层 (Hardware & Kernel)                                     │
│                                                                         │
│   U-Boot 引导程序 (选槽与验签)          Linux 内核 + 设备树 (DTS)       │
│   板级硬件抽象 HAL (liboctopus_board: MAC / SN / LED / Reset 键 / 喂狗)  │
│   可选：交换芯片 (DSA)、NPU、HDMI/LCD 的 DRM 显示、硬件看门狗           │
└─────────────────────────────────────────────────────────────────────────┘
```



### 3.2 职责与禁止越界红线


| 层级         | 必须承担的核心职责                   | 严禁越界的行为（红线）                      |
| ---------- | --------------------------- | -------------------------------- |
| **硬件与内核层** | 硬件初始化、驱动抽象、标准设备暴露（`/dev/`*） | 严禁承载业务状态机，严禁硬编码产品业务逻辑            |
| **运行环境层**  | 合成只读根目录、收割僵尸进程、按依赖拉起与健康监护   | 严禁处理具体的网络/网关配置，严禁解析业务协议          |
| **主管理面进程** | 维护全系统配置唯一真相、处理并发事务、管理模块生命周期 | 严禁在主循环中执行重度耗时操作（如固件写盘、打包日志、模型推理） |
| **应用与扩展层** | 承载外围业务（本地屏界面、视频流推理、高阶动态路由协议） | 严禁私自落盘系统配置；HDMI/LCD 界面与 Web、CLI、云端一样，只能经管理面改配置 |




### 3.3 运行时进程拓扑（`ps` 视角）

```text
  [Linux 内核] (PID 0)
        │
        ▼ exec
  [octopus_supervisor] (PID 1)
        │
        ├── [轻量 Web 服务] (Hiawatha/FastCGI) ──JSON-RPC──► 转发至 octopus_app
        │
        ├── [octopus_app] (核心主进程)
        │     │
        │     ├── [Reactor 主循环] (配置事务、IPC 监听、RPC 分发)
        │     ├── [Worker 线程池] (异步任务分发)
        │     ├── [Watchdog 专属线程] (心跳巡检与 /dev/watchdog 独占喂狗)
        │     │
        │     ├── 启动辅进程：[firmware_worker] (固件擦写与校验，按需拉起)
        │     └── 启动辅进程：[diag_collector] (日志与 coredump 收集，按需拉起)
        │
        ├── [octopus_cli] (串口/SSH 交互式登录终端) ──IPC──► 接入 octopus_app
        │
        ├── (可选) [local_ui] (HDMI/LCD：Qt 或 LVGL) ──JSON-RPC──► 同一份运行配置
        │
        ├── (可选) [FRR zebra/ospfd] (动态路由进程) ──Netlink──► Linux 内核 FIB
        │
        └── (可选) [ai_vision_daemon] (AI 视觉进程) ──SHM 环形区──► 共享内存数据面
```

---

### 3.4 「不掉电变砖」与「7×24 自愈」的兑现前提

本架构把「防变砖」与「故障自愈」作为核心卖点，但这两项承诺并不是纯软件就能成立的，它们各自依赖若干**硬件与平台前提**。以下前提必须在 bring-up 阶段逐项实测，**任一项不成立时，对应承诺在该机型上即为不可兑现**，必须在产品文档中如实降级说明，不能默认成立：

| # | 前提 | 不成立时的后果 | 验证方式 |
| :--- | :--- | :--- | :--- |
| 1 | 硬件看门狗实际探测成功，且 `octopus_app` 能独立打开 | 用户态死锁时无人复位，系统永久假死 | 打开 `/dev/watchdog`，故意挂起 `octopus_app`，确认 20 秒内整机复位 |
| 2 | 看门狗复位后 `pstore` / `ramoops` 记录仍存活 | 看门狗复位后现场零痕迹，「死机可溯源」失效 | 每块板子冷/热复位各 20 次，核对崩溃记录存在且内容完整 |
| 3 | SoC 能区分**掉电复位**与**看门狗复位**（复位原因寄存器可靠性） | bootcount 会把电网抖动误判为新固件缺陷，累积若干次后把一个完好固件判死 | 分别制造掉电、看门狗、软复位各 50 次，核对 `octopus_board_get_reset_reason()` 判定准确率 100% |
| 4 | 温度传感器可读，且过温不受看门狗覆盖 | 高温下只能靠看门狗反复重启，重启不解决发热，形成热循环 | 满载拷机至热稳态，确认 9.5 的分级降级动作按预期触发 |
| 5 | BootROM 支持多副本 SPL 搜索（或 eMMC boot0/boot1 轮询） | Bootloader 单点无备份，只能依赖 Maskrom 按键救砖 | 按芯片手册确认 + 破坏主 SPL 后上电，验证能否从副本启动 |
| 6 | U-Boot 具备读取 UBI 静态卷的能力（`CONFIG_CMD_UBI` + 静态卷 + 对应 MTD 驱动） | 4.1 的「两端同一套 UBI 逻辑地址」论证落空，NAND 机型上 A/B 无法成立 | 在目标 U-Boot 版本上 `ubi read` 读出一个静态卷并与原文件比对 |
| 7 | 分区边界落在介质擦除块边界上（NOR 常见 64KB 擦除块） | 擦写某分区时会损坏相邻分区 | 核对 DTS 分区偏移与器件 datasheet；实测擦写 SLOT_A 后 SLOT_B 头部完好 |
| 8 | SoC 硬件 RNG 可用 | 首启生成的 SSH 主机密钥与 TLS 证书偏弱，多台设备可能雷同 | 首启后统计多台设备的密钥指纹，确认无重复 |

> **设计约束**：第 1、4 两项不成立时，架构层面不得以「重启」作为兜底手段 —— 重启既不能让失效的看门狗复活，也不能降低芯片温度。这两类故障必须各有**非复位类**的降级路径（前者应在启动阶段即告警并要求更换硬件；后者见 9.5）。



---



## 4. 存储抽象架构、固件布局与 A/B 容灾引导状态机



### 4.1 存储抽象架构：6 大标准逻辑卷模型（SAL Logical Volume Model）

#### (1) 物理介质天然断层与抽象必要性
嵌入式硬件产品矩阵因成本、体积与供应链经常变更存储方案（超低成本网关用单 SPI-NOR，中端用单 SPI-NAND，工业高可靠用 NOR+eMMC/NAND 双介质，高性能 AI 边缘用单 eMMC）。然而各类介质的物理特性存在天然断层：

| 介质类型 | 最小擦除单位 | 读写特性 | 坏块与磨损问题 | Linux 访问接口 | 典型容量与选型定位 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **SPI-NOR Flash** | 扇区 (4KB / 64KB) | 随机字节读，页写，块擦除 | **100% 无坏块**，极高可靠性，寿命 ~10 万次 | MTD 字符设备 (`/dev/mtdX`) | 16MB ~ 64MB，低成本/高可靠引导 |
| **SPI-NAND Flash** | 块 (128KB / 256KB) | 必须整页读写 (2KB/4KB) | **出厂即有坏块**，运行期增加，需 ECC 与跳坏块 | MTD / UBI 卷 (`/dev/ubiX_Y`) | 128MB ~ 512MB，中端高性价比 |
| **eMMC / UFS** | 扇区 (512 Byte) | 内部自带 FTL 控制器，表现为标准块设备 | 硬件自动管理坏块与磨损，系统无感 | 块设备 (`/dev/mmcblk0pX`) | 4GB ~ 64GB+，AI 边缘与大数据 |

若由上层业务（OTA、配置、引导）直接面向物理设备，换一颗料就会导致 U-Boot 脚本、Loader 挂载、OTA 刷写工具全面写满 `#ifdef`。  
**Octopus 确立存储抽象层（Storage Abstraction Layer, SAL）核心准则**：屏蔽底层物理设备节点，向上暴露 **6 大标准逻辑卷（Logical Volumes）**：

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                    全系统统一的 6 大逻辑卷 (Logical Volumes)           │
 ├──────────┬──────────┬──────────────┬──────────────┬─────────────┬──────┤
 │ 1. BOOT  │ 2. ENV   │ 3. FACTORY   │ 4. SLOT_A    │ 5. SLOT_B   │6.DATA│
 │ SPL+Boot │ 双副本   │ 出厂基因     │ 主固件 (OFI) │ 备固件 (OFI)│ 用户 │
 │ 产线固化 │ 启动参数 │ MAC/SN/校准  │ 完整镜像     │ 完整镜像    │ 配置 │
 │ 不进 OTA │ 计数仲裁 │ 物理写保护   │ (A/B 热备)   │ (A/B 热备)  │ 日志 │
 └──────────┴──────────┴──────────────┴──────────────┴─────────────┴──────┘
```

#### (2) 4 类典型硬件拓扑的物理映射矩阵

| 逻辑卷 | 拓扑 1：单 SPI-NOR (64MB) | 拓扑 2：单 SPI-NAND (256MB) | 拓扑 3：混合 NOR (16M) + eMMC (8G) | 拓扑 4：单 eMMC (8G~32G) |
| :--- | :--- | :--- | :--- | :--- |
| **硬件代表** | SSD20X / 全志超低成本网关 | 工业无线路由器 / 监控 IPC | 工业智能网关 / 高可靠工控设备 | RK3568 / 高算力智能边缘 |
| **底层总线** | SPI (MTD 架构) | SPI (MTD/UBI 架构，有坏块) | SPI (NOR) + SDIO/MMC (eMMC) | eMMC 5.1 (标准块设备) |
| **1. BOOT** | NOR 头部 0~1MB | NAND 头部 0~2MB (裸 MTD，多副本冗余) | **NOR 头部 0~2MB (绝不被擦写)** | `mmcblk0boot0`（SoC BootROM 支持从 boot 分区启动时），否则为 user 区头部厂商约定位置 |
| **2. ENV** | NOR 1~2MB (双扇区互备) | NAND 2~4MB (裸 MTD，双副本 + 坏块余量) | **NOR 2~4MB (双扇区互备)** | `mmcblk0boot1` 或 GPT 分区 `env` |
| **3. FACTORY** | NOR 2~3MB (只读保护) | NAND 4~6MB (裸 MTD，双副本 + CRC) | **NOR 4~6MB (物理写保护)** | eMMC 只读 GPT 分区 `factory` |
| **4. SLOT_A** | NOR 3~32MB (29MB) | UBI 设备（NAND 6~256MB）内静态卷 `slot_a`（100MB） | **eMMC GPT 分区 `slot_a` (2GB)** | eMMC GPT 分区 `slot_a` (2GB) |
| **5. SLOT_B** | NOR 32~61MB (29MB) | 同一 UBI 设备内静态卷 `slot_b`（100MB） | **eMMC GPT 分区 `slot_b` (2GB)** | eMMC GPT 分区 `slot_b` (2GB) |
| **6. DATA** | NOR 61~64MB (3MB JFFS2)| 同一 UBI 设备内动态卷 `data`（UBIFS，约 40MB） | **eMMC GPT 分区 `data` (4GB ext4)**| eMMC GPT 分区 `data` (4GB ext4) |

> **NAND 上的 SLOT 与 DATA 必须放在同一个 UBI 设备里**，由 UBI 统一处理坏块、磨损均衡和读干扰（read disturb）后的数据搬移（scrub）。裸 MTD 加“跳坏块”只用于 BOOT / ENV / FACTORY 这类产线一次写入、很少改写的区域，并且分区末尾要预留坏块余量。UBI 设备还需要预留约 2%~3% 的 PEB 作为坏块替换池，实际可用容量比表中略小。

> **工业混合双介质（拓扑 3）高可靠准则**：将 `BOOT`、`ENV` 与 `FACTORY` 固化在寿命极长、无坏块的 SPI-NOR Flash；将体积庞大的操作系统、AI 视觉模型与业务数据放入大容量 eMMC。即使 eMMC 在恶劣断电中文件系统受损，NOR 中的 U-Boot 永远死不掉，设备永远能进入网络/串口救援模式。

#### (3) 物理隔离红线
业务层（OTA 升级、配置中心、系统自省）**严禁直接硬编码任何物理设备路径（如 `/dev/mtd3` 或 `/dev/mmcblk0p2`）**，全系统必须统一通过逻辑卷名称或 `/dev/octopus/by-vol/*` 规范路径寻址。

---

### 4.2 U-Boot 引导存储抽象与 A/B 仲裁驱动 (`cmd_ofi.c`)

U-Boot 是引导 A/B 槽位的第一道防线，通过驱动操作集抽象屏蔽底层介质：

```c
/* U-Boot 存储驱动操作集接口 */
struct ofi_storage_ops {
    const char *name;
    int (*init)(void);
    int (*read_vol)(const char *vol_name, uint32_t offset, size_t size, void *dst_buf);
};

/* 编译期根据单板硬件自动绑定具体后端：
 * - ofi_storage_mtd.c (对应 NOR: mtd_read / NAND: nand_read_skip_bad)
 * - ofi_storage_mmc.c (对应 eMMC: mmc_read / blk_dread)
 * - ofi_storage_ubi.c (对应 UBI 动态卷)
 */
```

- **双副本环境变量区（Redundant Env）**：无论底层介质是 NOR（`CONFIG_ENV_IS_IN_SPI_FLASH`）、NAND（`CONFIG_ENV_IS_IN_NAND`）还是 eMMC（`CONFIG_ENV_IS_IN_MMC`），均开启 `CONFIG_SYS_REDUNDAND_ENVIRONMENT`，规划两个相互独立的擦除块。采用 U-Boot 标准冗余格式（CRC32 + 1 字节 flags/计数），Linux 侧统一用 `libubootenv`（`fw_printenv` / `fw_setenv`）读写，`/etc/fw_env.config` 由板级配置生成，保证两端对格式理解一致；
- **复用 U-Boot 标准启动计数**：启用 `CONFIG_BOOTCOUNT_LIMIT`，变量名沿用 U-Boot 约定的 `bootcount` / `bootlimit` / `upgrade_available` / `altbootcmd`，Octopus 只增加 `boot_slot`（a/b）和 `slot_a_state` / `slot_b_state`（valid / trying / bad）。这样 SWUpdate、RAUC 等工具读到的语义也一致；
- **统一引导命令**：`ofi boot a` 命令解析目标槽位的 OFI 头部、验证签名与段哈希，加载内核至 RAM 执行，并通过 bootargs 传入 `octopus.slot=a`，供 Loader 确定挂载哪个槽位。

---

### 4.3 Linux 内核与 Loader 层的卷发现机制（Volume Discovery）

系统进入 Linux 内核阶段后，由设备树标准标签与 initramfs 动态装配逻辑卷符号链接：

#### (1) 设备树（DTS）与 GPT 标准命名规范
1. **MTD 设备（NOR / NAND）**：在 DTS 的 `partitions` 节点中强制使用标准 `label`：
   ```dts
   partitions {
       compatible = "fixed-partitions";
       #address-cells = <1>;
       #size-cells = <1>;

       partition@0       { label = "boot";      reg = <0x00000000 0x00100000>; };
       partition@100000  { label = "env";       reg = <0x00100000 0x00100000>; };
       partition@200000  { label = "factory";   reg = <0x00200000 0x00100000>; read-only; };
       partition@300000  { label = "slot_a";    reg = <0x00300000 0x01D00000>; };
       partition@2000000 { label = "slot_b";    reg = <0x02000000 0x01D00000>; };
       partition@3D00000 { label = "data";      reg = <0x03D00000 0x00300000>; };
   };
   ```
2. **块设备（eMMC）**：采用标准 GPT 分区表，分区标签（`PARTLABEL`）严格命名为 `slot_a`、`slot_b`、`factory`、`data`。
3. **UBI 卷（NAND）**：UBI 卷名严格命名为 `slot_a`、`slot_b`、`data`；内核需开启 `CONFIG_MTD_UBI_BLOCK`，以便把只读静态卷暴露为块设备供 SquashFS 挂载。

#### (2) `octopus_loader` 动态卷解析与 OverlayFS 合成
在 initramfs 阶段，`octopus_loader` 启动后执行通用的“**卷发现器（Volume Discoverer）**”。initramfs 里没有 udev，`/dev/disk/by-partlabel/` 不存在，只能直接读 sysfs；SquashFS 也只能挂在块设备上，MTD 字符设备和 UBI 卷字符设备都不能直接挂：

```text
 0. 读取 /proc/cmdline 中 U-Boot 传入的 octopus.slot=a|b，确定本次启动的槽位
          │
          ▼
 1. 扫描并转换为可挂载的块设备:
    ├── eMMC: 遍历 /sys/class/block/*/uevent，匹配 PARTNAME=slot_a  → /dev/mmcblk0pN
    ├── NOR : 在 /proc/mtd 中匹配 "slot_a"                         → /dev/mtdblockN（只读）
    └── NAND: ubiattach 后在 /sys/class/ubi/ubi0_*/name 匹配 slot_a
              → ubiblock --create /dev/ubi0_N                      → /dev/ubiblock0_N
          │
          ▼
 2. 映射为统一的符号链接:
    ln -s <块设备> /dev/octopus/by-vol/slot_a
    ln -s <块设备> /dev/octopus/by-vol/slot_b
    ln -s <对应设备> /dev/octopus/by-vol/data
          │
          ▼
 3. 定位 RootFS 段（OFI 头部之后，段按 4KB 对齐存放）:
    读取槽位头部的 OFI 段表，得到 RootFS 段的 offset / size
    losetup -r -o <offset> --sizelimit <size> /dev/loop0 /dev/octopus/by-vol/slot_${SLOT}
    （启用 dm-verity 时，改为在 loop0 之上建立 verity 映射，root hash 取自已验签的 OFI 头部）
          │
          ▼
 4. Loader 统一挂载:
    mount -t squashfs -o ro /dev/loop0 /ro_root
    mount -t tmpfs -o size=<上限>,mode=0755 tmpfs /rw_overlay
    mount -t overlay -o lowerdir=/ro_root,upperdir=/rw_overlay/upper,workdir=/rw_overlay/work overlay /new_root
    mount /dev/octopus/by-vol/data /new_root/data     (失败时见下方降级规则)
    mount --move /dev  /new_root/dev                  (保留 by-vol 符号链接)
    mount --move /proc /new_root/proc; mount --move /sys /new_root/sys
    exec switch_root /new_root /sbin/octopus_supervisor
```
**无论存储介质如何改变，第 2 步之后的挂载逻辑保持一致，介质差异只存在于第 1 步。**

- **tmpfs 必须限额**：OverlayFS 上层写入全部占用 RAM。紧凑型机型建议 `size=4m` 左右，写满时返回 `ENOSPC`，而不是把内存吃光触发 OOM；
- **DATA 挂载失败的降级规则**：`/data` 挂载失败（ext4 日志回放失败、UBIFS 元数据损坏）时，Loader 不得停在 initramfs。先尝试只读挂载并把可读取的配置拷到内存；仍失败则用 tmpfs 顶替 `/data` 继续启动，并在内核命令行和 `/run/octopus/degraded` 中留下标记。管理面启动后告警，提示“数据区损坏，已使用出厂配置运行”。**是否自动重新格式化 DATA 必须是可配置策略**，默认连续 3 次挂载失败后才重建，避免一次偶发错误清空用户数据；
- **需要持久化的系统状态**：OverlayFS 上层重启即丢失。SSH 主机密钥、TLS 证书、随机数种子、最后已知时间戳、`machine-id` 等必须显式放到 `/data/system/` 并通过符号链接或 bind mount 映射回原路径，不能指望写 `/etc` 能保存。

---

### 4.4 OFI (Octopus Firmware Image) 镜像规范

OFI 由“固定头部 + 段描述表 + 签名 + 按 4KB 对齐的段数据”组成，所有多字节字段一律 **小端序**：

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Magic: 'O' 'F' 'I' '!'                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Format Version (0x0001)     |        Header Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Section Count               |  Sig Algo     |  Flags        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Firmware Version (major.minor.patch.build, 4 x uint8)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Security Version (防降级计数，单调递增, uint32)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Board Compatible (如 "rk3568-gw-v1"，32 字节，NUL 填充)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  段描述表，每项 56 字节，共 Section Count 项:                   |
|    Section Type (uint32: Kernel / RootFS / Model / ...)       |
|    Reserved     (uint32，保持 8 字节对齐)                      |
|    Section Offset (uint64，相对镜像起点，4KB 对齐)              |
|    Section Size   (uint64)                                    |
|    SHA-256 of Section (32 字节)                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  dm-verity root hash + salt（可选，对应 RootFS 段）            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Signature（ECDSA-P256 64B / RSA-2048 256B，由 Sig Algo 决定）  |
|  覆盖范围：从 Magic 到 verity 字段结束的全部头部字节              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  段数据（4KB 对齐）...                                         |
```

- **签名只签头部**：头部包含每个段的 SHA-256，签名覆盖整个头部，就等于间接覆盖了全部段。这样可以**先验签头部、再流式写入并逐段校验哈希**，64MB 内存的设备无需把几十 MB 镜像完整缓存到 RAM；
- **兼容性与防降级**：`Board Compatible` 不匹配直接拒绝；`Security Version` 小于设备记录的最小值（存于 OTP / eMMC RPMB / 受保护 ENV）直接拒绝，防止攻击者刷回一个签名合法但存在已知漏洞的旧固件。`Firmware Version` 只用于展示和配置迁移，不作为安全判断依据；
- **段独立性**：Kernel 段（FIT-Image 含 DTB 与 initramfs）、RootFS 段（SquashFS 基础系统）、Model 段（AI 模型层，可选）。段数据按 4KB 对齐，Loader 可以直接用 loop 设备偏移挂载，不需要先解包；
- **增量更新**：OTA 可以只下载变化的段（如只更新 AI 模型权重），但目标槽位写入的必须是完整镜像：未变化的段从当前运行槽拷贝，写完后按新头部逐段校验。

---

### 4.5 A/B 双槽引导仲裁状态机

```text
                      [产线初始烧录完成]
                               │
                               ▼
                       ┌───────────────┐
                       │  SLOT_A_VALID │◄─────────────────────────┐
                       └───────┬───────┘                          │
                               │ 触发 OTA: 标记 B=bad → 写入 B    │
                               │ 校验通过后 upgrade_available=1   │
                               │ 设置 boot_slot=b, bootcount=0    │
                               ▼                                  │
                       ┌───────────────┐                          │
                       │ SLOT_B_TRYING │                          │
                       └───────┬───────┘                          │
                               │                                  │
                 U-Boot 启动   │ bootcount++                      │
                               │ 校验 OFI 签名                    │
                               ▼                                  │
                     [进入 Linux 用户态?]                         │
                    /                    \                        │
             成功  /                      \  启动挂死 / 看门狗复位 /
                  ▼                        ▼ 校验失败 / Panic
        [octopus_app 自检通过]      [U-Boot 检测: bootcount > 3?]
                  │                                │
                  │                                ├─ 否: 重新尝试引导 B
                  │ 健康门槛通过后调用 HAL 确认    │
                  │ upgrade_available=0            ▼ 是: 触发回退
                  │ slot_b_state=valid     ┌───────────────┐
                  ▼                        │ ROLLBACK_TO_A │
           ┌───────────────┐               └───────┬───────┘
           │  SLOT_B_VALID │                       │ boot_slot=a, slot_b_state=bad
           └───────────────┘                       │ 记录回退原因，用户态上报
                                                   └──────────────────────┘
```

> **⚠️ 待修订（见 13.1 第 9 项）**：本节步骤 1「先把目标槽标记为 `bad` 再开始擦写」虽然防止了半截镜像被当作有效槽，但会导致写入窗口内**运行槽成为全机唯一有效槽**。该窗口的并发与掉电语义尚未定稿，Phase 3 实现 `cmd_ofi.c` / `firmware_worker` 前必须先收敛本项。同样，下文 `bootcount > 3` 的判定尚未区分复位原因（见 13.1 第 10 项）。

- **引导计数器（bootcount）保护**：
  - 启动参数配置在 U-Boot 双副本环境变量区；
  - 每次 U-Boot 尝试引导处于 `TRYING` 状态的槽位前，先将 `bootcount` 递增并落盘；
  - 若整机在用户态完全启动并确认前突发掉电或死锁复位，`bootcount` 会持续累加；
  - 一旦 `bootcount > 3`，U-Boot 强制判定该槽损坏，自动回退到稳定槽引导，避免设备陷入无限重启循环。
- **写入顺序（任何一步掉电都可恢复）**：
  1. 先把目标槽位标记为 `bad` 并落盘 ENV，**再开始擦写**。否则目标槽如果原本是回退用的旧版本，写到一半掉电，U-Boot 可能把半截镜像当成有效槽；
  2. 验签头部 → 流式写入 → 逐段回读校验 SHA-256；
  3. 全部校验通过后，一次性写 ENV：`boot_slot=b`、`slot_b_state=trying`、`upgrade_available=1`、`bootcount=0`；
  4. 回退发生时，U-Boot 把失败槽标记为 `bad`，不再尝试，直到下一次 OTA 覆盖它。用户态启动后读取 ENV 发现“刚发生过回退”，要上报云端并记录原因（bootcount 超限、验签失败、看门狗复位）。
- **U-Boot 到用户态之间必须有看门狗覆盖**：“启动挂死 → 看门狗复位 → bootcount 累加”这条回退路径成立的前提是，硬件看门狗在 U-Boot 中就已开启，并一直计时到 `octopus_app` 接管。内核需开启 `CONFIG_WATCHDOG_HANDLE_BOOT_ENABLED`，并用 `watchdog.open_timeout=` 限定用户态必须在多少秒内打开设备。否则内核或 initramfs 卡死时，没有任何机制触发复位，设备会停在半启动状态。
- **确认生效点（Confirmation Point）**：
  - 最低要求：`octopus_app` 正常拉起、核心模块初始化完毕、看门狗线程开始正常调度；
  - **可配置的健康门槛**：对远程部署的设备，仅“进程起来了”不够，新固件可能网络驱动有问题，导致设备再也连不上管理平台。建议默认要求“在 N 分钟内与管理平台完成一次成功通信”，或者通过产品自定义自检项后才确认；超时未满足则主动重启，交由 bootcount 回退；
  - 确认成功后清除 `upgrade_available`、写 `slot_b_state=valid`，并更新防降级计数（若本次固件的 `Security Version` 更高）。**防降级计数只能在确认成功之后更新**，否则新固件启动失败时就再也回退不了旧版本。
- **配置与槽位的联动**：新固件可能在首次启动时迁移配置格式。OTA 切换槽位前，先把当前 `startup-config` 快照为 `startup-config.slot_a`；如果发生回退，旧固件启动时检测到回退标记，优先加载自己的快照，而不是用旧解析器去读新格式配置（宽容解析会把不认识的行丢掉，安全相关配置被静默丢弃是不可接受的）。

---

### 4.6 介质物理容错与 Bootloader 损坏防范

#### (1) SPI-NAND Flash 物理坏块容错与对齐机制（Bad Block Skipping）
- **问题与风险**：SPI-NAND 属于裸 Flash，芯片出厂即允许存在 1%~2% 的物理坏块，且使用寿命期内会产生新坏块；长期只读的数据还会因读干扰和数据保持期衰减出现位翻转。严禁使用 `dd` 直接写 MTD 分区；
- **固件槽位走 UBI**：SLOT_A / SLOT_B 是 UBI 静态卷，OTA 通过 `UBI_IOCVOLUP`（即 `ubiupdatevol` 的机制）整卷写入，坏块替换、磨损均衡、位翻转后的搬移都由 UBI 完成。U-Boot 用 `ubi read` 读取同一个卷，两端看到的是同一套逻辑地址，不存在“跳块拓扑不对称”的问题；
- **裸 MTD 区域的跳坏块写入**：BOOT / ENV / FACTORY 仍是裸 MTD。产线或救援工具写入时调用 `ioctl(fd, MEMGETBADBLOCK)` 检测坏块，**遇到坏块跳过并顺延至下一块**；U-Boot 读取时使用同样的跳块语义（raw NAND 用 `nand_read_skip_bad()`，SPI-NAND 走 MTD 子系统的 `mtd read`，它会自动跳过坏块）。分区末尾必须预留坏块余量，否则运行期新增一个坏块，镜像尾部就被挤出分区。

#### (2) Bootloader 绝对禁止 OTA 的顶级安全准则
- **单点变砖风险**：在绝大多数硬件上，Flash 头部（Sector 0）只存放了一份 SPL / U-Boot，无 A/B 备份。刷写瞬间断电必变砖；
- **Octopus 强制要求**：OFI 固件升级包**严禁携带 Bootloader**。U-Boot 在产线一次性固化，OTA 永远只更新内核与根文件系统（A/B 双槽）。

#### (3) 硬件极端救砖保底
1. **主备 SPL 机制（取决于 SoC）**：若 SoC BootROM 支持多副本搜索，在其约定的位置放两份相互独立的 SPL 拷贝，主 SPL 校验失败时 BootROM 自动尝试备用副本。是否支持、副本放在哪里，必须以芯片手册为准，不同 SoC 差异很大；
2. **硬件救援接口（Maskrom）**：硬件 PCB 必须强制引出 Recovery 按键或特定上拉测试点。当 Flash 内容全损时，按住该键上电可强制 SoC 进入 USB 烧录模式，避免拆芯片救砖。

---

### 4.7 安全启动链（Secure Boot Chain）

1. **信任根（Root of Trust）**：芯片内部 OTP / eFuse 烧录 OEM 公钥哈希，开启硬件强制验签；
2. **二级加载验签**：ROM Code 验证 SPL 签名 -> SPL 验证 U-Boot 镜像签名；
3. **内核与系统验签**：U-Boot 解析 OFI 头部，利用内置公钥验证段校验和及 RSA-2048 / ECDSA 签名后，加载 Kernel + DTB；
4. **根文件系统保护**：Linux 内核通过 `dm-verity` 挂载 SquashFS 只读根文件系统，防止运行时篡改。verity 的 root hash 必须来自已验签的 OFI 头部，由 U-Boot 经内核命令行或 FIT 内的 initramfs 传入；如果 root hash 能被随意修改，dm-verity 就没有意义；
5. **防降级（Anti-Rollback）**：签名只能证明“这是厂商发布的固件”，不能阻止刷回有漏洞的旧版本。设备在 OTP / eFuse / eMMC RPMB 中保存最小可接受的 `Security Version`，U-Boot 和用户态升级服务都要检查。没有这类硬件存储的机型，只能退而记录在 ENV 中，此时需在文档中写明这只能防误操作、防不了物理攻击；
6. **密钥管理**：签名私钥只存放在离线签名机或 HSM 中，CI 通过受控接口请求签名，私钥不进入构建服务器。由于 Bootloader 不进 OTA，U-Boot 出厂时就要内置多把公钥（或公钥哈希），吊销状态记录在 OTP 位中。某把私钥泄露后，用备用密钥签名的新固件通过 OTA 下发并烧写吊销位，设备无需返厂。

---

### 4.8 用户态存储抽象库（`liboctopus_storage`）与 DATA 卷自适应文件系统

#### (1) 统一用户态 C 接口：`octopus_storage.h`
为应用层（固件升级辅进程 `firmware_worker`、配置管理）提供统一接口，彻底消除硬件差异：

```c
/**
 * @file octopus_storage.h
 * @brief Octopus Embedded Storage Abstraction Layer (SAL)
 */

#ifndef OCTOPUS_STORAGE_H
#define OCTOPUS_STORAGE_H

#include <stdint.h>
#include <stddef.h>
#include <sys/types.h>   /* ssize_t */

#ifdef __cplusplus
extern "C" {
#endif

typedef enum {
    OCTOPUS_VOL_BOOT = 0,
    OCTOPUS_VOL_ENV,
    OCTOPUS_VOL_FACTORY,
    OCTOPUS_VOL_SLOT_A,
    OCTOPUS_VOL_SLOT_B,
    OCTOPUS_VOL_DATA,
    OCTOPUS_VOL_MAX
} octopus_vol_id_t;

typedef enum {
    OCTOPUS_STORAGE_RDONLY = 0,
    OCTOPUS_STORAGE_UPDATE = 1   /* 整卷重写，仅允许 SLOT_A / SLOT_B */
} octopus_storage_mode_t;

typedef struct octopus_vol_handle octopus_vol_handle_t;

/* 打开逻辑卷（内部自动识别底层为 MTD、UBI 或 MMC 块设备）。
 * UPDATE 模式必须给出 total_size：UBI 整卷更新要求预先声明字节数，
 * NOR 后端据此预擦除，eMMC 后端据此检查分区容量。 */
octopus_vol_handle_t* octopus_storage_open(octopus_vol_id_t vol_id,
                                           octopus_storage_mode_t mode,
                                           uint64_t total_size);

ssize_t octopus_storage_read(octopus_vol_handle_t *handle, uint64_t offset,
                             void *buf, size_t count);

/* 顺序流式写入（内部完成：裸 NAND 跳坏块、NOR 扇区擦除、eMMC 块对齐） */
ssize_t octopus_storage_write(octopus_vol_handle_t *handle, const void *buf, size_t count);

int     octopus_storage_sync(octopus_vol_handle_t *handle);
void    octopus_storage_close(octopus_vol_handle_t *handle);
uint64_t octopus_storage_get_size(octopus_vol_id_t vol_id);

#ifdef __cplusplus
}
#endif

#endif /* OCTOPUS_STORAGE_H */
```

#### (2) 持久数据区（`DATA` 卷）文件系统自适应选型
`DATA` 卷承载着用户的配置文件（`startup-config`）和系统崩溃黑匣子，要求必须具备强抗突发断电能力：

```text
 ┌────────────────────────────────────────────────────────┐
 │ 统一挂载点: /data (通过 POSIX open/fsync/rename 操作)   │
 └──────────────────────────┬─────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
 【单 NOR Flash (64M)】  【单 NAND Flash】    【eMMC / NVMe】
   容量: 2MB ~ 4MB        容量: 30MB ~ 100MB   容量: 1GB ~ 32GB
   文件系统: JFFS2        文件系统: UBIFS      文件系统: ext4 (data=journal)
   (主线内核、小分区友好)  (磨损均衡、坏块感知)  (高性能、日志预写)
```

- **为什么 NOR 用 JFFS2 而不是 LittleFS**：LittleFS 抗掉电设计很好，但它不在 Linux 主线内核中，只能通过 FUSE 使用，要在 initramfs 阶段挂载 `/data` 还得额外带 FUSE 守护进程，不适合作为系统数据区。3MB 左右的 NOR 分区上 UBI + UBIFS 的固定开销又太大，JFFS2 是主线内核里最实际的选择。它的代价是挂载时间随数据量增长，所以 NOR 机型的 `/data` 只放配置和少量日志；
- **eMMC 本身也会因掉电出错**：文件系统日志只能保护元数据一致性，保护不了 eMMC 控制器内部 FTL 在掉电瞬间的状态。工业机型应选带掉电保护的工业级 eMMC，硬件上建议增加掉电检测（电压监测 GPIO + 储能电容），检测到掉电时立即停止写入并 `sync`；
- **统一写盘红线**：无论底层是 JFFS2、UBIFS 还是 ext4，管理面存盘统一遵守 `tmp -> fsync(file) -> rename -> fsync(dir)` POSIX 原子替换通道，详见 7.4。

---



## 5. 运行环境：Loader 根文件系统与 Supervisor (PID 1)



### 5.1 octopus_loader：根文件系统动态合成

在进入用户态主系统之前，由 initramfs 中的 `octopus_loader` 完成根文件系统装配：

1. **确定槽位**：从 `/proc/cmdline` 读取 `octopus.slot`；
2. **处理启动早期出厂恢复（兜底路径）**：读取 `/proc/cmdline` 中的 `octopus.factory_reset=1`。若存在，**必须在挂载 DATA 之前**按 [reliability 1.2](reliability/00-critical-topics.md) 的清空清单处理用户数据区（保留 FACTORY 分区与产线校准值），并在 `/run/octopus/` 留下 `factory-reset-applied` 标记供用户态上报。这条路径存在的原因：若恰好是某条用户配置导致 `octopus_app` 启动即崩溃，用户态永远走不到按键检测那一步，只有 Loader 能救；
3. **挂载只读底座**：通过 loop 偏移（或 dm-verity）把当前槽位 OFI 中的 RootFS 段挂载至 `/ro_root`；
4. **挂载内存临时层**：将限额的 `tmpfs` 挂载至 `/rw_overlay`；
5. **合成 OverlayFS**：将只读层与可写层联合挂载至 `/new_root`；
6. **挂载持久数据**：将 DATA 卷挂载至 `/new_root/data`，失败时按 4.3 的降级规则处理；
7. **切换根并移交控制权**：`mount --move` 迁移 `/dev`、`/proc`、`/sys` 后执行 `exec switch_root /new_root /sbin/octopus_supervisor`。initramfs 场景只能用 `switch_root`，不能用 `pivot_root`。

具体命令见 4.3 (2)。



### 5.2 PID 1 选型裁决表：为什么自研轻量级 Supervisor


| 候选方案                   | 内存消耗        | 进程监护能力                    | 崩溃自愈与依赖管理                  | 嵌入式适用性评估                                        | 裁决结果     |
| ---------------------- | ----------- | ------------------------- | -------------------------- | ----------------------------------------------- | -------- |
| **systemd**            | 十余 MB ~ 数十 MB（视组件） | 极强，完善的 cgroup / socket 激活 | 完善，但机制极其复杂                 | 强依赖 glibc，组件多、裁剪维护成本高，64MB 机型开销占比过大                | **否决**   |
| **busybox init**       | < 1MB       | 弱，inittab 串行启动 + `respawn`    | 无依赖关系、无就绪通知、无退避与健康检查            | 功能过于原始，无法支撑关键服务崩溃自动隔离与恢复                        | **否决**   |
| **runit / s6**         | 2MB ~ 4MB   | 强，基于文件描述符与管道管理            | 具备进程拉起，但缺少拓扑感知与系统自省集成      | 目录与脚本结构复杂，与内部配置事务、健康上报缺乏原生联动                    | **参考借鉴** |
| **octopus_supervisor** | **< 2MB**   | **专精设计，仅覆盖必要系统职责**        | **声明式 DAG 依赖启动 + 阶梯式退避重启** | **专为嵌入式量身打造，基于纯 C 实现（<2000 行），透明且可控** | **正式采纳** |




### 5.3 octopus_supervisor 的核心职责与边界

1. **收割僵尸进程**：作为 PID 1 循环调用 `waitpid(-1, &status, WNOHANG)`，彻底解决子进程变孤儿后的僵尸资源泄漏；
2. **捕获系统信号**：监听 `SIGTERM`、`SIGINT`、`SIGPWR`，实施两阶段有序关机（先通知各服务保存数据，超时后发送 `SIGKILL` 强杀并卸载文件系统）；
3. **声明式服务监护**：
  - 读取 `/etc/octopus/services.d/` 中的轻量级服务配置文件（内核模块加载也是其中一个服务，不另设 `/etc/init.d` 脚本）；
  - 依据依赖关系（Dependency DAG）并发拉起服务。依赖的判定依据是**就绪**而不是“进程已 fork”：服务就绪时向 supervisor 传入的通知 fd 写一行（与 s6 的 readiness 机制同思路），未就绪超时视为启动失败；
  - 监听子进程退出事件：若核心管理面（`octopus_app`）退出，记录退出码并实施指数退避重启；若短时间内连续崩溃超过阈值，停止重启并记录黑匣子，由硬件看门狗完成整机复位（见下方看门狗归属）。
4. **看门狗归属（与 9.1 一致）**：`/dev/watchdog` 由 `octopus_app` 独占，supervisor 不喂狗。内核开启 `CONFIG_WATCHDOG_NOWAYOUT`，`octopus_app` 崩溃导致 fd 关闭时看门狗不会停止，因此：
  - `octopus_app` 重启 + 初始化到重新打开看门狗的总时间必须小于硬件超时（20 秒），指数退避的上限要按这个预算设置；
  - 超过崩溃阈值后 supervisor 什么都不用做，看门狗超时自然复位整机，复位原因由下一次启动读取。
5. **PID 1 自身的兜底**：PID 1 退出会导致内核 panic。内核命令行必须带 `panic=5`（panic 后 5 秒自动重启，而不是永久挂起），并建议开启 `panic_on_oops=1`；supervisor 自身保持单线程、无动态加载、启动后不再分配内存。
6. **服务日志**：各服务的 stdout / stderr 由 supervisor 接管，写入内存环形日志并按级别转发到 syslog，不允许服务直接往 `/data` 写无限增长的日志文件。

---



## 6. 管理面核心：octopus_app 与三大最小接口契约



### 6.1 octopus_app 定位与内部线程模型

`octopus_app` 是系统的**管理面编排与状态中心**，内部严禁执行重度阻塞 I/O。

```text
 ┌──────────────────────── octopus_app ────────────────────────┐
 │                                                             │
 │  ┌───────────────────────────────────────────────────────┐  │
 │  │ 主事件循环 (Reactor Loop: epoll / libevent)            │  │
 │  │  - CLI 终端命令交互                                   │  │
 │  │  - Web JSON-RPC 请求分发                              │  │
 │  │  - 本机 IPC 请求分发 (UDS)                             │  │
 │  │  - 配置事务提交校验                                   │  │
 │  └──────────────────────────┬────────────────────────────┘  │
 │                             │ 异步派发                      │
 │                             ▼                               │
 │  ┌───────────────────────────────────────────────────────┐  │
 │  │ Worker 线程池 (固定 2~4 个线程)                       │  │
 │  │  - 执行板级 HAL 慢速操作 (LED 切换、读取 EEPROM)       │  │
 │  │  - 配置文件后台序列化与原子落盘                       │  │
 │  │  - 唤醒外部固件升级辅助进程 (firmware_worker)         │  │
 │  └───────────────────────────────────────────────────────┘  │
 │                                                             │
 │  ┌───────────────────────────────────────────────────────┐  │
 │  │ Watchdog 专属巡检线程 (独占高优先级)                   │  │
 │  │  - 收集全模块心跳令牌 (Heartbeat Matrix)             │  │
 │  │  - 巡检内部互斥锁健康状态 (lock_watchdog 锁监护)       │  │
 │  │  - 全员健康时执行 ioctl(wdt_fd, WDIOC_KEEPALIVE) 喂狗 │  │
 │  └───────────────────────────────────────────────────────┘  │
 └─────────────────────────────────────────────────────────────┘
```

---



### 6.2 冻结接口 1：板级硬件抽象 `board_hal.h`

这是系统与具体芯片解耦的物理边界。所有板级移植只需在 `liboctopus_board` 中实现此标准 C 头文件：

```c
/**
 * @file board_hal.h
 * @brief Octopus Embedded Board Hardware Abstraction Layer
 */

#ifndef OCTOPUS_BOARD_HAL_H
#define OCTOPUS_BOARD_HAL_H

#include <stdint.h>
#include <stddef.h>
#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef enum {
    OCTOPUS_LED_SYS = 0,
    OCTOPUS_LED_RUN,
    OCTOPUS_LED_ALARM,
    OCTOPUS_LED_MAX
} octopus_led_id_t;

typedef enum {
    OCTOPUS_LED_OFF = 0,
    OCTOPUS_LED_ON,
    OCTOPUS_LED_BLINK_SLOW,  /* 1 Hz, 50% duty */
    OCTOPUS_LED_BLINK_FAST   /* 5 Hz, 50% duty */
} octopus_led_state_t;

typedef enum {
    OCTOPUS_SLOT_A = 0,
    OCTOPUS_SLOT_B = 1
} octopus_fw_slot_t;

typedef enum {
    OCTOPUS_SLOT_STATE_VALID = 0,
    OCTOPUS_SLOT_STATE_TRYING,
    OCTOPUS_SLOT_STATE_BAD
} octopus_slot_state_t;

typedef enum {
    OCTOPUS_RESET_POWER_ON = 0,  /* Cold boot / power loss */
    OCTOPUS_RESET_WATCHDOG,      /* Hardware watchdog timeout */
    OCTOPUS_RESET_PANIC,         /* Kernel panic (pstore record present) */
    OCTOPUS_RESET_SOFTWARE,      /* Orderly reboot */
    OCTOPUS_RESET_UNKNOWN
} octopus_reset_reason_t;

typedef struct {
    uint8_t base_mac[6];     /* First MAC; interface N uses base_mac + N */
    uint8_t mac_count;       /* Number of consecutive MACs allocated to this board */
    char serial_number[32];  /* Unique factory device serial number */
    char hw_version[16];     /* PCB revision string, e.g., "V1.0.2" */
    char board_name[32];     /* Must match OFI "Board Compatible" */
} octopus_board_info_t;

/* --- Lifecycle & Device Genetic Info --- */
int  octopus_board_init(void);
void octopus_board_deinit(void);
int  octopus_board_get_info(octopus_board_info_t *info);

/* Factory calibration blob by type; returns bytes copied or negative errno.
 * A field re-calibration stored under /data overrides the factory copy. */
int  octopus_board_get_calibration(uint32_t type, void *buf, size_t buf_len);

octopus_reset_reason_t octopus_board_get_reset_reason(void);

/* --- User Feedback & Inputs --- */
int  octopus_board_set_led(octopus_led_id_t led, octopus_led_state_t state);
bool octopus_board_get_reset_button(void); /* Returns true if physically pressed */

/* --- Factory Reset (hold detection, debounce and LED feedback are board timing) ---
 * The 5-second hold decision and the LED progress feedback live in the HAL,
 * because they depend on board-level timing and the LED wiring. The user-space
 * management plane only consumes the verdict; the U-Boot rescue path reads the
 * same pin and passes octopus.factory_reset=1 (see 5.1). */
bool octopus_board_factory_reset_triggered(void);

/* --- Thermal ---
 * Returns 0 on success and writes millidegrees Celsius; negative errno if the
 * board has no reachable sensor. The value feeds the L0~L3 policy in 9.5. */
int  octopus_board_get_temperature_millicelsius(int32_t *out_mc);

/* --- Hardware Watchdog --- */
int  octopus_board_wdt_open(uint32_t timeout_sec);
int  octopus_board_wdt_feed(void);
void octopus_board_wdt_close(void);

/* --- A/B Bootloader & Firmware Arbitration --- */
int  octopus_board_get_active_slot(octopus_fw_slot_t *slot);
int  octopus_board_get_slot_state(octopus_fw_slot_t slot, octopus_slot_state_t *state);
int  octopus_board_mark_slot_bad(octopus_fw_slot_t slot);     /* Before erasing it */
int  octopus_board_set_upgrade_target(octopus_fw_slot_t target_slot);
int  octopus_board_confirm_boot_success(void);
bool octopus_board_rollback_happened(void);                   /* Last boot fell back */

#ifdef __cplusplus
}
#endif

#endif /* OCTOPUS_BOARD_HAL_H */
```

> 本头文件是 Board HAL 的**唯一权威定义**。`reliability/00-critical-topics.md` 与 `guides/02-switch-hal-drivers.md` 中出现的 HAL 片段只是示意，以本节为准。

---



### 6.3 冻结接口 2：模块生命周期与注册契约 `module.h`

`octopus_app` 内部所有业务模块（系统、网络、固件、路由适配）必须遵守此通用契约：

```c
/**
 * @file module.h
 * @brief Octopus Embedded Module Registration & Lifecycle Contract
 */

#ifndef OCTOPUS_MODULE_H
#define OCTOPUS_MODULE_H

#include <stdint.h>
#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef struct octopus_module octopus_module_t;

typedef enum {
    OCTOPUS_MOD_STATE_UNINITIALIZED = 0,
    OCTOPUS_MOD_STATE_INITIALIZED,
    OCTOPUS_MOD_STATE_RUNNING,
    OCTOPUS_MOD_STATE_STOPPED,
    OCTOPUS_MOD_STATE_FAULT
} octopus_module_state_t;

struct octopus_module {
    const char *name;             /* Unique module identifier */
    const char *description;      /* Human-readable description */
    uint32_t   priority;          /* Startup priority (0 = Highest) */

    /* Lifecycle Hooks */
    int  (*init)(octopus_module_t *self);
    int  (*start)(octopus_module_t *self);
    int  (*stop)(octopus_module_t *self);
    int  (*heartbeat_check)(octopus_module_t *self); /* Returns 0 if healthy */

    /* Private Data Pointer */
    void *priv_data;
};

/* Module Registry API */
int  octopus_module_register(octopus_module_t *mod);
int  octopus_module_start_all(void);
int  octopus_module_stop_all(void);
octopus_module_state_t octopus_module_get_state(const char *name);

#ifdef __cplusplus
}
#endif

#endif /* OCTOPUS_MODULE_H */
```

---



### 6.4 冻结接口 3：统一配置模型与事务接口 `config.h`

这是管理面多入口并发写配置的唯一通道，确立 Candidate-Running 事务模型：

```c
/**
 * @file config.h
 * @brief Octopus Embedded Configuration Transaction & Persistence Interface
 */

#ifndef OCTOPUS_CONFIG_H
#define OCTOPUS_CONFIG_H

#include <stddef.h>
#include <stdint.h>
#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef enum {
    OCTOPUS_CFG_OK                 =  0,
    OCTOPUS_CFG_ERR_INVALID_PATH   = -1,
    OCTOPUS_CFG_ERR_INVALID_VAL    = -2,
    OCTOPUS_CFG_ERR_VERIFY_FAILED  = -3,
    OCTOPUS_CFG_ERR_LOCKED         = -4,
    OCTOPUS_CFG_ERR_IO             = -5,
    OCTOPUS_CFG_ERR_CONFLICT       = -6,  /* Running changed since begin() */
    OCTOPUS_CFG_ERR_APPLY_FAILED   = -7,  /* A module failed to apply; rolled back */
    OCTOPUS_CFG_ERR_EXPIRED        = -8,  /* Transaction lease timed out */
    OCTOPUS_CFG_ERR_DENIED         = -9   /* Origin lacks permission for this path */
} octopus_cfg_err_t;

/* Who is changing the config: used for permission checks and the audit log */
typedef struct {
    const char *channel;   /* "cli" / "web" / "cloud" / "local_ui" / "ipc" */
    const char *user;      /* Authenticated user or peer process name */
} octopus_cfg_origin_t;

/* Opaque transaction handle */
typedef struct octopus_cfg_tx octopus_cfg_tx_t;

/* --- Transaction Management (Candidate-Running Model) --- */
/* Snapshot the current running revision. lease_ms bounds how long an idle
 * transaction (e.g. an abandoned Web session) may hold resources. */
octopus_cfg_tx_t* octopus_cfg_begin(const octopus_cfg_origin_t *origin, uint32_t lease_ms);
int  octopus_cfg_set(octopus_cfg_tx_t *tx, const char *json_pointer_path, const char *val_json);
int  octopus_cfg_verify(octopus_cfg_tx_t *tx, char *err_msg_buf, size_t err_msg_len);
/* Fails with ERR_CONFLICT if running revision moved since begin(). */
int  octopus_cfg_commit(octopus_cfg_tx_t *tx);
/* Commit, then auto-revert unless octopus_cfg_confirm() is called within
 * confirm_timeout_s. Required for remote changes that may cut off access. */
int  octopus_cfg_commit_confirmed(octopus_cfg_tx_t *tx, uint32_t confirm_timeout_s);
int  octopus_cfg_confirm(void);
void octopus_cfg_abort(octopus_cfg_tx_t *tx);

/* --- Read & Query Operations --- */
int      octopus_cfg_get(const char *json_pointer_path, char *out_buf, size_t out_buf_len);
uint64_t octopus_cfg_get_revision(void);

/* --- Change Notification (Web / local UI / cloud refresh) --- */
typedef void (*octopus_cfg_notify_fn)(uint64_t new_revision, const char *changed_path, void *ctx);
int  octopus_cfg_subscribe(const char *path_prefix, octopus_cfg_notify_fn fn, void *ctx);

/* --- Persistence Operations --- */
int  octopus_cfg_save_running(const char *target_file);
int  octopus_cfg_load_and_replay(const char *source_file);

#ifdef __cplusplus
}
#endif

#endif /* OCTOPUS_CONFIG_H */
```

---



## 7. 配置中心与并发事务语义



### 7.1 单一真相（Single Source of Truth）内存模型

串口 CLI、Web、云端 JSON-RPC、本机 HDMI/LCD 界面和其它本机进程，全部读写同一份配置，禁止各自落盘。内存中始终维护两颗由读写锁保护的树形状态结构：

- **Running Config**：当前系统正在实际生效的运行态配置；
- **Candidate Config**：正在被修改、校验、尚未提交的候选态配置。



### 7.2 Candidate-Running 两阶段提交事务机制

```text
  [CLI 会话]    [Web]    [云端 RPC]    [HDMI/LCD 本地界面]    [本机 IPC]
       │            │          │                │                  │
       └────────────┴──────────┴────────────────┴──────────────────┘
                                       │ 竞争获取事务锁
                                       ▼
                         ┌───────────────────────────┐
                         │  创建事务: cfg_begin()     │
                         │  克隆一份 Candidate 副本   │
                         └─────────────┬─────────────┘
                                       │
                                       ▼
                         ┌───────────────────────────┐
                         │  批量执行: cfg_set(...)    │
                         └─────────────┬─────────────┘
                                       │
                                       ▼
                         ┌───────────────────────────┐
                         │  事务预校验: cfg_verify()  │
                         │  全模块联动检查参数与冲突  │
                         └─────────────┬─────────────┘
                                      / \
                               校验  /   \ 校验失败
                             通过   /     \
                                   ▼       ▼
           ┌───────────────────────────┐  ┌───────────────────────────┐
           │ 提交事务: cfg_commit()     │  │ 放弃事务: cfg_abort()     │
           │ 1. 检查基线 revision      │  │ 释放 Candidate 副本       │
           │ 2. 按依赖顺序调用模块     │  │ 释放事务锁                │
           │    apply Hook             │  └───────────────────────────┘
           │ 3. 全部成功才替换 Running │
           │    revision + 1，发通知   │
           │ 4. 标记“待落盘”           │
           └───────────────────────────┘
```

- **乐观并发而不是长时间持锁**：`cfg_begin()` 只记录当时的 Running revision，不长期持有全局锁。提交时若 revision 已被别的入口推进，返回 `ERR_CONFLICT`，由调用方重新读取、合并后再提交。这样 Web 页面打开一个编辑框后放着不管，也不会锁死串口和本地屏；闲置事务超过 `lease_ms` 自动失效。真正的临界区只有第 1~3 步，时间很短；
- **应用失败必须回滚**：第 2 步某个模块应用失败（如网卡拒绝新 IP），已应用成功的模块用旧配置重新 apply，Running 保持不变，返回 `ERR_APPLY_FAILED`。不允许出现“内存树已是新值、硬件停在一半”的状态；
- **确认式提交（commit confirmed）**：从云端或 Web 远程修改 IP、路由、防火墙、VLAN 时，一旦配置错误，设备会与管理端失联，而且再也无法远程改回来。这类修改应使用 `cfg_commit_confirmed()`：提交后在限定时间内（如 120 秒）必须收到一次 `cfg_confirm()`，否则自动恢复到提交前的配置。这是远程运维网关必需的能力；
- **审计**：每次提交记录来源通道、用户、revision 与变更路径摘要，写入 `/data/log/audit`，受日志配额约束。


### 7.3 持久化存储格式

持久化格式借鉴网络设备业界成熟的 running-config / startup-config 做法，采用**分层纯文本（Cisco 风格）**，例如：

```text
!version 1.0.4
!hostname octopus-node-01
system
  log-level info
  ntp server 0.pool.ntp.org
!
interface eth0
  ip address 192.168.1.1 255.255.255.0
  dhcp-server enable
!
watchdog
  timeout 20
  feed-interval 5
!
```

- **优势**：
  - 掉电恢复时人类可读，现场串口调试可通过文本直接查错；
  - 易于进行 Git-diff 式版本比对；
  - 解析器支持逐行扫描，对内存占用极低。
- **Schema 单一来源**：CLI 命令树（`.ocli`）、JSON-RPC 路径（JSON Pointer）和上面的文本格式必须由同一份 Schema 生成，不能分别手写，否则 1.3 所说的“多套解析器各写各的”只是从各入口挪到了三种表示之间；
- **敏感字段不能明文**：用户密码只存加盐哈希（如 `sha512-crypt` / `yescrypt`）；Wi-Fi 密钥、VPN 私钥、云端令牌等可逆秘密，使用由 SoC 硬件唯一密钥派生的密钥加密后存储，文本中显示为 `secret enc:<base64>`。`show running-config` 和配置导出默认打码。“人类可读”不等于“秘密可读”。



### 7.4 掉电写入原子性（Power-loss Atomicity）

严禁直接以覆盖写模式打开 `startup-config`。所有配置存盘严格遵守 POSIX 原子替换管道：

```text
 1. 写入临时文件: open("/data/config/startup-config.tmp", O_CREAT|O_WRONLY|O_TRUNC)
    文件末尾追加一行校验尾: "!checksum crc32=<hex> length=<bytes>"
        │
        ▼
 2. 强制物理落盘: fsync(fd)，然后 close(fd)
        │
        ▼
 3. 保留上一版: rename("startup-config", "startup-config.bak")   (首次保存时跳过)
        │
        ▼
 4. POSIX 原子替换: rename("startup-config.tmp", "startup-config")
        │
        ▼
 5. 目录落盘: fsync(open("/data/config", O_RDONLY|O_DIRECTORY))
    (rename 只保证原子性，不保证持久性；不 fsync 目录，掉电后可能回到改名之前)
```

- **加载顺序**：`startup-config` 校验尾正确 → 使用；否则尝试 `startup-config.bak`；两者都坏 → 加载只读根文件系统里的 `default-config`，并告警“配置已回退到出厂值”。没有校验尾，就无法区分“完整的旧文件”和“内容被截断的文件”，也就谈不上选择备份；
- **提交与落盘分离，并做写入合并**：`cfg_commit()` 只改 Running，落盘由后台合并执行，例如“最后一次提交后 2 秒无新提交才写”，并设置每小时写入次数上限。本地屏的亮度滑块、音量旋钮一秒能产生几十次提交，逐次落盘会迅速磨损 NOR/NAND。对“只影响运行态、不需要保存”的字段（如临时调试开关），Schema 中标记为 `volatile`，不参与落盘。



### 7.5 跨固件版本 Schema 迁移

固件从 V1.0 升级至 V2.0 时，由于模块增删字段可能导致参数不兼容。Octopus 确立三条解析准则：

1. **版本元数据标记**：文本第一行必须标明固件生成版本（如 `!version 1.0.4`）；
2. **宽容重放（Tolerant Parsing）**：若新固件遇到已废弃或无法识别的历史语法行，**记录一条 Warning 日志并自动跳过**，绝不允许终止解析或导致开机 Panic；
3. **出厂缺省兜底**：新固件新增的配置项若在历史配置中缺失，自动使用当前版本代码内置的出厂默认值；
4. **显式迁移规则**：字段改名、单位变化（如秒改毫秒）、一拆二这类变化，仅靠“跳过 + 默认值”会悄悄丢掉用户设置。每个模块按 `!version` 提供有序的迁移函数（v1→v2→v3 逐级执行），迁移结果在首次保存时写回；
5. **安全相关行不得静默跳过**：防火墙、访问控制、认证相关的配置行解析失败时，该模块进入“拒绝全部”的安全默认状态并告警，而不是当作不存在，否则一条规则被跳过就可能意味着端口被打开；
6. **降级场景**：A/B 回退到旧固件时，旧解析器面对的是新格式配置。处理方式见 4.5 的“配置与槽位的联动”：优先加载切换前的快照。

---



## 8. 控制通道、外设与扩展子系统



### 8.1 命令行引擎 `octopus_cli`

- **语法 DSL（**`.ocli`**）**：采用轻量声明式 DSL 描述命令树、参数补全与权限级别；
- **实现策略**：`ocli_tool` 主机代码生成器编译生成纯 C 结构体描述表；
- **底层引擎务实绑定**：首选接入成熟轻量的 `linenoise` 或 `libcli` 提供终端驱动、Tab 补全与行编辑能力，避免重新自研 VT100/ANSI 终端控制台，大幅降低研发成本。



### 8.2 Web 接入与安全防护基线

针对工业网关暴露在网络中的安全风险，确立安全基线：

1. **架构解耦**：采用经过工业检验的高安全 HTTP 服务器（如 Hiawatha），仅提供静态页面（HTML/JS）与 FastCGI 代理，前端页面无任何直接执行 shell 的权限；
2. **JSON-RPC 通道保护**：
  - **会话鉴权**：基于安全 HTTP-only Cookie 或 Bearer Token，支持超时自动登出；
  - **防 CSRF**：每个非安全变更请求强制携带 Anti-CSRF 随机令牌；
  - **传输安全**：最低 TLS 1.2，库支持时优先 TLS 1.3，支持设备自签名证书与出厂 CA 预置。TLS 1.3 需要 OpenSSL ≥ 1.1.1 或 mbedTLS 3.x，部分老 BSP 自带的 OpenSSL 达不到，这也是安全组件不能交给厂商 BSP 提供的原因之一（见 [reliability 议题 4](reliability/00-critical-topics.md)）；
3. **无通用默认密码**：ETSI EN 303 645、欧盟 CRA、英国 PSTI 等法规已禁止所有设备共用同一个默认密码。初始密码按设备生成并印在铭牌上（产线写入 FACTORY），或首次登录强制设置。出厂恢复后回到“设备唯一初始密码”，而不是回到 `admin/admin`；
4. **本机 IPC 同样需要鉴权**：`/var/run/json_ipc.socket` 能改全部系统配置。管理面通过 `SO_PEERCRED` 获取对端 uid/pid，按调用方身份做路径级授权：Web 服务进程只能代理已鉴权用户的请求，AI 进程只能读写 `ai.*` 和自身状态，第三方进程默认只读。socket 文件权限设为 `0660` 并归属专用用户组；
5. **最小权限运行**：除 supervisor 与必须操作硬件的进程外，Web 服务、AI 进程、第三方进程均以独立非 root 用户运行，只授予必要的 capabilities（如 `CAP_NET_BIND_SERVICE`）；
6. **调试入口的量产管控**：串口 root shell、`debug memory dump`、`debug i2c/spi` 透传等诊断命令是很大的攻击面。量产固件默认关闭或需要厂商签名的一次性调试令牌才能开启，开启与使用都写入审计日志；
7. **首次启动的熵**：SSH 主机密钥、自签名 TLS 证书若在首次启动时生成，而此时内核随机数池尚未初始化（老内核的 `/dev/urandom` 不会阻塞），生成的密钥可能很弱且多台设备雷同。必须开启 SoC 硬件 RNG（`CONFIG_HW_RANDOM` 对应驱动），用 `getrandom()` 阻塞等待就绪，或者在产线为每台设备预生成密钥；
8. **时间可信**：很多低成本板子没有 RTC 电池，上电时间是 1970 年，TLS 证书校验会失败，日志和崩溃文件名也会错乱。启动时从 `/data/system/last-time` 恢复最后已知时间（只增不减），联网后用 NTP 校准；日志同时记录单调时钟和 `boot_id`，不依赖墙上时间排序。



### 8.3 本机 HDMI / LCD 界面：显示在本地，配置走同一条通道

带屏设备（工业 HMI、机器人面板、网关状态屏）的画面由独立界面进程绘制，配置不允许另存一份。

- **显示留在内核**：HDMI、MIPI-DSI、RGB LCD 由设备树和 DRM/KMS 驱动点亮，背光和触摸归板级设备树。界面进程只消费标准显示设备，不把 GPU 合成器塞进 `octopus_app`；
- **配置与 Web、CLI、云端一致**：亮度、网络、业务参数的读取和修改全部走 JSON-RPC，进入同一套 Candidate-Running 事务。在屏上改完，网页、命令行和云端立刻看到同一份运行配置；从云端下发的变更，本地屏下一次读取或变更通知后同步刷新；
- **界面进程不持有系统配置文件**：禁止在 Qt/LVGL 进程里单独写数据库或 JSON 作为系统配置。界面崩溃由 `octopus_supervisor` 重拉，不影响管理面里已经提交的配置；
- **像素与配置分开**：视频预览、摄像头画面走 DRM dmabuf 或共享内存，不经过配置事务。

### 8.4 动态路由与交换芯片硬件卸载（Switch Offload）

- **路由数据面唯一真实源**：Linux 内核网络协议栈（Kernel FIB）是唯一的真实源；
- **动态路由协议进程**：FRR（zebra/ospfd/bgpd）作为独立第三方进程运行，计算路由后通过标准 Netlink 接口注入 Linux 内核；
- **硬件转发表卸载（**`switch_offload`**）**：
  - 作为 `octopus_app` 的可选插件编译；
  - 通过 Netlink（`RTM_NEWROUTE` / `RTM_DELROUTE`）异步监听内核路由变更；
  - 自动将匹配网段的路由项与二层 FDB 表项下发至以太网交换芯片（DSA / Switchdev），实现线速硬件转发；
  - **解耦优势**：若无交换芯片，系统自然退化为 Linux 内核软转发，上层业务与管理面完全无感。



### 8.5 AI 视觉推理扩展：控制面与数据面彻底分离

```text
  [octopus_app 主管理面]                           [ai_vision_daemon 视觉进程]
  (运动控制 / 参数下发 / 告警处理)                (独占 NPU / V4L2 摄像头)
           │                                                │
           │ 1. 控制面 (UNIX Domain Socket: JSON-RPC)       │
           ├───────────────────────────────────────────────►│  (启停/切换模型/调节阈值)
           │◄───────────────────────────────────────────────┤  (识别事件/坐标/心跳上报)
           │                                                │
           │ 2. 数据面 (POSIX 共享内存 SHM 零拷贝环形队列)   │
           │    [ RingBuffer: 视频帧 / 特征张量元数据 ]     │
           │◄═══════════════════════════════════════════════╡  (高带宽传递原始画面)
```

- **物理隔离保障**：NPU 相关用户态库与厂商 AI 运行时（由设备方按厂商许可另行获取，**不进入本仓库**）仅加载在 `ai_vision_daemon` 独立进程。若该进程发生泄漏或致命 Crash，仅自身退出，由 `octopus_supervisor` 自动重拉；
- **模型更新有两种模式，不能混为一谈**：
  - **随槽位更新（默认）**：模型是 OFI 的 Model 段，跟着 A/B 槽位走，受 U-Boot 回退保护，但更新后需要重启才能切换槽位；
  - **热更新（可选）**：模型包单独签名，存放在 `/data/models/{a,b}/`，由用户态维护一个 A/B 指针。更新时写入非活动目录、验签，然后只重启 `ai_vision_daemon`；新模型加载失败或健康检查不通过，由升级服务把指针切回旧目录。这条路径**不受 U-Boot 保护**，回退完全由用户态负责，而且占用 DATA 空间，NOR/NAND 小容量机型不适用。

---



## 9. 系统级高可靠性、自省与可观测性



### 9.1 硬件看门狗联动心跳矩阵（Heartbeat Matrix）

严禁无脑单线程盲目喂狗。系统实施**全子系统多维心跳巡检**：

```text
 [IPC 处理线程] ──── 心跳打卡 ───► ┌─────────────────────────────────────┐
 [路由监听线程] ──── 心跳打卡 ───► │ 看门狗巡检管理模块                  │
 [主事件循环]   ──── 心跳打卡 ───► │ (Watchdog Runner)                   │
 [驱动轮询线程] ──── 心跳打卡 ───► │                                     │
                                   │ 规则：                             │
                                   │ 1. 只有全部关键线程心跳都在10秒以内 │
                                   │ 2. 且内部无互斥锁死锁挂起           │
                                   └──────────────────┬──────────────────┘
                                                      │ 全员健康才喂狗
                                                      ▼ (每 5 秒执行一次)
                                        [硬件看门狗: /dev/watchdog]
                                        (硬件硬超时设为 20 秒)
```



### 9.2 互斥锁看门狗（Lock Watchdog）与死锁溯源

基于 `liboctopus_basics` 中的 `lock_watchdog` 设施包装系统互斥锁：

- 线程获取关键互斥锁时记录时间戳与线程 ID；
- 若持有锁超过 5 秒，锁看门狗打印告警日志并抓取持锁线程调用栈；
- 若超过 10 秒仍未释放，判定为死锁：先把全部锁状态写入黑匣子，再停止喂狗，由硬件看门狗复位。
- **统一时间线**（全仓库以此为准）：心跳 2 秒一次 → 巡检 5 秒一次 → 任一关键心跳或持锁超过 10 秒判定故障 → 停止喂狗 → 最迟 20 秒后硬件复位。从故障发生到整机复位，最坏约 30 秒。



### 9.3 内存压力阶梯与防 OOM 击穿

在紧凑型网关上建立三级防御。水位按“已用内存 = MemTotal − MemAvailable”计算：

1. **警戒水位（75%）**：通知各模块释放非必要缓存，截断诊断日志内存缓存；
2. **危机水位（88%）**：降级运行，主动暂停 AI 视觉推理进程或停止非必要后台扫描；
3. **OOM 保护机制**：为 `octopus_app` 设置 `oom_score_adj = -1000`，确保系统级 OOM-Killer 触发时，优先清理外围第三方应用，保全主控制面。

- **按内核能力选择检测手段**：PSI（Pressure Stall Information）需要 4.20 以上内核并开启 `CONFIG_PSI`，SSD20X 这类 4.9 内核的 BSP 没有。基线实现轮询 `/proc/meminfo` 的 `MemAvailable`（3.14 起即有）；内核支持时再叠加 PSI 触发器，作为“内存紧张导致卡顿”的更早信号。cgroup 同理：老内核只有 cgroup v1，资源限制写法要分别适配；
- **-1000 的副作用**：`octopus_app` 被设为不可杀后，如果泄漏的恰恰是它自己，OOM-Killer 会反复杀掉其它进程，supervisor 又把它们拉起来，形成循环。因此 `octopus_app` 必须自检：常驻内存超过自身预算上限时主动写黑匣子并受控退出，由 supervisor 重启。**该预算必须在 2.2 中单独给出**（现有 NFR 只规定了「supervisor + app + cli」合计 ≤ 12 MB，无法直接作为自检阈值），并按模块在 9.7 的健康快照中暴露峰值；
- **zram（可选）**：64MB 机型可启用小容量 zram 作为压缩交换区，缓解突发峰值，但它会增加 CPU 开销，并让“内存耗尽”发生得更晚、更隐蔽，默认关闭。



### 9.4 崩溃黑匣子与持久化诊断日志

- **保留内存（pstore / ramoops）**：利用保留的一小块 RAM 区域（默认 1MB，布局见 [reliability 议题 2](reliability/00-critical-topics.md)），在内核 Panic 或重启瞬间记录最后的崩溃堆栈与控制台日志。**它只在热复位时有效，断电后数据消失**。而且看门狗复位时 DDR 是否保持供电、SPL/U-Boot/TF-A 是否会覆盖这块内存，都与具体 SoC 和板子相关，每块板子 bring-up 时必须实测“看门狗复位后 pstore 仍有记录”（列为 3.4 的前提 2）；
- **符号与离线解析**：量产固件是 strip 过的，现场抓到的只有地址。每次构建保留带调试符号的 ELF，并以 GNU build-id 为索引归档；崩溃记录中写入 build-id、各模块加载基址（取自 `/proc/self/maps`）和相对偏移，回到研发侧再离线还原为函数名和行号；
- **持久化诊断配额**：`/data/log/` 严格限制总空间配额（如最大 10MB），采用多文件滚动覆盖（Log Rotation），杜绝因日志暴涨耗尽存储盘空间。

---



### 9.5 温度与热管理：过温不得依赖看门狗

对 7×24 现场运行且带 NPU 的机型，持续推理的发热是必然而非偶发。**过温必须走分级降级，绝不允许靠看门狗复位"解决"** —— 复位既降低不了芯片温度，也定位不了问题，只会形成"重启 → 迅速再升温 → 再重启"的热循环，反而加速器件老化。

温度来源有两个，二者都要用：SoC 内部热敏（`thermal_zone`）覆盖 CPU/GPU/NPU 结温；板级热敏（HAL 接口 `octopus_board_get_temperature_millicelsius()`）覆盖靠近功率器件的板面温度。

| 级别 | 触发条件 | 动作 |
| :--- | :--- | :--- |
| **L0 正常** | 低于预警阈值 | 无 |
| **L1 预警** | 预警阈值（建议器件上限 − 15 °C） | 记录并上报；降低 `ai_vision_daemon` 帧率与推理分辨率；暂停非必要后台扫描 |
| **L2 危险** | 危险阈值（建议器件上限 − 5 °C） | 停止 `ai_vision_daemon`；CPU/GPU 限制到最低可用频点；抑制非关键日志写入；持续告警 |
| **L3 临界** | 超过器件上限 | 先 `sync` 落盘，写黑匣子并标注过温，再受控关机；**由外部条件（降温/断电检修）恢复，不自动重启** |

- **与看门狗的关系**：L1/L2 期间心跳与喂狗照常，系统仍视为健康（温度是工况问题，不是故障）；进入 L3 后按正常优雅关机流程停喂，由硬超时完成最终复位，并保证黑匣子已落盘；
- **阈值必须可配置且按板型标定**：不同 SoC、不同散热方案的阈值不同，禁止在代码中写死；配置项纳入 `octopus_config`（`thermal.*`），标定值在 bring-up 阶段实测填入；
- **禁止掩盖**：任何情况下不得通过"提高告警阈值"或"关闭温度采样"来消除过温告警，此类改动等同于放弃 NFR 的温度指标；
- **验证方式**：高低温箱内按 L0→L3 逐级加热并满载拷机，确认每一级的动作按上表触发、且在 L3 之前设备始终可管理（能登录、能查配置、能上报）。



### 9.6 Flash 写入预算：把"10 年寿命"算出来

器件级擦写寿命（NOR 约 10 万次）是**器件**指标，不等于**系统**寿命。系统寿命由「各功能每日写入量 ÷ 介质可承受写入量」决定，必须在设计阶段对账，不能留到现场才发现写穿。测试手段已在 [测试策略](guides/01-qemu-test-strategy.md) 的长期拷机与 Flash 磨损章节给出（eMMC 读 `EXT_CSD` 寿命字段、UBI 读擦除计数），本节规定**设计侧**的预算要求。

**写入预算表（每个机型一份，随架构文档维护）**：

| 写入源 | 触发频率 | 单次字节 | 日写入量 | 已有约束 |
| :--- | :--- | :--- | :--- | :--- |
| `startup-config` 原子落盘 | 合并后落盘，见 7.4 | 约 4 KB | 待填 | 每小时写入次数上限 |
| 审计日志（`/data/log/audit`） | 每次配置提交 | 约 200 B | 待填 | 受日志总配额约束 |
| 诊断日志滚动 | 持续 | — | 待填 | 总配额 10 MB |
| 崩溃转储 | 崩溃时 | ≤ 10 MB | 视现场 | 触发频率应极低 |
| **AI 模型热更新**（`/data/models/`） | 每次模型变更 | 数 MB ~ 数十 MB | 视业务 | **该源最可能主导寿命，必须单独核算** |

- **通过标准（纳入 NFR 验收）**：按上表加速模拟 10 年累计写入量，**介质擦除计数不得超过器件标称寿命的 50%**（保留 2 倍余量应对写放大与坏块替换）；
- **必须暴露的指标**：UBI 擦除计数 / eMMC `EXT_CSD` 寿命消耗百分比，作为 9.7 健康快照中的只读指标；
- **若预算不通过，按此顺序调整**：先降低日志级别与保留期 → 再对 AI 模型改用"随槽位更新"（8.5 默认模式，不占 DATA 写入）→ 最后才考虑换更大容量或更高耐久介质。**不允许通过延长落盘间隔到影响配置可靠性的程度来凑数**。



### 9.7 统一健康快照（只读）

为了让"系统自省"可被外部观测（Web 状态页、云端巡检、售后取证），管理面暴露一个**只读、低成本、无副作用**的健康快照接口：

```c
typedef struct {
    uint64_t uptime_s;
    octopus_reset_reason_t last_reset_reason;
    uint32_t rollback_count;        /* 历史回退次数，见 4.5 */
    uint8_t  slot_state[2];         /* 与 octopus_slot_state_t 对应 */
    uint32_t temp_mc;               /* 当前温度，毫摄氏度 */
    uint8_t  thermal_level;         /* 见 9.5 的 L0~L3 */
    uint32_t free_bytes;            /* DATA 卷剩余 */
    uint32_t write_budget_pct;      /* 介质寿命已消耗百分比，见 9.6 */
    uint32_t peak_rss_kb;           /* 底座常驻内存峰值，见 2.2 */
    uint8_t  degraded_flags;        /* bit 定义见下 */
} octopus_health_snapshot_t;

int octopus_get_health_snapshot(octopus_health_snapshot_t *out);
```

`degraded_flags` 至少覆盖：`DATA_REBUILT`（DATA 卷被重建过）、`CONFIG_FALLBACK`（配置已回退出厂值）、`DATA_TMPFS`（DATA 挂载失败、以 tmpfs 顶替）、`WDT_ABSENT`（硬件看门狗未能打开，见 3.4 前提 1）、`NO_RTC`、`FACTORY_RESET_APPLIED`。

> 该接口是 3.4 各项前提与 9.5 / 9.6 分级策略的**唯一客观观测面**：没有它，上述策略是否生效无法被验证，也无法被售后与云端巡检使用。它必须是只读查询，禁止承担任何控制职责。



---

## 10. 架构决策与权衡辩论（Architectural RFC & Decision Records / Trade-offs）

本章节记录项目在架构收敛过程中最关键的 7 项**技术决策论证（RFC / ADR）**，阐明“为什么选 A 而不是 B”、“放弃了什么”与“核心权衡考量”，作为开源社区讨论的基石。

### RFC-01: 核心底座开发语言——为什么坚持“轻量 C++”而不是“纯 C 原教旨主义”？

- **争议焦点**：不少嵌入式底层开发者主张全系统必须用纯 C 编写，认为 C 最贴近底层且 ABI 稳定。
- **决策结论**：**“离硬件越近越纯 C，管理面核心用轻量 C++，对外 ABI 边界一律纯 C，上层业务自由多语言。”**
- **权衡论证**：
  1. **管理面反射效率**：高可靠系统底座能够高效支撑海量业务模块的根本优势在于 `StructReadWrite` 模板元编程。一行模板声明即可自动完成 JSON-RPC 树挂载、参数校验、IPC 路由与配置重放。如果退回纯 C，每个结构体都要手写冗长的 `cJSON` 解析与序列化匹配，代码量将暴增 3~5 倍且极易在字段改动时引发段错误；
  2. **并发安全与 RAII**：`octopus_app` 包含多线程锁与动态资源，纯 C 极易发生 `goto out` 漏解锁导致的死锁；C++ 的 `std::lock_guard` / RAII 机制能从语法层面杜绝锁泄漏；
  3. **克制原则（Embedded C++ 准则）**：严禁使用异常（-fno-exceptions）、严禁运行时 RTTI（-fno-rtti）、严禁在主循环中频繁动态申请内存、严禁深层多继承。



### RFC-02: PID 1 路线——为什么自研轻量级 Supervisor 而不直接采用现有开源方案？

- **争议焦点**：是否有必要重新编写一个 PID 1 进程？直接用 systemd、runit 或 s6 不好吗？
- **决策结论**：**自研 <2000 行纯 C 的** `octopus_supervisor`**，坚持极简、透明与受控的工程哲学。**
- **权衡论证**：
  1. **对决 systemd**：systemd 连同 journald、udevd 等配套组件常驻十余 MB 到数十 MB，强依赖 glibc，在 SSD20X（64MB RAM）这类机型上开销占比过高，裁剪后的长期维护成本也高；
  2. **对决 BusyBox init**：BusyBox 只能串行执行脚本，`respawn` 能重拉进程，但没有依赖拓扑、就绪通知和退避策略，无法承担高可用守护重任；
  3. **对决 runit/s6**：runit 和 s6 功能优秀，但其配置目录结构割裂，且无法与嵌入式场景下的硬件看门狗心跳、A/B 槽位确认、`lock_watchdog` 锁死锁与死机转储深度闭环。自研纯 C supervisor 专为工业网关定制，小巧、透明且可控。



### RFC-03: 进程隔离与单点风险——如何兼顾高性能共享与故障域隔离？

- **争议焦点**：传统架构常将所有代码塞进一个巨型单进程，若模块崩溃则管理面全挂；但若全部拆为微服务，进程间 IPC 拷贝与上下文切换开销巨大。
- **决策结论**：**管理面状态内聚（In-Tree）+ 重 I/O 辅进程外挂 + AI/协议重业务独立进程（Out-of-Tree）的双轨架构。**
- **权衡论证**：
  1. **配置与状态管理必须内聚**：系统参数、网络配置、设备信息保持在 `octopus_app` 同一地址空间，消除了成百上千个配置项频繁 cross-process IPC 的开销；
  2. **剥离高危/耗时逻辑**：将容易阻塞的固件 Flash 写入（`firmware_worker`）、日志打包（`diag_collector`）剥离为独立辅进程，按需拉起；
  3. **物理隔离高内存与厂商加速运行时**：AI 视觉进程（加载厂商 NPU 用户态库）与复杂动态路由（FRR）必须是独立进程，其崩溃由 `octopus_supervisor` 毫秒级重启，绝不污染管理面。



### RFC-04: OTA 升级范围——为什么 Bootloader 坚持不进远程 OTA？

- **争议焦点**：部分系统支持整盘包括 Bootloader 在内的全量升级。
- **决策结论**：**远程升级只覆盖内核与根文件系统（OFI），Bootloader 产线固化，不进 OTA。**
- **权衡论证**：
  1. **物理单点风险**：绝大多数嵌入式 SoC（如 RK3568、SSD20X、全志）的 SPL / U-Boot 物理分区只有一份。一旦在擦写 Bootloader 时突发掉电，设备将瞬间变砖，现场只能返厂；
  2. **业务解耦**：产品的业务功能演进、Bug 修复 99.9% 集中在 Linux 内核与应用层，只要 Bootloader 固化了稳定的 A/B 选槽与验签逻辑，根本无需频繁触碰 Bootloader；
  3. **极端保底**：即使两套固件槽全损，PCB 上的硬件 Maskrom 按键仍可触发 ROM Code 救砖。



### RFC-05: 配置中心架构——为什么坚持纯文本分层配置而不用 SQLite 或 JSON？

- **争议焦点**：现代系统偏好使用 SQLite 或整文件 JSON 保存配置。
- **决策结论**：**采用 Cisco 风格的分层纯文本 running-config / startup-config。**
- **权衡论证**：
  1. **现场可救治性（Human-readable Debugging）**：现场工程师通过串口调试时，纯文本配置可直接阅读、修改并排错；若为二进制 SQLite 或复杂的格式化 JSON，在串口或救援 Shell 下排查将极其痛苦；
  2. **原子性与容错性**：纯文本支持宽容重放（Tolerant Parsing），某一行语法废弃直接跳过即可开机；SQLite 库若在突发掉电时发生 journal 损坏，往往导致数据库完全打不开而开机 Panic；
  3. **内存与存储足迹**：纯文本解析器仅需几百行代码，开销几 KB，对超低成本硬件极度友好。



### RFC-06: 网络软硬件协同——为什么以 Linux 内核 FIB 为唯一真实源？

- **争议焦点**：传统网络设备以专用交换芯片私有 SDK 为中心配置转发表，还是以 Linux 内核网络协议栈为中心？
- **决策结论**：**Linux 内核网络协议栈是全系统唯一的真实路由与转发表源头（Single Source of Truth），交换芯片仅作为硬件卸载加速器（Switch Offload）。**
- **权衡论证**：
  1. **统一管理模型**：没有交换芯片时，系统自然作为 Linux 软路由器运行；有交换芯片时，通过 Netlink 监听无缝下发硬件线速转发表；
  2. **生态无缝复用**：FRR、iptables、WireGuard、Docker 等海量开源网络生态均基于标准 Linux 内核网络栈开发，以此为真实源可直接继承全部开源网络资产，无需为某款交换 ASIC 的私有用户态模型改写上层协议。



### RFC-07: 存储架构解耦——为什么坚持 6 大逻辑卷抽象而非直面物理分区？

- **争议焦点**：是否可以直接由业务脚本或 Makefile 按板子硬编码 `/dev/mtdX` 或 `/dev/mmcblk0pX`？
- **决策结论**：**建立存储抽象层（SAL），全系统统一规范 6 大逻辑卷（BOOT/ENV/FACTORY/SLOT_A/SLOT_B/DATA），严禁业务层直接硬编码物理设备。**
- **权衡论证**：
  1. **屏蔽物理介质天然断层**：NOR、NAND 与 eMMC 的擦写单位、坏块特性、磨损均衡与 Linux 访问接口（MTD 字符设备 vs MMC 块设备）完全不同。若不抽象，换一颗存储芯片将导致 U-Boot 引导命令、Loader 挂载、OTA 升级工具全面推倒重写；
  2. **规避供应链断供与快速改版**：商业产品常因芯片缺货或成本优化，在单 NOR (64M)、单 NAND (256M)、NOR+eMMC 双介质之间切换。通过逻辑卷与驱动操作集（Ops）解耦，上层业务与 OTA 逻辑 100% 保持不动，只需调整设备树与存储后端配置；
  3. **读写坏块拓扑严格对称**：在 SPI-NAND 场景下，固件槽位放在 UBI 卷中，用户态写入与 U-Boot 读取访问的是同一套 UBI 逻辑地址；只有 BOOT / ENV / FACTORY 等裸 MTD 区域才由两端按同样规则跳坏块，从根上避免裸 Flash 坏块导致的升级失败。

---



## 11. 术语词汇表（Glossary）与工程路线对齐



### 11.1 术语词汇表


| 术语                   | 英文全称                             | 权威定义与在 Octopus 中的职责                                       |
| -------------------- | -------------------------------- | --------------------------------------------------------- |
| **OFI**              | Octopus Firmware Image           | Octopus 统一段式固件镜像封装格式，含内核、只读根文件系统与可选模型层。                   |
| **SAL**              | Storage Abstraction Layer        | 存储抽象层，屏蔽 NOR/NAND/eMMC 物理差异，向上暴露 6 大标准逻辑卷与统一读写接口。          |
| **Logical Volume**   | Logical Volume                   | 逻辑卷（BOOT/ENV/FACTORY/SLOT_A/SLOT_B/DATA），系统通用寻址单元。         |
| **Board HAL**        | Board Hardware Abstraction Layer | 纯 C 定义的板级接口（`board_hal.h`），隔离 MAC、SN、LED、按键及看门狗硬件差异。      |
| **Supervisor**       | Octopus Process Supervisor       | PID 1 极简进程监护器，负责僵尸进程收割、信号转发与服务依赖生命周期管理。                   |
| **Loader**           | Octopus RootFS Loader            | initramfs 阶段执行的微型工具，负责将 SquashFS 与 tmpfs 联合挂载为 OverlayFS。 |
| **Running-Config**   | Running Configuration            | 系统当前正在内存中生效的配置树，受 Candidate 事务机制与读写锁保护。                   |
| **Startup-Config**   | Startup Configuration            | 保存在持久化存储区（`/data/config/`）的纯文本配置文件，开机时重放恢复状态。             |
| **Heartbeat Matrix** | Watchdog Heartbeat Matrix        | 主进程内部各核心模块向看门狗线程打卡的心跳矩阵，全员健康方可喂硬件看门狗。                     |
| **Lock Watchdog**    | Mutex Lock Watchdog              | 互斥锁监护器（`lock_watchdog`），负责互斥锁持有时长追踪、死锁告警与调用栈溯源。             |




### 11.2 与仓库路线图（ROADMAP.md）里程碑对齐


| 阶段          | 目标版本号          | 本架构规范对应的具体交付物约束                                                                                                            |
| ----------- | -------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **Phase 1** | `v0.2.0-alpha` | 确立 CMake monorepo；基于 `liboctopus_basics` 实现 `board_hal.h` / `module.h` / `config.h` 桩定义；x86 单元测试；单体主程序骨架与基础 JSON-RPC 通信验证；配置解析器与 OFI 解析器接入模糊测试。 |
| **Phase 2** | `v0.3.0-alpha` | 落地 `octopus_config` 事务（revision 冲突检测、apply 回滚、确认式提交）与原子落盘机制；基于 `linenoise` 构建 `octopus_cli`；完成 RK3568 与 SSD20X 的板级 HAL 驱动实装；`liboctopus_client`（C + Python）。 |
| **Phase 3** | `v0.4.0-beta`  | 交付 U-Boot `cmd_ofi.c` 与 A/B 双槽仲裁状态机；`liboctopus_storage`（NOR / UBI / eMMC 三种后端）；`octopus_loader` 与 `octopus_supervisor` (PID 1)；`ofi_pack` 签名工具链与防降级计数；QEMU ARM64 / ARM32 全流程端到端自动化测试。 |
| **Phase 4** | `v1.0.0-rc1`   | Web 安全访问接入（Hiawatha + FastCGI）与本机 IPC 鉴权；本地 HDMI/LCD 界面示例；可选 AI 视觉进程；10,000 次随机突发掉电容灾老化测试；产线烧录与设备唯一凭证写入工具；板级 Bring-up 指南与 GPL 源码交付流程。 |

---

## 12. 许可证边界（License Boundary）

仓库整体采用 Apache-2.0，但 Octopus 有一部分代码必须进入 GPL 项目，二者不能混为一谈：

| 组件 | 所在项目 | 许可证要求 |
| :--- | :--- | :--- |
| `cmd_ofi.c` 等 U-Boot 补丁 | U-Boot（GPL-2.0+） | 必须以 GPL-2.0-or-later 发布。**Apache-2.0 与 GPLv2 不兼容**，Apache-2.0 代码不能直接合入 U-Boot |
| 内核驱动、DTS | Linux（GPL-2.0） | 驱动 GPL-2.0；DTS 建议 `GPL-2.0 OR MIT` 双许可，与上游习惯一致 |
| `octopus_loader`、`octopus_supervisor`、`liboctopus_*`、`octopus_app` | 本仓库用户态代码 | Apache-2.0 |
| U-Boot 与用户态共用的 OFI 解析代码 | 两边都要用 | 建议以 `Apache-2.0 OR GPL-2.0-or-later` 双许可单独成目录，或 U-Boot 侧独立实现 |

- 源码目录中 GPL 部分单独存放（如 `bootloader/`、`kernel/`），每个文件带 SPDX 标识；
- 设备固件中包含 U-Boot、Linux、BusyBox 等 GPL 组件，量产发货时必须履行源码提供义务。构建系统要能为每个发布版本导出对应的 GPL 源码包与 SBOM；
- 厂商以二进制形式交付的 NPU 运行库只能在用户态以独立进程或动态链接方式使用，不能静态链接进 GPL 组件；此类库由设备方按厂商许可自行获取，**本仓库不收录其源码或预编译包**。

---

## 13. 尚未覆盖的议题（Open Issues）

> **阅读提示（开源发布）：** 本章列出的是**尚未定稿**的议题。请勿把第 4–5 章正文中的 A/B 流程读成「已冻结、可直接按此量产实现」。其中 **§13.1** 为 **Phase 3 编码前 Blockers**；§13 正文条目多为产品面 / 治理面，可与底座编码并行讨论，但不构成「永不变砖」已闭环的证明。
>
> 漏洞与安全设计缺陷的报告渠道见仓库根目录 [SECURITY.md](../../SECURITY.md)。

以下问题会直接影响量产，但目前仍没有设计结论，欢迎在 RFC 中讨论：

1. **实时控制的归属**：机器人电机闭环（1ms 级）不应放在 `octopus_app` 这种带 Reactor、RPC 与配置事务的管理进程里。候选方案有三种：独立的实时进程（`SCHED_FIFO` + `mlockall` + CPU 隔离，必要时使用 PREEMPT_RT 内核）、外挂 MCU 协处理器，或者两者结合。无论选哪种，都要定义控制进程或 AI 进程崩溃时电机进入安全状态的机制（硬件急停、驱动器侧独立看门狗）。这属于功能安全范畴，不能只靠 supervisor 重启兜底；具身 / AI 扩展见 `guides/03-ai-robot-vision.md`，**不在阶段 1–3 底座主路径**；
2. **网络连接管理**：蜂窝模组（4G/5G）拨号与复位、多 WAN 切换、Wi-Fi 配网，是网关断连的最大来源，目前只有“网络配置模块”一句话；
3. **远程运维通道**：大多数设备处在 NAT 之后，需要定义设备主动建立的反向运维通道（隧道或云端中继）、鉴权方式和审计；
4. **OTA 下发协议**：设备端下载协议（断点续传、限速、电量/时间窗口检查）、清单格式和灰度发布策略尚未定义；
5. **设备身份与凭证下发**：私钥应存放在安全元件（SE）或 TEE 中，而不是明文放在 FACTORY 分区；没有 SE 的机型采用什么降级方案，需要单独评估；
6. **构建系统与 SBOM**：用户态根文件系统用 Buildroot 还是 Yocto 构建，如何与厂商 BSP 解耦，如何为每个版本产出 SBOM 并跟踪 CVE（欧盟 CRA 的硬性要求）；**Phase 1 交叉编译前建议至少选定其一**；
7. **隐私合规**：摄像头识别人物并上传云端，涉及《个人信息保护法》、GDPR 等，需要设计本地打码、数据保留期和用户授权机制；
8. **仓库治理**：根目录已提供 [SECURITY.md](../../SECURITY.md)（漏洞报告渠道与响应时限）。仍待开源上线后启用 GitHub Discussions、按实际 org/仓库名核对 Issue 模板与 Advisory 链接，并在有维护者公共邮箱后回填到 `CODE_OF_CONDUCT.md`。

### 13.1 Phase 3 编码前 Blockers（必须定稿）

> **BLOCKERS — before Phase 3 coding.**  
> 以下条目**直接影响 4.5 的状态机与「防变砖」论证**。在编写 `cmd_ofi.c` / `octopus_loader` / `firmware_worker` / U-Boot ENV 协议之前，必须通过 RFC 收敛并回写第 4 章正文；**不得**在未决状态下把 4.5 现行叙述当作唯一实现规格。

| 编号 | 议题 | 可选方向（摘要） | 阻塞对象 |
| :--- | :--- | :--- | :--- |
| **B1** (§13.1-9) | 半写入窗口「单一有效槽」 | `upgrade_phase` vs 先标 `bad`；窗口内禁并发 ENV 写；是否保留最小救援 | A/B 写入顺序、永不变砖话术 |
| **B2** (§13.1-10) | bootcount 分类与阈值 | 掉电/外复是否计数；TRYING vs VALID 不同阈值 | 误判完好固件为 `bad` |
| **B3** (§13.1-11) | 救援模式（Rescue Mode）定义 | 进入条件；串口/网/`ofi` 写槽；退出路径 | 「只剩 Bootloader 也能救」闭环 |
| **B4** | OFI 解析许可落地 | `Apache-2.0 OR GPL-2.0-or-later` 单目录 vs U-Boot/用户态双实现 | 第 12 章合规、双份漂移 |
| **B5** | 用户态构建体系 | Buildroot external vs Yocto（与 §13-6 联动） | Phase 1 起交叉编译契约 |

详细待决说明（与上表对应）：

9. **半写入窗口内的「单一有效槽」状态（B1）**：4.5 现规定在开始擦写前先把目标槽标记 `bad` 并落盘 ENV，这在「ENV 落盘」到「新镜像写完并校验通过」之间使**运行槽成为全机唯一有效槽**。若该窗口内发生掉电打断擦除、或运行槽同期损坏，两个槽将同时不可用，而 4.6 与 RFC-04 的「永不变砖」论证只覆盖了 Bootloader 单点，未覆盖此状态。待决：(a) 改以 ENV 中的 `upgrade_phase` 记录升级进行中，目标槽在写入期间保留原状态，由 U-Boot 的镜像校验（而非提前写 `bad`）承担「启动不了」的表达；(b) 该窗口内禁止并发升级与其他 ENV 写入；(c) 是否需要在该窗口保留最小救援能力；
10. **bootcount 的分类计数与阈值判定（B2）**：4.5 现规定 `bootcount > 3` 即判定该槽损坏，不区分复位原因。掉电与外部复位反映的是电网而非固件缺陷，累积若干次会把一个完好固件判死。待决：按 `octopus_board_get_reset_reason()` 分类计入（掉电/外部复位不计或降权）、已验证槽与试运行槽采用不同阈值、`VALID` 槽不因 bootcount 被判 `bad`；
11. **救援模式（Rescue Mode）的定义（B3）**：4.6 与 RFC-04 反复依赖「只剩 Bootloader 也能恢复」，但未定义该状态的进入条件、最小能力（串口 / 网络 / `ofi` 写槽 / 配置恢复）、界面与退出路径。缺此定义，「永不变砖」的论证不闭环；
12. **内核 LTS 维护路径**：2.2 允许最低内核 4.9（厂商 BSP 现状），但 10 年生命周期内的安全补丁获取方式未定（跟 BSP 长期打补丁 / 逐步主线化 / 双轨回移），这直接决定 CVE 响应能力；
13. **ABI 演进流程**：6.2 / 6.3 / 6.4 三份冻结接口在需要新增字段或改变语义时的流程未定（版本化结构体 / 仅允许新增函数 / 小版本允许破坏性变更）。冻结接口一旦被打破，「零迁移成本」即失效，该流程本身即承诺的一部分；
14. **ADR / RFC 决策流程**：第 10 章有 7 篇 RFC，但未定义批准人、意见冲突的解决方式与 ADR 状态标记（proposed / accepted / superseded / deprecated）。开源后社区第一个问题必然涉及此点；
15. **验收判据表**：2.2 的 NFR 均有指标但无判定方法（何谓「变砖」、样本量、批量抽样规则），且缺少「模拟 10 年写入量」的 Flash 寿命验收（见 9.6）。需求追踪矩阵与风险登记册同样缺失；
16. **产线流程与供应链**：烧录、密钥注入、单板测试、不良品返修、出货固件追溯，以及 NAND 停产、工业级 eMMC 掉电保护等级等长期供货风险。第 13 章现有条目 5 只覆盖了「设备身份凭证下发」；
17. **易失路径清单**：4.3 为 OverlayFS 上层的 tmpfs 设了限额（建议 4m），但未规定 `/etc`、`/var`、`/tmp` 各自归属与配额。第三方组件往这些路径写是常态，写满即 `ENOSPC`。需要一份「易失路径 + 限额分配」表，而非举例说明。


