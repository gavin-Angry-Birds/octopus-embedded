<p align="center">
  <img src="assets/banner.svg" alt="Octopus Embedded Banner" width="100%" style="max-width: 840px; border-radius: 12px;" />
</p>

<p align="center">
  <strong>English</strong> | <a href="README.zh-CN.md">简体中文</a>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-Apache%202.0-blue.svg" alt="License"></a>
  <img src="https://img.shields.io/badge/Phase-0%20Architecture%20draft-orange.svg" alt="Status">
  <img src="https://img.shields.io/badge/Platform-Embedded%20Linux-green.svg" alt="Platform">
  <a href="docs/README.md"><img src="https://img.shields.io/badge/Docs-Architecture%20Index-teal.svg" alt="Docs"></a>
</p>

# Octopus Embedded

> Cross-SoC high-reliability **embedded Linux** software platform for industrial gateways, mission-critical networked devices, and AI edge robotics.

**Octopus Embedded** is an open-source embedded platform built for **7×24 unattended field operation**. Initial bring-up targets Rockchip RK3568, SigmaStar SSD20X, Allwinner, and similar SoCs; the architecture is meant to extend to most comparable ARM application processors used in industrial gateways and edge computing. It provides a unified approach to anti-brick firmware, process supervision, a single management plane, storage abstraction, and modular business extension.

> **Current phase:** The architecture specification and core RFCs are public and open for discussion. **Source coding starts in Phase 1** — see [ROADMAP.md](ROADMAP.md).
>
> **Language policy:** Simplified Chinese (`README.zh-CN.md` and the architecture docs under `docs/`) is the source of truth. This English README is kept in sync with that Chinese text. Detailed design documents under `docs/` are currently Chinese-first.

---

## Glossary

These four terms recur throughout this document. They describe one practice: how the system image is stored on the device, and which copy boots after power-on or after an upgrade. Read the diagrams first, then the definitions.

### Figure 1: Flash physical layout

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

**What each block does**

| Block | Role | Touched by remote upgrade? |
|:---|:---|:---|
| `1. BOOT` | Bootloader (SPL + U-Boot). Flashed once on the factory line | No |
| `2. ENV` | Boot parameters (redundant copy). Records which slot to boot, per-slot state, and consecutive boot-failure count | Yes (read/write during upgrade) |
| `3. FACTORY` | Factory data: MAC, serial number, calibration. Physically write-protected | No |
| `4. SLOT_A` / `5. SLOT_B` | **Firmware slots**: each holds one full bootable system (kernel + rootfs) in the unified OFI format | Yes — write the slot that is not running |
| `6. DATA` | User config and logs (including audit and diagnostics). **Not part of any slot** | Not overwritten by upgrade; factory reset clears only this block |

> **Reading tip:** Blocks `4` and `5` are **two complete system images of the same format** (A/B dual slots). The left column (`1`, `2`, `3`) belongs to the boot chain and is not part of remote OTA after factory flash. Block `6` is user data and is independent of firmware version.
>
> **Layout note:** Boxes use ASCII only (`+ - |`) so alignment does not depend on font width.

### Figure 2: Dual-slot OTA and rollback

```text
  normal operation
  +--------------------------+   +--------------------------+
  |  SLOT_A                  |   |  SLOT_B                  |
  |  system image 1          |   |  system image 2          |
  |  [ RUNNING ]             |   |  [ STANDBY ]             |
  +--------------------------+   +--------------------------+
               |
               |  OTA: write new image into the slot that is NOT running
               |  the running slot stays untouched, so rollback stays possible
               v
  first boot after OTA
  +--------------------------+   +--------------------------+
  |  SLOT_A                  |   |  SLOT_B                  |
  |  old version             |   |  new version             |
  |  [ still bootable ]      |   |  [ TRYING ]              |
  +--------------------------+   +--------------------------+
               |                              |
        boot ok / self-check ok          boot fail / self-check fail
               |                              |
               v                              v
     confirm slot B: mark VALID        bootloader follows ENV, boots A again
     next boot uses slot B             -> version rollback, device stays bootable
```

