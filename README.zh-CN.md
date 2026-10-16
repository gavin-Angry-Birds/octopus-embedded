![Octopus Embedded Banner](assets/banner.png)

[English](README.md) | **简体中文**

![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)![Status](https://img.shields.io/badge/阶段-0%20架构草案-orange.svg)![Platform](https://img.shields.io/badge/平台-Embedded%20Linux-green.svg)![Docs](https://img.shields.io/badge/文档-架构索引与导航-teal.svg)

# Octopus Embedded

> 面向工业网关、长期在线设备和边缘视觉类产品的跨 SoC 嵌入式 Linux 软件底座。

Octopus Embedded 是一套开源架构与后续代码仓的设计起点，面向需要长期无人值守运行的现场设备。首批适配目标包括瑞芯微 RK3568、星宸 SSD20X、全志等；架构按同档 ARM 应用处理器可扩展来写。关注点集中在：固件升级可回退、进程监护、统一管理面、存储抽象，以及业务模块的边界。

当前处于阶段 0：公开的是架构草案与 RFC，源码从阶段 1 开始写。进度见 [ROADMAP.md](ROADMAP.md)。

### 架构总览

下图为整机分层与存储逻辑卷的一页总览。矢量源文件见 `[assets/architecture-overview.svg](assets/architecture-overview.svg)`。

![Octopus Embedded Architecture Overview](assets/architecture-overview.png)

自上而下：客户端（Web / 屏 / CLI / 云，以及可选视觉进程）→ 管理面 `octopus_app`（统一配置与诊断）→ 运行时（`octopus_loader` + `octopus_supervisor`）→ 内核与薄 Board HAL。底部六卷是跨介质的存储抽象；固件槽走 OFI，与 `DATA` 分离。

---



## 为什么需要 Octopus Embedded？

做嵌入式 Linux 量产时，下面这类问题会反复出现。表中第三列是本项目打算怎么处理（设计阶段，尚未全部落地成代码）。

文中会用到「固件槽位 / 双槽 / A/B 仲裁 / OFI」等说法：大致是 Flash 上放两份可启动系统、升级写空闲那一份、启动失败则回退；OFI 是统一的固件包格式。详细图解见后文[关键术语](#关键术语)。


| 常见问题                           | 现场或研发后果                                           | 本项目的对应设计                                            |
| ------------------------------ | ------------------------------------------------- | --------------------------------------------------- |
| 芯片 SDK 绑死全生命周期                 | 换 SoC（如 RK3568 → SSD20X）后，烧录工具、打包格式、选槽启动脚本往往整套重写  | 统一 OFI 格式与双槽状态机；芯片差异尽量压在设备树和薄 Board HAL             |
| 换 Flash 介质就打穿 OTA              | NOR / SPI-NAND / eMMC 换料后，坏块、MTD 与块设备差异导致写入与引导对不上 | 存储抽象层（SAL）：六类逻辑卷；SPI-NAND 上固件槽走 UBI；数据区按介质选文件系统     |
| CLI / Web / 云 / 本地屏各写各的        | 串口改过的值屏上看不到；屏上保存盖掉云端；断电写配置文件截断                    | 一份 Candidate–Running 配置，多入口共用；冲突检测与确认提交；文本配置 + 原子落盘 |
| PID 1 两极：systemd 太重，BusyBox 太弱 | 小内存网关上 systemd 难裁；BusyBox 缺依赖顺序、就绪与退避             | 自研轻量 `octopus_supervisor`（目标：纯 C、体积小，带依赖启动与退避重启）    |
| 现场死机后无现场                       | 看门狗复位后日志空；Flash 放不下 core dump；泄漏和死锁难定位            | 锁监护、按模块的堆用量统计、调试版越界校验、崩溃栈写入 RAMoops/Flash           |
| 掉电损坏可写根文件系统                    | 元数据坏掉后无法挂载，只能返厂                                   | 只读 SquashFS + Overlay；持久数据与根分离                      |
| 运动控制与 NPU 推理同进程                | NPU 用户态库故障拖垮整机；高清帧走 Socket 拷贝打满 CPU               | 控制面 JSON-RPC；图像/张量走共享内存；视觉进程独立、由 supervisor 拉起      |
| 套用 SWUpdate / RAUC 仍缺一截        | 框架假定 Bootloader 已会选槽与坏块处理，也不管配置迁移和进程编排            | U-Boot 与用户态对称实现选槽/坏块语义；升级、配置、监护放在同一套底座里协同           |


---



## 适用场景

- 工业或长期在线网关：路由、双槽 OTA、现场诊断
- 带本地屏的设备：屏、Web、CLI、云端共用一份配置
- 边缘视觉 / 机器人类产品：主进程与可选 NPU 视觉进程分离
- 多芯片产品线：业务与生命周期规范共用，工具链按板型切换

---



## 关键术语

下面四个词在架构文档里会出现很多次，指的是同一件事：系统镜像怎么放在 Flash 上，以及上电或升级后启动哪一份。

### 图 1：Flash 布局

```text
  +---------------+  +---------------------+ +---------------------+
  |  1. BOOT      |  |  4. SLOT_A          | |  5. SLOT_B          |
  |  SPL + U-Boot |  |  firmware slot A    | |  firmware slot B    |
  +---------------+  +---------------------+ +---------------------+
                     |  kernel + rootfs    | |  kernel + rootfs    |
                     +---------------------+ +---------------------+

  +---------------+  +---------------------------------+
  |  2. ENV       |  |  6. DATA                        |
  |  boot_slot    |  |  user config + logs             |
  |  slot states  |  +---------------------------------+
  |  bootcount    |  +-----------------+
  +---------------+  |  3. FACTORY     |
                     |  MAC / SN       |
                     +-----------------+
```


| 块                         | 作用                               | 远程升级是否改写        |
| ------------------------- | -------------------------------- | --------------- |
| `1. BOOT`                 | SPL + U-Boot，产线烧一次               | 否               |
| `2. ENV`                  | 启动参数（双副本）：当前槽、槽状态、连续启动失败次数       | 升级过程会读写         |
| `3. FACTORY`              | MAC、序列号、产线校准；宜写保护                | 否               |
| `4. SLOT_A` / `5. SLOT_B` | 各放一份完整可启动系统（内核 + 根文件系统），包格式为 OFI | 写入当前未运行的槽       |
| `6. DATA`                 | 配置与日志；不属于任一槽                     | 升级不覆盖；出厂恢复主要清这里 |


`4` / `5` 是同格式的两份系统；`1`–`3` 属引导链；`6` 与固件版本脱钩。图中框线只用 ASCII，避免中文字宽把对齐弄乱。

### 图 2：双槽升级与回退

```text
  normal operation
  +--------------------------+   +--------------------------+
  |  SLOT_A                  |   |  SLOT_B                  |
  |  system image 1          |   |  system image 2          |
  |  [ RUNNING ]             |   |  [ STANDBY ]             |
  +--------------------------+   +--------------------------+
               |
               |  OTA: write new image into the slot that is NOT running
               v
  first boot after OTA
  +--------------------------+   +--------------------------+
  |  SLOT_A                  |   |  SLOT_B                  |
  |  old version             |   |  new version             |
  |  [ still bootable ]      |   |  [ TRYING ]              |
  +--------------------------+   +--------------------------+
               |                              |
        boot / self-check ok             boot / self-check fail
               v                              v
     mark B VALID, next boot B         bootloader returns to A
```

`RUNNING` 运行中；`STANDBY` 备用；`TRYING` 试运行未确认。下次从哪槽启动记在 `ENV`。


| 词      | 含义                                                           |
| ------ | ------------------------------------------------------------ |
| 固件槽位   | `SLOT_A` 或 `SLOT_B`：一整份可启动系统所在区域                             |
| 双槽     | 两份对等槽，同时只跑一个；升级写空闲槽，失败则回到旧槽                                  |
| A/B 仲裁 | 按 `ENV` 里的槽状态与启动失败次数决定启动 A 还是 B。原厂 SDK 常写成芯片专用脚本；本项目收成统一状态机  |
| OFI    | Octopus Firmware Image（`.ofi`）：统一的段式固件包（头、签名、内核/根文件系统/可选模型段） |


容易混的两点：

1. 「槽」只指系统固件，不是 `DATA`。配置和日志在 `DATA`，升级换槽默认不覆盖。只读根 + Overlay 的机型上，要持久保存的文件须放 `/data`，写 `/etc` 重启即丢。
2. 双槽不是两套系统同时跑；空闲槽只是可启动的备份。

状态机里仍有未定稿项（例如半写入窗口、救援模式），见架构文档 [§13.1](docs/architecture/00-platform-design.md#131-phase-3-编码前-blockers必须定稿)。

---



## 文档

入口：[docs/README.md](docs/README.md)。设计文档均为 **中文正文 + `.en.md` 英文对照**；架构意图以中文为准，冲突时以中文为准。

| 领域 | 中文 | English | 说明 |
|------|------|---------|------|
| 核心架构 | [00-platform-design.md](docs/architecture/00-platform-design.md) | [`.en.md`](docs/architecture/00-platform-design.en.md) | 分层、状态机、HAL、RFC |
| 高可靠与量产 | [00-critical-topics.md](docs/reliability/00-critical-topics.md) | [`.en.md`](docs/reliability/00-critical-topics.en.md) | 看门狗、OOM、存储、诊断 |
| 语言选型 | [00-language-cpp-guidelines.md](docs/guides/00-language-cpp-guidelines.md) | [`.en.md`](docs/guides/00-language-cpp-guidelines.en.md) | C / C++ 边界 |
| 测试 | [01-qemu-test-strategy.md](docs/guides/01-qemu-test-strategy.md) | [`.en.md`](docs/guides/01-qemu-test-strategy.en.md) | x86 / QEMU / 真机掉电 |
| 交换与驱动 | [02-switch-hal-drivers.md](docs/guides/02-switch-hal-drivers.md) | [`.en.md`](docs/guides/02-switch-hal-drivers.en.md) | HAL / 可选交换卸载 |
| 视觉与机器人（远期） | [03-ai-robot-vision.md](docs/guides/03-ai-robot-vision.md) | [`.en.md`](docs/guides/03-ai-robot-vision.en.md) | 不在阶段 1–3 主路径 |
| 品牌 | [branding.md](docs/project/branding.md) | （单文件中英双语） | 命名与 Logo |


---



## 路线图

详见 [ROADMAP.md](ROADMAP.md)。


| 阶段  | 重点                                                | 标签              | 状态       |
| --- | ------------------------------------------------- | --------------- | -------- |
| 0   | 架构草案与 RFC（本仓库文档）                                  | `v0.1.0-design` | 已发布，开放讨论 |
| 0.5 | QEMU A/B OTA 最小演示（断电测试 + CI）                 | `v0.2.0-demo`   | 下一步      |
| 1   | `liboctopus_basics`、CMake、单测、解析器 fuzz、JSON-RPC 雏形 | `v0.3.0-alpha`  | 计划中      |
| 2   | CLI、配置事务、Board HAL、对外 SDK                         | `v0.4.0-alpha`  | 计划中      |
| 3   | U-Boot OFI、supervisor、双槽固件                        | `v0.5.0-beta`   | 计划中      |
| 4   | 集成、可选视觉、压测                                        | `v1.0.0-rc1`    | 计划中      |


---



## 参与贡献

设计阶段欢迎 RFC、文档修正和 Issue。请先读 [CONTRIBUTING.md](CONTRIBUTING.md)。

架构意图以简体中文（本文件与 `docs/`）为准；改结论时请同步英文入口。阶段 3 编码前必须收敛的议题见 [§13.1](docs/architecture/00-platform-design.md#131-phase-3-编码前-blockers必须定稿)。

---



## 安全

报告方式见 [SECURITY.md](SECURITY.md)。不要在公开 Issue 里贴可利用细节。

---



## 许可证

[Apache License 2.0](LICENSE)。

---



## 免责声明

Octopus Embedded 是独立开源项目，与 [Octopus Deploy](https://octopus.com) 及其他「Octopus」商标持有者无隶属关系。本仓库目前只有原创架构文档，不包含、不分发第三方专有/闭源源码或预编译包。文中提到的厂商 SDK、NPU 运行时、启动固件等只作集成边界说明，须由设备方按厂商许可自行获取。本文是架构草案，A/B 与救援等关键状态机仍有公开未决项（见架构第 13 章），不表示已经过量产验证。