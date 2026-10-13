# Switch Chip SDK Layer Analysis and Third-Party Kernel Driver (.ko) Integration Architecture Guide

**English** | [简体中文](./02-switch-hal-drivers.md)

> **Note**: This is an English translation. The Simplified Chinese version is authoritative; if the two differ, the Chinese version prevails.

> **Purpose**: Clarifies the point embedded developers most often confuse when building on the Octopus Embedded platform: **the relationship between the switch chip SDK and Linux kernel drivers (.ko)**. It explains in depth the standard integration paths and lifecycle management for hardware drivers in different business scenarios.

---

## 1. Core Concepts: What Exactly Is the Switch Chip SDK?

First, one fundamental principle must be made clear: **The switch chip SDK runs entirely in Linux user space. It is a pure C/C++ function library and is not a kernel module (.ko) at all!**

```text
 ┌────────────────────────────────────────────────────────┐
 │ octopus_app (user-space core process)                  │
 ├────────────────────────────────────────────────────────┤
 │ Switch chip SDK - user-space C library                 │ Handles generic "chip-level" logic
 ├────────────────────────────────────────────────────────┤
 │ Board Adaptation Layer - user-space C library          │ Handles "board-level hardware differences"
 └──────────────────────────┬─────────────────────────────┘
                            │ Interacts via UIO /dev nodes or character devices
 ┌──────────────────────────▼─────────────────────────────┐
 │ Linux kernel space (Kernel Driver / .ko driver layer)  │
 └────────────────────────────────────────────────────────┘
```

### 1.1 Switch Chip SDK (Chip-Level Generic API, Pure Software Abstraction)
- **Ownership**: Supplied by the switch chip vendor, usually licensed as a proprietary SDK, and not distributed in this repository.
- **Responsibility**: It knows only about **the switch chip core itself** (such as industrial gigabit switch chips or 10-gigabit multi-port Ethernet chips) and does not care at all which board the chip is soldered onto.
- **Scope**: Generic abstractions of the chip's internal functions, for example:
  - "Insert/delete static entries in the MAC address table";
  - "Create VLAN 10 and add physical ports 1–4 as members";
  - "Configure port rate-limiting queues and QoS policies";
  - "Set global link aggregation (LACP) control parameters".

### 1.2 Board Switch Adaptation Layer (Board Difference Adaptation Layer, the Board's "Translator")
- **Ownership**: Written per board by the board vendor or product team, located in `plugins/switch_offload/boards/`, and compiled with the optional plugin.
- **Pain point**: With the very same switch chip, different hardware vendors design wildly different boards:
  - Vendor A designs a 24-port all-fiber board, with SFP optical modules on the CPU's I2C-1 bus and status LEDs driven by GPIO 5;
  - Vendor B designs an 8-port copper board with an external PHY chip of a specific model at PHY address `0x10`, with its reset pin on a shift register;
  - Vendor C designs an industrial control board that must support feeding an external hardware watchdog and monitoring a specific power management chip.
- **Responsibility**: The Board Switch Adaptation layer (code prefix `bsa_`) implements a set of callback functions for each specific board (such as `bsa_board_rk3568_custom.c`).
- **How it works**: When the upper layer calls `bsa_port_admin_state_set(port, enable)`, the board switch adaptation layer internally drives the specific GPIO pin to enable power, or configures the optical module through a Linux I2C node.

---

## 2. How Does the Switch Chip Solution Operate Hardware Through `.ko` Drivers? (A Special High-Performance Design)

In the traditional Linux model, network communication goes through standard kernel network device drivers (`struct net_device`, such as `e1000.ko` and `r8169.ko`). Packets are processed by the kernel network stack (`sk_buff`).

In enterprise Ethernet switches, however, there are dozens of gigabit/10-gigabit ports; if all line-rate traffic went through the Linux kernel, the CPU would be saturated and grind to a halt immediately. Therefore, some high-performance networking solutions adopt an **unconventional bypass pass-through design**.

