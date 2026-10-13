# QEMU Emulation and End-to-End Embedded Testing Strategy Guide

**English** | [简体中文](./01-qemu-test-strategy.md)

> **Note**: This is an English translation. The Simplified Chinese version is authoritative; if the two differ, the Chinese version prevails.

> **Purpose**: Answers key engineering questions that arise during development of the Octopus platform, such as **"Can U-Boot be debugged in QEMU?", "How should work be split between QEMU and real hardware, and what are the trade-offs?", and "How do we build the most efficient test environment?"**. It proposes a three-tier test range that balances **development speed, seamless breakpoint debugging, CI automation, and real-hardware reliability**.

---

## 1. The Fundamental Question: Debug U-Boot in QEMU or Go Straight to Real Hardware?

Many embedded developers have this thought early on: *"Debugging U-Boot on the board is too painful — every change needs a reflash, and it can brick the board. Can we fully emulate the RK3568 or SSD20X U-Boot inside QEMU?"*

Here is the **honest truth and the dividing line**, learned from countless pitfalls across the industry:

### 1.1 The Reality: The Physical Hardware of Mainstream SoCs Cannot Be Fully Emulated by QEMU
- **There is no QEMU machine model for the specific SoC**:  
  Mainline QEMU supports the generic `virt` machine (a standard virtualization platform) and a very small number of open-source development boards (such as VersatilePB and Raspberry Pi). **QEMU simply has no `rk3568` or `ssd20x` board model!**
- **Chip-specific peripherals are missing**:  
  When the RK3568 boots, it depends on the on-SoC BootROM, vendor-supplied boot firmware such as DDR initialization (usually delivered as binary packages with the chip BSP and **not provided by this repository**), a complex clock tree (CRU), a power management IC (PMIC), and a secure execution environment (ATF/TrustZone). None of this hardware is emulated in QEMU. If you force a U-Boot built for the RK3568 to run in QEMU, it will crash on the very first assembly instruction.

---

### 1.2 The Practical Dividing Line: When to Use QEMU and When Real Hardware Is Mandatory

| Debug Task / Content | Suitable Platform | Why? |
| :--- | :--- | :--- |
| **OFI firmware parsing command (`cmd_ofi.c`)** | **QEMU (strongly recommended)** | Pure software logic! Header validation, TLV parsing, memory copying, and decompression can be single-stepped with breakpoints in QEMU with results in seconds. |
| **A/B partition arbitration algorithm and environment variable logic** | **QEMU (strongly recommended)** | Pure state-machine logic! Testing `boot_count` increments, `boot_active` switching, and timeout rollback — QEMU lets you script 1000 simulated reboots. |
| **octopus_loader and OverlayFS mounting** | **QEMU (strongly recommended)** | Involves SquashFS verification, loop mounts at offsets, tmpfs, and `switch_root`; the QEMU kernel supports all of it perfectly, and it is 10× faster than real hardware. |
| **UBI / NAND power-loss and bad-block logic** | **QEMU + nandsim (recommended)** | The kernel's `nandsim` emulates NAND and can inject bad blocks and bit flips, and UBI's debugfs supports simulated power loss, covering SLOT writes and DATA corruption recovery without real NAND. |
| **Octopus core main process and Web/IPC interfaces** | **QEMU / local host** | These are entirely POSIX-level applications; attach GDB and capture core dumps directly in the emulator — no board needed at all. |
| **DDR memory timing, chip clocks, PMIC power supply** | **Real hardware required** | Involves physical electrical characteristics that QEMU simply cannot emulate. |
| **Actual eMMC / SPI NAND physical read/write and flashing** | **Real hardware required** | Chip vendors' USB production-line flashing protocols (such as Rockchip Maskrom and Sigmastar ISP); the tools and protocols are supplied by the vendors. |
| **Ethernet switch chip (switch chip UIO / register access)** | **Real hardware required** | Physical register access to the switch ASIC; there is no generic QEMU emulation model. |

---

## 2. Industry Best Practice: The Most Efficient Three-Tier Test Pyramid

