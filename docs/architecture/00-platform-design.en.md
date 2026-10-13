# Octopus Embedded Core Architecture Design & Engineering Decision Specification (Architecture Specification & RFCs)

**English** | [简体中文](./00-platform-design.md)

> **Note**: The Simplified Chinese version is authoritative for architecture intent. This English twin must be kept aligned with it; in case of discrepancy, the Chinese text prevails.

> **Document scope**: This document is the **overall architecture specification and core decision rationale (Architecture Specification & Decision Records)** of the Octopus Embedded open-source project.  
> **Applicable phases**: It guides engineering implementation from Phase 1 (core skeleton) through Phase 4 (mass-production hardening), and serves as the fundamental technical reference for all system code, cross-SoC porting, and open-source community collaboration.

---

## 1. Business Pain Points and Core Mission

Embedded Linux is widely deployed in industrial smart gateways, edge computing boxes, energy-storage controllers, mobile robots, and in-vehicle terminals. Yet for a long time most embedded teams have been stuck in a vicious cycle of **"constantly rebuilding the lower layers from scratch, three disjointed middleware stacks, no way to diagnose sporadic field failures, and painful re-work every time the hardware changes."**

A close look at today's embedded development and mass-production lifecycle reveals the following 8 major engineering pain points:

### 1.1 Tight Coupling to SoC SDKs and Fragmented Lifecycle "Shell" Mechanisms
SoC vendor SDKs (e.g., Rockchip, Allwinner, SigmaStar) are fundamentally delivered with the goal of "getting the reference demo board running," not of supporting "the full lifecycle of an industrial-grade, high-reliability product":
- **Divergent image formats and flashing tools**: Each vendor invents its own SoC-specific image packaging format and companion host-side flashing tool, mutually incompatible;
- **Arbitrary boot and partition naming**: Partition-table parameter formats (parameter/env/sys_config/DTS) vary wildly, and U-Boot environment variable structures and boot scripts each go their own way;
- **Deeply entangled shell mechanisms**: Watchdog startup, process supervision, and factory-reset scripts are scattered across assorted `/etc/init.d/` scripts or vendor-private kernel patches.

When hardware must move from RK3568 to SSD20X or Allwinner due to shortages, cost reduction, or performance upgrades, business logic may only need a cross-recompile, but **the entire firmware packaging scripts, production-line flashing fixtures, OTA server upgrade protocol, and U-Boot slot selection and power-loss fallback logic must all be rewritten from scratch**. R&D teams spend 70% of their effort passively coping with vendor shell fragmentation and cannot accumulate real core software assets.

```text
 Rockchip SDK (RK3568)             Allwinner SDK                     SigmaStar SDK (SSD20X)
 ┌──────────────────────┐          ┌──────────────────────┐          ┌──────────────────────┐
 │ Private image fmt /  │          │ Private image fmt /  │          │ Private IPL /        │
 │ partition table      │          │ partition descriptor │          │ flashing format      │
 │ Vendor flash & pack  │          │ Vendor flash tools & │          │ Vendor upgrade &     │
 │ tools                │          │ scripts              │          │ flashing path        │
 │ SDK built-in recovery│          │ Vendor boot fallback │          │ Vendor boot chain &  │
 │ logic                │          │ logic                │          │ watchdog             │
 └──────────┬───────────┘          └──────────┬───────────┘          └──────────┬───────────┘
            ▼                                 ▼                                 ▼
       Three non-reusable shell toolsets ───► Every SoC change rewrites upgrade/production/after-sales
```

### 1.2 Frequent Storage Chip Changes and the Physical Gap Between Raw Flash and Block Devices
In long-lifecycle industrial hardware, Flash storage chips change very frequently:
- **Board-level cost-down part substitution**: Moving from expensive eMMC to SPI-NAND, or to a single SPI-NOR for extreme cost compression;
- **Inherent gaps in physical characteristics**:
  - **SPI-NAND / SLC NAND (raw Flash)**: Allowed to ship with 1%–2% initial bad blocks, and new bad blocks appear dynamically over its service life. Writing partitions directly with plain `dd` will inevitably corrupt data; if the bad-block skipping algorithm in the upgrade program is not perfectly symmetric with that in the U-Boot loader, the device will brick after upgrade;
  - **NOR Flash**: Linearly readable but with very small erase blocks and limited capacity, requiring a read-only file system with a very high compression ratio;
  - **eMMC / SD**: Block devices with a built-in FTL controller; sudden power loss easily corrupts file system metadata;
- **Upper-layer disaster due to lack of abstraction**: Without a unified Storage Abstraction Layer (SAL), OTA flashing tools, boot-time mount scripts, and boot configurations must hard-code `/dev/mtdX` or `/dev/mmcblk0pX` per board for the specific medium. Every board revision breaks the entire storage pipeline once.

### 1.3 Fragmented Management Plane: CLI, Web, RPC, Cloud, and Local Display Each Built Separately
A field device has more than one configuration entry point. Operators use a serial console or SSH CLI; the LAN uses Web; the cloud and mobile apps use JSON-RPC; gateways, robots, and industrial PCs with displays also run a local UI on HDMI or LCD (Qt, LVGL, etc.):
- **Multiple parsers**: Every new parameter must be implemented separately in the CLI, web pages, cloud protocol, local display, and persistence module. Miss one, and different entry points see different values;
- **Local display keeps a private copy**: The HDMI/LCD UI writes brightness, network, and business parameters into its own database or JSON. A value changed via serial console still shows the old value on screen; pressing Save on the screen then overwrites configuration just pushed by Web or the cloud;
- **Concurrent dirty writes**: There is no unified in-memory configuration and no transaction lock. While a field engineer edits parameters over the serial console, someone else clicks Save on the web page or local display, and whoever writes last overwrites the entire configuration;
- **Truncation on power loss**: Configuration files are overwritten in place; if power fails mid-write, the file becomes empty or half-written, and the business service cannot start on next boot.

### 1.4 The PID 1 Orchestration Dilemma: Bloated systemd Exhausting Memory vs. Thin BusyBox Collapsing on Crash
Embedded Linux has long been awkwardly polarized on system initialization and service supervision (PID 1):
- **Desktop-derived approach (systemd)**: Feature-complete, but PID 1 plus companions like journald and udevd consume from over ten MB to tens of MB of resident memory (depending on enabled components), depend heavily on glibc (musl needs extra patches), and without deep trimming take several to over ten seconds to boot. On a compact 64MB-RAM gateway, this overhead crowds out memory available to the business, and trimming and maintenance costs are high;
- **Primitive approach (BusyBox init / SysVinit)**: Relies on `/etc/inittab` and rc scripts to start serially. `respawn` can restart a single process, but there is no dependency DAG (directed acyclic graph) among services, so developers can only guess timing with `sleep 2` in scripts. There is no readiness notification, backoff restart, or health check; a repeatedly crashing process is either respawned infinitely fast or stuck in a zombie-like "process alive, service dead" state.

### 1.5 Black-Box Field Crash Diagnosis and the "Cannot Reproduce" After-Sales Quagmire
Industrial devices are deployed on remote poles, in substations, on factory lines, or in deep mines, making sporadic field crashes extremely painful to diagnose:
- **Blind watchdog resets leave zero field traces**: After a deadlock or crash, the only last resort is a hard reset by the hardware watchdog. After reset, volatile RAM data is completely gone, and field logs are empty;
- **Flash too small for core dumps**: Writable Flash partitions on embedded devices are typically only a few MB to tens of MB. Desktop Linux core dumps, often hundreds of MB, simply cannot be saved, and forcing them to disk would rapidly wear out the Flash;
- **Hidden multi-threaded deadlocks**: Lock-hold timeouts, ABBA cross-deadlocks, and slow-callback blocking often reproduce only once after weeks of continuous high load. In the field, GDB cannot be attached, let alone JTAG;
- **Slow, small memory leaks**: A communication submodule leaks 10KB per hour; after three months of continuous operation it finally triggers an OOM crash. The development team guesses blindly across tens of thousands of lines of code, and the fix takes months.

### 1.6 Root File System Corruption and Bricking Under Sudden Power Loss
In industrial, agricultural, and special-purpose equipment sites, yanked power cords, production-line breaker trips, or momentary bus undervoltage are extremely frequent (weekly or even daily):
- If the device uses a single read-write file system (e.g., a single root partition formatted directly as ext4 or ubifs), sudden power loss during log flushing or configuration writes easily corrupts file system metadata (Superblock/Journal/Inode);
- On next power-up the kernel cannot mount the root partition, and the screen or serial console prints `Kernel panic: VFS: Unable to mount root fs`. The field device is completely bricked and must be returned to the factory, disassembled, and reflashed.

### 1.7 The "Big-Brain/Little-Brain Split and Throughput Bottleneck" in the Era of Embodied AI and Edge AI
For modern smart edge gateways or mobile robots (AGV/AMR/inspection robotic arms) with NPU compute, the architecture faces new challenges:
- **Single-point-of-failure risk from mixing control and inference**: Vendor NPU user-space runtimes and third-party neural network stacks are typically heavier than management-plane processes and more prone to leaks or sporadic faults. If real-time motion control (the "cerebellum") and AI vision (the "cerebrum") are crammed into one giant process, an AI-side crash directly causes loss of motor control and can lead to serious physical collisions;
- **Cross-process video throughput saturating the CPU**: If the two are split into separate processes and 1080P/4K video frames are transferred via conventional local sockets or HTTP, frequent memory copies and kernel context switches push embedded CPU usage to 100%, causing end-to-end latencies of over a hundred milliseconds and stealing real-time scheduling bandwidth from core control threads.

### 1.8 The Deployment Gap of General-Purpose Update Frameworks
Open-source update frameworks such as SWUpdate and RAUC provide generic logic for user-space updates and dual-slot switching, but there are inherent gaps when deploying them in industrial-grade embedded products:
1. **Boot-chain vacuum**: General frameworks assume U-Boot already has complete slot-selection and bad-block handling logic, and are powerless against Bootloader differences across SoCs;
2. **Lack of system coordination**: General frameworks only write images to partitions; they do not manage runtime configuration version migration, process lifecycle orchestration, lock monitoring, or crash self-healing;
3. **High resource threshold**: On ultra-compact IoT gateways with 64MB RAM / 16MB Flash, general frameworks are hard to trim deeply.

### 1.9 Octopus's Core Mission and Approach
Octopus Embedded was founded to **fully standardize and productize the "full-lifecycle infrastructure" of industrial products**.
Through in-house specifications, it masks the inherent gaps of underlying SoCs and storage media, fully confining SoC vendor differences to the Linux kernel device tree and an extremely thin board-level interface (Board HAL), achieving **"one foundation for many SoCs; write once, reuse across the product line"**:

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │ Business application layer (gateway routing protocols, embodied robot  │ Unchanged when
 │ control, AI vision inference processes)                                │ SoC changes
 ├────────────────────────────────────────────────────────────────────────┤
 │ Octopus common product infrastructure (fixed lifecycle)                │
 │  · Unified firmware image format (OFI) · A/B dual-slot boot & anti-    │ Fully unified
 │                                          brick state machine           │ across SoCs
 │  · Lightweight PID 1 (supervisor)      · Single-source-of-truth config │ Zero migration
 │                                          center (octopus_config)       │ cost
 │  · Unified hierarchical CLI            · Hardware watchdog integration │
 │    (octopus_cli)                         & crash black box             │
 ├────────────────────────────────────────────────────────────────────────┤
 │ Linux kernel + device tree + thin board HAL (liboctopus_board)         │ Only this layer
 │                                                                        │ adapts per SoC
 └────────────────────────────────────────────────────────────────────────┘