> This is an optional approach, not Octopus's default path. Octopus treats the Linux kernel FIB as the single source of truth (RFC-06) and prefers DSA / switchdev kernel drivers. Only when the switch chip has no mainline kernel driver does it fall back to the UIO + user-space SDK approach below, isolated as the `switch_offload` plugin. Under the UIO approach, ports are no longer standard netdevs, so ecosystems that depend on the kernel network stack, such as FRR and iptables, cannot be used directly — which is exactly what RFC-06 aims to avoid.

```text
 [ User-space main process ]
   octopus_app (includes the switch chip SDK)
       │
       │ 1. open("/dev/uio0")
       │ 2. mmap() maps the switch chip registers into its own virtual memory
       ▼
 ┌────────────────────────────────────────────────────────┐
 │ Linux kernel space                                     │
 │                                                        │
 │   Loads a lightweight driver: switch_uio.ko            │
 │   (based on the native Linux UIO framework)            │
 │   Role: only exports the physical register memory      │
 │   region to user space; no business logic              │
 └────────────────────────────┬───────────────────────────┘
                              │
                              ▼
 ┌────────────────────────────────────────────────────────┐
 │ Physical hardware: Ethernet switch chip register space │
 └────────────────────────────────────────────────────────┘
```

1. **An extremely minimal UIO driver**: The kernel loads only a tiny driver, `switch_uio.ko` (a few KB);
2. **Direct memory mapping**: At startup, the main program calls `open("/dev/uio0")` and then `mmap()` to pull the entire control register address space of the switch chip directly into user-space memory;
3. **Maximum performance**: When the switch chip SDK configures the hardware from user space, it is essentially writing registers directly through pointers into this memory region — **zero system call overhead and zero kernel-mode transitions**.

---

## 3. When a User Has Their Own Special `.ko` Driver, How Exactly Should It Be Integrated?

Developers often wonder: *"If my board has my own special hardware attached (for example a dedicated FPGA, a special SFP controller, or an encryption chip) and I've written a kernel driver `.ko` for it, whom do I talk to? How do they interact?"*

Depending on which business domain the hardware belongs to, the architecture defines **three standard scenarios**:

---

### Scenario 1: The User Driver Is an "Ethernet Network Peripheral" (Must Integrate with the Board Switch Adaptation Layer)

> **Typical case**: A customer builds a custom switch board with a special **SFP optical module power chip** or a particular **Ethernet PHY chip** attached. The customer writes a dedicated kernel driver, `my_sfp_ctrl.ko`, which creates the character device `/dev/sfp_ctrl` when loaded and enables optical modules via `ioctl`.

**Integration mechanism and code flow**:
This hardware is part of the Ethernet port subsystem, so it **must be integrated in the board switch adaptation layer code**.

```text
 ┌──────────────────────────────────────────────┐
 │ octopus_app core business logic              │
 └──────────────────────┬───────────────────────┘
                        │ Calls the standard API: bsa_port_admin_state_set(port, enable)
                        ▼
 ┌──────────────────────────────────────────────┐
 │ Board switch adaptation layer code           │
 │ (e.g. bsa_custom_board.c)                    │
 │                                              │
 │  /* Operate the customer's special character │
 │     device inside the BSA callback */        │
 │  int fd = open("/dev/sfp_ctrl", O_RDWR);     │
 │  ioctl(fd, SFP_CMD_ENABLE_POWER, &port);     │
 │  close(fd);                                  │
 └──────────────────────┬───────────────────────┘
                        │ VFS system calls (open/ioctl)
                        ▼
 ┌──────────────────────────────────────────────┐
 │ Linux kernel: customer driver                │
 │ (my_sfp_ctrl.ko)                             │
 │ Creates device node: /dev/sfp_ctrl           │
 └──────────────────────┬───────────────────────┘
                        │ Toggles hardware pins
                        ▼
 ┌──────────────────────────────────────────────┐
 │ Physical hardware (SFP controller /          │
 │ optical module power supply)                 │
 └──────────────────────────────────────────────┘
```

- **Architectural benefit**: The upper-layer switch chip SDK and `octopus_app` business code need no changes at all and stay clean; hardware differences are fully sealed inside a single board-level file in the board switch adaptation layer.