Don't pile every test onto real hardware (too slow and hard to diagnose), and don't pin all your hopes on QEMU (it cannot replace physical hardware). The smartest test architecture is **layered and progressive**:

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │ Layer 3: Physical hardware range (RK3568 / SSD20X real boards)         │
 │  - Validate physical peripheral drivers (UIO, PHY, Ethernet, hardware  │
 │    watchdog, physical buttons)                                         │
 │  - Automated serial test rig: batch power-loss tests and A/B partition │
 │    anti-hang disaster-recovery tests                                   │
 ├────────────────────────────────────────────────────────────────────────┤
 │ Layer 2: QEMU ARM64 / ARM32 full-system black-box emulation range      │
 │  - Machine models: qemu-system-aarch64 -M virt (maps to RK3568)        │
 │                    qemu-system-arm -M virt (maps to SSD20X and other   │
 │                    32-bit ARMv7)                                       │
 │  - Run generic ARM64 U-Boot: debug cmd_ofi and the A/B dual-boot       │
 │    algorithm (with GDB single-stepping)                                │
 │  - Run the full business loop: initramfs + OverlayFS +                 │
 │    octopus_supervisor + octopus_app                                    │
 │  - Port forwarding: the host browser accesses the Web UI and JSON-RPC  │
 │    inside QEMU directly                                                │
 ├────────────────────────────────────────────────────────────────────────┤
 │ Layer 1: Local PC (x86_64 Linux) white-box ultra-fast unit tests       │
 │  - Build x86 versions of liboctopus_basics, liboctopus_json,           │
 │    octopus_cli, octopus_config                                         │
 │  - Second-level builds, AddressSanitizer leak detection, GoogleTest    │
 │    automated tests                                                     │
 │  - libFuzzer fuzzing: OFI header parsing, config text parsing,         │
 │    JSON-RPC request parsing                                            │
 └────────────────────────────────────────────────────────────────────────┘
```

- **Why Layer 2 must also run ARM32**: The SSD20X is 32-bit ARMv7, where `time_t`, `long`, pointer width, struct alignment, and 64-bit atomic operations all differ from ARM64. If you test only on ARM64, these issues won't surface until you reach real hardware;
- **Why the parsers must be fuzzed**: The OFI header parsing code also runs inside U-Boot; any out-of-bounds read or write could be exploited by a crafted firmware package to bypass signature verification. The configuration parser and the JSON-RPC parser face network input directly. Running libFuzzer + ASan continuously against these three at Layer 1 has the lowest cost and the highest payoff;
- **Static analysis**: In CI, run `clang-tidy` and `cppcheck` on `octopus-core`, and `checkpatch.pl` on U-Boot / kernel patches.

---

## 3. Hands-On: Building a QEMU ARM64 Emulation Environment Step by Step

The goal of this setup: **On your development machine (Linux or WSL2), with no hardware at all, run the entire flow — U-Boot reading the OFI image, booting Linux, octopus_supervisor launching the main program, and the host browser opening the Web page!**

### 3.1 Core Idea: Use QEMU's Standard `virt` Platform
We don't need to emulate the RK3568 board; instead we use the standard ARM64 `virt` architecture:
1. Build a generic U-Boot from `qemu_arm64_defconfig`;
2. Compile our `cmd_ofi.c` into this generic U-Boot;
3. Build a generic ARM64 Linux kernel (with OverlayFS and SquashFS support);
4. Use a virtual disk (VirtIO disk) to emulate the eMMC storage partitions.

### 3.2 Creating the Virtual Disk and Partitions
Generate a virtual disk on the host to emulate the on-board eMMC:
```bash
# 1. Create a 512MB virtual disk file
qemu-img create -f raw emmc_disk.img 512M

# 2. Partition it; GPT partition names (PARTLABEL) follow the naming convention in section 4.3 of the architecture document
parted -s emmc_disk.img mklabel gpt
parted -s emmc_disk.img mkpart env     1MiB   2MiB   # Partition 1: env (U-Boot redundant environment)
parted -s emmc_disk.img mkpart factory 2MiB   4MiB   # Partition 2: factory (factory data)
parted -s emmc_disk.img mkpart slot_a  4MiB   132MiB # Partition 3: slot_a (128MB)
parted -s emmc_disk.img mkpart slot_b  132MiB 260MiB # Partition 4: slot_b (128MB)
parted -s emmc_disk.img mkpart data    260MiB 511MiB # Partition 5: data (user persistent area)

