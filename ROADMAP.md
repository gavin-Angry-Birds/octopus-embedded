# Octopus Embedded — Roadmap

**English** | [简体中文](#简体中文)

This roadmap tracks public delivery milestones. Tags follow [Semantic Versioning](https://semver.org/) where applicable.

---

## Phase 0 — Architecture draft (current)

**Target tag:** `v0.1.0-design`

- [x] Publish the architecture draft for public discussion
- [x] Establish brand: **Octopus Embedded**
- [x] Document 7 production-critical topics (watchdog, OOM, storage, diagnostics, …)
- [x] Logo & visual identity (`assets/logo.svg`, `assets/banner.svg`, `assets/icon.svg`)
- [x] Add `SECURITY.md` (vulnerability reporting channel and response times)
- [x] Add `CHANGELOG.md` and document language policy (Chinese authoritative + `*.en.md` twins for all design docs)
- [x] GitHub public repository live
- [x] Enable GitHub Discussions for RFCs
- [x] Point Issue-template / Security-advisory / CHANGELOG URLs at `gavin-Angry-Birds/octopus-embedded`
- [x] Verify Discussions and private vulnerability reporting are enabled after first push
- [x] Interim v0.x decisions for **Phase 3 blockers** recorded in architecture §13.1.0 (ADR-0001); formal closure deferred to Phase 3
- [ ] Continue RFC discussion on other chapter 13 open issues (real-time control ownership, network/cellular, remote access, OTA download protocol, …)

**Deliverable:** This repository (`docs/` only, no application source yet).

---

## Phase 0.5 — QEMU A/B OTA minimal demo

**Target tag:** `v0.2.0-demo`
**Goal:** prove the §4.5 A/B rollback design with a runnable, reproducible demo before building the full platform.
**Interim decisions:** see architecture §13.1.0 (ADR-0001).

- [ ] Buildroot `BR2_EXTERNAL` tree + `board/qemu-arm64/` (start from `qemu_aarch64_virt_defconfig`)
- [ ] genimage disk layout: ENV / SLOT_A / SLOT_B / DATA
- [ ] U-Boot boot script: select slot from ENV `boot_slot`, increment `bootcount` for `TRYING` slots, fall back after `bootcount > 3`
- [ ] `scripts/run-qemu.sh` — one-command boot
- [ ] `octopus-ota` (C): install (mark `bad` → write → verify SHA-256 → set `trying`) and `confirm`; called from an init script (BusyBox init is PID 1 for now)
- [ ] Power-cut test harness (Python): kill QEMU at a random point during upgrade, reboot, check bootable + expected version; report
- [ ] GitHub Actions: build + short power-cut run (50 cycles) on every push
- [ ] README: demo GIF + test results

**Done when:** 1000 random power-cut cycles in QEMU with 0 unbootable outcomes, reproducible with one command.

> Phase 1 (`liboctopus_basics`, JSON-RPC, …) target tag moves to `v0.3.0-alpha`; later phases shift by one minor version. The custom PID 1 supervisor is planned after this demo.

---

## Phase 1 — Core skeleton

**Target tag:** `v0.2.0-alpha`

- [ ] New `octopus-core` layout inside this repo (CMake monorepo)
- [ ] Extract `liboctopus_basics` (OSAL, lock_watchdog, trace, dual-track alloc stats)
- [ ] PC x86 native build + GoogleTest unit tests (Layer 1 pyramid)
- [ ] libFuzzer targets for the OFI header, config text, and JSON-RPC parsers
- [ ] Minimal `octopus_app` main process stub
- [ ] JSON-RPC skeleton + `/var/run/json_ipc.socket` proof-of-concept

---

## Phase 2 — Management plane & Board HAL

**Target tag:** `v0.3.0-alpha`

- [ ] octopus_cli engine + `.ocli` compile-time code generator
- [ ] octopus_config running-config serialize / replay
- [ ] Config transactions: revision conflict detection, apply rollback, confirmed commit, change notifications
- [ ] `liboctopus_board` — MAC, SN, LED, factory-reset button (RK3568 / SSD20X)
- [ ] Out-of-tree SDK: `liboctopus_client` (C + Python)

---

## Phase 3 — Boot resilience & firmware

**Target tag:** `v0.4.0-beta`

- [ ] U-Boot `cmd_ofi.c` port (RK3568 + SSD20X parity, GPL-2.0-or-later)
- [ ] A/B firmware arbitration + boot-count rollback
- [ ] `liboctopus_storage` backends: NOR (MTD), SPI-NAND (UBI), eMMC
- [ ] `ofi_pack` signing via offline key / HSM, security version (anti-rollback)
- [ ] `octopus_loader` + `octopus_supervisor` PID 1 + OverlayFS rootfs
- [ ] Layer 2: QEMU ARM64 and ARM32 virt full boot path, nandsim power-cut tests

---

## Phase 4 — Integration & hardening

**Target tag:** `v1.0.0-rc1`

- [ ] Web (Hiawatha + FastCGI) + unified CLI/Web/IPC config
- [ ] Local IPC authorization (`SO_PEERCRED`) and non-root service accounts
- [ ] Optional local HDMI/LCD UI sample on the shared config
- [ ] Optional `ai_vision_daemon` + `model.squashfs` hot upgrade
- [ ] Layer 3: 10,000-cycle power-loss stress test
- [ ] Factory provisioning tool (per-device initial password, keys, MAC/SN)
- [ ] GPL source bundle + SBOM generation per release
- [ ] Production documentation & board bring-up guides

---

## Version history (releases)

| Tag | Date | Notes |
|-----|------|-------|
| `v0.1.0-design` | TBD | Initial public architecture release |

---

## 简体中文

### 阶段 0 — 架构草案（当前）

**目标标签：** `v0.1.0-design`

- [x] 发布架构草案，供公开讨论
- [x] 确立品牌：**Octopus Embedded**
- [x] 七大量产关键议题文档化
- [x] Logo 与视觉规范 (`assets/logo.svg`, `assets/banner.svg`, `assets/icon.svg`)
- [x] 补充 `SECURITY.md`（漏洞报告渠道与响应时限）
- [x] 补充 `CHANGELOG.md`，并写明中文为规范权威源
- [ ] GitHub 公开仓库上线
- [ ] 开启 GitHub Discussions 收 RFC
- [x] Issue 模板 / Security Advisory / CHANGELOG 链接已指向 `gavin-Angry-Birds/octopus-embedded`
- [ ] 首次推送后启用 Discussions 与私有漏洞报告（Private vulnerability reporting）
- [x] §13.1 Blockers 已形成 v0.x 临时决议（架构规范 §13.1.0，ADR-0001），正式收敛留待 Phase 3
- [ ] 继续讨论第 13 章其余开放议题（实时控制归属、网络/蜂窝、远程运维、OTA 下发协议等）

### 阶段 0.5 — QEMU A/B OTA 最小演示

**目标标签：** `v0.2.0-demo` — Buildroot 外部树 + QEMU arm64、ENV/SLOT_A/SLOT_B/DATA 分区、U-Boot 按 ENV 选槽与 bootcount 回退、`octopus-ota` 安装/确认、随机断电测试脚本与 CI。**完成标准：** QEMU 中 1000 次随机断电，0 次无法启动，一条命令可复现。临时决议见架构规范 §13.1.0（ADR-0001）。阶段 1 顺延为 `v0.3.0-alpha`，自研 PID 1 监视进程排在本演示之后。

### 阶段 1 — 核心骨架

**目标标签：** `v0.2.0-alpha` — `liboctopus_basics`、CMake、x86 单元测试、解析器模糊测试、JSON-RPC 雏形。

### 阶段 2 — 管理面与 Board HAL

**目标标签：** `v0.3.0-alpha` — octopus_cli、octopus_config（冲突检测、回滚、确认式提交）、板级 HAL、对外 SDK。

### 阶段 3 — 引导与固件容灾

**目标标签：** `v0.4.0-beta` — U-Boot OFI（GPL-2.0）、A/B 双固件、`liboctopus_storage`、固件签名与防降级、octopus_loader / octopus_supervisor、QEMU ARM64 / ARM32 全流程。

### 阶段 4 — 集成与量产压测

**目标标签：** `v1.0.0-rc1` — Web 联调与本机 IPC 鉴权、本地屏示例、AI 视觉、万次掉电压测、产线凭证写入工具、GPL 源码包与 SBOM。