---

### Scenario 2: The User Driver Is a "General-Purpose Industrial/Business Peripheral" (Completely Independent of the Switch Chip SDK!)

> **Typical case (an important conceptual breakthrough)**:  
> A customer develops an edge gateway product on the RK3568 or SSD20X, with an **FPGA data acquisition card**, a **Chinese national-cryptography (SM-series) hardware encryption chip**, or a **multi-channel RS485 expansion UART chip** attached to the board. The customer writes their own kernel driver, `my_fpga.ko` (part of the customer's project, not part of this Octopus repository), which creates `/dev/my_fpga0`.

**Key conceptual breakthrough**:  
**In this case there is no need to go through the board switch adaptation layer or the switch chip SDK at all!**  
The responsibilities of the board switch adaptation layer and the switch chip SDK are limited to Ethernet networking. For general-purpose industrial hardware peripherals, the Octopus platform provides two minimal, efficient integration paths:

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                       Customer hardware solution                       │
 │       (e.g. an FPGA data acquisition card with a my_fpga.ko driver)    │
 └──────────────────────────────────┬─────────────────────────────────────┘
                                    │ insmod creates device node /dev/my_fpga0
                                    ▼
       ┌────────────────────────────────────────────────────────────┐
       │        How should the user layer choose to integrate?      │
       └──────────────┬──────────────────────────────┬──────────────┘
                      │ Option A: In-Tree            │ Option B: Out-of-Tree
                      │ (high-performance mode)      │ (loosely coupled mode)
                      ▼                              ▼
 ┌─────────────────────────────────────────┐  ┌──────────────────────────┐
 │ Write a module inside the Octopus core  │  │ Customer writes a        │
 │ platform: modules/fpga_mgr/             │  │ standalone process:      │
 │  - open("/dev/my_fpga0") directly to    │  │ /usr/bin/fpga_daemon     │
 │    read/write data                      │  │  - Freely reads/writes   │
 │  - Register structs in the JSON-RPC     │  │    /dev/my_fpga0         │
 │    method tree                          │  │  - Calls the Unix socket │
 │  - Automatically gets Web API / CLI /   │  │    /var/run/             │
 │    octopus_config persistent config     │  │    json_ipc.socket       │
 │    saving                               │  │                          │
 └─────────────────────────────────────────┘  └──────────────────────────┘
```

#### Option A: In-Tree Mode (Tightly Integrated with the Platform, Full Management Plane in One Step)
- **Approach**: Add a new module under Octopus's `modules/` directory (such as `modules/fpga_mgr/`);
- **Operation**: In the module's C++ business thread, call `open("/dev/my_fpga0", O_RDWR)` directly for routine reads and writes;
- **Big payoff**: Just register the acquired parameters once with the `StructReadWrite` template:
  1. The **Web UI** automatically gets Ajax endpoints for querying and configuring the FPGA;
  2. The **octopus_cli console** automatically gets commands like `show fpga status`;
  3. **octopus_config configuration saving** automatically supports persisting parameters across power loss and restoring them at boot.

#### Option B: Out-of-Tree Mode (Standalone External Process, Zero-Risk Isolation)
- **Approach**: Without changing a single line of the Octopus main program's source, the customer writes a standalone Linux daemon in C/C++/Python (such as `/usr/bin/fpga_daemon`);
- **Operation**: This process exclusively owns `/dev/my_fpga0` for high-frequency data acquisition. When it needs to sync status to the system or receive parameters pushed from the Web UI, it calls the lightweight `liboctopus_client` to communicate with the local `/var/run/json_ipc.socket`;
- **Big payoff**: **Complete fault isolation**. Even if the customer's driver has a bug that makes this child process segfault and exit, the Octopus main process stays rock solid, and octopus_supervisor immediately restarts the exited child process to self-heal!

---

### Scenario 3: Driver Lifecycle — Who Loads the `.ko`, and When?

Many engineers are confused by the filesystem environment of OFI, two-stage boot (initramfs/rootfs), SquashFS, and OverlayFS, and don't know where drivers should be placed or when they take effect.

In the standard Octopus boot flow, the kernel driver loading lifecycle is as follows:

```text
           U-Boot boots initramfs (Kernel + minimal Initramfs)
                                       │
                                       ▼
           octopus_loader runs in the initramfs stage and mounts the root filesystem
           (read-only SquashFS + writable OverlayFS merged on top)
                                       │
                                       ▼
           [Key moment: once rootfs is ready, exec /sbin/octopus_supervisor (PID 1)]
                                       │
                                       ├─► Step 1: octopus_supervisor first runs the kernel driver loading service
                                       │           Service definition: /etc/octopus/services.d/kmod.conf
                                       │           Module list: /etc/octopus/modules.list
                                       │           ─────────────────────────────────────────
                                       │           ① Load basic Linux drivers:
                                       │              modprobe uio
                                       │              modprobe i2c-dev
                                       │           ② Load user-defined business drivers:
                                       │              modprobe my_fpga (module placed in extra/ and depmod run)
                                       │           ─────────────────────────────────────────
                                       │           ==> The /dev/my_fpga0 device node now exists!
                                       │
                                       ├─► Step 2: Once drivers are ready, octopus_supervisor starts application services by dependency
                                       │           - Start /usr/bin/octopus_app (main process)
                                       │           - Start /usr/bin/fpga_daemon (customer standalone process)
                                       ▼
           The main application and customer process safely open("/dev/my_fpga0"); business starts normally!
```

- **Storage path**: The user's `.ko` driver files are packaged into `rootfs.squashfs` and placed in the fixed directory `/lib/modules/$(uname -r)/extra/`, with `depmod` run at build time. The root filesystem is read-only, so modules cannot be copied in at runtime; they can only be updated via OFI upgrades;
- **Load configuration**: Simply append a line with the module name to `/etc/octopus/modules.list`, and the supervisor's `kmod` service will `modprobe` them in order. Services that depend on the driver declare `after = kmod` in their own definitions; do not add `sleep` calls to wait for device nodes;
- **Kernel version binding and signing**: A `.ko` must strictly match the kernel version and configuration; when the kernel is upgraded, drivers must be rebuilt together and placed in the same OFI. When kernel module signing is enabled (`CONFIG_MODULE_SIG_FORCE`), third-party drivers must be signed with the same key or they will be refused. This is the continuation of the Secure Boot chain of trust beyond the kernel.

---

## 4. Summary and Decision Quick Reference

| Peripheral Category | Typical Examples | Switch Chip SDK Required? | Where Driver Operation and Interaction Happen |
| :--- | :--- | :--- | :--- |
| **Ethernet switch chip** | Industrial/enterprise Ethernet switch chips | **Loaded as needed** | The kernel loads `switch_uio.ko` or a DSA driver; the switch chip driver either reads/writes physical hardware registers directly from user space via `mmap`, or the kernel takes over. |
| **Switch board port peripherals** | Port SFP optical modules, external PHY chips, port status LEDs | **Required** | The driver creates `/dev/i2c-*` or a character device in the kernel; `open/ioctl` calls to drive the hardware are written **in the board-level adaptation source of the board switch adaptation layer**. |
| **General-purpose industrial business hardware** | FPGA acquisition cards, encryption chips, multi-channel UART chips, ADC sampling chips | **Not required at all!** | The kernel loads `my_hw.ko` normally, creating `/dev/my_hw`.<br>Users can `open` it directly with standard POSIX APIs from an **Octopus In-Tree module** or an **external standalone child process**. |

---

## 5. Core Architectural Question: Do We Actually Need a User-Space HAL Layer?

After clarifying the boundary between the switch chip SDK and kernel drivers, many developers raise a deeper question:  
**"Since we need to span chip platforms (RK3568, SSD20X, Allwinner), does Octopus need to write a large, generic user-space hardware abstraction layer (HAL, such as GPIO_HAL, I2C_HAL, UART_HAL) like Android does?"**

**Core conclusion: It's not that no HAL is needed — it's that "the Linux kernel itself is already the ultimate HAL for the vast majority of general-purpose hardware"! We must never reinvent the wheel in user space.**

---

### 5.1 Why General-Purpose Peripherals Don't Need a User-Space HAL (The Linux Kernel Is a Natural HAL)

In embedded Linux, **"kernel driver frameworks + the Device Tree (DTS)"** already form the most standardized, most thoroughly cross-chip hardware abstraction layer in the industry:

```text
               Octopus application layer (business main program)
                 │
                 │ Sees fully cross-chip, generic POSIX / Linux VFS standard interfaces:
                 │  - Serial communication: /dev/ttyS0, /dev/ttyFIQ0 (standard termios API)
                 │  - NIC configuration: eth0, eth1 (standard Linux Socket / Netlink / ioctl API)
                 │  - I2C bus: /dev/i2c-1 (standard i2c-dev ioctl API)
                 │  - Hardware watchdog: /dev/watchdog (standard Linux Watchdog API)
                 │  - Block devices and partitions: /dev/mmcblk0, /dev/mtd0
                 ▼
 ┌────────────────────────────────────────────────────────────┐
 │     The Linux kernel itself is the natural, ultimate HAL   │
 ├─────────────────┬──────────────────┬───────────────────────┤
 │ Rockchip RK3568 │ SigmaStar SSD20X │ Allwinner T507 / H616 │
 │ Device tree     │ Device tree      │ Device tree           │
 │ (.dts)          │ (.dts)           │ (.dts)                │
 └─────────────────┴──────────────────┴───────────────────────┘
```

#### A Real-World Comparison:
- On the **RK3568**, the I2C-1 controller's physical base address is `0xFE5A0000` and its pin is `GPIO0_B3`;
- On the **SSD20X**, the I2C-0 controller's physical base address is `0x1F220000` and its pin is `PAD_I2C0_SCL`;
- **Where are the differences smoothed out?** They are absorbed in each chip's **kernel driver source and board-level device tree (DTS)**!
- **What does the application layer see?** On both chips, the application layer sees the standard `/dev/i2c-0` device file. The application layer simply uses `open()`, `read()`, and `write()`; **there is no need to wrap yet another so-called `i2c_hal_read()` in the application layer**.

---

### 5.2 Why Did Some Early Solutions Build a Switch Chip SDK / Board Switch Adaptation Layer in User Space?

Because, in pursuit of maximum performance, early switch chip solutions did something **"unconventional"**:
To let the Ethernet switch chip reach hardware line-rate forwarding of tens of Gbps, they **completely abandoned the Linux kernel network driver framework (`struct net_device`)** and used UIO to map the chip's physical registers directly into user space.

**The resulting side effect:**
Once you voluntarily give up the Linux kernel driver, every abstraction the Linux kernel would have provided for you (such as the netdev abstraction, the PHY state machine abstraction, and the MDIO bus abstraction) **stops working entirely**!
As a result, vendors had no choice but to painstakingly rebuild their own HAL in user space:
1. **Switch chip SDK**: the switch chip register abstraction layer (a chip-level user-space HAL);
2. **Board switch adaptation layer**: the pin and optical port/PHY chip abstraction layer for each switch board (a board-level user-space HAL).

---

### 5.3 What Kind of HAL Do We Actually Need in the Octopus Platform?

Based on the principles above, Octopus establishes a clear HAL architecture philosophy:

> **Core principles:**  
> 1. **Never reinvent driver abstractions that Linux already provides** (UART, I2C, SPI, Watchdog, and standard NICs go directly through the Linux kernel's standard interfaces);  
> 2. **Keep only one "ultra-lightweight board difference adaptation interface (Board HAL)"**, dedicated to the handful of "board-level chores" that the Linux kernel does not define uniformly.

These "board-level chores" are usually limited to the following 4 items:

```text
                       Octopus core business layer
                               │
                               ▼
 ┌────────────────────────────────────────────────────────────┐
 │       Octopus ultra-lightweight board adaptation layer     │
 │       (Board HAL)                                          │
 ├────────────────────────────────────────────────────────────┤
 │ 1. Unique device identity (board_get_mac /                 │
 │    board_get_serial_no)                                    │
 │ 2. Factory reset button detection                          │
 │    (board_is_factory_reset_triggered)                      │
 │ 3. System status LEDs (board_set_status_led)               │
 │ 4. [Optional plugin] Low-level switch network adaptation   │
 │    (board switch adaptation layer interface)               │
 └─────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
        Concrete low-level implementations (each board writes one standalone .c adaptation file)
```

#### Why Do Only These Few Things Need a Board HAL?
Take **"reading the device's factory physical MAC address and serial number"** as an example:
- On an **RK3568 industrial control board**: the MAC address may be burned into the CPU's internal OTP / eFuse area, or derived from the eMMC CID;
- On an **SSD20X core board**: the MAC address is stored at offsets `0x00~0x05` of an external I2C EEPROM;
- On an **Allwinner gateway mainboard**: the MAC address is stored in a dedicated `factory` MTD partition on SPI NAND Flash.

The business layer (the network configuration module, the Web status page) only needs to know what the device's physical MAC actually is; it **should never care whether the current board reads it from an EEPROM or from eFuses**.

Therefore, Octopus only needs to provide an extremely simple, compact board-level C interface. Below is a **simplified illustration** of the idea; the actual interface is the `board_hal.h` (in `liboctopus_board`) frozen in section 6.2 of the architecture document:

```c
// Illustrative code, not the official interface

typedef enum {
    LED_SYS_RUN,
    LED_SYS_ALARM,
    LED_SYS_CLOUD
} board_led_id_t;

typedef enum {
    LED_OFF,
    LED_ON,
    LED_BLINK_SLOW,
    LED_BLINK_FAST
} board_led_state_t;

typedef struct {
    // 1. Read the factory-programmed MAC address of the mainboard
    int (*get_factory_mac)(uint8_t mac_out[6]);
    
    // 2. Read the mainboard serial number (SN)
    int (*get_serial_number)(char *sn_out, size_t max_len);
    
    // 3. Control the system run status LEDs
    void (*set_status_led)(board_led_id_t led, board_led_state_t state);
    
    // 4. Detect whether the physical factory reset button (Reset Button) was long-pressed
    //    The official interface is octopus_board_factory_reset_triggered() (long-press detection and LED feedback live inside the HAL);
    //    the name here is only this section's illustrative spelling; section 6.2 of the architecture document is authoritative
    bool (*is_factory_reset_triggered)(void);
} octopus_board_ops_t;

// Register the current board's implementation
int octopus_board_register(const octopus_board_ops_t *ops);
```

For specific boards:
- Write a `board_rk3568_evb.c` for the RK3568 (reading the MAC from eFuses and the button via Linux `gpio-keys`);
- Write a `board_ssd20x_gw.c` for the SSD20X (reading the MAC from the EEPROM over I2C);
- The upper-layer main program is completely unaware of board hardware differences, and the core code stays extremely clean.

---

## 6. Guidance for the Octopus Generic Architecture Design

1. **Clear boundaries; never abuse the switch chip SDK**:  
   When building a generic platform spanning chip platforms (RK3568, SSD20X, Allwinner), do not stuff non-networking code into the switch chip SDK. The switch chip SDK should be fully isolated in the optional plugin package `plugins/switch_offload/`. If the hardware has no dedicated Ethernet switch ASIC, simply turn off the build macro and the platform remains complete.
2. **Stick to the standard Linux philosophy**:  
   For all kinds of user extension peripherals, the kernel provides standard character device nodes (`/dev/xxx`) through drivers; Octopus provides an extremely convenient integration framework in user space (In-Tree modules auto-generate Web/CLI; Out-of-Tree processes get a high-speed IPC socket).
3. **A minimal Board HAL is good enough**:  
   Don't write lengthy, heavyweight user-space driver wrappers. Keep only the lightweight `board_hal` to absorb differences in MAC/SN, status LEDs, and the reset button, so the whole software framework is both highly cohesive and extremely easy to port quickly between chips!