# 3. Write the packaged Octopus .ofi firmware into slot_a (4MiB offset = sector 8192 of 512 bytes)
#    This is a virtual eMMC block device, so dd is fine; never use dd on real raw NAND, see reliability topic 6
dd if=octopus_fw_a.ofi of=emmc_disk.img seek=8192 bs=512 conv=notrunc
```

### 3.3 Launching QEMU with U-Boot in One Command
```bash
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a57 \
    -m 1024M \
    -nographic \
    -bios u-boot.bin \
    -drive file=emmc_disk.img,format=raw,id=emmc,if=none \
    -device virtio-blk-device,drive=emmc \
    -netdev user,id=net0,hostfwd=tcp::8080-:80,hostfwd=tcp::2222-:22 \
    -device virtio-net-device,netdev=net0
```
**Key parameters explained:**
- `-bios u-boot.bin`: QEMU jumps straight into the compiled generic ARM64 U-Boot at power-on;
- `-device virtio-blk-device`: Attaches the virtual disk as a VirtIO device (recognized as a disk by both U-Boot and Linux);
- `hostfwd=tcp::8080-:80`: **Handy port forwarding!** Maps port 80 inside QEMU (the Hiawatha Web server) to `http://localhost:8080` on the host. You can open the device's Web management page directly in your desktop browser!

### 3.4 The Killer Advantage of Debugging U-Boot in QEMU: GDB Breakpoints
On real hardware, U-Boot debugging usually comes down to `printf` debugging; as soon as memory gets corrupted or a pointer goes out of bounds, the board silently hangs, which is extremely painful.

**Debugging U-Boot in QEMU gives you a "god's-eye view":**
1. Add `-s -S` when launching QEMU (`-s` starts a GDB server on port 1234; `-S` halts at the very first assembly instruction after power-on and waits);
2. In another terminal, start the cross-platform GDB debugger:
   ```bash
   aarch64-linux-gnu-gdb u-boot
   (gdb) target remote localhost:1234
   (gdb) b do_ofi_boot          # Set a breakpoint right at the entry of your OFI parsing function!
   (gdb) continue
   ```
3. U-Boot stops precisely on your line of `cmd_ofi.c` code! You can `step` through at any time, print variables with `p *header`, and inspect TLV lengths. **This is a debugging experience real hardware can never give you!**

---

## 4. How Do We Make Real-Hardware Testing "Convenient and Efficient"? (Avoiding the Nightmare of Repeated Flashing)

Once the code passes in QEMU and you port it to real RK3568 / SSD20X development boards, **never reflash the entire firmware into eMMC/Flash for every change** — that is extremely inefficient and wears out the storage chips.

The **"three key techniques"** for efficient industrial-grade debugging on real hardware:

### Technique 1: U-Boot Network Loading (TFTP + NFS Network Root Filesystem)
This is the golden rule for embedded driver and low-level debugging:
```text
  Dev board (RK3568)                    Dev host (PC Linux / Ubuntu)
  [ U-Boot ] ────────── TFTP protocol ─────────> [ TFTP server ]
  Flashed onto the board only once               Downloads the latest kernel Image
        │
        ▼ Boot the kernel
  [ Linux kernel ] ───── NFS protocol ─────────> [ NFS server directory ]
                                                  Mounts the rootfs on the PC
```
- **How it works**: The board's eMMC only needs U-Boot flashed once; after that, the kernel and filesystem live entirely in a directory on your computer;
- **Huge payoff**:
  After building `octopus_app` on your computer, with no flashing at all, press Enter in the board's serial console and the latest program runs immediately! The edit-build-test cycle takes just 5 seconds!