```

---



## 2. Architecture Design Principles and Non-Functional Requirements (NFR) Baseline



### 2.1 Core Design Philosophy

1. **Preserve the business across SoC changes**: SoC-specific differences are pushed entirely down into the device tree and a thin board-level HAL written in pure C;
2. **Self-healing after power loss**: The whole system tolerates sudden power loss at any moment; dual firmware slots arbitrate and fall back automatically; configuration files are written atomically;
3. **Physical isolation of fault domains**: Memory-hungry AI vision and heavyweight dynamic routing protocols run in separate processes; their crashes never affect the main management plane;
4. **Single source of truth for configuration**: Only one running configuration is maintained in memory. Serial CLI, Web, JSON-RPC, cloud, and the local HDMI/LCD UI are all clients of this configuration; a change from any entry point is immediately visible to all others; persistence uses human-readable, auditable text configuration;
5. **Restrained evolution, no over-engineering**: The core foundation prefers lightweight C/C++, avoids heavy external dependencies, and fits low-memory hardware.

> **Boundary of "zero migration cost" (important qualification)**: Principle 1 aims for "business code and OTA logic stay unchanged when the SoC changes," but this holds only under one precondition — **the target platform's libc and kernel capabilities are on the same baseline**. A verified exception is described in [reliability 4.2](reliability/00-critical-topics.md): older BSPs such as SSD20X are stuck on glibc 2.24 / uClibc-ng + kernel 4.9, where `_TIME_BITS=64` has no effect, `time_t` remains 32-bit, and the Year 2038 problem cannot be fully eliminated. The degradation on such platforms is **irreducible**; the support period must be stated in product documentation, and "zero migration cost" must not be claimed indiscriminately.



### 2.2 System Resource and Non-Functional Requirements (NFR) Baseline


| Dimension          | Compact gateway (e.g., SSD20X)                     | Industrial smart edge (e.g., RK3568)          | Verification & constraint method                |
| ----------- | ----------------------------------- | ------------------------- | ---------------------- |
| **Memory hardware spec**  | 64 MB ~ 128 MB DDR                  | 1 GB ~ 4 GB LPDDR4        | Hardware baseline configuration                 |
| **Persistent storage medium**  | 16 MB ~ 32 MB SPI-NOR / 128MB NAND  | 8 GB ~ 32 GB eMMC / NVMe  | Partition planning baseline                 |
| **Foundation resident memory**  | **≤ 12 MB** (incl. supervisor, app, cli) | **≤ 35 MB** (incl. base routing plugins)      | `ps_mem` / cgroup limits   |
| **Firmware root file system** | **≤ 8 MB** (SquashFS compressed)             | **≤ 64 MB** (full toolchain and models)     | OFI packaging size gate             |
| **Cold boot time**   | **≤ 3.5 s** (power-on to main control plane available)              | **≤ 8.0 s** (power-on to NPU/network ready) | Serial timestamps (grabserial) + GPIO markers timed by oscilloscope |
| **Sudden power-loss tolerance**  | **10,000** random power cuts with no bricking or corruption            | **10,000** random power cuts with no configuration loss    | Production-line power-cut relay burn-in test            |
| **Watchdog response**   | Hardware watchdog hard timeout 20 s, heartbeat 5 s                | Hardware watchdog hard timeout 20 s, heartbeat 5 s      | Exclusive lock on `/dev/watchdog`   |
| **Minimum kernel version**  | 4.9 (current vendor BSP; no PSI, no 64-bit time syscalls) | 4.19 / 5.10                | Capability probing + degradation path; see 9.3 memory pressure ladder |
| **Operating temperature range**  | **-20 ~ +70 °C** (whole unit at full load, incl. NPU models)      | **-20 ~ +70 °C**              | Thermal chamber + full-load burn-in; criteria in 9.5 |
| **Storage temperature range**  | **-40 ~ +85 °C** (unpowered)              | **-40 ~ +85 °C**              | Thermal chamber, unpowered |
| **`octopus_app` single-process memory** | **≤ 8 MB** (RSS, excluding cli and supervisor) | **≤ 28 MB**              | `ps_mem` / cgroup; used as the self-check threshold in 9.3 |

> **Note on temperature figures**: The table above states **whole-unit** commitments, not component specs. Individual components (DDR, eMMC, PHY, NPU) each have wider allowed ranges; the whole-unit figure is determined by the narrowest link together with the thermal design. Exact numbers must be fixed during hardware selection based on thermal measurements; **the software side must provide a temperature sampling channel and a tiered degradation strategy (see 9.5), otherwise this figure can neither be verified nor upheld**.

> **Note on minimum capacity**: A/B dual slots require Flash to hold at least two copies of "kernel + root file system." With the 8MB root file system above plus roughly 3MB of kernel, **a 16MB SPI-NOR cannot fit full A/B**. 16MB models must choose one: compress the root file system to under 5MB, or fall back to an asymmetric dual slot of "main system + minimal recovery system," where the recovery system only connects to the network to re-download the main system. This trade-off must be settled during hardware selection, not deferred to mass production.


---



## 3. System Layering and Runtime Process Topology



### 3.1 Four-Layer System Architecture

```text
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. Application & Extension Layer (Application & Ext Layer)              │
│                                                                         │
│   Static Web pages  Local HDMI/LCD UI    3rd-party processes  Optional: │
│   (HTML/JS)         (Qt / LVGL, display  (C / Go / Python)    AI vision │
│                      only)                                    (owns NPU)│
│         │                 │                    │                 │      │
│         └──── JSON-RPC ───┴──── Local IPC ─────┴─────────────────┘      │
│       (Web / local display / CLI / cloud read & write the same running  │
│        configuration)                                                   │
├─────────────────────────────────────────────────────────────────────────┤
│ 3. Main Management Plane octopus_app (Management Plane Core)            │
│                                                                         │
│   ┌────────────┐ ┌────────────┐ ┌──────────────┐ ┌──────────────────┐   │
│   │ octopus_cli│ │ JSON-RPC   │ │ Local IPC    │ │ octopus_config   │   │
│   │ CLI engine │ │ server     │ │ server (UDS) │ │ config engine    │   │
│   └────────────┘ └────────────┘ └──────────────┘ └──────────────────┘   │
│                  │ Shared in-memory data (Candidate-Running txn guard)  │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │ Built-in modules: system mgmt / network config / firmware       │   │
│   │                   upgrade orchestration / diagnostics           │   │
│   │ Optional plugin: switch_offload (switch-chip HW forwarding      │   │
│   │                  table offload)                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────────┤
│ 2. Runtime Environment Layer (Runtime Environment)                      │
│                                                                         │
│   octopus_loader (early boot composition) octopus_supervisor (PID 1)    │
│   Assembles read-only SquashFS +          Starts services by dependency │
│   in-memory OverlayFS                     topology, auto-restarts       │
├─────────────────────────────────────────────────────────────────────────┤
│ 1. Hardware & Kernel Layer (Hardware & Kernel)                          │
│                                                                         │
│   U-Boot bootloader (slot selection     Linux kernel + device tree (DTS)│
│   & signature verification)                                             │
│   Board HAL (liboctopus_board: MAC / SN / LED / Reset key / watchdog)   │
│   Optional: switch chip (DSA), NPU, HDMI/LCD DRM display, HW watchdog   │
└─────────────────────────────────────────────────────────────────────────┘
```



### 3.2 Responsibilities and No-Overreach Red Lines


| Layer         | Mandatory core responsibilities                   | Strictly prohibited overreach (red lines)                      |
| ---------- | --------------------------- | -------------------------------- |
| **Hardware & kernel layer** | Hardware initialization, driver abstraction, standard device exposure (`/dev/`*) | Must not host business state machines; must not hard-code product business logic            |
| **Runtime environment layer**  | Compose the read-only root, reap zombie processes, start by dependency and supervise health   | Must not handle specific network/gateway configuration; must not parse business protocols          |
| **Main management plane process** | Maintain the system-wide single source of truth for configuration, handle concurrent transactions, manage module lifecycle | Must not perform heavy, time-consuming operations in the main loop (e.g., writing firmware to disk, packaging logs, model inference) |
| **Application & extension layer** | Host peripheral business (local display UI, video stream inference, advanced dynamic routing protocols) | Must not persist system configuration privately; the HDMI/LCD UI, like Web, CLI, and cloud, may change configuration only via the management plane |




### 3.3 Runtime Process Topology (`ps` View)

```text
  [Linux kernel] (PID 0)
        │
        ▼ exec
  [octopus_supervisor] (PID 1)
        │
        ├── [Lightweight Web server] (Hiawatha/FastCGI) ──JSON-RPC──► forwarded to octopus_app
        │
        ├── [octopus_app] (core main process)
        │     │
        │     ├── [Reactor main loop] (config transactions, IPC listening, RPC dispatch)
        │     ├── [Worker thread pool] (async task dispatch)
        │     ├── [Dedicated watchdog thread] (heartbeat inspection & exclusive /dev/watchdog feeding)
        │     │
        │     ├── Helper process: [firmware_worker] (firmware erase/write & verification, on demand)
        │     └── Helper process: [diag_collector] (log & coredump collection, on demand)
        │
        ├── [octopus_cli] (serial/SSH interactive login shell) ──IPC──► attaches to octopus_app
        │
        ├── (optional) [local_ui] (HDMI/LCD: Qt or LVGL) ──JSON-RPC──► same running configuration
        │
        ├── (optional) [FRR zebra/ospfd] (dynamic routing processes) ──Netlink──► Linux kernel FIB
        │
        └── (optional) [ai_vision_daemon] (AI vision process) ──SHM ring──► shared-memory data plane
```

---

### 3.4 Preconditions for Delivering "No Bricking on Power Loss" and "24×7 Self-Healing"

This architecture positions "anti-bricking" and "fault self-healing" as core selling points, but neither promise can be achieved by software alone; each depends on several **hardware and platform preconditions**. The following preconditions must be measured one by one during bring-up. **If any one fails, the corresponding promise cannot be delivered on that model** and must be honestly downgraded in product documentation; none may be assumed by default:

| # | Precondition | Consequence if not met | Verification method |
| :--- | :--- | :--- | :--- |
| 1 | The hardware watchdog is actually probed successfully, and `octopus_app` can open it independently | A user-space deadlock goes unreset; the system hangs forever | Open `/dev/watchdog`, deliberately hang `octopus_app`, confirm the whole unit resets within 20 s |
| 2 | `pstore` / `ramoops` records survive a watchdog reset | Zero field traces after watchdog reset; "traceable crashes" fails | 20 cold and 20 warm resets per board; verify crash records exist and are complete |
| 3 | The SoC can distinguish **power-loss reset** from **watchdog reset** (reset-reason register reliability) | bootcount misjudges grid glitches as new-firmware defects and, after several accumulate, condemns a healthy firmware | Induce 50 each of power-loss, watchdog, and soft resets; verify `octopus_board_get_reset_reason()` is 100% accurate |
| 4 | The temperature sensor is readable, and over-temperature is not handled by the watchdog | At high temperature the only recourse is repeated watchdog reboots, which do not reduce heat, creating a thermal loop | Full-load burn-in to thermal steady state; confirm the tiered degradation actions in 9.5 trigger as expected |
| 5 | The BootROM supports multi-copy SPL search (or eMMC boot0/boot1 polling) | Single point of failure in the Bootloader with no backup; recovery only via Maskrom button | Confirm against the SoC manual + corrupt the primary SPL and power on to verify boot from the copy |
| 6 | U-Boot can read UBI static volumes (`CONFIG_CMD_UBI` + static volumes + corresponding MTD driver) | The "same UBI logical addressing on both ends" argument in 4.1 collapses; A/B cannot work on NAND models | On the target U-Boot version, `ubi read` a static volume and compare with the original file |
| 7 | Partition boundaries align with medium erase-block boundaries (NOR commonly 64KB erase blocks) | Erasing one partition corrupts the adjacent partition | Check DTS partition offsets against the device datasheet; after erasing SLOT_A, verify SLOT_B header is intact |
| 8 | SoC hardware RNG is available | SSH host keys and TLS certificates generated at first boot are weak and may collide across devices | After first boot, collect key fingerprints from multiple devices and confirm no duplicates |

> **Design constraint**: When items 1 or 4 do not hold, the architecture must not use "reboot" as a fallback — a reboot can neither revive a failed watchdog nor lower chip temperature. Each of these two fault classes must have a **non-reset** degradation path (the former should raise an alarm at boot and require hardware replacement; for the latter see 9.5).



---



## 4. Storage Abstraction Architecture, Firmware Layout, and A/B Disaster-Recovery Boot State Machine



### 4.1 Storage Abstraction Architecture: The 6 Standard Logical Volume Model (SAL Logical Volume Model)

#### (1) Inherent Physical Media Gaps and the Need for Abstraction
Embedded hardware product lines frequently change storage schemes due to cost, size, and supply chain (ultra-low-cost gateways use a single SPI-NOR, mid-range uses a single SPI-NAND, high-reliability industrial uses NOR+eMMC/NAND dual media, high-performance AI edge uses a single eMMC). However, the physical characteristics of these media have inherent gaps:

| Medium type | Minimum erase unit | Read/write characteristics | Bad blocks & wear | Linux access interface | Typical capacity & positioning |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **SPI-NOR Flash** | Sector (4KB / 64KB) | Random byte read, page write, block erase | **100% free of bad blocks**, very high reliability, ~100K cycle endurance | MTD character device (`/dev/mtdX`) | 16MB ~ 64MB, low-cost / high-reliability boot |
| **SPI-NAND Flash** | Block (128KB / 256KB) | Must read/write whole pages (2KB/4KB) | **Bad blocks from the factory**, more appear at runtime; requires ECC and bad-block skipping | MTD / UBI volume (`/dev/ubiX_Y`) | 128MB ~ 512MB, mid-range cost-effective |
| **eMMC / UFS** | Sector (512 Byte) | Built-in FTL controller; appears as a standard block device | Hardware manages bad blocks and wear transparently | Block device (`/dev/mmcblk0pX`) | 4GB ~ 64GB+, AI edge and big data |

If upper-layer business (OTA, configuration, boot) faces physical devices directly, every part substitution will litter U-Boot scripts, Loader mounting, and OTA flashing tools with `#ifdef`s.  
**Octopus establishes the core principle of a Storage Abstraction Layer (SAL)**: hide underlying physical device nodes and expose **6 standard Logical Volumes** upward:

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │              System-wide unified 6 Logical Volumes                     │
 ├──────────┬──────────┬──────────────┬──────────────┬─────────────┬──────┤
 │ 1. BOOT  │ 2. ENV   │ 3. FACTORY   │ 4. SLOT_A    │ 5. SLOT_B   │6.DATA│
 │ SPL+Boot │ Dual-copy│ Factory DNA  │ Primary FW   │ Backup FW   │ User │
 │ Fixed at │ boot     │ MAC/SN/      │ (OFI) full   │ (OFI) full  │config│
 │ factory  │ params   │ calibration  │ image        │ image       │ logs │
 │ Not OTA'd│ Counter  │ Physical     │ (A/B hot     │ (A/B hot    │      │
 │          │ arbitr.  │ write-protect│ standby)     │ standby)    │      │
 └──────────┴──────────┴──────────────┴──────────────┴─────────────┴──────┘
