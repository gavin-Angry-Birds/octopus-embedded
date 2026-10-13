# Octopus Generic Platform: Key Topics on Mass-Production Deployment and Engineering Boundaries

**English** | [简体中文](./00-critical-topics.md)

> **Note**: The Chinese version is authoritative. This English twin must stay aligned with it; when the two differ, the Chinese version prevails.

> **Document purpose**: Before formal coding begins in the `octopus-core` repository, this document systematically examines and reasons through the **7 core engineering blind spots and architectural boundaries** that cannot be avoided as the system moves from a "lab prototype" toward "commercial mass production and high-reliability, industrial-grade availability". It serves as technical guidance for subsequent dedicated decisions and solution implementation.

---

## Table of Contents

- [Topic 1: Configuration Lifecycle, Version Migration, and Factory Reset (Persistence & Lifecycle)](#topic-1-configuration-lifecycle-version-migration-and-factory-reset-persistence--lifecycle)
- [Topic 2: Hardware Watchdog Integration, OOM Protection Tiers, and Crash Black Box (Reliability & Faults)](#topic-2-hardware-watchdog-integration-oom-protection-tiers-and-crash-black-box-reliability--faults)
- [Topic 3: Factory Data Isolation, Device Credentials, and Security Compliance (Security & Provisioning)](#topic-3-factory-data-isolation-device-credentials-and-security-compliance-security--provisioning)
- [Topic 4: Third-Party Dependency Supply Model and Cross-SoC libc Compatibility Boundaries (Dependencies & Sysroot)](#topic-4-third-party-dependency-supply-model-and-cross-soc-libc-compatibility-boundaries-dependencies--sysroot)
- [Topic 5: Generic High-Performance Shared-Memory Data-Plane Channel and Dual-Plane Coordination (liboctopus_shm)](#topic-5-generic-high-performance-shared-memory-data-plane-channel-and-dual-plane-coordination-liboctopus_shm)
- [Topic 6: Physical Boundaries of Flash Storage Media and Brick-Proof Bootloader Upgrades (Storage & Bootloader Resilience)](#topic-6-physical-boundaries-of-flash-storage-media-and-brick-proof-bootloader-upgrades-storage--bootloader-resilience)
- [Topic 7: System Introspection and High-Reliability Advanced Diagnostics (Diagnostics, Lock Watchdog & Memory Introspection)](#topic-7-system-introspection-and-high-reliability-advanced-diagnostics-diagnostics-lock-watchdog--memory-introspection)

---

## Topic 1: Configuration Lifecycle, Version Migration, and Factory Reset (Persistence & Lifecycle)

System configuration persistence (Running-Config / Startup-Config) is stored in plain-text octopus_config format in the writable persistent data partition on Flash. Over a product lifecycle spanning several years, the following three core engineering specifications have been established:

### 1.1 Configuration Migration Across Firmware Versions (Schema Migration)
- **Problem and challenge**:
  A device is upgraded from V1.0 to V2.0 firmware and reboots. In the V2.0 firmware, some business modules have added new parameters, changed the valid range of parameter defaults, or deprecated old parameter names.
- **Finalized specification: tolerant parsing + default fallback + version metadata**:
  1. **First-line version metadata**: The first line of every file serialized by octopus_config must contain a version tag (e.g. `!version 1.0.4`) identifying the firmware version that generated the configuration;
  2. **Tolerant replay parsing (Tolerant Parsing)**: When the main application replays an old configuration and the parser encounters an unknown legacy command or deprecated syntax, it **logs a Syslog warning and automatically skips it**, continuing with the next line. It never blocks system startup;
  3. **Factory default fallback**: For configuration fields newly added in the new firmware, if the old configuration file does not contain the corresponding line, the business module automatically uses the factory default built into the current firmware source, achieving 100% smooth backward compatibility and a stable boot.

---

### 1.2 Tiered Definition and Trigger Paths of Factory Reset
- **Key boundaries and trigger paths (finalized: Linux user-space detection + visual feedback)**:
  1. **Soft reset**: Entering the `factory-default` command in the Web UI or the octopus_cli command line causes the main application to safely clear the configuration and trigger a reboot;
  2. **Hard reset (physical button)**:
     - **Detection timing**: After the device has booted normally into Linux, the `liboctopus_board` module of the main platform registers a GPIO interrupt for the physical tactile button and starts a debounce software timer;
     - **Long-press criterion**: A continuous press of **5 seconds** by the user is judged as a factory reset trigger;
     - **Visual feedback**: During the long press and at the moment of triggering, the system's bi-color LED blinks rapidly at high frequency (or alternates red/green in a chase pattern) to give clear visual feedback, telling the user "factory reset confirmed, rebooting now" and preventing accidental triggering.
  3. **Early-boot reset (fallback path)**: Detecting the button only in Linux user space has a blind spot: if a particular configuration item is exactly what makes `octopus_app` crash immediately at startup, the device never reaches the button detection step. Therefore U-Boot must also read the same button at power-on: if the button is held during power-on for more than N seconds, U-Boot adds `octopus.factory_reset=1` to bootargs, and the Loader performs the wipe before mounting `/data`. Combined with the supervisor's "consecutive crash counter", the device can also automatically boot once with the factory configuration after several consecutive boot failures (user configuration is not deleted, only not loaded).
- **Data wipe scope and the "absolute protection red line"**:
  - **Items to be fully wiped (user private data)**:
    - `/data/config/startup-config*` (user-defined configuration);
    - `/data/log/*` (system historical runtime logs and crash dumps);
    - Temporary scripts or business data uploaded by the user;
  - **Items that must never be erased (the device's factory hardware DNA)**:
    - Physical MAC address, device unique serial number (SN), PCB hardware revision;
    - Production-line physical zero-offset calibration parameters of robot servos/motors (Calibration Data);
    - Cloud-provisioned device root certificate and factory keys;
  - **Post-reset state**: The main application detects that the user configuration file does not exist, then automatically loads and replays the golden read-only factory configuration (`default-config`) baked into the read-only layer of the root filesystem, restoring the default IP (e.g. `192.168.1.1`). The login password is restored to **the device's unique initial password** (written to FACTORY on the production line and printed on the nameplate), not a default password shared by all devices; the latter is prohibited by regulations such as ETSI EN 303 645 and the EU CRA;
  - **Field recalibration**: Replacing a servo or motor requires recalibration, but the FACTORY partition is read-only. Field calibration values are written to `/data/calib/` and take precedence over the production-line values in FACTORY when read; a factory reset deletes field calibration and falls back to the production-line values.

---

### 1.3 Atomic Configuration File Writes Under Sudden Power Loss (Power-loss Atomicity)
- **Power-loss corruption risk**:
  If the target file is opened directly with `open("startup-config", "w")` and power is suddenly lost during the few milliseconds when Flash is being erased/written, the target file content may be emptied or left as fragmented garbage, completely corrupting parsing at the next boot.
- **Finalized specification: POSIX atomic replace + mirrored backup double safeguard**:
  ```text
             [User or system triggers configuration save]
                        │
                        ▼
   1. Write temp file:         open("/data/config/startup-config.tmp", O_CREAT|O_WRONLY|O_TRUNC)
                        │
                        ▼
   2. Force flush to media:    fsync(fd), then close(fd) (file ends with a CRC32 + length check trailer)
                        │
                        ▼
   3. Keep previous version:   rename("startup-config", "startup-config.bak")
                        │
                        ▼
   4. POSIX atomic replace:    rename("startup-config.tmp", "startup-config")
                        │
                        ▼
   5. Flush directory:         fsync(/data/config directory fd)
                      (rename only guarantees atomicity; without fsync on the directory,
                       a power loss may revert to the state before the rename)
  ```
  On load, `startup-config` → `startup-config.bak` → `default-config` are tried in order, using the check trailer to determine whether a file is intact. For detailed rules and the requirements of "separating commit from flush to media, and coalescing writes", see Section 7.4 of the architecture document.

---

## Topic 2: Hardware Watchdog Integration, OOM Protection Tiers, and Crash Black Box (Reliability & Faults)

Process-level monitoring by octopus_supervisor alone is not sufficient to handle abnormal deadlocks and loss of hardware control across the entire embedded system; a closed-loop hardware/software fallback mechanism must be established. This topic defines the complete technical specification for system-level fault self-healing, memory fallback, and crash diagnostics.

### 2.1 Hardware Watchdog Ownership and the "Anti-Fake-Kick" Model

#### (1) Three Anti-Patterns and Pitfalls of Traditional Embedded Watchdogs
1. **Anti-pattern A (kernel kicks on behalf of user space)**: Letting the kernel keep kicking the watchdog on behalf of user space (for example, enabling `CONFIG_WATCHDOG_HANDLE_BOOT_ENABLED` without setting `watchdog.open_timeout`, or using a "watchdog" driven by a software timer). When the core user-space process `octopus_app` deadlocks, kernel scheduling still runs normally, the watchdog never times out, and the whole device hangs while never resetting. Note that `CONFIG_WATCHDOG_NOWAYOUT` means exactly the opposite: it forbids closing a started watchdog by any means, and it is an option Octopus **requires to be enabled**;
2. **Anti-pattern B (single thread kicking in a blind loop)**: Starting a dedicated thread inside `octopus_app` that runs `while(1) { write(wdt_fd); sleep(5); }`. Once other worker threads (such as network I/O, IPC handling, or motor drivers) hang on a mutex deadlock, this kicking thread keeps happily kicking the watchdog on its own, and the system still hangs;
3. **Anti-pattern C (uncoordinated contention)**: The system has Busybox's `watchdogd` running while the main process also writes `/dev/watchdog`, causing multiple processes to interfere with each other.

#### (2) Octopus Specification: Exclusive Ownership by the Main Process + Multi-Dimensional Subsystem Health Heartbeat Matrix (Heartbeat Matrix)

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                    Core main process (octopus_app)                     │
 │                                                                        │
 │  ┌───────────────────────┐  heartbeat   ┌───────────────────────────┐  │
 │  │ Thread 1: IPC handler │ ──────────>  │                           │  │
 │  └───────────────────────┘ (every 2 s)  │                           │  │
 │  ┌───────────────────────┐  heartbeat   │  Watchdog patrol manager  │  │
 │  │ Thread 2: Net/routing │ ──────────>  │  (Watchdog Supervisor)    │  │
 │  └───────────────────────┘ (every 2 s)  │                           │  │
 │  ┌───────────────────────┐  heartbeat   │ - Tracks module heartbeat │  │
 │  │ Thread 3: Ctrl/engine │ ──────────>  │   timestamps              │  │
 │  └───────────────────────┘ (every 2 s)  │ - Checks system-wide      │  │
 │                                         │   deadlock state          │  │
 │                                         └─────────────┬─────────────┘  │
 └───────────────────────────────────────────────────────┼────────────────┘
                                                         │ only when all submodules are healthy:
                                                         │ write once every 5 s
                                                         ▼
                                           ┌────────────────────────────┐
                                           │ HW watchdog: /dev/watchdog │
                                           │ (timeout window: 20 s)     │
                                           └────────────────────────────┘
```

- **Implementation details**:
  1. **Exclusive ownership**: After system startup, `octopus_app` exclusively opens `/dev/watchdog` and sets the hardware timeout to **20 seconds** (no other external program may touch this device node);
  2. **Health token registration**: Core modules (such as the IPC thread, main event loop, and critical driver threads) register a "health token" with the watchdog manager during initialization;
  3. **Heartbeat check-in**: After completing each normal event-loop iteration, every module must update its own timestamp (`heartbeat_tick(MODULE_ID)`);
  4. **Kick only when all are healthy**: The watchdog manager thread iterates over all modules every 5 seconds: **`ioctl(WDIOC_KEEPALIVE)` is issued only if every critical module has updated within the last 10 seconds**;
  5. **Deadlock penalty and hardware reset**: If any critical module's heartbeat has not been updated for more than 10 seconds (consistent with the unified timeline in Section 9.2 of the architecture document), the watchdog thread immediately stops kicking, attempts to write the deadlock context to pstore/logs, and then waits for the hardware watchdog to forcibly assert the hardware reset signal (RESET pin pulled low) when the 20-second timeout expires, fully resetting the entire SoC;
  6. **Normal reboot does not disable the watchdog**: The kernel enables `CONFIG_WATCHDOG_NOWAYOUT`, so Magic Close (writing `'V'`) has no effect. On `reboot` or an OTA reboot, `octopus_app` raises the timeout to the upper bound required by the shutdown sequence (e.g. 60 seconds) and keeps the watchdog running. If unmounting filesystems or a driver hangs during shutdown, the watchdog can still reset the device as a fallback; if the watchdog were disabled first "to avoid false triggers", a single shutdown hang would leave an unattended device stuck there forever;
  7. **Coverage during boot**: U-Boot enables the hardware watchdog; the kernel uses `CONFIG_WATCHDOG_HANDLE_BOOT_ENABLED` to kick it on behalf of user space until user space takes over, with `watchdog.open_timeout=` bounding the takeover deadline. Without coverage of this phase, the A/B fallback path "boot hang → watchdog reset → bootcount increment" does not hold;
  8. **External watchdog**: On some SoCs the internal watchdog does not reset thoroughly (peripherals and the PMIC are not reset), or DDR contents are lost after reset, rendering pstore ineffective. For high-reliability models, an external standalone watchdog chip is recommended (kicked via GPIO, directly asserting PMIC reset), with the differences abstracted by the Board HAL's `octopus_board_wdt_*`.

---

### 2.2 Linux OOM-Killer Protection Tiers

On devices with limited memory resources (SSD20X has only 64–128 MB of RAM; RK3568 also sees burst overhead when running AI vision), once physical memory is exhausted the Linux kernel invokes the OOM-Killer to kill processes based on its scoring. If it kills the core management process or PID 1, the system collapses instantly.

#### (1) Tiered Configuration (controlled via `/proc/<pid>/oom_score_adj`, range -1000 to +1000)

```text
  [Survival tier]                      [Process]                            [oom_score_adj value]
  ═══════════════════════════════════════════════════════════════════════════════════════════════
  Top privilege (never killed) ────>   octopus_supervisor (PID 1)           (no setting needed: the kernel never selects PID 1)
                                       octopus_app (core platform)          -1000 (keeps the hub alive; must be paired with its own memory-cap self-check)
  ───────────────────────────────────────────────────────────────────────────────────────────────
  Standard services (medium) ──────>   dropbear (SSH), hiawatha (Web)       0 (participates normally in kernel scoring)
                                       zebra / ospfd (dynamic routing)      0
  ───────────────────────────────────────────────────────────────────────────────────────────────
  Sacrificial (killed first) ──────>   ai_vision_daemon (NPU vision)        +500 (large memory footprint, sacrificed first)
                                       Customer third-party child procs     +700 (prevents customer programs from dragging down the platform)
```

#### (2) Closed-Loop Self-Healing
When a sudden memory surge triggers OOM, the kernel preferentially kills `ai_vision_daemon` or customer external processes occupying tens of megabytes, instantly freeing a large amount of physical memory and keeping `octopus_app` and the system alive. Then `octopus_supervisor` (PID 1) catches the child exit signal (`SIGCHLD`) and automatically restarts the AI vision process once physical memory has fallen back, achieving **"local sacrifice, system stays up, self-healing within seconds"**.

---

### 2.3 Crash Black Box and On-Site Dumps (Crash Diagnostics & PStore)

Industrial devices are deployed in the field. If a device crashes due to a segmentation fault (SIGSEGV) or is reset by the hardware watchdog and no logs survive the reboot, troubleshooting becomes an unsolvable mystery.

#### (1) Reserve a Dedicated 1 MB at the Top of DDR (RAMoops)
Configure a reserved memory region (Reserved Memory) in the device tree (DTS), for example the last 1 MB of physical memory, marked as excluded from kernel dynamic allocation. The addresses in the example below correspond to a 64-bit platform with 4 GB of DDR; 1 GB / 2 GB models and 32-bit SoCs (`#address-cells = <1>`) must be rewritten according to the actual memory layout.

**Prerequisites must be verified board by board**: During a warm reset (Warm Reset / Watchdog Reset), DDR must remain in self-refresh without losing power, and none of SPL, U-Boot, or TF-A may use or zero this memory. On some boards a watchdog reset causes the PMIC to cut power entirely; in that case ramoops is completely ineffective and only the Flash black box can be relied upon.
```dts
reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    ramoops@fff00000 {
        compatible = "ramoops";
        reg = <0x0 0xfff00000 0x0 0x100000>; /* 1MB reserved memory */
        record-size     = <0x20000>; /* 128KB kernel log */
        console-size    = <0x20000>; /* 128KB console log */
        pmsg-size       = <0x40000>; /* 256KB user-space messages */
    };
};
```

#### (2) Dual-Channel Dumps for Kernel Panics and User-Space Segmentation Faults
1. **Automatic kernel crash dump**: At the moment of a kernel crash, the `ramoops` driver copies the pre-crash `dmesg` ring buffer directly into this 1 MB of memory;
2. **User-space main application crash capture (PMSG)**:
   `octopus_app` registers signal handlers for `SIGSEGV`, `SIGBUS`, and `SIGABRT`. When an illegal memory access occurs, the signal handler captures the current call stack and writes it directly to `/dev/pmsg0` (the write node that kernel PStore exposes to user space).
   What can be done inside a signal handler is very limited. The following constraints must be observed; otherwise the crash handling itself will crash a second time or deadlock:
   - Pre-allocate an alternate stack with `sigaltstack()`; otherwise a `SIGSEGV` caused by stack overflow has no stack to run on;
   - The first call to `backtrace()` loads libgcc and allocates memory, so it must be called once during initialization to "warm it up";
   - Use only async-signal-safe functions such as `write()` and pre-allocated buffers; `printf`, `malloc`, and taking locks are forbidden;
   - ARM32 must be compiled with `-funwind-tables`; otherwise the backtrace yields only one frame;
   - After the dump completes, restore the default handler and re-`raise()` the original signal so that the process exits with its original signal; the supervisor then decides, based on the exit reason, whether to restart the process or reset the whole device.

#### (3) Boot-Time Check and Extraction/Export
After the system reboots, the kernel automatically exposes this memory as the `/sys/fs/pstore/` filesystem. During the `EARLY_INIT` phase, `octopus_app` checks this directory for records of a previous crash:
- If present, they are automatically copied to persistent storage at `/data/log/crash/crash_<seq>_<boot_id>.log`, and the pstore cache is cleared. File names use a monotonically increasing sequence number rather than a date, because on devices without an RTC the wall-clock time at power-on is untrustworthy; the date is recorded inside the file content, annotated with its time source (NTP-synchronized / last known time);
- The Web UI and the octopus_cli command `show system crash-log` provide one-click viewing and bundled export.

---

## Topic 3: Factory Data Isolation, Device Credentials, and Security Compliance (Security & Provisioning)

For commercial mass production, the device must achieve "separation of code and credentials, and isolation of factory reset from private data". The following three asset protection and security compliance specifications are established:

### 3.1 Physical Storage and Isolation of Critical Production Data (Factory Information)
- **Asset classification**:
  1. **Basic hardware identity**: Ethernet MAC address (6 bytes), device unique serial number SN (16–32 characters), hardware PCB revision;
  2. **Production calibration parameters (Calibration Data)**: physical mechanical zero offsets of robot servos/motors, ADC battery voltage calibration coefficients, RF transmit power calibration parameters;
  3. **Cloud security identity**: the device private key (`device.key`) and X.509 client certificate (`device.crt`) required to connect to AWS IoT / Alibaba Cloud.
- **Finalized storage scheme: a dedicated read-only `factory` partition on Flash as primary + a reserved I2C EEPROM driver abstraction**:
  - **Default storage medium**: A dedicated `factory` partition (128 KB–1 MB) is carved out of Flash (eMMC or SPI NAND) and mounted read-only during normal operation. When `factory-reset` is executed, the lower layers of the system are **absolutely forbidden from erasing or formatting this partition**;
  - **Integrity and redundancy**: With only a single copy of FACTORY, one accidental erase/write or bit flip would lose the MAC and SN, and the device could no longer be identified. Data is stored as two copies in "version + length + content + CRC32" format; on read, both are verified and the intact copy is selected automatically;
  - **Unified driver abstraction (Board HAL)**: Access goes through the `board_hal.h` frozen in Section 6.2 of the architecture document (`octopus_board_get_info()`, `octopus_board_get_calibration()`); no separate interface is defined here. The lower layer adapts based on the device tree (DTS): if an I2C EEPROM is present on the board, data is physically read from the EEPROM; otherwise it is read from `/dev/octopus/by-vol/factory`, fully transparent to upper-layer business logic;
  - **Private keys should not be stored in plaintext**: If `device.key` is stored in plaintext in the FACTORY partition, clipping onto the Flash chip is enough to clone the device identity. On models with a secure element (SE, e.g. ATECC608, SE050) or a TEE, the private key should be generated inside it and be non-exportable; on models without one, it should at least be stored encrypted with the SoC hardware unique key, and the protection boundary must be documented in the threat model.

---

### 3.2 Firmware Anti-Tamper Signing and Secure Boot Specification (Secure Boot)
- **Threats defended against**: Preventing third parties from flashing unauthorized firmware, tampering with the kernel, or counterfeiting hardware.
- **Established technical flow**:
  1. **Release stage (Sign)**: On an offline signing machine or HSM, the OFI header is signed with the vendor private key (**ECDSA-P256 or RSA-2048**). The header contains the SHA-256 of each segment, the firmware version, the anti-rollback counter, and the board identifier; the signature covers the entire header (format in Section 4.4 of the architecture document);
  2. **Upgrade verification (Verify in User Space)**: `firmware_worker` embeds the corresponding public key. It first verifies the header signature and checks the board identifier and anti-rollback counter; if they pass, it streams the image into the inactive slot, computing each segment's hash while writing, and switches the boot slot only if all hashes match. **A header signature failure is rejected outright; if any segment hash mismatches, the inactive slot is kept marked bad and an alarm is raised**. This avoids caching the entire image in memory;
  3. **Bootloader verification (Verify in Bootloader)**: The U-Boot `cmd_ofi.c` driver, after loading the kernel image from Flash into RAM and before jumping to it, calls a built-in lightweight public-key library to verify the signature; if verification fails, it stops booting and switches back to the backup partition.

---

### 3.3 Anti-Theft Protection and Hardware Binding of Core Algorithm Model Weights
- **IP protection requirement**: Core AI vision weight files that customers have trained with significant manpower and compute (model packages exported by the vendor's NPU toolchain, e.g. the common `.rknn` format) must be protected from easy theft by outsiders reading the Flash with a programmer clip. Model formats and inference libraries are provided separately by chip/toolchain vendors under their own licenses; they are **not included in this repository's source tree, and this project does not distribute their source code or binaries**.
- **Why a key cannot simply be derived from the Chip ID**: A common approach is "derive a key from the SoC Chip ID via a derivation algorithm and encrypt `model.squashfs` with AES-GCM". However, the Chip ID can be read by software and the derivation algorithm lives in the Loader; an attacker who obtains the Flash image and the Chip ID of any device can compute the key. This is "security through obscurity of the algorithm". Moreover, since each device's Chip ID differs at packaging time, the same image cannot be shipped in volume.
- **Established strategy: hardware key wrapping + block-device encryption**:
  1. At packaging time, the model image is encrypted with a randomly generated model key K. The `aes-xts-plain64` format supported by dm-crypt is used instead of whole-file AES-GCM, enabling random block-wise reads and direct mounting;
  2. On the production line, a hardware unique key that **software cannot read** inside the device (the HUK in SoC secure storage / OTP, used via OP-TEE or a hardware crypto engine) wraps K into a "device-specific ciphertext", which is written to FACTORY;
  3. At boot, the TEE or crypto engine unwraps K, which is then mapped via dm-crypt and mounted; K is never persisted and never appears on Flash;
  4. **The protection boundary must be stated clearly**: This prevents "desoldering the Flash and copying the image" to run on other devices. However, an attacker with root privileges on a running device can still read the decrypted model from memory. Defending against that layer requires running inference inside the TEE or NPU driver support for encrypted models, depending on the specific SoC's capabilities. SoCs without a HUK / TEE cannot provide this protection, and this must be stated truthfully.

---

## Topic 4: Third-Party Dependency Supply Model and Cross-SoC libc Compatibility Boundaries (Dependencies & Sysroot)

The `octopus-core` repository is a standalone modern CMake codebase, but it runs in a Linux world made up of underlying cross toolchains and libraries. Regarding the source of third-party libraries and C runtime library compatibility, the following two core specifications are established:

### 4.1 Third-Party Open-Source Library Supply Model: Layered Hybrid Supply (Finalized: Mode B)

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │ [Mode B: Layered hybrid supply model]                                  │
 │   (minimal core bundled in-tree + heavyweight linked externally)       │
 │                                                                        │
 │  1. Lightweight pure C/C++ core libraries (in octopus-core/3rdparty/): │
 │     - e.g. cJSON parser, lightweight embedded MQTT client,             │
 │       tiny Base64/SHA libraries                                        │
 │     - Approach: built by CMake alongside the octopus-core sources      │
 │     - Benefit: whether on x86 PC, WSL2, Docker, SSD20X or RK3568,      │
 │       the core libraries and unit-test suites [build in one step,      │
 │       fast, with zero environment dependencies]!                       │
 │                                                                        │
 │  2. Heavyweight system services and network components                 │
 │     (provided by each SoC BSP's Buildroot/Sysroot):                    │
 │     - e.g. Hiawatha web server, Dropbear SSH, OpenSSL crypto suite     │
 │     - Approach: cross-compiled by the unified user-space build layer   │
 │       (Buildroot external tree, etc.) and packaged into                │
 │       rootfs.squashfs; the octopus-core build only links headers and   │
 │       libraries via CMake find_package                                 │
 │     - Benefit: no duplicate builds of large, complex software in the   │
 │       platform repo; fast builds, lightweight architecture             │
 └────────────────────────────────────────────────────────────────────────┘
```

- **Security components must not be left to the vendor BSP**: Components such as OpenSSL, Dropbear, and the web server frequently require urgent CVE patches. If they come with the vendor BSP, every SoC carries a different version, the upgrade cadence is dictated by the vendor, and a single vulnerability has to be chased across three different chip vendors — exactly a repeat of the "tight coupling to the SoC SDK" problem in Section 1.1. Vendor BSPs typically deliver the kernel, U-Boot, and **boot firmware and acceleration libraries provided only in binary form** (such as the NPU user-space runtime); such binaries are obtained and integrated by the device maker under the vendor license and **must not enter this open-source repository**. All other user-space components are maintained by Octopus's unified build layer, with all SoCs using the same version;
- **SBOM and vulnerability tracking**: The unified build layer produces an SBOM (SPDX or CycloneDX) for each release, used for CVE matching and EU CRA compliance.

---

### 4.2 Cross-SoC C Runtime Library (libc) Compatibility Coding Guidelines

- **Reality of cross-SoC toolchain differences**:
  - **SigmaStar SSD20X**: a relatively early 32-bit ARM platform, with a toolchain based on the conservative **glibc 2.24** or even **uClibc-ng**;
  - **Rockchip RK3568**: a modern 64-bit ARM64 platform, whose toolchains commonly use **glibc 2.31** or **glibc 2.35**.
- **Three established coding and compilation guidelines**:
  1. **Never use private extension functions of newer glibc versions**:  
     For example, `getentropy()` (introduced in glibc 2.25), `copy_file_range()`, and certain C11 threading extensions produce undefined-symbol errors at link time on older toolchains. The platform uniformly uses the most robust standard **POSIX.1-2008 / C99 interfaces** (e.g. reading `/dev/urandom` for true random numbers, and standard `read`/`write` for data copying);
  2. **Never depend on the size of libc-specific internal data structures**:  
     All cross-module and cross-process communication data uses explicitly fixed-width strong types (such as `uint8_t`, `uint32_t`, `int64_t`); the use of `long` (whose width is not fixed) or compiler-specific bit fields is strictly forbidden;
  3. **CMake must inject 64-bit file offset and timestamp macros**:  
     In the top-level `CMakeLists.txt`, the following compile definitions are injected globally for all build targets:
     ```cmake
     add_compile_definitions(_FILE_OFFSET_BITS=64 _TIME_BITS=64)
     ```
     `_FILE_OFFSET_BITS=64` is effective on all target toolchains, so even the 32-bit SSD20X can handle files larger than 2 GB.
     **`_TIME_BITS=64`, however, has prerequisites**: it requires glibc ≥ 2.34 and a kernel ≥ 5.1 that provides 64-bit time system calls. With the glibc 2.24 / uClibc-ng + 4.9 kernel combination common on SSD20X, this macro has no effect, `time_t` remains 32-bit, and the year-2038 problem **cannot be fully eliminated** on such platforms. Feasible mitigations:
     - Our own file formats, IPC protocols, and log timestamps always use `int64_t` (seconds or microseconds); `time_t` is never serialized directly;
     - Time intervals are always computed with `CLOCK_MONOTONIC`, unaffected by wall-clock wraparound;
     - CMake detects `sizeof(time_t)`; when it is 32-bit, an explicit warning is emitted in the build log, and the support period is stated in the product documentation.

---

## Topic 5: Generic High-Performance Shared-Memory Data-Plane Channel and Dual-Plane Coordination (liboctopus_shm)

The management plane and event reporting have already been decided to use a Unix Domain Socket (`/var/run/json_ipc.socket`) + JSON-RPC. This is ideal for 95% of management-plane scenarios (Web configuration, status polling, alarm push, at frequencies < 30 Hz).

However, **generic infrastructure must never be specialized for a single business scenario (such as robot servos)**. In industrial control, high-frequency telemetry, and intelligent edge domains, there are many high-throughput scenarios that require **microsecond-level latency, zero-copy, and nanosecond-level lock-free communication**. Therefore, Octopus establishes a generic dual-plane architecture with **"physical separation of the control plane and the data plane"**.

### 5.1 Four Generic Business Needs Addressed by the Shared-Memory Data Channel
1. **Industrial high-frequency measurement & control and power waveform recording (DAQ & Power Grid)**:  
   High-precision ADC current/voltage sampling (e.g. microprocessor-based power protection relays require 12.8 kHz sampling for FFT analysis) and mechanical vibration monitoring (tens of thousands of samples per second). Passing this through socket string parsing would instantly drive the CPU to 100% and collapse; it must flow zero-copy through shared memory;
2. **Industrial real-time fieldbus and PLC process data image (Process Data Image / Fieldbus)**:  
   External sensors and PLCs are polled over RS485 / CAN / EtherCAT, and all applications must share one up-to-date snapshot of all I/O registers every 1–5 ms;
3. **High-density AI vision feature vectors, LiDAR point clouds, and raw full-duplex audio (Feature / PointCloud / Audio)**:  
   512-dimensional float vectors output by face/object recognition, hundreds of thousands of spatial points per second from LiDAR, and 16 kHz 16-bit full-duplex raw audio PCM frames are continuously flowing high-throughput binary streams;
4. **Robot high-frequency hard-real-time motion control (Robot Motion Control)**:  
   Robot joint servos/brushless motors require angle and torque refresh at 200–1000 Hz (1–5 ms closed loop), as one typical Channel attached to the generic data channel.
   **Note: shared memory only solves "fast transfer", not "on-time arrival".** On a standard Linux kernel, the scheduling latency of ordinary threads can reach several milliseconds or more under heavy load. A 1 ms closed loop requires a PREEMPT_RT kernel, `SCHED_FIFO` threads, `mlockall()`, and CPU isolation — or simply moving the closed loop onto an MCU with Linux only sending setpoints. See Chapter 13 of the architecture document, "Ownership of Real-Time Control".

---

### 5.2 Unified Data Channel Design: `liboctopus_shm`

To accommodate all of the above scenarios, the platform plans a generic common library, **`liboctopus_shm`**. Its core philosophy is: **"one generic multiplexed cell framework, with the payload format entirely defined by the business"**.

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │         Application layer (multiple generic workloads coexist)         │
 │                                                                        │
 │  [A: HF power sampling]   [B: industrial bus I/O]   [C: robot motors]  │
 └─────────────┬────────────────────────┬──────────────────────┬──────────┘
               │                        │                      │
               ▼                        ▼                      ▼
 ┌────────────────────────────────────────────────────────────────────────┐
 │           Unified generic data-channel layer: liboctopus_shm           │
 │                      (/dev/shm/octopus_channels)                       │
 │                                                                        │
 │  ┌──────────────────────────────────────────────────────────────────┐  │
 │  │ Generic Cell Header (strictly 32-byte aligned):                  │  │
 │  │  - channel_id: channel number (0~255, dynamic channel extension) │  │
 │  │  - payload_type: payload type (RAW_WAVEFORM / BUS_IO / MOTION...)│  │
 │  │  - sequence_id: 64-bit monotonic counter (detects skipped frames)│  │
 │  │  - timestamp_us: high-precision microsecond system timestamp     │  │
 │  │  - payload_len: actual payload length in bytes                   │  │
 │  │  - flags: reserved bits (e.g. frame-drop alarm, key-frame marker)│  │
 │  ├──────────────────────────────────────────────────────────────────┤  │
 │  │ Self-describing application payload area (Payload Data Buffer):  │  │
 │  │  - Could be: industrial power waveform (float waveform[1024])    │  │
 │  │  - Could be: fieldbus I/O state image (uint8_t io_image[512])    │  │
 │  │  - Could be: LiDAR point-cloud coordinates / raw audio PCM frames│  │
 │  │  - Could be: robot joint angle/torque (merely Channel_ID = 1)    │  │
 │  └──────────────────────────────────────────────────────────────────┘  │
 │                                                                        │
 │  Concurrency: single-producer/multi-consumer lock-free ring queue      │
 │  built on C11 atomic variables (stdatomic)                             │
 └────────────────────────────────────────────────────────────────────────┘
```

- **Cell structure definition (C11, natural alignment)**:
  ```c
  /* include/octopus/shm_channel.h */
  typedef struct {
      uint16_t channel_id;    /* Channel ID: 0~255 */
      uint16_t payload_type;  /* Business type: 0x01-sampling, 0x02-bus, 0x03-motor... */
      uint32_t payload_len;   /* Payload length */
      uint64_t sequence_id;   /* Frame sequence number (drop detection) */
      uint64_t timestamp_us;  /* CLOCK_MONOTONIC microsecond timestamp (latency and jitter analysis) */
      uint32_t flags;         /* Flags */
      uint32_t reserved;      /* Padding to 32 bytes */
      uint8_t  payload[];     /* C99 flexible array member */
  } octopus_shm_cell_t;

  _Static_assert(sizeof(octopus_shm_cell_t) == 32, "cell header must stay 32 bytes");
  ```
  The fields are already arranged for natural alignment; `__attribute__((packed))` is neither needed nor appropriate: packed makes the compiler assume the struct may be unaligned, generating byte-by-byte accesses on ARM, so 64-bit fields can no longer be read or written atomically. `payload[0]` is a GNU extension; the C99 standard form is `payload[]`. When 32-bit and 64-bit processes share the same memory region, this layout is identical on both sides (under ARM EABI, `uint64_t` is 8-byte aligned in both cases).
- **Core performance benefit**:
  Based on `/dev/shm` memory mapping and atomic sequence advancement, it achieves **zero memory copy, with CPU overhead per transfer under a microsecond**. Actual end-to-end latency depends on how consumers are woken: busy polling gives the lowest latency but consumes an entire core; the recommended approach is for the producer to wake consumers via `eventfd` (with the fd passed over UDS) or futex, in which case latency is determined by scheduling.
- **Reliability constraints**:
  - **Sequence-lock (seqlock) read protocol**: The consumer reads the sequence number once before and once after reading the data, and discards and re-reads if they differ. When it falls more than one full lap behind (data overwritten), dropped frames are detected through a jump in `sequence_id`, rather than reading a half-new, half-old record;
  - **Process crashes**: Shared memory does not use inter-process mutexes, avoiding permanent blocking of other processes after a lock-holding process crashes. After restarting, the producer first increments a "generation" field; when consumers see the generation change, they resynchronize;
  - **Permissions**: Each channel is one `/dev/shm` object, with file permissions set as "producer writable, consumers read-only", and consumers map it with `PROT_READ`.

---

### 5.3 Dual-Plane Coordination Matrix (Control Plane vs Data Plane)

| Dimension | Control Plane | Data Plane |
| :--- | :--- | :--- |
| **Underlying channel** | Unix Domain Socket (`/var/run/json_ipc.socket`) | POSIX shared memory (`/dev/shm/octopus_channels`) |
| **Common library** | `liboctopus_json` / `liboctopus_client` | `liboctopus_shm` |
| **Transport protocol** | Text-based standard **JSON-RPC** | Binary **POD cell header + business-defined fixed-length / lock-free ring buffer** |
| **Interaction frequency** | Low frequency (< 30 Hz) or event-driven (on demand) | High frequency (100 Hz – 10000 Hz continuous streaming) |
| **Core responsibilities** | Device configuration, parameter delivery, status queries, saving to storage, and O&M | Hard-real-time motion control, industrial waveform acquisition, bus I/O snapshots |
| **Typical interfaces** | `system.status.get`, `network.ip.set` | `shm_channel_publish()`, `shm_channel_subscribe()` |

---

## Topic 6: Physical Boundaries of Flash Storage Media and Brick-Proof Bootloader Upgrades (Storage & Bootloader Resilience)

The A/B dual-partition mechanism inherently protects the kernel and the application root filesystem, but for the underlying physical medium (NAND bad blocks) and the single point of weakness in the Bootloader itself, uncrossable engineering lines of defense must be established.

### 6.1 Physical Bad-Block Tolerance and Alignment for SPI NAND Flash (Bad Block Skipping)

#### (1) Hardware Physical Characteristics and Differences
- **eMMC / UFS (e.g. RK3568)**: Contains a built-in hardware FTL (Flash Translation Layer) controller; physical bad-block management, wear leveling, and ECC correction are completely transparent to the operating system. The driver accesses it directly as a standard block device (`/dev/mmcblk0`), and bad blocks very rarely leak through;
- **SPI NAND Flash (e.g. SSD20X or low-cost gateways)**: Raw Flash. The manufacturer allows 1%–2% factory bad blocks at shipment, and new runtime bad blocks continuously appear through erase/write cycles over the entire service life.

#### (2) Three Hard Rules for the Platform and Toolchain
1. **Never use `dd` to write MTD partitions directly**:
   `dd` treats the target device as physically contiguous space and has no awareness of MTD bad blocks at all. Once a write hits a bad-block sector, the kernel raises an I/O error and aborts the write, corrupting the firmware.
2. **Firmware slots live in UBI volumes, with UBI handling bad blocks**:
   SLOT_A / SLOT_B are static volumes in the same UBI device. `firmware_worker` writes them via whole-volume update with `UBI_IOCVOLUP`, and U-Boot reads them via `ubi read`. Bad-block replacement, wear leveling, and data relocation after bit flips (scrub) are all handled identically by UBI on both ends.
   This is far more reliable than "raw MTD + manual bad-block skipping", because the latter has two problems it cannot handle:
   - If a new bad block appears in the middle of a slot at runtime, all subsequent data shifts back, and if the spare margin at the end of the partition is insufficient, the image no longer fits;
   - Firmware is read-only for long periods, and NAND gradually accumulates bit flips due to read disturb and data-retention decay, so someone must move the data to new blocks. UBI automatically relocates data when it reads a correctable bit flip; under raw MTD nobody does this, and after a few years the same block of data will exceed the ECC correction capability.
3. **Manual bad-block skipping is needed only for raw MTD regions, and both ends must be symmetric**:
   BOOT / ENV / FACTORY remain on raw MTD and are written by production-line or rescue tools:
   - Iterate over the target Flash blocks, calling `ioctl(fd, MEMGETBADBLOCK)` to check whether the current block is bad;
   - **Automatically skip bad blocks (Skip Bad Block)** and continue writing to the next healthy physical block, reserving spare margin at the end of the partition;
   - Read back and verify after writing. When `ECCGETSTATS` shows an increase in correctable bit flips, the correct action is to rewrite that block's content to a new block (scrub), not to mark it bad directly; a block is marked bad only when an erase/write fails.
   When U-Boot reads these regions it uses the same skip semantics: raw NAND uses `nand_read_skip_bad()`, and SPI-NAND goes through the MTD subsystem's `mtd read` (which skips bad blocks automatically).

---

### 6.2 Bootloader (U-Boot) Upgrade Strategy: Should OTA Be Allowed?

In the vast majority of embedded hardware designs, the very beginning of physical Flash (starting at Sector 0) holds only a single copy of SPL / U-Boot. **This is usually a single point of weakness with no A/B backup**. If power is suddenly lost during the brief window in which the Bootloader is being flashed, the device is completely bricked and can only be returned to the factory for recovery with a fixture or a programming clip.

#### (1) Three Engineering Strategies and Trade-off Matrix

| Strategy | Approach | Bricking risk | Applicable scenarios and rationale |
| :--- | :--- | :--- | :--- |
| **Strategy A: Absolutely forbid OTA upgrade of the Bootloader (mandatory default, strongly recommended)** | OFI firmware upgrade packages **must never contain the U-Boot binary**. U-Boot is flashed and fixed once on the production line; all subsequent upgrades only ever target the kernel and root filesystem (A/B dual partitions). | **No bricking caused by OTA writing the Bootloader** (as long as the Bootloader survives, the device can always recover via automatic A/B arbitration or network rescue). The cost is that U-Boot vulnerabilities and signing keys can never be updated remotely, so U-Boot must ship with built-in backup public keys and a revocation mechanism. | General best practice for industrial switches, high-availability gateways, and unattended field devices. **The Octopus default firmware specification strictly enforces this strategy.** |
| **Strategy B: Restricted engineering rescue mode (Local Recovery Only)** | Normal OTA firmware packages must never carry the Bootloader. Only when the hardware has a serious DDR timing bug is a separate dedicated tool provided, which flashes it on its own under the constraints of stable external power and a manually re-entered confirmation command, followed immediately by a full read-back verification. | **Medium risk** (the device bricks if power is forcibly removed during the brief flashing window). | Restricted to after-sales technicians doing repairs, production-line rework, or special laboratory environments. |
| **Strategy C: Hardware-supported Bootloader dual backup (requires physical chip support)** | Relies on the chip BootROM's multi-copy loading mechanism (e.g. searching for multiple SPL copies at agreed offsets, or booting by polling the eMMC boot0 / boot1 hardware partitions). Support varies greatly across SoCs and must be confirmed against the chip manual and by testing. Achieves true A/B hot standby of U-Boot at the physical layer. | **Very low** (but strongly dependent on the specific SoC's BootROM hardware strap configuration or eFuse definitions). | High-budget custom hardware solutions. |

#### (2) Explicit Decision for the Octopus Platform
**"Strategy A (forbid OTA upgrade of the Bootloader)" is fixed as the system's default top-level security rule**:
1. The `ofi_pack` packaging script by default refuses to include the Bootloader in a generic `.ofi` firmware image;
2. When the `firmware` user-space upgrade service validates the OFI header TLVs and finds a Bootloader chunk, it by default refuses to flash it and returns an invalid-type error;
3. This ensures that no remote upgrade, Web UI upgrade, or bulk push upgrade — regardless of the step at which power is lost — can leave the device without a bootable slot.

---

## Topic 7: System Introspection and High-Reliability Advanced Diagnostics (Diagnostics, Lock Watchdog & Memory Introspection)

For high-reliability industrial devices to support **long-term continuous operation, self-healing from deadlocks, and traceability of failure sites**, the platform must have built-in **system introspection and diagnostic supervision** capabilities. This topic specifies 8 classes of diagnostic mechanisms that are **original designs of Octopus**, and records the solution reasoning and architectural decision process. This repository **does not contain or distribute** proprietary/closed-source code from any third-party vendor or third-party product.

```text
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│        Octopus Embedded diagnostics & supervision overview (industrial high-reliability practice)       │
├───────────────────────────────────┬────────────────────────────────┬────────────────────────────────────┤
│ Concurrency & lock supervision    │ Memory & lifecycle             │ Introspection & snapshots          │
│ • lock_watchdog deadlock detection│ • octopus_alloc memory redzone │ • crashhandler all-thread snapshot │
│ • Lock hold-time profiling        │ • Per-module heap leak stats   │ • thread_load_monitor thread CPU   │
├───────────────────────────────────┼────────────────────────────────┼────────────────────────────────────┤
│ Comms & business performance      │ Logging & hardware self-test   │ On-site deep debugging             │
│ • Slow-callback capture (tick)    │ • octopus_trace 3-D tracing    │ • debug memory dump (memory)       │
│ • Buffer leak troubleshooting     │ • syslog_flash black box       │ • debug i2c/spi online bus access  │
│ • Queue overflow/underflow        │ • POST power-on hardware BIST  │ • debug thread online call stacks  │
└───────────────────────────────────┴────────────────────────────────┴────────────────────────────────────┘
```

---

### 7.1 In-Depth Analysis of Core Diagnostic and Introspection Mechanisms

#### (1) `lock_watchdog`: Critical-Section Mutex Deadlock Monitor and Lock-Hold Performance Analyzer (`lock_watchdog` module)
In a high-reliability architecture, `lock_watchdog` is far more than a plain `pthread_mutex` wrapper; it is a fully featured **lock-state supervision and contention diagnostics subsystem**:
*   **Anti-corruption magic number (Canary)**: The head of the `lock_watchdog_t` struct holds `cookie = 0x0BADBABE`. This magic number is verified before every lock/unlock; if the mutex struct is corrupted by an external wild pointer or buffer overrun, the error is reported and intercepted immediately;
*   **Lock history tracking (Flight Recorder)**: The struct embeds a ring array of depth 16, `lock_attempt_thread_id[16]`, recording the thread TID, file name, source line number, and pending state of the last 16 lock attempts;
*   **Background patrol and deadlock timeout detection**: A dedicated background thread, `lock_watchdog_thread`, walks every mutex in the global registration list every 5 seconds. If a lock has been held continuously by the same thread beyond the threshold (default `max_lock_time = 10s`, consistent with the unified timeline in Section 9.2 of the architecture document; the threshold must be smaller than the hardware watchdog timeout, otherwise the whole device is reset before the deadlock is detected and nothing is left at the scene), it is immediately judged a **deadlock**;
*   **Lock-hold performance bottleneck profiling (Profiling)**: For each lock, it records the historical **maximum hold time** (`max_lock_tick_cnt`) and the source location where that maximum occurred (`max_lock_file`, `max_lock_line`), helping developers precisely pinpoint multithreaded lock contention bottlenecks online;
*   **Defensive assertion**: Provides `lock_watchdog_assert_locked()`. Adding this assertion at the entry of every internal function that assumes "the caller already holds the lock" fully eliminates missing-lock races during multithreaded refactoring.

#### (2) `crashhandler`: System-Wide Multithreaded Cooperative Crash Snapshot System (`crashhandler` module)
In a multithreaded Linux program, when a single thread triggers a segmentation fault (`SIGSEGV`), the default behavior is to produce a core dump or simply exit, with no way to know **what the other surviving sibling threads were doing at the moment of the crash**. The system defines a highly cooperative on-site dump protocol:
*   **First-dying-thread leadership**: The thread that catches the fault signal is marked as `first_thread_to_die` and is responsible for recording its own registers (PC, SP, general-purpose registers) and call stack;
*   **All-thread wake-up broadcast**: The first-dying thread opens the `/proc/self/task/` directory to enumerate the TIDs of all sibling threads in the process, and sends a dedicated dump signal to every sibling thread via the system call `syscall(SYS_tgkill, pid, tid, SIGRTMIN + 1)`;
*   **Cooperative concurrent dumps across threads**: Each surviving worker thread prints its thread name and TID in its own context and runs `_Unwind_Backtrace()` to print its own call stack at the moment of the crash, assembling a whole-device **panoramic call-stack snapshot**;
*   **Black-box persistence and controlled exit**: The first-dying thread waits for all sibling threads to finish dumping (at most 3 seconds, which must be far less than the watchdog timeout), appends key information such as `/proc/self/maps` and the build-id, writes it to pstore, then makes a best effort to write to `/data/log/crash/`, and finally exits the process with the original signal. **Whether to reset the whole device is decided by the supervisor**: an occasional crash only restarts the process, and only consecutive crashes beyond a threshold are escalated to a watchdog reset. If the crash handler itself called `reboot()`, it would defeat the supervisor's back-off restart policy and turn a process crash that could have been recovered locally into a full device reboot.

#### (3) `octopus_alloc`: Modular Heap Memory Safety Canary and Leak Tracking (`octopus_alloc` module)
Targets memory leaks and buffer-overflow corruption (Heap Corruption) in long-running embedded environments:
*   **Dual magic-number canary (Canary & Redzone)**:
    *   An extra 8 bytes are allocated at the head: the first 3 bytes are set to the magic number `0xC0FFEE`, the high byte records the `modid` (module ID), and the last 4 bytes record the user-requested size `sz`;
    *   1 byte is appended at the tail: set to the tail magic number `0xBA`;
    *   On free, the head and tail magic numbers are strictly verified. If a magic number has been overwritten, the pointer address at which the overrun occurred, the name of the corrupted module, and the source line number are printed immediately;
*   **Per-module heap statistics and leak pinpointing within seconds**:
    *   Independent counters are maintained for every module in the system: current usage (`usage`), historical peak (`max`), allocation count (`allocs`), and free count (`frees`);
    *   By entering `show memory` in the CLI, if a module's `allocs` is far greater than its `frees` and its `usage` keeps increasing monotonically, **you can confirm online which module is leaking memory without attaching a debugger**.

#### (4) `thread_load_monitor`: Thread-Level CPU Load and Scheduling Thrash Monitor (`thread_load_monitor` module)
High overall CPU load is not the scary part; the scary part is not knowing which specific thread is spinning:
*   **High-precision per-thread load calculation**: Periodically samples `/proc/stat` and `/proc/self/task/<tid>/stat` to compute system-wide idle load, interrupt/peripheral process load, and the 1-second instantaneous load and 10-second sliding-average load of **every worker thread in the system** (an `htop` dedicated to multithreaded embedded systems);
*   **Context switches and scheduling bottlenecks**: Counts each thread's context switches (voluntary vs. involuntary context switches); if a thread yields the CPU frequently, intense lock contention or unreasonable sleep granularity can be spotted directly;
*   **Page fault monitoring (Page Faults)**: Parses the rate of change of `pgfault` in `/proc/vmstat` and raises an early warning when memory pressure causes page thrashing or swap thrashing.

#### (5) `octopus_trace`: Three-Dimensional Dynamic Log Tracing System (`octopus_trace` module)
*   **Three-dimensional fine-grained filtering**: Every trace statement is identified by three dimensions, **[module ID, group ID, thread TID]**, and has its own level (`ERROR` -> `RACKET`, 6 levels in total);
*   **Zero-overhead compile-time trimming**: With `octopus_trace_LVL_MIN`, debug-level macros can be expanded to nothing in Release builds, with zero instructions and zero string footprint;
*   **Hot-pluggable runtime adjustment**: In production everything defaults to the `ERROR` level (zero cost). Once a function in the field (such as LACP, the motor driver, or MQTT) shows an intermittent anomaly, the field engineer only needs to enter a single command to dynamically raise that one module to `DEBUG` or `NOISE`, without restarting the process, and turn it back off once the logs have been captured.

#### (6) Business Performance and Queue Watermark Monitoring (`packet_dbg` and `msg_dbg`)
At the communication and packet forwarding layer, the system adds capture of **pathologically slow callbacks**:
*   **Slow-callback capture (`longest_rx_callback_ticks`)**: The dispatcher thread timestamps before and after dispatching data to a business module's callback, recording the maximum time consumed by each registered callback, to prevent business modules from performing disk writes or sleeping inside callbacks and blocking the entire pipeline;
*   **Buffer watermark and outstanding statistics (Outstanding Buffers)**: Globally counts `Allocations`, `Deallocations`, and the number of buffers currently `Outstanding` in the system, to prevent packet pool leaks;
*   **FIFO over/under statistics**: Counts overflow occurrences of hardware FIFOs and software ring queues.

#### (7) `post`: Power-On Hardware Self-Test and Peripheral Enumeration (`post` module)
At power-on, a self-test is performed on each hardware bus and device (BIST, DDR stress verification, EEPROM read/write loopback, I2C bus device scan, Ethernet port physical loopback test). Results are stored in `post_status_table`, forming a power-on hardware health report after system startup.

#### (8) `syslog_flash`: Power-Loss-Proof Hardware Black Box (`syslog_flash` module)
In an ordinary system, all logs live in RAM or the Linux log buffer; once the system suffers a power loss, a hard watchdog reset, or a kernel panic, the logs from the moment of the final crash vanish entirely. The high-reliability system sets aside a dedicated Flash ring log area (`syslog_flash_log`) and atomically flushes the most critical fatal errors (`OCTOPUS_ASSERT`, deadlock alarms, fatal exceptions) directly to Flash, so the device's "last words" can be retrieved after reboot.

---

### 7.2 Component Plan Mapped onto the Octopus Generic Platform

During the coding phase, the above mechanisms will be implemented in the following **Octopus-original** components (planning sketch; no source code is copied from any third-party product):

```text
octopus-core/
├── libs/
│   ├── liboctopus_basics/
│   │   ├── include/octopus/basics/
│   │   │   ├── octopus_lock_watchdog.h <-- Component 1: lock supervisor and deadlock detector (RAII C++ wrapper)
│   │   │   ├── octopus_alloc.h         <-- Component 2: heap memory canary and per-module usage statistics
│   │   │   ├── octopus_crashdump.h     <-- Component 3: all-thread cooperative call-stack snapshot and black box
│   │   │   ├── octopus_trace.h         <-- Component 4: multi-module three-dimensional runtime dynamic trace system
│   │   │   └── octopus_thread_mon.h    <-- Component 5: thread-level CPU/scheduling monitor
│   │   └── src/
│   └── liboctopus_board/
│       └── include/octopus/
│           ├── octopus_post.h          <-- Component 6: cross-SoC generic power-on hardware self-test framework (I2C/SPI/DDR/Ethernet)
│           └── octopus_blackbox.h      <-- Component 7: Flash / RAMoops power-loss-proof black box
```

| Mechanism | Planned location | Evolution and improvements for modern embedded scenarios | Priority |
| :--- | :--- | :--- | :--- |
| **`lock_watchdog` lock supervision** | `liboctopus_basics` | 1. Pure C API + C++14 RAII `LockGuard`;<br>2. Combined with Topic 2: when a deadlock exceeds 10 s, immediately stop kicking the hardware watchdog, output all lock states, then reset | **P0 (mandatory)** |
| **`octopus_alloc` memory canary** | `liboctopus_basics` | 1. Uses a **tiered dual-track scheme** (lock-free statistics in production builds + canary in test builds);<br>2. Automated module IDs; per-module memory allocation exposed via JSON-RPC | **P0 (mandatory)** |
| **`crashhandler` thread snapshot** | `liboctopus_basics` | 1. Supports register dumps on ARM64 (RK3568) / ARM32 (SSD20X) / x86_64;<br>2. Uses modern `libunwind` instead of the outdated `_Unwind_Backtrace`;<br>3. Snapshot output written first to RAMoops/pstore, with Flash as fallback | **P0 (mandatory)** |
| **`octopus_trace` dynamic tracing** | `liboctopus_basics` | 1. Keeps three-dimensional (Module + Group + Level) control;<br>2. Adds support for remote dynamic log-level changes via JSON-RPC, and colored highlighted terminal output | **P1 (strongly recommended)** |
| **`thread_load_mon`** | `liboctopus_basics` | 1. Reads `/proc/` only, zero intrusion;<br>2. Runs as a background patrol service (every 1 or 5 seconds), raising an alarm when any thread exceeds 95% usage for more than 10 seconds | **P1 (strongly recommended)** |
| **`post` hardware self-test** | `liboctopus_board` | 1. Abstracted into a generic hardware self-test plugin mechanism: after power-on, RK3568/SSD20X sequentially self-test I2C peripherals (RTC/EEPROM), PHY chips, and the shared-memory loopback channel | **P2 (recommended)** |
| **Slow-callback capture monitoring** | `liboctopus_shm` / `bus` | In Octopus's RPC / SHM data processing pipelines, record the maximum time of each processing callback, to prevent business code from blocking threads | **P1 (strongly recommended)** |

---

### 7.3 Key Engineering Disputes, Solution Reasoning, and Decision Records

#### (1) Decision 1: Default Enablement Policy for `lock_watchdog` Lock Supervision
*   **Options in dispute**:
    *   **Option A**: **Enabled by default for the entire lifecycle** (microsecond-level overhead is negligible, in exchange for round-the-clock deadlock detection and lock-hold performance supervision);
    *   **Option B**: Enabled only in Debug / CI test firmware; in Release firmware the macros degrade to native `pthread_mutex`.
*   **Reasoning and rationale**:
    *   **Decision: adopt Option A (enabled by default for the entire lifecycle)**;
    *   **Core rationale**: High-reliability industrial systems keep lock supervision enabled by default even in commercial shipping builds. Many intermittent deadlocks in the field only reproduce once under specific timing after weeks or even months of operation. If it is disabled in production builds, then once a device hangs at a customer site and is reset by the external watchdog, no evidence is left at all, and the deadlock point can never be located;
    *   **Deep integration with Topic 2 (system fault tolerance)**: When the background patrol thread finds any lock held for more than 10 seconds, it immediately judges it a deadlock. Step one: call `lock_watchdog_dump` to output the system-wide lock ownership relations and call stacks to the black-box log. Step two: **proactively stop kicking the watchdog**, leaving the hardware watchdog to perform a full hardware reset after the 20-second timeout.

---

#### (2) Decision 2: Reasoning and Recommendation for the `octopus_alloc` Memory Canary and Per-Module Usage Statistics

Regarding the overhead and production strategy of memory monitoring, the team carried out an in-depth engineering analysis:

##### ① Why Fully Disabling It in Production Builds Is Not Recommended (the Flaw of Option A)
If `octopus_alloc` is completely disabled in production builds and passes straight through to glibc `malloc/free`:
*   **Field incidents become a black box**: One of the most fatal failures of embedded devices at customer sites is **slow, small memory leaks** (e.g. a module leaking 10 KB per hour, triggering an OOM crash after 3 months of operation);
*   Once OOM occurs, you can only see that the whole process occupies 500 MB, but **there is no way to know which module leaked the memory**;
*   With no way to attach GDB or run Valgrind in the field (memory and compute cannot support them), the team would be forced to guess blindly through tens of thousands of lines of code, with a fix cycle of several weeks.

##### ② Why the Traditional Implementation Described in 7.1(3) Cannot Be Used As-Is on Modern Multi-Core Systems (the Trap of Option B)
The traditional implementation in 7.1(3) has two hard flaws on modern SoCs (such as the quad-core Cortex-A55 RK3568 and dual-core SSD20X):
1.  **Global mutex bottleneck**: Every `alloc` / `free` must acquire a global mutex, `alloc_crit`. The impact was small in the single-core era, but on a 4-core ARM64 concurrently handling multithreaded network packets and JSON-RPC parsing, this global lock causes severe CPU context switching and lock contention;
2.  **Pointer alignment risk**: An 8-byte header is prepended. On ARM64, glibc `malloc` guarantees **16-byte-aligned** pointers (meeting the requirements of NEON, `long double`, and similar types). With only an 8-byte header, the pointer handed to the caller is only 8-byte aligned, and storing objects of 16-byte-aligned types will fault or slow down. The single-byte tail magic does not itself affect the returned pointer, but it puts the actual block size on an odd boundary.

##### ③ Final Decision: Option C — "Tiered Dual-Track Scheme" (Modern Engineering Best Practice)

```text
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                     Octopus modular memory health system (dual-track)                       │
├──────────────────────────────────────────────┬──────────────────────────────────────────────┤
│ Track 1: Production / field-resident         │ Track 2: R&D / CI / automated test (Debug)   │
├──────────────────────────────────────────────┼──────────────────────────────────────────────┤
│ • Lock-free atomic counters (C11 stdatomic)  │ • Front/rear redzone memory canary (Canary)  │
│ • Strictly 16-byte aligned header            │ • Precise overrun capture with line tracing  │
│ • Pinpoints "which module leaks" in seconds  │ • Paired with GCC AddressSanitizer (ASan)    │
│ • Overhead < 1%, nanosecond-level cost       │ • Strictly catches Use-After-Free            │
└──────────────────────────────────────────────┴──────────────────────────────────────────────┘
```

*   **Production track (enabled round-the-clock by default in Release firmware) — completely solves "who is leaking memory"**:
    *   **Lock-free atomic counters (Zero-Lock)**: Counters use C11 `atomic_uint_least64_t` (32-bit counters would wrap within days on modules with high-frequency allocation), and the `alloc_crit` mutex is removed, so concurrent multi-core allocation never blocks;
    *   **Standard 16-byte header**: Aligned metadata is placed at the head of each memory block (containing `uint32_t module_id` and `uint32_t size`, padded to 16 bytes), guaranteeing that the pointer returned to the upper layer fully meets ARM64 memory alignment requirements;
    *   **Module attribution and coverage**: C++ code is hooked in by overriding the global `operator new/delete`; the module ID is taken from a thread-local "current module" marker (set at the entry of module callbacks). Memory that third-party libraries allocate by calling `malloc` directly does not go through `octopus_alloc`, and is listed separately in the report as `UNTRACKED` (computed as the process's total heap usage minus the tracked amount), so that no one mistakenly assumes the report covers all memory;
    *   **Online field introspection**: Whenever a problem occurs in the field, invoking the diagnostic interface via either the CLI or JSON-RPC instantly produces a per-module memory health report:
        ```text
        octopus# show memory modules
        Module             Allocs       Frees      In-Use (B)    Peak (B)   Status
        ------------------ ----------- ----------- ----------- ----------- --------
        SYS_MAIN                  1240        1240           0      131072   OK
        NET_LACP                 85200       85200           0       16384   OK
        RPC_SERVER              128400      128400        2048       65536   OK
        MQTT_CLIENT             450120       12000     8912896     8912896   WARN: LEAKING!
        UNTRACKED                    -           -     1245184     1310720   -
        ```
        **Benefit**: At a glance you can see that the `MQTT_CLIENT` module's `Allocs` far exceeds its `Frees` and its usage keeps rising. Field troubleshooting capability is maximized, while the performance cost is barely measurable (< 1%). (AI vision runs as a separate process and is not in `octopus_app`'s report; its memory is accounted per process by the supervisor.)
*   **Test track (enabled for R&D, CI automated testing, and burn-in stress testing) — completely solves "who is corrupting memory"**:
    *   The front/rear magic-number canary (Redzone) is enabled at firmware build time via `-DOCTOPUS_MEM_CANARY=ON`;
    *   Within the three-layer test pyramid (L1 unit tests, L2 module-level integration tests), native GCC/Clang `-fsanitize=address` (ASan) is enabled directly. During host simulation and dev-board testing, modern ASan catches memory corruption, wild pointers, and use-after-free far more precisely than simple hand-written magic numbers.

---

#### (3) Decision 3: Persistence Path and Coordination Mechanism for Panoramic Crash Dumps (`crashhandler`)
*   **Multithreaded cooperative snapshot**: Adopt the mechanism from 7.1(2) in which the first-dying thread broadcasts a signal via `/proc/self/task/` (the broadcast uses a dedicated real-time signal such as `SIGRTMIN+1`, not `SIGABRT`, to avoid confusion with a genuine abort), ensuring that at the moment of a segmentation fault every surviving worker thread (network, IPC, motor control, etc.) dumps its own call stack at that instant, forming a panoramic multithreaded snapshot;
*   **Modern stack unwinding**: Use the modern standard `libunwind` instead of the outdated `_Unwind_Backtrace`, with full support for printing ARM64, ARM32, and x86_64 register state;
*   **Double-safeguard persistence path strategy**:
    1.  **First priority (RAMoops / pstore)**: Write the crash snapshot into the 1 MB RAMoops region reserved by the kernel. At the moment of a memory crash the filesystem may be locked up or unstable, whereas a pure in-memory RAMoops write completes in nanoseconds and eliminates blocking on persistence I/O;
    2.  **Second priority (persistent Flash file)**: Attempt to write to `/data/log/crash/` (the crash directory unified across the repository, subject to the log quota), and print to the serial console at the same time;
    3.  **Controlled exit**: After the dump completes, exit the process with the original signal; the supervisor restarts it according to the back-off policy, and only consecutive crashes beyond a threshold lead to a full-device reset by the watchdog (see 7.1(2) and Section 5.3 of the architecture document).

---

## 8. Next-Step Action Guide

The 7 topics covered in this technical document are interrelated but have different focuses. It is recommended to reason through each of them in order of business urgency and turn them into technical decisions and detailed designs:

1. **First priority (most critical to mass production and hardware selection)**:  
   👉 **Topic 2 (watchdog and OOM)**, **Topic 6 (Flash bad blocks and U-Boot upgrade strategy)**, **Topic 7 (lock supervision and dual-track memory diagnostics)**;
2. **Second priority (most critical to platform coding)**:  
   👉 **Topic 1 (configuration lifecycle and atomic writes)**, **Topic 4 (third-party dependency supply model)**;
3. **Third priority (implemented on demand depending on the specific product form)**:  
   👉 **Topic 3 (security compliance and model encryption)**, **Topic 5 (high-frequency control shared-memory channel)**.