### Definitions

| Term | Meaning |
|:---|:---|
| **Firmware slot** | `SLOT_A` or `SLOT_B` in Figure 1. A region of Flash that holds **one full bootable system** (kernel + rootfs). After power-on, the bootloader loads the system from one of these slots into RAM and boots it. |
| **Dual slot** | Keeping both `SLOT_A` and `SLOT_B` (Figure 2). The device runs only one of them at a time. Remote upgrade writes the new image into the idle slot; after the new image proves it can boot, the next boot uses the new slot; if the new image fails boot repeatedly, the bootloader returns to the still-good old slot. |
| **A/B arbitration** | The logic in Figure 2 that chooses which slot to boot, based on slot state in `ENV` (valid / trying / bad) and consecutive boot-failure count. Vendor SDKs usually encode this as chip-specific boot scripts — the pain table below calls those **A/B arbitration scripts**. Octopus encodes the same rules as a unified **dual-slot state machine** that stays unchanged across SoCs. |
| **OFI image format** | The unified package format written into `SLOT_A` / `SLOT_B`: Octopus Firmware Image, extension `.ofi`. It defines how the header, signature, and kernel / rootfs / AI-model sections are laid out, so packaging stays the same when the SoC changes. |

### Two easy mix-ups

1. **"Slot" means system firmware only, not user data.** Config and logs live in block `6. DATA` in Figure 1. They belong to no slot and are not overwritten by upgrade or slot switch. (On machines with a read-only assembled rootfs, files that must survive reboot must be placed under `/data` explicitly; paths under `/etc` are lost on reboot.)
2. **Dual slot ≠ two systems running at once.** Only one slot runs at any time; the other is a standby that can boot. Writing the idle slot does not disturb the running system, and a failed upgrade can still boot the old version.

---

## Why Octopus Embedded?

In embedded Linux product development and mass production, teams commonly face **8** real engineering dilemmas:

| Real-world pain | Field / R&D consequence | Octopus Embedded approach |
|:---|:---|:---|
| **SoC SDK lock-in** | Switching SoCs for cost or supply (e.g. RK3568 → SSD20X) forces a rewrite of flashing tools, package formats, and the A/B arbitration scripts that choose which slot to boot | **Unified platform shell**: freeze the lifecycle rules (OFI image format, dual-slot state machine: how two images switch and how failure rolls back); SoC differences stay in the device tree and a thin Board HAL |
| **Storage media churn** | Board cost-downs changing NOR / SPI-NAND / eMMC break OTA write and boot logic because of bad-block skipping and MTD vs block-device mismatches | **Storage Abstraction Layer (SAL)**: six logical volumes across all media; firmware slots on SPI-NAND managed by UBI for bad blocks; data area uses adaptive filesystems (JFFS2 / UBIFS / ext4) |
| **Fractured management plane** | CLI, Web, cloud RPC, and the local HDMI/LCD UI each keep their own parser; a serial change is invisible on the panel, and a panel save overwrites the cloud; power cuts truncate the config file | **Single source of truth & transactions**: local screen, Web, CLI, RPC, and cloud all read/write one Candidate-Running config (candidate vs running) with conflict detection and confirmed commits; layered plain-text config with tolerant replay and a POSIX safe-save pipeline |
| **PID 1 supervision dilemma** | `systemd` plus companions often cost tens of MB of RAM and heavy trimming on small gateways; `BusyBox init` can respawn but has no dependency order, readiness, or backoff | **Lightweight PID 1**: pure-C `octopus_supervisor` (<2000 LOC, <2 MB RAM) with declarative DAG startup, stepped backoff restart, and clean shutdown |
| **Black-box field hangs** | Occasional deadlocks or crashes leave an empty log after hardware watchdog reset; tiny Flash cannot store core dumps; R&D falls into a “cannot reproduce” support loop | **Introspection & crash blackbox**: built-in `lock_watchdog` mutex monitor; per-module heap alloc/free counters to see which module is leaking; debug builds also write fixed guard values before/after each heap block (a **canary**)—if they are overwritten at free time, a buffer overrun is proven; on crash, capture all-thread stacks into RAMoops/Flash blackbox |
| **Power-cut rootfs corruption** | Factory or field power pulls corrupt metadata on a read-write root, causing Kernel Panic on mount and a return-to-factory | **Read-only base, assembled at boot**: initramfs `octopus_loader` combines read-only SquashFS with tmpfs into an OverlayFS root (reboot restores the base); persistent data is strictly isolated |
| **AI-edge “two-brain” IPC bottleneck** | Motion control and a vendor NPU user-space runtime in one process (an NPU-side fault takes down the machine); local sockets for HD video copy too much and saturate the CPU | **Dual-plane isolation**: control plane over lightweight JSON-RPC (fault-domain isolation); high-throughput image/tensor data plane over POSIX shared-memory zero-copy rings |
| **Generic updater landing gap** | SWUpdate, RAUC, and similar frameworks write images and assume U-Boot already does slot selection and bad-block handling; they do not own config migration or process lifecycle, and are hard to trim on a 64 MB RAM gateway | **Full chain, not a thin wrapper**: slot selection and bad-block semantics are implemented symmetrically in U-Boot and user space; upgrade, config migration, process orchestration, and crash recovery share one platform |