```

#### (2) Physical Mapping Matrix for 4 Typical Hardware Topologies

| Logical volume | Topology 1: single SPI-NOR (64MB) | Topology 2: single SPI-NAND (256MB) | Topology 3: hybrid NOR (16M) + eMMC (8G) | Topology 4: single eMMC (8G~32G) |
| :--- | :--- | :--- | :--- | :--- |
| **Representative hardware** | SSD20X / Allwinner ultra-low-cost gateway | Industrial wireless router / surveillance IPC | Industrial smart gateway / high-reliability industrial control | RK3568 / high-compute smart edge |
| **Underlying bus** | SPI (MTD architecture) | SPI (MTD/UBI architecture, with bad blocks) | SPI (NOR) + SDIO/MMC (eMMC) | eMMC 5.1 (standard block device) |
| **1. BOOT** | NOR head 0~1MB | NAND head 0~2MB (raw MTD, multi-copy redundancy) | **NOR head 0~2MB (never erased)** | `mmcblk0boot0` (when the SoC BootROM supports booting from the boot partition), otherwise a vendor-defined location at the head of the user area |
| **2. ENV** | NOR 1~2MB (dual mutually-backing sectors) | NAND 2~4MB (raw MTD, dual copies + bad-block margin) | **NOR 2~4MB (dual mutually-backing sectors)** | `mmcblk0boot1` or GPT partition `env` |
| **3. FACTORY** | NOR 2~3MB (read-only protected) | NAND 4~6MB (raw MTD, dual copies + CRC) | **NOR 4~6MB (physical write protection)** | eMMC read-only GPT partition `factory` |
| **4. SLOT_A** | NOR 3~32MB (29MB) | Static volume `slot_a` (100MB) in a UBI device (NAND 6~256MB) | **eMMC GPT partition `slot_a` (2GB)** | eMMC GPT partition `slot_a` (2GB) |
| **5. SLOT_B** | NOR 32~61MB (29MB) | Static volume `slot_b` (100MB) in the same UBI device | **eMMC GPT partition `slot_b` (2GB)** | eMMC GPT partition `slot_b` (2GB) |
| **6. DATA** | NOR 61~64MB (3MB JFFS2)| Dynamic volume `data` (UBIFS, ~40MB) in the same UBI device | **eMMC GPT partition `data` (4GB ext4)**| eMMC GPT partition `data` (4GB ext4) |

> **On NAND, SLOT and DATA must reside in the same UBI device**, so that UBI uniformly handles bad blocks, wear leveling, and data relocation (scrub) after read disturb. Raw MTD with "bad-block skipping" is used only for regions like BOOT / ENV / FACTORY that are written once on the production line and rarely rewritten, and a bad-block margin must be reserved at the end of each such partition. The UBI device also needs about 2%–3% of PEBs reserved as a bad-block replacement pool, so actual usable capacity is slightly smaller than shown.

> **High-reliability principle for industrial hybrid dual media (Topology 3)**: Place `BOOT`, `ENV`, and `FACTORY` on long-life, bad-block-free SPI-NOR Flash; place the bulky operating system, AI vision models, and business data on high-capacity eMMC. Even if the eMMC file system is damaged by harsh power loss, U-Boot in NOR never dies, and the device can always enter network/serial rescue mode.

#### (3) Physical Isolation Red Line
The business layer (OTA upgrade, configuration center, system introspection) **must never hard-code any physical device path (such as `/dev/mtd3` or `/dev/mmcblk0p2`)**. The entire system must address storage via logical volume names or the canonical `/dev/octopus/by-vol/*` paths.

---

### 4.2 U-Boot Boot Storage Abstraction and A/B Arbitration Driver (`cmd_ofi.c`)

U-Boot is the first line of defense for booting A/B slots; it hides the underlying medium through an abstract driver operation set:

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

*(Code comments above, translated: "U-Boot storage driver operation set interface"; "At compile time, the concrete backend is bound automatically per board: `ofi_storage_mtd.c` for NOR `mtd_read` / NAND `nand_read_skip_bad`; `ofi_storage_mmc.c` for eMMC `mmc_read` / `blk_dread`; `ofi_storage_ubi.c` for UBI dynamic volumes.")*

- **Redundant environment area (Redundant Env)**: Regardless of whether the underlying medium is NOR (`CONFIG_ENV_IS_IN_SPI_FLASH`), NAND (`CONFIG_ENV_IS_IN_NAND`), or eMMC (`CONFIG_ENV_IS_IN_MMC`), `CONFIG_SYS_REDUNDAND_ENVIRONMENT` is always enabled with two independent erase blocks. The standard U-Boot redundant format (CRC32 + 1-byte flags/counter) is used; Linux reads and writes uniformly via `libubootenv` (`fw_printenv` / `fw_setenv`), and `/etc/fw_env.config` is generated from the board configuration, ensuring both sides interpret the format identically;
- **Reuse U-Boot's standard boot counting**: Enable `CONFIG_BOOTCOUNT_LIMIT` and keep U-Boot's conventional variable names `bootcount` / `bootlimit` / `upgrade_available` / `altbootcmd`. Octopus only adds `boot_slot` (a/b) and `slot_a_state` / `slot_b_state` (valid / trying / bad). This way tools like SWUpdate and RAUC see consistent semantics;
- **Unified boot command**: `ofi boot a` parses the target slot's OFI header, verifies the signature and section hashes, loads the kernel into RAM and executes it, and passes `octopus.slot=a` via bootargs so the Loader knows which slot to mount.

---

### 4.3 Volume Discovery in the Linux Kernel and Loader Layers

Once the system enters the Linux kernel stage, logical volume symlinks are assembled dynamically from standard device tree labels and initramfs logic:

#### (1) Device Tree (DTS) and GPT Standard Naming Conventions
1. **MTD devices (NOR / NAND)**: The `partitions` node in DTS must use standard `label`s:
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
2. **Block devices (eMMC)**: Use a standard GPT partition table with partition labels (`PARTLABEL`) strictly named `slot_a`, `slot_b`, `factory`, `data`.
3. **UBI volumes (NAND)**: UBI volume names are strictly `slot_a`, `slot_b`, `data`; the kernel must enable `CONFIG_MTD_UBI_BLOCK` so read-only static volumes can be exposed as block devices for SquashFS mounting.

#### (2) `octopus_loader` Dynamic Volume Resolution and OverlayFS Composition
In the initramfs stage, `octopus_loader` runs a generic "**Volume Discoverer**" after startup. There is no udev in initramfs, so `/dev/disk/by-partlabel/` does not exist and sysfs must be read directly; SquashFS can only be mounted on block devices — neither MTD character devices nor UBI volume character devices can be mounted directly:

```text
 0. Read octopus.slot=a|b passed by U-Boot from /proc/cmdline to determine this boot's slot
          │
          ▼
 1. Scan and convert to mountable block devices:
    ├── eMMC: iterate /sys/class/block/*/uevent, match PARTNAME=slot_a  → /dev/mmcblk0pN
    ├── NOR : match "slot_a" in /proc/mtd                               → /dev/mtdblockN (read-only)
    └── NAND: after ubiattach, match slot_a in /sys/class/ubi/ubi0_*/name
              → ubiblock --create /dev/ubi0_N                           → /dev/ubiblock0_N
          │
          ▼
 2. Map to unified symlinks:
    ln -s <block device> /dev/octopus/by-vol/slot_a
    ln -s <block device> /dev/octopus/by-vol/slot_b
    ln -s <corresponding device> /dev/octopus/by-vol/data
          │
          ▼
 3. Locate the RootFS section (after the OFI header; sections are 4KB-aligned):
    Read the OFI section table in the slot header to get the RootFS section offset / size
    losetup -r -o <offset> --sizelimit <size> /dev/loop0 /dev/octopus/by-vol/slot_${SLOT}
    (When dm-verity is enabled, build a verity mapping on top of loop0 instead; the root hash
     comes from the signature-verified OFI header)
          │
          ▼
 4. Loader unified mounting:
    mount -t squashfs -o ro /dev/loop0 /ro_root
    mount -t tmpfs -o size=<limit>,mode=0755 tmpfs /rw_overlay
    mount -t overlay -o lowerdir=/ro_root,upperdir=/rw_overlay/upper,workdir=/rw_overlay/work overlay /new_root
    mount /dev/octopus/by-vol/data /new_root/data     (on failure see degradation rules below)
    mount --move /dev  /new_root/dev                  (preserves by-vol symlinks)
    mount --move /proc /new_root/proc; mount --move /sys /new_root/sys
    exec switch_root /new_root /sbin/octopus_supervisor
```
**No matter how the storage medium changes, the mounting logic from step 2 onward stays the same; medium differences exist only in step 1.**

- **tmpfs must be size-limited**: All OverlayFS upper-layer writes consume RAM. For compact models, about `size=4m` is recommended; when full it returns `ENOSPC` rather than exhausting memory and triggering OOM;
- **Degradation rules when DATA mount fails**: If mounting `/data` fails (ext4 journal replay failure, UBIFS metadata corruption), the Loader must not stall in initramfs. It first tries a read-only mount and copies readable configuration into memory; if that still fails, it substitutes tmpfs for `/data`, continues booting, and leaves a marker on the kernel command line and in `/run/octopus/degraded`. After the management plane starts it raises an alarm: "Data area corrupted; running with factory configuration." **Whether to automatically reformat DATA must be a configurable policy**; by default DATA is rebuilt only after 3 consecutive mount failures, to avoid wiping user data due to a single sporadic error;
- **System state that must persist**: The OverlayFS upper layer is lost on reboot. SSH host keys, TLS certificates, random seed, last-known timestamp, `machine-id`, etc. must be placed explicitly in `/data/system/` and mapped back to their original paths via symlinks or bind mounts; writing to `/etc` cannot be relied on to persist them.

---

### 4.4 OFI (Octopus Firmware Image) Image Specification

OFI consists of "fixed header + section descriptor table + signature + 4KB-aligned section data"; all multi-byte fields are **little-endian**:

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
|        Security Version (anti-rollback counter, monotonic,    |
|                          uint32)                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Board Compatible (e.g. "rk3568-gw-v1", 32 bytes,       |
|                          NUL-padded)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Section descriptor table, 56 bytes per entry, Section Count  |
|  entries:                                                     |
|    Section Type (uint32: Kernel / RootFS / Model / ...)       |
|    Reserved     (uint32, keeps 8-byte alignment)              |
|    Section Offset (uint64, relative to image start, 4KB-      |
|                    aligned)                                   |
|    Section Size   (uint64)                                    |
|    SHA-256 of Section (32 bytes)                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  dm-verity root hash + salt (optional, for the RootFS section)|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Signature (ECDSA-P256 64B / RSA-2048 256B, per Sig Algo)     |
|  Coverage: all header bytes from Magic to end of verity field |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Section data (4KB-aligned) ...                               |
```

- **Only the header is signed**: The header contains each section's SHA-256, and the signature covers the whole header, so it indirectly covers all sections. This allows **verifying the header signature first, then streaming the write and verifying each section's hash**; a 64MB-RAM device need not buffer a tens-of-MB image in RAM;
- **Compatibility and anti-rollback**: A mismatched `Board Compatible` is rejected outright; a `Security Version` lower than the device's recorded minimum (stored in OTP / eMMC RPMB / protected ENV) is rejected outright, preventing an attacker from flashing back an old firmware that is validly signed but has known vulnerabilities. `Firmware Version` is used only for display and configuration migration, never as a basis for security decisions;
- **Section independence**: Kernel section (FIT image containing DTB and initramfs), RootFS section (SquashFS base system), Model section (AI model layer, optional). Section data is 4KB-aligned, so the Loader can mount directly via loop-device offsets without unpacking first;
- **Incremental updates**: OTA may download only changed sections (e.g., only AI model weights), but what is written to the target slot must be a complete image: unchanged sections are copied from the currently running slot, and after writing, each section is verified against the new header.

---

### 4.5 A/B Dual-Slot Boot Arbitration State Machine