### Technique 2: Automated Serial Interaction and Regression Testing (Python `pexpect` / `pytest`)
Don't sit watching the serial console and typing commands by hand; write a simple Python script that sends and receives over the serial port automatically:
```python
# test_reboot_cycle.py (automated disaster-recovery test script)
import serial, time

ser = serial.Serial('/dev/ttyUSB0', 115200, timeout=10)

def wait_for(keyword):
    start = time.time()
    buf = ""
    while time.time() - start < 30:
        line = ser.readline().decode('utf-8', errors='ignore')
        buf += line
        if keyword in line:
            return True
    raise TimeoutError(f"Timed out waiting for: {keyword}")

# Automatically test 100 reboots and A/B switching
for i in range(100):
    print(f"=== Automated reboot test #{i} ===")
    ser.write(b"reboot\n")
    wait_for("U-Boot 2020.10")
    wait_for("OFI Header Valid")
    wait_for("Starting octopus_supervisor...")
    wait_for("Octopus application ready")
    print("Boot healthy, self-check passed!")
```

### Technique 3: Smart Power Strip / Relay for "Automated Forced Power-Loss Testing"
For a highly reliable dual-backup firmware system, the biggest fear is: **if power is pulled halfway through a firmware write, or halfway through boot, will the device brick?**
- Buy an inexpensive programmable USB relay or Wi-Fi smart power strip;
- Write a script: issue a firmware upgrade command -> random sleep of 2–8 seconds -> relay cuts power -> power back on -> check whether U-Boot automatically detects the fault and rolls back to the backup partition;
- Run 500 automated power cuts overnight; only if the device is still online and working normally the next day does it pass this round. The NFR requires 10,000 cycles.
- **Power-cut points must cover the critical windows**: Purely random sleeps rarely hit millisecond-scale windows such as "during an ENV write" or "between rename and the directory fsync". Have the firmware under test emit markers over the serial port or GPIO before and after critical steps, and have the relay script trigger power cuts on those markers to cover them deliberately;
- **After each round, check more than "it boots"**: Also check that the configuration is intact (correct checksum trailer), that it did not unexpectedly fall back to the factory configuration, whether DATA was rebuilt, and that bootcount and slot states match expectations;
- **Long-term soak testing and Flash wear**: Run continuously for weeks, tracking memory trends (`show memory modules`) and Flash write volume (the eMMC `EXT_CSD` lifetime fields and UBI erase counters), and extrapolate from the write rate whether the flash would wear out within the product's lifetime.

> **Note**: The NFS root filesystem in Technique 1 bypasses `octopus_loader`, OverlayFS, and the A/B flow, so it is only suitable for debugging application logic. Tests related to the boot chain and power loss must be run with production firmware on real storage.

---

## 5. Summary: Test Platform Deployment Matrix

| Development Phase | Recommended Test Environment | Goals and Deliverables |
| :--- | :--- | :--- |
| **Phase 1: Extracting core business libraries**<br>(liboctopus_basics/json/octopus_cli) | **Local PC (x86_64)** | Run gcc/cmake directly on Ubuntu/WSL2 with second-level builds, and pass unit tests for all basic data structures. |
| **Phase 2: OFI boot and A/B arbitration logic**<br>(cmd_ofi.c / octopus_loader / octopus_supervisor) | **QEMU ARM64 (virt)** | Run generic U-Boot on a virtual disk and use GDB single-step breakpoints to get OFI extraction, OverlayFS mounting, and Web responses working. |
| **Phase 3: RK3568 low-level peripheral bring-up**<br>(DDR / PMIC / NIC / eMMC drivers) | **RK3568 real hardware (TFTP/NFS)** | Get networking and the device tree working, avoid frequent flashing, and enable fast cross-compile verification. |
| **Phase 4: Full-system acceptance and reliability soak testing** | **RK3568 physical hardware (automated rig)** | Flashing real physical eMMC partitions, relay-driven abnormal power-loss tests, and long-connection high-concurrency stress tests. |

This combination of **"ultra-fast PC unit tests + QEMU architecture emulation + real-hardware network debugging + automated power-loss soak testing"** not only frees you from the pain of frequent flashing and bricked boards, but also dramatically shortens the project's development cycle!