### How these 8 dilemmas show up in a product cycle

The eight items below match the table above, written as what a team typically lives through on a real product. If you have shipped embedded devices, several will feel familiar.

1. **"New SoC, new nightmare"**: Vendor SDKs deliver not only kernel drivers but a chip-specific firmware lifecycle (custom packagers, flashing tools, hardcoded partition tables and upgrade scripts). Swapping SoCs burns most of the team on factory and OTA plumbing.
2. **"Flash respin OTA disaster"**: Cost-down from eMMC to SPI-NAND exposes factory and runtime bad blocks; plain `dd` bricks the part; U-Boot read topology and user-space write topology disagree, so the image never boots.
3. **"Split CLI, Web, RPC, and local screen"**: A serial edit never shows on the Web UI; the HDMI/LCD panel keeps a private copy. Saving on the panel overwrites cloud and CLI changes. A power cut during save leaves an empty file; the service fails on next boot.
4. **"systemd is too heavy, BusyBox is too fragile"**: On 64 MB–128 MB gateways, a full `systemd` stack eats a large share of RAM and needs heavy trimming; BusyBox has no dependency DAG or health checks, so a crashing service either spins forever or sits half-alive.
5. **"Field crash reports with zero clues"**: After months in the field, a watchdog reset leaves a clean slate. Tiny Flash cannot hold multi-megabyte core dumps; slow leaks and ABBA deadlocks stay untraceable.
6. **"Power-cut bricking"**: Frequent power pulls on a read-write ext4/ubifs root corrupt metadata; the machine will not boot; return rates explode.
7. **"NPU dies, whole machine dies"**: Motion needs millisecond determinism; vision depends on a vendor-delivered NPU user-space runtime. One process couples them so an NPU-side fault kills the box; moving HD video over local sockets burns CPU on copies.
8. **"Drop in an updater, still dig your own trench"**: SWUpdate/RAUC cover generic user-space write and slot switch, but assume the bootloader already selects slots and skips bad blocks, and they do not own config migration, process respawn, or crash healing. After integration, the hard middle is still yours—and on 64 MB RAM you must trim the framework first.

---

## Supported scenarios

- **Industrial & high-reliability gateways** — routing, dual-slot firmware OTA (failed upgrade can roll back to slot A), field diagnostics
- **Devices with a local panel** — HDMI / LCD UI shares one config with Web, CLI, and the cloud
- **AI edge & embodied robotics** — “small brain” main process + optional NPU vision process
- **Cross-chip product lines** — same codebase, different toolchains (RK3568 / SSD20X / …)