```text
                   [Initial production flashing complete]
                               │
                               ▼
                       ┌───────────────┐
                       │  SLOT_A_VALID │◄─────────────────────────┐
                       └───────┬───────┘                          │
                               │ OTA triggered: mark B=bad →      │
                               │ write B                          │
                               │ after verification               │
                               │ upgrade_available=1              │
                               │ set boot_slot=b, bootcount=0     │
                               ▼                                  │
                       ┌───────────────┐                          │
                       │ SLOT_B_TRYING │                          │
                       └───────┬───────┘                          │
                               │                                  │
                 U-Boot boot   │ bootcount++                      │
                               │ verify OFI signature             │
                               ▼                                  │
                   [Reached Linux user space?]                    │
                    /                    \                        │
          Success  /                      \  Boot hang / watchdog reset /
                  ▼                        ▼ verification failure / Panic
     [octopus_app self-check passed] [U-Boot check: bootcount > 3?]
                  │                                │
                  │                                ├─ No: retry booting B
                  │ After health gate passes,      │
                  │ call HAL to confirm            │
                  │ upgrade_available=0            ▼ Yes: trigger rollback
                  │ slot_b_state=valid     ┌───────────────┐
                  ▼                        │ ROLLBACK_TO_A │
           ┌───────────────┐               └───────┬───────┘
           │  SLOT_B_VALID │                       │ boot_slot=a, slot_b_state=bad
           └───────────────┘                       │ record rollback reason, report
                                                   │ from user space
                                                   └──────────────────────┘
```

> **⚠️ Pending revision (see item 9 in 13.1)**: Step 1 in this section, "mark the target slot `bad` before starting to erase/write," prevents a half-written image from being treated as a valid slot, but during the write window it makes **the running slot the only valid slot on the entire device**. The concurrency and power-loss semantics of this window are not yet finalized; this item must be resolved before implementing `cmd_ofi.c` / `firmware_worker` in Phase 3. Likewise, the `bootcount > 3` check below does not yet distinguish reset reasons (see item 10 in 13.1).

- **Boot counter (bootcount) protection**:
  - Boot parameters are stored in U-Boot's dual-copy environment area;
  - Each time U-Boot is about to boot a slot in the `TRYING` state, it first increments `bootcount` and persists it;
  - If the unit suffers sudden power loss or a deadlock reset before fully booting into user space and confirming, `bootcount` keeps accumulating;
  - Once `bootcount > 3`, U-Boot forcibly deems the slot broken and automatically falls back to booting the stable slot, preventing the device from entering an infinite reboot loop.
- **Write order (recoverable at any power-loss point)**:
  1. First mark the target slot `bad` and persist ENV, **then begin erasing/writing**. Otherwise, if the target slot originally held the old fallback version and power fails mid-write, U-Boot might treat the half-written image as a valid slot;
  2. Verify header signature → stream write → read back and verify SHA-256 section by section;
  3. Once all verification passes, write ENV in one go: `boot_slot=b`, `slot_b_state=trying`, `upgrade_available=1`, `bootcount=0`;
  4. When a rollback occurs, U-Boot marks the failed slot `bad` and no longer tries it until the next OTA overwrites it. After user space starts and reads ENV to find that "a rollback just happened," it must report to the cloud and record the reason (bootcount exceeded, signature verification failure, watchdog reset).
- **Watchdog coverage is required between U-Boot and user space**: The fallback path "boot hang → watchdog reset → bootcount accumulates" holds only if the hardware watchdog is already enabled in U-Boot and keeps counting until `octopus_app` takes over. The kernel must enable `CONFIG_WATCHDOG_HANDLE_BOOT_ENABLED` and use `watchdog.open_timeout=` to bound how many seconds user space has to open the device. Otherwise, if the kernel or initramfs hangs, nothing triggers a reset and the device is stuck half-booted.
- **Confirmation Point**:
  - Minimum requirement: `octopus_app` starts normally, core modules finish initialization, and the watchdog thread begins scheduling normally;
  - **Configurable health gate**: For remotely deployed devices, "the process is up" is not enough — a new firmware may have a broken network driver and never reconnect to the management platform. By default it is recommended to require "one successful communication with the management platform within N minutes," or to confirm only after product-defined self-checks pass; if not satisfied before timeout, the device reboots proactively and lets bootcount handle the rollback;
  - After successful confirmation, clear `upgrade_available`, write `slot_b_state=valid`, and update the anti-rollback counter (if this firmware's `Security Version` is higher). **The anti-rollback counter may only be updated after successful confirmation**; otherwise, if the new firmware fails to boot, it can never roll back to the old version.
- **Linking configuration with slots**: A new firmware may migrate the configuration format on first boot. Before OTA switches slots, snapshot the current `startup-config` as `startup-config.slot_a`; if a rollback occurs, the old firmware detects the rollback marker on boot and loads its own snapshot first, rather than reading the new-format configuration with the old parser (lenient parsing would drop unrecognized lines, and silently discarding security-related configuration is unacceptable).

---

### 4.6 Physical Media Fault Tolerance and Bootloader Corruption Prevention

#### (1) SPI-NAND Flash Physical Bad-Block Tolerance and Alignment (Bad Block Skipping)
- **Problem and risk**: SPI-NAND is raw Flash; chips are allowed to ship with 1%–2% physical bad blocks, and new bad blocks appear over the service life; data that stays read-only for long periods also suffers bit flips due to read disturb and retention decay. Writing MTD partitions directly with `dd` is strictly prohibited;
- **Firmware slots use UBI**: SLOT_A / SLOT_B are UBI static volumes. OTA writes whole volumes via `UBI_IOCVOLUP` (the mechanism behind `ubiupdatevol`); bad-block replacement, wear leveling, and relocation after bit flips are all handled by UBI. U-Boot reads the same volume with `ubi read`, so both ends see the same logical addressing and there is no "asymmetric skip topology" problem;
- **Bad-block-skipping writes for raw MTD regions**: BOOT / ENV / FACTORY remain raw MTD. When production or rescue tools write them, they call `ioctl(fd, MEMGETBADBLOCK)` to detect bad blocks, **skipping a bad block and continuing to the next one**; U-Boot reads with the same skip semantics (raw NAND uses `nand_read_skip_bad()`; SPI-NAND goes through the MTD subsystem's `mtd read`, which skips bad blocks automatically). A bad-block margin must be reserved at the end of each partition; otherwise a single new bad block at runtime would push the image tail out of the partition.

#### (2) Top-Level Security Rule: The Bootloader Must Never Be OTA-Updated
- **Single-point bricking risk**: On the vast majority of hardware, the Flash head (Sector 0) holds only one copy of SPL / U-Boot with no A/B backup. A power loss during flashing will certainly brick the device;
- **Octopus mandatory requirement**: OFI firmware upgrade packages **must never carry a Bootloader**. U-Boot is fixed once on the production line; OTA only ever updates the kernel and root file system (A/B dual slots).

#### (3) Last-Resort Hardware Unbricking
1. **Primary/backup SPL mechanism (SoC-dependent)**: If the SoC BootROM supports multi-copy search, place two independent SPL copies at its defined locations; when the primary SPL fails verification, the BootROM automatically tries the backup copy. Whether this is supported and where copies go must follow the SoC manual; SoCs differ greatly;
2. **Hardware rescue interface (Maskrom)**: The hardware PCB must expose a Recovery button or a specific pull-up test point. When Flash contents are entirely corrupted, holding this key during power-on forces the SoC into USB flashing mode, avoiding chip removal for unbricking.

---

### 4.7 Secure Boot Chain

1. **Root of Trust**: Burn the OEM public key hash into the SoC's internal OTP / eFuse and enable hardware-enforced signature verification;
2. **Second-stage loader verification**: ROM code verifies the SPL signature -> SPL verifies the U-Boot image signature;
3. **Kernel and system verification**: U-Boot parses the OFI header, uses its built-in public key to verify the section checksums and the RSA-2048 / ECDSA signature, then loads Kernel + DTB;
4. **Root file system protection**: The Linux kernel mounts the read-only SquashFS root file system via `dm-verity` to prevent runtime tampering. The verity root hash must come from the signature-verified OFI header and be passed by U-Boot via the kernel command line or the initramfs inside the FIT; if the root hash can be modified freely, dm-verity is meaningless;
5. **Anti-Rollback**: A signature only proves "this firmware was released by the vendor"; it cannot prevent flashing back a vulnerable old version. The device stores the minimum acceptable `Security Version` in OTP / eFuse / eMMC RPMB, and both U-Boot and the user-space upgrade service must check it. Models without such hardware storage can only fall back to recording it in ENV; in that case documentation must state that this guards only against operator mistakes, not physical attacks;
6. **Key management**: Signing private keys are stored only on an offline signing machine or HSM; CI requests signatures through a controlled interface, and private keys never enter build servers. Because the Bootloader is not OTA-updated, U-Boot must ship with multiple public keys (or public key hashes) built in, with revocation status recorded in OTP bits. If one private key leaks, new firmware signed with a backup key is delivered via OTA and burns the revocation bit, without returning devices to the factory.

---

### 4.8 User-Space Storage Abstraction Library (`liboctopus_storage`) and Adaptive File System for the DATA Volume

#### (1) Unified User-Space C Interface: `octopus_storage.h`
Provides a unified interface to the application layer (firmware upgrade helper `firmware_worker`, configuration management), fully eliminating hardware differences:

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

*(Code comments above, translated: `OCTOPUS_STORAGE_UPDATE` — "whole-volume rewrite, allowed only for SLOT_A / SLOT_B"; `octopus_storage_open` — "opens a logical volume (internally detects whether the backend is MTD, UBI, or an MMC block device); UPDATE mode must supply `total_size`: UBI whole-volume update requires declaring the byte count up front, the NOR backend uses it to pre-erase, and the eMMC backend uses it to check partition capacity"; `octopus_storage_write` — "sequential streaming write (internally handles raw-NAND bad-block skipping, NOR sector erase, eMMC block alignment)".)*

#### (2) Adaptive File System Selection for the Persistent Data Area (`DATA` Volume)
The `DATA` volume holds user configuration files (`startup-config`) and the system crash black box, and must be highly resistant to sudden power loss:

```text
 ┌────────────────────────────────────────────────────────┐
 │ Unified mount point: /data (via POSIX open/fsync/rename)│
 └──────────────────────────┬─────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
 [Single NOR Flash (64M)] [Single NAND Flash]  [eMMC / NVMe]
   Capacity: 2MB ~ 4MB    Capacity: 30~100MB   Capacity: 1GB ~ 32GB
   FS: JFFS2              FS: UBIFS            FS: ext4 (data=journal)
   (mainline kernel,      (wear leveling,      (high performance,
    small-partition        bad-block aware)     write-ahead journal)
    friendly)
```

- **Why NOR uses JFFS2 rather than LittleFS**: LittleFS has an excellent power-loss-resilient design, but it is not in the mainline Linux kernel and can only be used via FUSE; mounting `/data` in the initramfs stage would require shipping an extra FUSE daemon, which is unsuitable for a system data area. On a ~3MB NOR partition, the fixed overhead of UBI + UBIFS is too large, so JFFS2 is the most practical choice in the mainline kernel. Its cost is that mount time grows with data volume, so on NOR models `/data` holds only configuration and a small amount of logs;
- **eMMC itself can also fail on power loss**: File system journaling protects only metadata consistency; it cannot protect the state of the eMMC controller's internal FTL at the instant of power loss. Industrial models should choose industrial-grade eMMC with power-loss protection, and hardware should add power-loss detection (voltage-monitoring GPIO + hold-up capacitor), stopping writes and calling `sync` immediately when power loss is detected;
- **Unified persistence red line**: Whether the backend is JFFS2, UBIFS, or ext4, management-plane persistence uniformly follows the POSIX atomic replacement path `tmp -> fsync(file) -> rename -> fsync(dir)`; see 7.4.

---



## 5. Runtime Environment: Loader Root File System and Supervisor (PID 1)



### 5.1 octopus_loader: Dynamic Root File System Composition

Before entering the user-space main system, `octopus_loader` in the initramfs assembles the root file system:

1. **Determine the slot**: Read `octopus.slot` from `/proc/cmdline`;
2. **Handle early-boot factory reset (fallback path)**: Read `octopus.factory_reset=1` from `/proc/cmdline`. If present, **before mounting DATA**, process the user data area according to the wipe list in [reliability 1.2](reliability/00-critical-topics.md) (preserving the FACTORY partition and production calibration values), and leave a `factory-reset-applied` marker under `/run/octopus/` for user space to report. This path exists because if a particular user configuration causes `octopus_app` to crash on startup, user space never reaches the button-detection step, and only the Loader can recover;
3. **Mount the read-only base**: Mount the RootFS section from the current slot's OFI at `/ro_root` via loop offset (or dm-verity);
4. **Mount the in-memory temporary layer**: Mount a size-limited `tmpfs` at `/rw_overlay`;
5. **Compose OverlayFS**: Union-mount the read-only and writable layers at `/new_root`;
6. **Mount persistent data**: Mount the DATA volume at `/new_root/data`; on failure, apply the degradation rules in 4.3;
7. **Switch root and hand over control**: After moving `/dev`, `/proc`, `/sys` via `mount --move`, execute `exec switch_root /new_root /sbin/octopus_supervisor`. In the initramfs scenario only `switch_root` may be used, not `pivot_root`.

See 4.3 (2) for the exact commands.



### 5.2 PID 1 Selection Verdict: Why a Custom Lightweight Supervisor


| Candidate                   | Memory footprint        | Process supervision                    | Crash self-healing & dependency management                  | Embedded suitability assessment                                        | Verdict     |
| ---------------------- | ----------- | ------------------------- | -------------------------- | ----------------------------------------------- | -------- |
| **systemd**            | Over ten MB ~ tens of MB (component-dependent) | Very strong; full cgroup / socket activation | Complete, but extremely complex mechanisms                 | Heavy glibc dependency, many components, high trimming/maintenance cost; overhead too large on 64MB models                | **Rejected**   |
| **busybox init**       | < 1MB       | Weak; serial inittab startup + `respawn`    | No dependencies, no readiness notification, no backoff or health checks            | Too primitive to support automatic isolation and recovery of critical service crashes                        | **Rejected**   |
| **runit / s6**         | 2MB ~ 4MB   | Strong; managed via file descriptors and pipes            | Can start processes, but lacks topology awareness and system introspection integration      | Complex directory and script structure; no native linkage with internal configuration transactions and health reporting                    | **Reference** |
| **octopus_supervisor** | **< 2MB**   | **Purpose-built, covering only necessary system duties**        | **Declarative DAG dependency startup + stepped backoff restart** | **Tailor-made for embedded, pure C implementation (<2000 lines), transparent and controllable** | **Adopted** |




### 5.3 Core Responsibilities and Boundaries of octopus_supervisor

1. **Reap zombie processes**: As PID 1, loop on `waitpid(-1, &status, WNOHANG)` to fully eliminate zombie resource leaks after child processes are orphaned;
2. **Capture system signals**: Listen for `SIGTERM`, `SIGINT`, `SIGPWR` and perform a two-phase orderly shutdown (first notify services to save data; after a timeout, send `SIGKILL` and unmount file systems);
3. **Declarative service supervision**:
  - Read lightweight service configuration files from `/etc/octopus/services.d/` (kernel module loading is also one of these services; there are no separate `/etc/init.d` scripts);
  - Start services concurrently according to the Dependency DAG. Dependencies are judged on **readiness**, not "process has forked": when a service is ready it writes one line to the notification fd passed by supervisor (the same idea as s6's readiness mechanism); a readiness timeout is treated as a startup failure;
  - Monitor child exit events: if the core management plane (`octopus_app`) exits, record the exit code and apply exponential backoff restart; if it crashes consecutively beyond a threshold within a short time, stop restarting, record to the black box, and let the hardware watchdog reset the whole unit (see watchdog ownership below).
4. **Watchdog ownership (consistent with 9.1)**: `/dev/watchdog` is owned exclusively by `octopus_app`; supervisor does not feed the watchdog. The kernel enables `CONFIG_WATCHDOG_NOWAYOUT`, so the watchdog does not stop when an `octopus_app` crash closes the fd. Therefore:
  - The total time for `octopus_app` to restart and initialize up to reopening the watchdog must be less than the hardware timeout (20 seconds); the exponential backoff ceiling must be set within this budget;
  - After the crash threshold is exceeded, supervisor need do nothing; the watchdog timeout naturally resets the unit, and the reset reason is read on the next boot.
5. **Fallback for PID 1 itself**: PID 1 exiting causes a kernel panic. The kernel command line must include `panic=5` (auto-reboot 5 seconds after panic instead of hanging forever), and enabling `panic_on_oops=1` is recommended; supervisor itself stays single-threaded, does no dynamic loading, and allocates no memory after startup.
6. **Service logs**: Each service's stdout / stderr is taken over by supervisor, written to an in-memory ring log, and forwarded to syslog by level; services are not allowed to write unbounded log files directly to `/data`.

---



## 6. Management Plane Core: octopus_app and the Three Minimal Interface Contracts



### 6.1 octopus_app Positioning and Internal Thread Model

`octopus_app` is the system's **management-plane orchestration and state center**; heavy blocking I/O inside it is strictly prohibited.

```text
 ┌──────────────────────── octopus_app ────────────────────────┐
 │                                                             │
 │  ┌───────────────────────────────────────────────────────┐  │
 │  │ Main event loop (Reactor Loop: epoll / libevent)      │  │
 │  │  - CLI terminal command interaction                   │  │
 │  │  - Web JSON-RPC request dispatch                      │  │
 │  │  - Local IPC request dispatch (UDS)                   │  │
 │  │  - Configuration transaction commit validation        │  │
 │  └──────────────────────────┬────────────────────────────┘  │
 │                             │ Async dispatch                │
 │                             ▼                               │
 │  ┌───────────────────────────────────────────────────────┐  │
 │  │ Worker thread pool (fixed 2~4 threads)                │  │
 │  │  - Execute slow board HAL ops (LED toggling, EEPROM)  │  │
 │  │  - Background config serialization & atomic persist   │  │
 │  │  - Wake external firmware upgrade helper              │  │
 │  │    (firmware_worker)                                  │  │
 │  └───────────────────────────────────────────────────────┘  │
 │                                                             │
 │  ┌───────────────────────────────────────────────────────┐  │
 │  │ Dedicated watchdog inspection thread (exclusive high  │  │
 │  │ priority)                                             │  │
 │  │  - Collect heartbeat tokens from all modules          │  │
 │  │    (Heartbeat Matrix)                                 │  │
 │  │  - Inspect internal mutex health (lock_watchdog)      │  │
 │  │  - When all healthy, ioctl(wdt_fd, WDIOC_KEEPALIVE)   │  │
 │  └───────────────────────────────────────────────────────┘  │
 └─────────────────────────────────────────────────────────────┘
```

---



### 6.2 Frozen Interface 1: Board Hardware Abstraction `board_hal.h`

This is the physical boundary decoupling the system from specific SoCs. All board ports need only implement this standard C header in `liboctopus_board`:

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

> This header is the **sole authoritative definition** of the Board HAL. HAL snippets appearing in `reliability/00-critical-topics.md` and `guides/02-switch-hal-drivers.md` are illustrative only; this section prevails.

---



### 6.3 Frozen Interface 2: Module Lifecycle and Registration Contract `module.h`

All business modules inside `octopus_app` (system, network, firmware, routing adapters) must follow this common contract:

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



### 6.4 Frozen Interface 3: Unified Configuration Model and Transaction Interface `config.h`

This is the sole channel for concurrent configuration writes from multiple management-plane entry points, establishing the Candidate-Running transaction model:

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



## 7. Configuration Center and Concurrent Transaction Semantics



### 7.1 Single Source of Truth In-Memory Model

Serial CLI, Web, cloud JSON-RPC, the local HDMI/LCD UI, and other local processes all read and write the same configuration; private persistence by any of them is prohibited. Memory always maintains two tree-shaped state structures protected by a read-write lock:

- **Running Config**: The runtime configuration currently in effect on the system;
- **Candidate Config**: The candidate configuration being modified and validated, not yet committed.



### 7.2 Candidate-Running Two-Phase Commit Transaction Mechanism

```text
  [CLI session]  [Web]  [Cloud RPC]  [HDMI/LCD local UI]   [Local IPC]
       │            │          │                │                  │
       └────────────┴──────────┴────────────────┴──────────────────┘
                                       │ Compete for transaction lock
                                       ▼
                         ┌───────────────────────────┐
                         │ Create txn: cfg_begin()   │
                         │ Clone a Candidate copy    │
                         └─────────────┬─────────────┘
                                       │
                                       ▼
                         ┌───────────────────────────┐
                         │ Batch execute: cfg_set()  │
                         └─────────────┬─────────────┘
                                       │
                                       ▼
                         ┌───────────────────────────┐
                         │ Pre-validate: cfg_verify()│
                         │ Cross-module param and    │
                         │ conflict checks           │
                         └─────────────┬─────────────┘
                                      / \
                       Validation    /   \ Validation
                           passed   /     \ failed
                                   ▼       ▼
           ┌───────────────────────────┐  ┌───────────────────────────┐
           │ Commit txn: cfg_commit()  │  │ Abort txn: cfg_abort()    │
           │ 1. Check base revision    │  │ Release Candidate copy    │
           │ 2. Call module apply hooks│  │ Release txn lock          │
           │    in dependency order    │  └───────────────────────────┘
           │ 3. Replace Running only if│
           │    all succeed; revision  │
           │    + 1, send notification │
           │ 4. Mark "pending persist" │
           └───────────────────────────┘
```

- **Optimistic concurrency rather than long-held locks**: `cfg_begin()` only records the Running revision at that moment and does not hold a global lock for long. At commit time, if the revision has been advanced by another entry point, it returns `ERR_CONFLICT`, and the caller re-reads, merges, and commits again. This way, a web page that opens an edit box and is left idle does not lock out the serial console or local display; idle transactions expire automatically after `lease_ms`. The real critical section is only steps 1–3, which is very short;
- **Apply failures must roll back**: If a module fails to apply in step 2 (e.g., the NIC rejects a new IP), modules that applied successfully re-apply the old configuration, Running stays unchanged, and `ERR_APPLY_FAILED` is returned. A state where "the in-memory tree has new values but hardware is stuck halfway" is not allowed;
- **Confirmed commit (commit confirmed)**: When IP, routing, firewall, or VLAN is modified remotely from the cloud or Web, a misconfiguration may disconnect the device from management, with no way to change it back remotely. Such changes should use `cfg_commit_confirmed()`: after commit, a `cfg_confirm()` must be received within a bounded time (e.g., 120 seconds), otherwise the configuration automatically reverts to its pre-commit state. This is an essential capability for remotely operated gateways;
- **Audit**: Each commit records the source channel, user, revision, and a summary of changed paths, written to `/data/log/audit` and subject to log quotas.


### 7.3 Persistent Storage Format

The persistence format borrows the mature running-config / startup-config practice from the networking equipment industry and uses **hierarchical plain text (Cisco style)**, for example:

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

- **Advantages**:
  - Human-readable during power-loss recovery; field serial debugging can inspect errors directly in the text;
  - Easy Git-diff style version comparison;
  - The parser supports line-by-line scanning with very low memory usage.
- **Single source of Schema**: The CLI command tree (`.ocli`), JSON-RPC paths (JSON Pointer), and the text format above must all be generated from the same Schema rather than hand-written separately; otherwise the "multiple parsers each built separately" problem in 1.3 merely moves from the entry points to the three representations;
- **Sensitive fields must not be in plain text**: User passwords are stored only as salted hashes (e.g., `sha512-crypt` / `yescrypt`); reversible secrets such as Wi-Fi keys, VPN private keys, and cloud tokens are encrypted with a key derived from the SoC's unique hardware key and shown in the text as `secret enc:<base64>`. `show running-config` and configuration export mask them by default. "Human-readable" does not mean "secrets readable."



### 7.4 Power-loss Atomicity

Opening `startup-config` directly in overwrite mode is strictly prohibited. All configuration persistence strictly follows the POSIX atomic replacement pipeline:

```text
 1. Write temp file: open("/data/config/startup-config.tmp", O_CREAT|O_WRONLY|O_TRUNC)
    Append a checksum trailer line at end of file: "!checksum crc32=<hex> length=<bytes>"
        │
        ▼
 2. Force physical persistence: fsync(fd), then close(fd)
        │
        ▼
 3. Keep previous version: rename("startup-config", "startup-config.bak")   (skipped on first save)
        │
        ▼
 4. POSIX atomic replace: rename("startup-config.tmp", "startup-config")
        │
        ▼
 5. Persist directory: fsync(open("/data/config", O_RDONLY|O_DIRECTORY))
    (rename guarantees atomicity only, not durability; without fsync on the directory,
     a power loss may revert to before the rename)
```

- **Load order**: `startup-config` checksum trailer valid → use it; otherwise try `startup-config.bak`; if both are bad → load `default-config` from the read-only root file system and raise the alarm "configuration has reverted to factory defaults." Without a checksum trailer, there is no way to distinguish "a complete old file" from "a truncated file," and thus no basis for choosing the backup;
- **Separate commit from persistence, with write coalescing**: `cfg_commit()` only changes Running; persistence is coalesced in the background, e.g., "write only after 2 seconds with no new commits after the last one," with a cap on writes per hour. A brightness slider or volume knob on the local display can generate dozens of commits per second; persisting each one would rapidly wear out NOR/NAND. Fields that "only affect runtime state and need not be saved" (e.g., temporary debug switches) are marked `volatile` in the Schema and excluded from persistence.



### 7.5 Cross-Firmware-Version Schema Migration

When firmware upgrades from V1.0 to V2.0, fields added or removed by modules may cause parameter incompatibility. Octopus establishes the following parsing rules:

1. **Version metadata tag**: The first line of the text must state the firmware version that generated it (e.g., `!version 1.0.4`);
2. **Tolerant Parsing**: If new firmware encounters a deprecated or unrecognized historical syntax line, it **logs a Warning and skips it automatically**; it must never abort parsing or cause a boot Panic;
3. **Factory-default fallback**: Configuration items newly added in the new firmware that are missing from the historical configuration automatically use the factory defaults built into the current version's code;
4. **Explicit migration rules**: Changes such as field renames, unit changes (e.g., seconds to milliseconds), or splitting one field into two would silently lose user settings if handled only by "skip + default." Each module provides ordered migration functions keyed by `!version` (executed step by step v1→v2→v3), and the migration result is written back on the first save;
5. **Security-related lines must not be silently skipped**: When parsing of firewall, access control, or authentication configuration lines fails, the module enters a "deny all" secure default state and raises an alarm rather than treating them as absent; otherwise a single skipped rule could mean a port is opened;
6. **Downgrade scenarios**: When A/B falls back to old firmware, the old parser faces new-format configuration. See "Linking configuration with slots" in 4.5: load the pre-switch snapshot first.

---



## 8. Control Channels, Peripherals, and Extension Subsystems



### 8.1 CLI Engine `octopus_cli`

- **Syntax DSL (`.ocli`)**: A lightweight declarative DSL describes the command tree, argument completion, and privilege levels;
- **Implementation strategy**: The host-side code generator `ocli_tool` compiles it into pure C struct descriptor tables;
- **Pragmatic binding of the underlying engine**: Prefer integrating the mature, lightweight `linenoise` or `libcli` for terminal driving, Tab completion, and line editing, avoiding a home-grown VT100/ANSI terminal console and greatly reducing R&D cost.



### 8.2 Web Access and Security Baseline

Given the security risks of industrial gateways exposed on networks, the following security baseline is established:

1. **Architectural decoupling**: Use an industry-proven, high-security HTTP server (e.g., Hiawatha) that serves only static pages (HTML/JS) and a FastCGI proxy; front-end pages have no permission to execute shell commands directly;
2. **JSON-RPC channel protection**:
  - **Session authentication**: Based on secure HTTP-only cookies or Bearer tokens, with automatic logout on timeout;
  - **CSRF protection**: Every non-safe mutating request must carry an anti-CSRF random token;
  - **Transport security**: Minimum TLS 1.2, preferring TLS 1.3 when the library supports it; supports device self-signed certificates and factory-provisioned CAs. TLS 1.3 requires OpenSSL ≥ 1.1.1 or mbedTLS 3.x, which the OpenSSL bundled with some old BSPs does not meet — one of the reasons security components cannot be left to the vendor BSP (see [reliability topic 4](reliability/00-critical-topics.md));
3. **No universal default password**: Regulations such as ETSI EN 303 645, the EU CRA, and the UK PSTI prohibit all devices from sharing the same default password. The initial password is generated per device and printed on the nameplate (written to FACTORY on the production line), or a password must be set on first login. After factory reset, the device returns to its "device-unique initial password," not to `admin/admin`;
4. **Local IPC also requires authentication**: `/var/run/json_ipc.socket` can change all system configuration. The management plane obtains the peer uid/pid via `SO_PEERCRED` and performs path-level authorization based on caller identity: the Web server process may only proxy requests of authenticated users, the AI process may only read/write `ai.*` and its own status, and third-party processes are read-only by default. The socket file permission is `0660` and owned by a dedicated group;
5. **Run with least privilege**: Except for supervisor and processes that must operate hardware, the Web server, AI process, and third-party processes all run as separate non-root users, granted only necessary capabilities (e.g., `CAP_NET_BIND_SERVICE`);
6. **Production control of debug entry points**: Diagnostic commands such as the serial root shell, `debug memory dump`, and `debug i2c/spi` passthrough are a large attack surface. Production firmware disables them by default or requires a vendor-signed one-time debug token to enable them; both enabling and use are written to the audit log;
7. **First-boot entropy**: If SSH host keys and self-signed TLS certificates are generated at first boot while the kernel random pool is not yet initialized (on old kernels `/dev/urandom` does not block), the generated keys may be weak and identical across devices. The SoC hardware RNG must be enabled (the driver under `CONFIG_HW_RANDOM`), and `getrandom()` used to block until ready; alternatively, pre-generate keys per device on the production line;
8. **Trustworthy time**: Many low-cost boards have no RTC battery, so power-on time is 1970; TLS certificate validation fails, and log and crash file names become garbled. At boot, restore the last known time from `/data/system/last-time` (monotonically non-decreasing) and calibrate with NTP once online; logs record both the monotonic clock and `boot_id`, without relying on wall-clock time for ordering.


### 8.3 Local HDMI / LCD UI: Display Locally, Configure Through the Same Channel

On devices with displays (industrial HMIs, robot panels, gateway status screens), the screen is drawn by a separate UI process, and keeping a separate copy of configuration is not allowed.

- **Display stays in the kernel**: HDMI, MIPI-DSI, and RGB LCD are brought up by the device tree and DRM/KMS drivers; backlight and touch belong to the board device tree. The UI process only consumes standard display devices, and no GPU compositor is stuffed into `octopus_app`;
- **Configuration consistent with Web, CLI, and cloud**: Reading and modifying brightness, network, and business parameters all go through JSON-RPC into the same Candidate-Running transactions. After a change on screen, the web page, CLI, and cloud immediately see the same running configuration; changes pushed from the cloud are refreshed on the local display at the next read or change notification;
- **The UI process holds no system configuration files**: Writing a separate database or JSON as system configuration inside the Qt/LVGL process is prohibited. UI crashes are restarted by `octopus_supervisor` without affecting configuration already committed in the management plane;
- **Separate pixels from configuration**: Video preview and camera frames go through DRM dmabuf or shared memory, not through configuration transactions.

### 8.4 Dynamic Routing and Switch-Chip Hardware Offload (Switch Offload)

- **Single source of truth for the routing data plane**: The Linux kernel network stack (Kernel FIB) is the only source of truth;
- **Dynamic routing protocol processes**: FRR (zebra/ospfd/bgpd) runs as an independent third-party process and injects computed routes into the Linux kernel via the standard Netlink interface;
- **Hardware forwarding table offload (`switch_offload`)**:
  - Compiled as an optional plugin of `octopus_app`;
  - Asynchronously listens for kernel route changes via Netlink (`RTM_NEWROUTE` / `RTM_DELROUTE`);
  - Automatically programs matching-subnet route entries and L2 FDB entries into the Ethernet switch chip (DSA / Switchdev), achieving line-rate hardware forwarding;
  - **Decoupling advantage**: Without a switch chip, the system naturally degrades to Linux kernel software forwarding, completely transparent to upper-layer business and the management plane.



### 8.5 AI Vision Inference Extension: Full Separation of Control Plane and Data Plane

```text
  [octopus_app main management plane]              [ai_vision_daemon vision process]
  (motion control / param push / alarm handling)  (owns NPU / V4L2 camera)
           │                                                │
           │ 1. Control plane (UNIX Domain Socket: JSON-RPC)│
           ├───────────────────────────────────────────────►│  (start/stop/switch model/tune thresholds)
           │◄───────────────────────────────────────────────┤  (detection events/coords/heartbeat)
           │                                                │
           │ 2. Data plane (POSIX shared memory SHM         │
           │    zero-copy ring queue)                       │
           │    [ RingBuffer: video frames / feature        │
           │      tensor metadata ]                         │
           │◄═══════════════════════════════════════════════╡  (high-bandwidth raw frames)
```

- **Physical isolation guarantee**: NPU-related user-space libraries and the vendor AI runtime (obtained separately by the device maker under the vendor's license, **not included in this repository**) are loaded only in the separate `ai_vision_daemon` process. If that process leaks or crashes fatally, only it exits and is automatically restarted by `octopus_supervisor`;
- **There are two model update modes, which must not be conflated**:
  - **Update with the slot (default)**: The model is the OFI Model section and follows the A/B slot, protected by U-Boot rollback, but switching slots after an update requires a reboot;
  - **Hot update (optional)**: The model package is signed separately and stored in `/data/models/{a,b}/`, with user space maintaining an A/B pointer. An update writes to the inactive directory, verifies the signature, then restarts only `ai_vision_daemon`; if the new model fails to load or fails health checks, the upgrade service switches the pointer back to the old directory. This path is **not protected by U-Boot**; rollback is entirely the responsibility of user space, and it consumes DATA space, so it is not suitable for small-capacity NOR/NAND models.

---



## 9. System-Level High Reliability, Introspection, and Observability



### 9.1 Hardware Watchdog Linked Heartbeat Matrix

Mindless single-threaded blind watchdog feeding is strictly prohibited. The system implements **multi-dimensional heartbeat inspection across all subsystems**:

```text
 [IPC handler thread]   ── heartbeat ──► ┌─────────────────────────────────────┐
 [Route listener thread]── heartbeat ──► │ Watchdog inspection manager         │
 [Main event loop]      ── heartbeat ──► │ (Watchdog Runner)                   │
 [Driver polling thread]── heartbeat ──► │                                     │
                                         │ Rules:                              │
                                         │ 1. All critical thread heartbeats   │
                                         │    are within 10 seconds            │
                                         │ 2. And no internal mutex deadlock   │
                                         │    or hang                          │
                                         └──────────────────┬──────────────────┘
                                                            │ Feed only when all healthy
                                                            ▼ (every 5 seconds)
                                              [Hardware watchdog: /dev/watchdog]
                                              (hardware hard timeout set to 20 s)
```


### 9.2 Lock Watchdog and Deadlock Tracing

System mutexes are wrapped with the `lock_watchdog` facility in `liboctopus_basics`:

- When a thread acquires a critical mutex, the timestamp and thread ID are recorded;
- If a lock is held for more than 5 seconds, the lock watchdog prints a warning log and captures the call stack of the holding thread;
- If it is still not released after 10 seconds, a deadlock is declared: first write all lock states to the black box, then stop feeding the watchdog and let the hardware watchdog reset.
- **Unified timeline** (authoritative across the whole repository): heartbeat every 2 seconds → inspection every 5 seconds → any critical heartbeat or lock hold exceeding 10 seconds is declared a fault → stop feeding → hardware reset at most 20 seconds later. From fault occurrence to whole-unit reset, the worst case is about 30 seconds.



### 9.3 Memory Pressure Ladder and OOM Breakthrough Prevention

A three-tier defense is established on compact gateways. Watermarks are computed as "used memory = MemTotal − MemAvailable":

1. **Warning watermark (75%)**: Notify modules to release non-essential caches and truncate in-memory diagnostic log buffers;
2. **Critical watermark (88%)**: Run degraded, proactively pausing the AI vision inference process or stopping non-essential background scans;
3. **OOM protection mechanism**: Set `oom_score_adj = -1000` for `octopus_app`, ensuring that when the system OOM-Killer triggers, peripheral third-party applications are cleaned up first, preserving the main control plane.

- **Choose detection means by kernel capability**: PSI (Pressure Stall Information) requires kernel 4.20+ with `CONFIG_PSI`, which 4.9-kernel BSPs such as SSD20X lack. The baseline implementation polls `MemAvailable` in `/proc/meminfo` (available since 3.14); when the kernel supports it, PSI triggers are layered on top as an earlier signal of "stalls caused by memory pressure." Likewise for cgroup: old kernels only have cgroup v1, and resource limit configuration must be adapted separately;
- **Side effect of -1000**: Once `octopus_app` is made unkillable, if the leak is in `octopus_app` itself, the OOM-Killer will repeatedly kill other processes, supervisor restarts them, and a loop forms. Therefore `octopus_app` must self-check: when its resident memory exceeds its own budget ceiling, it proactively writes to the black box and exits in a controlled manner, to be restarted by supervisor. **This budget must be specified separately in 2.2** (the existing NFR only specifies a combined "supervisor + app + cli" ≤ 12 MB, which cannot serve directly as a self-check threshold), and peak values per module are exposed in the health snapshot in 9.7;
- **zram (optional)**: 64MB models may enable a small zram as compressed swap to absorb burst peaks, but it adds CPU overhead and makes "memory exhaustion" occur later and more covertly; disabled by default.



### 9.4 Crash Black Box and Persistent Diagnostic Logs

- **Reserved memory (pstore / ramoops)**: A small reserved RAM region (1MB by default; layout in [reliability topic 2](reliability/00-critical-topics.md)) records the last crash stack and console log at the moment of kernel Panic or reboot. **It is effective only across warm resets; data is lost on power loss.** Moreover, whether DDR stays powered during a watchdog reset, and whether SPL/U-Boot/TF-A overwrite this memory, depend on the specific SoC and board; every board must verify during bring-up that "pstore still has records after a watchdog reset" (listed as precondition 2 in 3.4);
- **Symbols and offline decoding**: Production firmware is stripped, so only addresses are captured in the field. Each build retains ELF files with debug symbols, archived and indexed by GNU build-id; crash records include the build-id, each module's load base address (from `/proc/self/maps`), and relative offsets, which are decoded offline back into function names and line numbers on the R&D side;
- **Persistent diagnostic quota**: `/data/log/` has a strict total space quota (e.g., at most 10MB) with multi-file rolling overwrite (Log Rotation), preventing log explosions from exhausting storage space.

---

### 9.5 Temperature and Thermal Management: Over-Temperature Must Not Rely on the Watchdog

For models with an NPU running 24×7 in the field, heat from continuous inference is inevitable, not occasional. **Over-temperature must be handled through tiered degradation; "solving" it with watchdog resets is never allowed** — a reset neither lowers chip temperature nor locates the problem; it only creates a "reboot → rapid reheating → reboot again" thermal loop that accelerates component aging.

There are two temperature sources, and both must be used: the SoC's internal thermal sensors (`thermal_zone`) cover CPU/GPU/NPU junction temperatures; the board-level thermal sensor (HAL interface `octopus_board_get_temperature_millicelsius()`) covers board surface temperature near power components.

| Level | Trigger condition | Action |
| :--- | :--- | :--- |
| **L0 Normal** | Below warning threshold | None |
| **L1 Warning** | Warning threshold (recommended: component limit − 15 °C) | Log and report; lower `ai_vision_daemon` frame rate and inference resolution; pause non-essential background scans |
| **L2 Danger** | Danger threshold (recommended: component limit − 5 °C) | Stop `ai_vision_daemon`; cap CPU/GPU at the lowest usable frequency; suppress non-critical log writes; continuous alarm |
| **L3 Critical** | Above component limit | First `sync` to disk, write the black box marked as over-temperature, then perform a controlled shutdown; **recovery is by external conditions (cooling / power-off maintenance); no automatic reboot** |

- **Relationship with the watchdog**: During L1/L2, heartbeats and watchdog feeding continue as usual, and the system is still considered healthy (temperature is an operating-condition issue, not a fault); upon entering L3, feeding stops as part of the normal graceful shutdown flow, the hard timeout performs the final reset, and the black box is guaranteed to be persisted;
- **Thresholds must be configurable and calibrated per board type**: Thresholds differ by SoC and thermal design and must not be hard-coded; configuration items belong to `octopus_config` (`thermal.*`), with calibrated values measured and filled in during bring-up;
- **No masking**: Under no circumstances may over-temperature alarms be eliminated by "raising the alarm threshold" or "disabling temperature sampling"; such changes amount to abandoning the NFR temperature figures;
- **Verification method**: In a thermal chamber, heat step by step from L0→L3 under full-load burn-in; confirm each level's actions trigger as in the table above, and that the device remains manageable before L3 (can log in, query configuration, report).



### 9.6 Flash Write Budget: Calculating the "10-Year Lifetime"

Component-level erase endurance (about 100K cycles for NOR) is a **component** metric, not the **system** lifetime. System lifetime is determined by "daily write volume of each function ÷ write volume the medium can withstand," and must be reconciled at design time rather than discovering wear-through in the field. Test methods are given in the long-term burn-in and Flash wear chapters of the [test strategy](guides/01-qemu-test-strategy.md) (eMMC reads the `EXT_CSD` lifetime fields, UBI reads erase counters); this section specifies the **design-side** budget requirements.

**Write budget table (one per model, maintained with the architecture document)**:

| Write source | Trigger frequency | Bytes per write | Daily write volume | Existing constraint |
| :--- | :--- | :--- | :--- | :--- |
| `startup-config` atomic persistence | Coalesced persistence, see 7.4 | ~4 KB | TBD | Cap on writes per hour |
| Audit log (`/data/log/audit`) | Each configuration commit | ~200 B | TBD | Subject to total log quota |
| Diagnostic log rotation | Continuous | — | TBD | Total quota 10 MB |
| Crash dump | On crash | ≤ 10 MB | Field-dependent | Trigger frequency should be very low |
| **AI model hot update** (`/data/models/`) | Each model change | Several MB ~ tens of MB | Business-dependent | **This source is the most likely to dominate lifetime and must be accounted for separately** |

- **Pass criterion (part of NFR acceptance)**: Simulate the 10-year cumulative write volume per the table above at an accelerated rate; **the medium erase count must not exceed 50% of the component's rated endurance** (keeping a 2× margin for write amplification and bad-block replacement);
- **Metrics that must be exposed**: UBI erase counts / eMMC `EXT_CSD` lifetime consumption percentage, as read-only metrics in the health snapshot in 9.7;
- **If the budget fails, adjust in this order**: first lower log level and retention period → then switch AI models to "update with the slot" (the default mode in 8.5, which does not consume DATA writes) → only last consider larger-capacity or higher-endurance media. **Lengthening the persistence interval to the point of compromising configuration reliability just to make the numbers work is not allowed.**



### 9.7 Unified Health Snapshot (Read-Only)

To make "system introspection" externally observable (Web status page, cloud inspection, after-sales forensics), the management plane exposes a **read-only, low-cost, side-effect-free** health snapshot interface:

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

*(Field comments above, translated: `rollback_count` — historical rollback count, see 4.5; `slot_state` — corresponds to `octopus_slot_state_t`; `temp_mc` — current temperature in millidegrees Celsius; `thermal_level` — L0~L3 per 9.5; `free_bytes` — remaining space on the DATA volume; `write_budget_pct` — percentage of medium lifetime consumed, see 9.6; `peak_rss_kb` — peak foundation resident memory, see 2.2; `degraded_flags` — bit definitions below.)*

`degraded_flags` covers at least: `DATA_REBUILT` (the DATA volume has been rebuilt), `CONFIG_FALLBACK` (configuration has reverted to factory defaults), `DATA_TMPFS` (DATA mount failed; tmpfs substituted), `WDT_ABSENT` (hardware watchdog could not be opened; see precondition 1 in 3.4), `NO_RTC`, `FACTORY_RESET_APPLIED`.

> This interface is the **only objective observation surface** for the preconditions in 3.4 and the tiered strategies in 9.5 / 9.6: without it, whether those strategies take effect cannot be verified, nor can they be used by after-sales or cloud inspection. It must be a read-only query and must not take on any control responsibility.



---

## 10. Architectural RFC & Decision Records / Trade-offs

This chapter records the 7 most critical **technical decision rationales (RFC / ADR)** made during architectural convergence, explaining "why A rather than B," "what was given up," and "the core trade-off considerations," serving as the cornerstone for open-source community discussion.

### RFC-01: Core Foundation Language — Why Insist on "Lightweight C++" Rather Than "Pure C Fundamentalism"?

- **Point of contention**: Many low-level embedded developers argue the entire system must be written in pure C, considering C closest to the hardware with a stable ABI.
- **Decision**: **"The closer to hardware, the purer the C; the management-plane core uses lightweight C++; all external ABI boundaries are pure C; upper-layer business is free to use multiple languages."**
- **Trade-off rationale**:
  1. **Management-plane reflection efficiency**: The fundamental advantage that lets a high-reliability foundation efficiently support a large number of business modules is `StructReadWrite` template metaprogramming. A single template declaration automatically completes JSON-RPC tree mounting, parameter validation, IPC routing, and configuration replay. Falling back to pure C would require hand-writing lengthy `cJSON` parsing and serialization matching for every struct, inflating code size 3–5× and easily causing segfaults when fields change;
  2. **Concurrency safety and RAII**: `octopus_app` contains multi-threaded locks and dynamic resources; in pure C, a `goto out` path that misses an unlock easily causes deadlocks; C++'s `std::lock_guard` / RAII eliminates lock leaks at the language level;
  3. **Restraint principles (Embedded C++ guidelines)**: Exceptions are prohibited (-fno-exceptions), runtime RTTI is prohibited (-fno-rtti), frequent dynamic memory allocation in the main loop is prohibited, and deep multiple inheritance is prohibited.



### RFC-02: PID 1 Approach — Why a Custom Lightweight Supervisor Instead of an Existing Open-Source Solution?

- **Point of contention**: Is it necessary to write a new PID 1 process? Why not just use systemd, runit, or s6?
- **Decision**: **Build a custom pure-C `octopus_supervisor` of <2000 lines, adhering to an engineering philosophy of minimalism, transparency, and control.**
- **Trade-off rationale**:
  1. **Versus systemd**: systemd together with companions like journald and udevd stays resident at over ten MB to tens of MB and depends heavily on glibc; on models like SSD20X (64MB RAM) its overhead share is too high, and long-term maintenance after trimming is also costly;
  2. **Versus BusyBox init**: BusyBox can only execute scripts serially; `respawn` can restart processes, but there is no dependency topology, readiness notification, or backoff strategy, so it cannot bear the responsibility of high-availability supervision;
  3. **Versus runit/s6**: runit and s6 are excellent, but their configuration directory structures are fragmented, and they cannot form a deep closed loop with the hardware watchdog heartbeat, A/B slot confirmation, `lock_watchdog` deadlock detection, and crash dumps in embedded scenarios. A custom pure-C supervisor tailored for industrial gateways is small, transparent, and controllable.



### RFC-03: Process Isolation and Single-Point Risk — How to Balance High-Performance Sharing with Fault-Domain Isolation?

- **Point of contention**: Traditional architectures often cram all code into one giant process, so a module crash takes down the whole management plane; but splitting everything into microservices incurs huge IPC copy and context-switch overhead.
- **Decision**: **A dual-track architecture: management-plane state kept cohesive (In-Tree) + heavy-I/O helper processes attached externally + heavy AI/protocol workloads in independent processes (Out-of-Tree).**
- **Trade-off rationale**:
  1. **Configuration and state management must be cohesive**: System parameters, network configuration, and device information stay in the same `octopus_app` address space, eliminating the overhead of frequent cross-process IPC for hundreds or thousands of configuration items;
  2. **Strip out high-risk / time-consuming logic**: Easily blocking firmware Flash writes (`firmware_worker`) and log packaging (`diag_collector`) are split into independent helper processes started on demand;
  3. **Physically isolate high-memory workloads and vendor acceleration runtimes**: The AI vision process (which loads the vendor NPU user-space libraries) and complex dynamic routing (FRR) must be independent processes; their crashes are restarted by `octopus_supervisor` within milliseconds and never contaminate the management plane.



### RFC-04: OTA Upgrade Scope — Why Keep the Bootloader Out of Remote OTA?

- **Point of contention**: Some systems support full-disk upgrades including the Bootloader.
- **Decision**: **Remote upgrades cover only the kernel and root file system (OFI); the Bootloader is fixed on the production line and not OTA-updated.**
- **Trade-off rationale**:
  1. **Physical single-point risk**: On the vast majority of embedded SoCs (e.g., RK3568, SSD20X, Allwinner), there is only one physical SPL / U-Boot partition. A sudden power loss while erasing/writing the Bootloader instantly bricks the device, which can then only be returned to the factory;
  2. **Business decoupling**: 99.9% of product feature evolution and bug fixes are concentrated in the Linux kernel and application layer. As long as the Bootloader has stable A/B slot-selection and signature-verification logic baked in, there is no need to touch the Bootloader frequently;
  3. **Last-resort fallback**: Even if both firmware slots are fully corrupted, the hardware Maskrom button on the PCB can still trigger ROM-code unbricking.



### RFC-05: Configuration Center Architecture — Why Insist on Hierarchical Plain-Text Configuration Instead of SQLite or JSON?

- **Point of contention**: Modern systems prefer to store configuration in SQLite or whole-file JSON.
- **Decision**: **Adopt Cisco-style hierarchical plain-text running-config / startup-config.**
- **Trade-off rationale**:
  1. **Field recoverability (Human-readable Debugging)**: When field engineers debug via serial console, plain-text configuration can be read, edited, and troubleshot directly; with binary SQLite or complex formatted JSON, troubleshooting from a serial or rescue shell is extremely painful;
  2. **Atomicity and fault tolerance**: Plain text supports Tolerant Parsing — a deprecated syntax line is simply skipped and the device boots; if the SQLite journal is corrupted by sudden power loss, the database often cannot be opened at all, causing a boot Panic;
  3. **Memory and storage footprint**: A plain-text parser needs only a few hundred lines of code and a few KB of overhead, which is extremely friendly to ultra-low-cost hardware.



### RFC-06: Network Hardware/Software Co-design — Why the Linux Kernel FIB as the Single Source of Truth?

- **Point of contention**: Should forwarding tables be configured centered on a dedicated switch chip's private SDK, as in traditional network equipment, or centered on the Linux kernel network stack?
- **Decision**: **The Linux kernel network stack is the system-wide Single Source of Truth for routing and forwarding tables; the switch chip serves only as a hardware offload accelerator (Switch Offload).**
- **Trade-off rationale**:
  1. **Unified management model**: Without a switch chip, the system naturally runs as a Linux software router; with a switch chip, hardware line-rate forwarding tables are programmed seamlessly by listening via Netlink;
  2. **Seamless ecosystem reuse**: A vast open-source networking ecosystem — FRR, iptables, WireGuard, Docker, and more — is built on the standard Linux kernel network stack. Using it as the source of truth directly inherits all open-source networking assets, without rewriting upper-layer protocols for some switch ASIC's private user-space model.



### RFC-07: Storage Architecture Decoupling — Why Insist on the 6 Logical Volume Abstraction Rather Than Facing Physical Partitions Directly?

- **Point of contention**: Could business scripts or Makefiles simply hard-code `/dev/mtdX` or `/dev/mmcblk0pX` per board?
- **Decision**: **Establish a Storage Abstraction Layer (SAL) that standardizes 6 logical volumes system-wide (BOOT/ENV/FACTORY/SLOT_A/SLOT_B/DATA); the business layer is strictly prohibited from hard-coding physical devices.**
- **Trade-off rationale**:
  1. **Mask inherent physical media gaps**: NOR, NAND, and eMMC differ completely in erase/write units, bad-block characteristics, wear leveling, and Linux access interfaces (MTD character devices vs. MMC block devices). Without abstraction, changing one storage chip would force a complete rewrite of U-Boot boot commands, Loader mounting, and OTA upgrade tools;
  2. **Mitigate supply-chain shortages and rapid board revisions**: Commercial products often switch among single NOR (64M), single NAND (256M), and NOR+eMMC dual media due to chip shortages or cost optimization. Decoupling via logical volumes and driver operation sets (Ops) keeps upper-layer business and OTA logic 100% unchanged; only the device tree and storage backend configuration need adjusting;
  3. **Strictly symmetric read/write bad-block topology**: In SPI-NAND scenarios, firmware slots live in UBI volumes, so user-space writes and U-Boot reads access the same UBI logical addressing; only raw MTD regions such as BOOT / ENV / FACTORY are skipped by both ends under identical rules, fundamentally avoiding upgrade failures caused by raw-Flash bad blocks.

---



## 11. Glossary and Engineering Roadmap Alignment



### 11.1 Glossary


| Term                   | Full name                             | Authoritative definition and responsibility in Octopus                                       |
| -------------------- | -------------------------------- | --------------------------------------------------------- |
| **OFI**              | Octopus Firmware Image           | Octopus's unified sectioned firmware image packaging format, containing the kernel, read-only root file system, and optional model layer.                   |
| **SAL**              | Storage Abstraction Layer        | Storage abstraction layer that masks physical differences among NOR/NAND/eMMC and exposes the 6 standard logical volumes and a unified read/write interface upward.          |
| **Logical Volume**   | Logical Volume                   | Logical volumes (BOOT/ENV/FACTORY/SLOT_A/SLOT_B/DATA), the system's common addressing unit.         |
| **Board HAL**        | Board Hardware Abstraction Layer | Board-level interface defined in pure C (`board_hal.h`), isolating hardware differences in MAC, SN, LED, buttons, and watchdog.      |
| **Supervisor**       | Octopus Process Supervisor       | Minimal PID 1 process supervisor responsible for zombie reaping, signal forwarding, and service dependency lifecycle management.                   |
| **Loader**           | Octopus RootFS Loader            | A small tool running in the initramfs stage that union-mounts SquashFS and tmpfs as OverlayFS. |
| **Running-Config**   | Running Configuration            | The configuration tree currently in effect in memory, protected by the Candidate transaction mechanism and a read-write lock.                   |
| **Startup-Config**   | Startup Configuration            | The plain-text configuration file kept in persistent storage (`/data/config/`), replayed at boot to restore state.             |
| **Heartbeat Matrix** | Watchdog Heartbeat Matrix        | The matrix of heartbeats that core modules inside the main process check in with the watchdog thread; the hardware watchdog is fed only when all are healthy.                     |
| **Lock Watchdog**    | Mutex Lock Watchdog              | Mutex supervisor (`lock_watchdog`) responsible for tracking mutex hold durations, deadlock alarms, and call-stack tracing.             |




### 11.2 Alignment with Repository Roadmap (ROADMAP.md) Milestones


| Phase          | Target version          | Concrete deliverable constraints from this architecture specification                                                                                                            |
| ----------- | -------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **Phase 1** | `v0.2.0-alpha` | Establish the CMake monorepo; implement `board_hal.h` / `module.h` / `config.h` stub definitions on top of `liboctopus_basics`; x86 unit tests; monolithic main program skeleton and basic JSON-RPC communication validation; fuzz testing for the configuration parser and OFI parser. |
| **Phase 2** | `v0.3.0-alpha` | Land `octopus_config` transactions (revision conflict detection, apply rollback, confirmed commit) and atomic persistence; build `octopus_cli` on `linenoise`; complete board HAL driver implementations for RK3568 and SSD20X; `liboctopus_client` (C + Python). |
| **Phase 3** | `v0.4.0-beta`  | Deliver U-Boot `cmd_ofi.c` and the A/B dual-slot arbitration state machine; `liboctopus_storage` (NOR / UBI / eMMC backends); `octopus_loader` and `octopus_supervisor` (PID 1); the `ofi_pack` signing toolchain and anti-rollback counter; full end-to-end automated testing on QEMU ARM64 / ARM32. |
| **Phase 4** | `v1.0.0-rc1`   | Secure Web access (Hiawatha + FastCGI) and local IPC authentication; local HDMI/LCD UI example; optional AI vision process; 10,000-cycle random sudden power-loss resilience burn-in testing; production-line flashing and device-unique credential provisioning tools; board bring-up guide and GPL source delivery process. |

---

## 12. License Boundary

The repository as a whole uses Apache-2.0, but part of Octopus's code must go into GPL projects, and the two must not be conflated:

| Component | Host project | License requirement |
| :--- | :--- | :--- |
| U-Boot patches such as `cmd_ofi.c` | U-Boot (GPL-2.0+) | Must be released as GPL-2.0-or-later. **Apache-2.0 is incompatible with GPLv2**; Apache-2.0 code cannot be merged into U-Boot directly |
| Kernel drivers, DTS | Linux (GPL-2.0) | Drivers GPL-2.0; DTS recommended as `GPL-2.0 OR MIT` dual license, consistent with upstream practice |
| `octopus_loader`, `octopus_supervisor`, `liboctopus_*`, `octopus_app` | User-space code in this repository | Apache-2.0 |
| OFI parsing code shared by U-Boot and user space | Needed on both sides | Recommended as a separate directory dual-licensed `Apache-2.0 OR GPL-2.0-or-later`, or an independent implementation on the U-Boot side |

- GPL parts are kept in separate source directories (e.g., `bootloader/`, `kernel/`), with an SPDX identifier in every file;
- Device firmware contains GPL components such as U-Boot, Linux, and BusyBox, so source-offer obligations must be fulfilled when shipping in volume. The build system must be able to export the corresponding GPL source package and SBOM for each release;
- NPU runtime libraries delivered by vendors in binary form may only be used in user space as separate processes or via dynamic linking, and must not be statically linked into GPL components; such libraries are obtained by the device maker under the vendor's license, and **this repository includes neither their source code nor prebuilt packages**.

---

## 13. Open Issues

> **Reading note (open-source release):** This chapter lists issues that are **not yet finalized**. Do not read the A/B flow in the body of chapters 4–5 as "frozen and ready for mass-production implementation as-is." **§13.1** contains the **Blockers before Phase 3 coding**; most items in the body of §13 are product- or governance-level and can be discussed in parallel with foundation coding, but they do not constitute proof that "never bricks" has been closed out.
>
> For reporting vulnerabilities and security design flaws, see [SECURITY.md](../../SECURITY.md) at the repository root.

The following issues directly affect mass production but still have no design conclusion; discussion via RFC is welcome:

1. **Ownership of real-time control**: Robot motor closed-loop control (1ms-class) should not live in a management process like `octopus_app` that has a Reactor, RPC, and configuration transactions. There are three candidate approaches: an independent real-time process (`SCHED_FIFO` + `mlockall` + CPU isolation, with a PREEMPT_RT kernel when necessary), an external MCU co-processor, or a combination of both. Whichever is chosen, a mechanism must be defined for motors to enter a safe state when the control process or AI process crashes (hardware emergency stop, independent watchdog on the drive side). This falls under functional safety and cannot rely solely on supervisor restarts; for embodied / AI extensions see `guides/03-ai-robot-vision.md`, which is **not on the Phase 1–3 foundation main path**;
2. **Network connection management**: Cellular module (4G/5G) dialing and reset, multi-WAN failover, and Wi-Fi provisioning are the largest source of gateway disconnections, yet currently there is only a single sentence about a "network configuration module";
3. **Remote operations channel**: Most devices sit behind NAT; a device-initiated reverse operations channel (tunnel or cloud relay), its authentication method, and auditing need to be defined;
4. **OTA delivery protocol**: The device-side download protocol (resumable download, rate limiting, battery/time-window checks), manifest format, and staged rollout strategy are not yet defined;
5. **Device identity and credential provisioning**: Private keys should be stored in a Secure Element (SE) or TEE, not in plain text in the FACTORY partition; what fallback scheme applies to models without an SE needs separate evaluation;
6. **Build system and SBOM**: Whether the user-space root file system is built with Buildroot or Yocto, how to decouple from vendor BSPs, and how to produce an SBOM for each release and track CVEs (a hard requirement of the EU CRA); **it is recommended to choose at least one before Phase 1 cross-compilation**;
7. **Privacy compliance**: Cameras recognizing people and uploading to the cloud involve the Personal Information Protection Law (PIPL), GDPR, etc.; local masking, data retention periods, and user consent mechanisms need to be designed;
8. **Repository governance**: [SECURITY.md](../../SECURITY.md) (vulnerability reporting channel and response timelines) is already provided at the repository root. Still pending after the open-source launch: enable GitHub Discussions, verify Issue templates and Advisory links against the actual org/repository name, and fill in a maintainer public email in `CODE_OF_CONDUCT.md` once available.

### 13.1 Blockers Before Phase 3 Coding (Must Be Finalized)

> **BLOCKERS — before Phase 3 coding.**  
> The following items **directly affect the state machine in 4.5 and the "anti-bricking" argument**. Before writing `cmd_ofi.c` / `octopus_loader` / `firmware_worker` / the U-Boot ENV protocol, they must be converged through RFCs and written back into the body of chapter 4; the current narrative in 4.5 **must not** be treated as the sole implementation specification while they remain unresolved.

| ID | Issue | Candidate directions (summary) | Blocks |
| :--- | :--- | :--- | :--- |
| **B1** (§13.1-9) | "Single valid slot" during the partial-write window | `upgrade_phase` vs. marking `bad` first; forbid concurrent ENV writes during the window; whether to retain minimal rescue | A/B write order, "never bricks" messaging |
| **B2** (§13.1-10) | bootcount classification and thresholds | Whether power-loss/external resets count; different thresholds for TRYING vs. VALID | Misjudging healthy firmware as `bad` |
| **B3** (§13.1-11) | Definition of Rescue Mode | Entry conditions; serial/network/`ofi` slot writing; exit path | Closing the loop on "recoverable even with only the Bootloader left" |
| **B4** | Licensing implementation for OFI parsing | Single `Apache-2.0 OR GPL-2.0-or-later` directory vs. dual U-Boot/user-space implementations | Chapter 12 compliance, drift between two copies |
| **B5** | User-space build system | Buildroot external vs. Yocto (linked with §13-6) | Cross-compilation contract from Phase 1 onward |

Detailed pending notes (corresponding to the table above):

9. **"Single valid slot" state during the partial-write window (B1)**: 4.5 currently specifies marking the target slot `bad` and persisting ENV before erasing begins, which makes **the running slot the only valid slot on the device** between "ENV persisted" and "new image fully written and verified." If a power loss interrupts the erase during this window, or the running slot is corrupted at the same time, both slots become unusable simultaneously; the "never bricks" argument in 4.6 and RFC-04 covers only the Bootloader single point, not this state. Pending: (a) instead record upgrade-in-progress via `upgrade_phase` in ENV, keep the target slot's original state during writing, and let U-Boot's image verification (rather than writing `bad` in advance) express "cannot boot"; (b) forbid concurrent upgrades and other ENV writes during this window; (c) whether minimal rescue capability needs to be retained during this window;
10. **Classified counting and threshold decisions for bootcount (B2)**: 4.5 currently declares a slot broken once `bootcount > 3`, without distinguishing reset reasons. Power loss and external resets reflect the power grid rather than firmware defects; after several accumulate they condemn a healthy firmware. Pending: count by `octopus_board_get_reset_reason()` classification (power-loss/external resets not counted or down-weighted), different thresholds for verified slots versus trial slots, and `VALID` slots never marked `bad` due to bootcount;
11. **Definition of Rescue Mode (B3)**: 4.6 and RFC-04 repeatedly rely on "recoverable even with only the Bootloader left," but do not define this state's entry conditions, minimal capabilities (serial / network / `ofi` slot writing / configuration recovery), interface, and exit path. Without this definition, the "never bricks" argument does not close;
12. **Kernel LTS maintenance path**: 2.2 allows a minimum kernel of 4.9 (current vendor BSP), but how security patches will be obtained over a 10-year lifecycle is undecided (long-term patching on the BSP / gradual mainlining / dual-track backporting), which directly determines CVE response capability;
13. **ABI evolution process**: The process for adding fields or changing semantics in the three frozen interfaces in 6.2 / 6.3 / 6.4 is undecided (versioned structs / only new functions allowed / breaking changes allowed in minor versions). Once a frozen interface is broken, "zero migration cost" no longer holds; this process is itself part of the promise;
14. **ADR / RFC decision process**: Chapter 10 has 7 RFCs, but approvers, conflict resolution, and ADR status markers (proposed / accepted / superseded / deprecated) are not defined. After open-sourcing, the community's first question will inevitably touch on this;
15. **Acceptance criteria table**: The NFRs in 2.2 all have figures but no judgment methods (what counts as "bricked," sample sizes, batch sampling rules), and a Flash lifetime acceptance test "simulating 10 years of write volume" is missing (see 9.6). A requirements traceability matrix and risk register are likewise missing;
16. **Production process and supply chain**: Flashing, key injection, board testing, defective-unit rework, shipped-firmware traceability, and long-term supply risks such as NAND end-of-life and the power-loss protection grade of industrial eMMC. Existing item 5 in chapter 13 only covers "device identity credential provisioning";
17. **Volatile path inventory**: 4.3 sets a size limit for the tmpfs backing the OverlayFS upper layer (4m recommended), but does not specify the ownership and quotas of `/etc`, `/var`, and `/tmp` individually. Third-party components routinely write to these paths, and once full they hit `ENOSPC`. A "volatile paths + quota allocation" table is needed rather than illustrative examples.