---

## Documentation

Start here → **[docs/README.md](docs/README.md)**

| Area | Document | Description |
|------|----------|-------------|
| **Core architecture** | [00-platform-design.md](docs/architecture/00-platform-design.md) | **Architecture specification & decision RFCs** (layers, state machines, frozen HAL APIs, trade-offs) — Chinese |
| **Reliability & production** | [00-critical-topics.md](docs/reliability/00-critical-topics.md) | Watchdog heartbeat matrix, OOM ladder, storage physical limits, introspection |
| **Language guidelines** | [00-language-cpp-guidelines.md](docs/guides/00-language-cpp-guidelines.md) | C vs C++ selection and restrained industrial embedded C++ rules |
| **Test pyramid** | [01-qemu-test-strategy.md](docs/guides/01-qemu-test-strategy.md) | Three-layer automation (x86 unit / QEMU / power-cut on hardware) |
| **Drivers & data-plane offload** | [02-switch-hal-drivers.md](docs/guides/02-switch-hal-drivers.md) | HAL boundaries and optional Ethernet switch hardware forwarding offload |
| **AI vision & robotics** | [03-ai-robot-vision.md](docs/guides/03-ai-robot-vision.md) | Dual-brain AI, shared-memory rings, vision pipeline |
| **Brand & visual identity** | [branding.md](docs/project/branding.md) | Naming, logo SVG rules, open-source copyright guidelines |

---

## Roadmap

See **[ROADMAP.md](ROADMAP.md)**:

| Phase | Focus | Target tag | Status |
|-------|--------|------------|--------|
| **0** | Architecture draft & design RFCs (this repo’s docs) | `v0.1.0-design` | **Draft published / open for RFC discussion** |
| **1** | `liboctopus_basics`, CMake skeleton, unit tests, parser fuzzing, JSON-RPC stub | `v0.2.0-alpha` | Planned |
| **2** | octopus_cli, octopus_config (conflict detection, rollback, confirmed commit), Board HAL, public SDK | `v0.3.0-alpha` | Planned |
| **3** | U-Boot OFI, octopus_supervisor, A/B dual-slot firmware | `v0.4.0-beta` | Planned |
| **4** | Full integration, AI vision, stress testing | `v1.0.0-rc1` | Planned |

The table lists phase focus only. Full milestone checklists and acceptance items are in [ROADMAP.md](ROADMAP.md).

---

## Contributing

Design-phase contributions welcome: architecture feedback, RFCs, and documentation improvements.

Please read [CONTRIBUTING.md](CONTRIBUTING.md) before opening issues or pull requests.

Architecture intent is authoritative in **Simplified Chinese** (`README.zh-CN.md` and `docs/`); keep English entry points aligned. Phase-3 coding blockers are listed in [architecture §13.1](docs/architecture/00-platform-design.md#131-phase-3-编码前-blockers必须定稿).

---

## Security

See **[SECURITY.md](SECURITY.md)** for how to report vulnerabilities and security-relevant design flaws. Do not paste exploit details into public issues.

---

## License

Licensed under the [Apache License, Version 2.0](LICENSE).

---

## Disclaimer

**Octopus Embedded** is an independent open-source project. It is **not affiliated with** [Octopus Deploy](https://octopus.com) or any other trademark holder of the word “Octopus”. This repository currently contains **original Octopus Embedded architecture documentation only**. It does **not** include or distribute any third-party proprietary/closed-source code or prebuilt binaries. Mentions of vendor SDKs, NPU runtimes, or boot firmware describe integration boundaries only; those materials must be obtained by the device maker under the vendor’s own license. The materials describe an **architecture draft**; critical A/B upgrade and rescue state machines still have open items (architecture chapter 13) and are **not** a claim of mass-production-proven implementation.
