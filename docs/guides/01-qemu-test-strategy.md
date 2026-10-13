# QEMU 仿真模拟与嵌入式全流程测试方案指南

> **文档定位**：深入解答在 Octopus 通用底座开发过程中，关于 **“U-Boot 调试能否用 QEMU”、“QEMU 与真实硬件的分工与优劣”、“如何搭建最高效的测试环境”** 等关键工程问题。提出一套兼顾**开发速度、无感断点调试、CI 自动化与真机可靠性**的三层测试靶场体系。

---

## 1. 灵魂拷问：调试 U-Boot 该用 QEMU 还是直接上真机？

很多嵌入式开发者在早期都会产生一个念头：*“在板子上调 U-Boot 太麻烦了，改一次就要烧录一次，还会变砖，能不能在 QEMU 里面完整模拟 RK3568 或 SSD20X 的 U-Boot？”*

我们直接给出经过工业界无数次踩坑后的**大实话与分水岭**：

### 1.1 现实痛点：主流 SoC 的物理硬件无法被 QEMU 完整模拟
- **没有针对具体 SoC 的 QEMU 机器模型**：  
  QEMU 官方主线支持的是通用的 `virt`（标准虚拟化平台）或极少数开源开发板（如 VersatilePB、Raspberry Pi）。**QEMU 根本没有 `rk3568` 或 `ssd20x` 的主板模型！**
- **芯片专属外设缺失**：  
  RK3568 启动时依赖 SoC 片内 BootROM、厂商提供的 DDR 初始化等启动固件（通常随芯片 BSP 以二进制包交付，**不由本仓库提供**）、复杂时钟树（CRU）、电源芯片（PMIC）、安全执行环境（ATF/TrustZone）。这些硬件在 QEMU 里通通没有模拟。如果硬要把编译给 RK3568 的 U-Boot 塞给 QEMU 跑，第一行汇编就会直接崩溃。

---

### 1.2 科学分水岭：什么时候用 QEMU？什么时候必须上真机？

| 调试任务 / 内容 | 适合平台 | 为什么？ |
| :--- | :--- | :--- |
| **OFI 固件解析命令 (`cmd_ofi.c`)** | **QEMU (强力推荐)** | 纯软件逻辑！校验 Header、解析 TLV、内存拷贝、解压，在 QEMU 上单步断点调试秒级出结果。 |
| **A/B 分区仲裁算法与环境变量逻辑** | **QEMU (强力推荐)** | 纯状态机逻辑！测试 `boot_count` 递增、`boot_active` 切换、超时回滚，在 QEMU 上可以用脚本模拟 1000 次重启。 |
| **octopus_loader 与 OverlayFS 挂载** | **QEMU (强力推荐)** | 涉及 Squashfs 检验、loop 偏移挂载、tmpfs、`switch_root` 切根，QEMU 内核支持完美，比真机快 10 倍。 |
| **UBI / NAND 掉电与坏块逻辑** | **QEMU + nandsim (推荐)** | 内核 `nandsim` 模拟 NAND 并可注入坏块与位翻转，UBI 的 debugfs 支持模拟掉电，可以在没有真实 NAND 的情况下覆盖 SLOT 写入与 DATA 损坏恢复。 |
| **Octopus 核心主进程与 Web/IPC 接口** | **QEMU / 本机** | 完全是 POSIX 级应用程序，在模拟器里直接挂 GDB、抓 Core Dump，根本不需要板子。 |
| **DDR 内存时序、芯片时钟、PMIC 供电** | **必须上真机** | 涉及物理电气特性，QEMU 根本模拟不了。 |
| **实际 eMMC / SPI NAND 物理读写与烧录** | **必须上真机** | 芯片厂商的 USB 产线烧录协议（如 Rockchip Maskrom、Sigmastar ISP），工具与协议由厂商提供。 |
| **以太网交换芯片 (交换芯片 UIO / 寄存器读写)** | **必须上真机** | 交换 ASIC 物理寄存器访问，无通用 QEMU 仿真模型。 |

---

## 2. 业界最佳实践：最高效的三层测试金字塔

不要把所有测试都堆在真机上（太慢、难排查），也不要把所有希望寄托在 QEMU 上（无法替代物理硬件）。最聪明的测试架构是**分层递进**：

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │ Layer 3: 物理硬件靶场 (RK3568 / SSD20X 真机)                           │
 │  - 验证物理外设驱动 (UIO、PHY、以太网、硬件看门狗、物理按键)          │
 │  - 自动化串口台架: 批量进行掉电测试、A/B 分区抗死机容灾测试            │
 ├────────────────────────────────────────────────────────────────────────┤
 │ Layer 2: QEMU ARM64 / ARM32 全系统黑盒模拟靶场                        │
 │  - 机器模型: qemu-system-aarch64 -M virt (对应 RK3568)                 │
 │            qemu-system-arm -M virt (对应 SSD20X 等 32 位 ARMv7)        │
 │  - 跑通用 ARM64 U-Boot: 调试 cmd_ofi 与 A/B 双启动算法 (带 GDB 单步)   │
 │  - 跑 initramfs + OverlayFS + octopus_supervisor + octopus_app 全套业务闭环         │
 │  - 端口映射: 宿主机浏览器直接访问 QEMU 里的 Web 界面和 JSON-RPC       │
 ├────────────────────────────────────────────────────────────────────────┤
 │ Layer 1: PC 本机 (x86_64 Linux) 白盒极速单元测试                       │
 │  - 编译 x86 版本的 liboctopus_basics, liboctopus_json, octopus_cli, octopus_config     │
 │  - 秒级编译、AddressSanitizer 内存泄漏检测、GoogleTest 自动化测试      │
 │  - libFuzzer 模糊测试: OFI 头部解析、配置文本解析、JSON-RPC 请求解析    │
 └────────────────────────────────────────────────────────────────────────┘
```

- **为什么 Layer 2 要同时跑 ARM32**：SSD20X 是 32 位 ARMv7，`time_t`、`long`、指针宽度、结构体对齐、64 位原子操作都与 ARM64 不同。只在 ARM64 上测，这类问题要等到真机才暴露；
- **为什么解析器必须做模糊测试**：OFI 头部解析代码同时运行在 U-Boot 中，一旦存在越界读写，就可能被构造的固件包利用来绕过签名校验；配置解析器和 JSON-RPC 解析器直接面对网络输入。这三处在 Layer 1 用 libFuzzer + ASan 持续跑，成本最低、收益最高；
- **静态检查**：CI 中对 `octopus-core` 运行 `clang-tidy` 与 `cppcheck`，对 U-Boot / 内核补丁运行 `checkpatch.pl`。

---

## 3. 实战方案：手把手搭建 QEMU ARM64 模拟环境

这个方案的目标是：**在你的开发机（Linux 或 WSL2）上，不依赖任何硬件，就能跑通从 U-Boot 读取 OFI 镜像、引导 Linux、octopus_supervisor 拉起主程序、宿主机浏览器打开 Web 网页的全流程！**

### 3.1 核心思路：使用 QEMU 标准 `virt` 平台
我们不需要模拟 RK3568 板子，而是用标准的 ARM64 `virt` 架构：
1. 编译一个针对 `qemu_arm64_defconfig` 的通用 U-Boot；
2. 把我们的 `cmd_ofi.c` 编译进这个通用 U-Boot；
3. 编译一个通用的 ARM64 Linux 内核（支持 OverlayFS、SquashFS）；
4. 用虚拟磁盘（VirtIO 磁盘）模拟 eMMC 存储分区。

### 3.2 制作虚拟硬盘与分区
在宿主机上生成一个虚拟硬盘，模拟板载 eMMC：
```bash
# 1. 创建一个 512MB 的虚拟硬盘文件
qemu-img create -f raw emmc_disk.img 512M

# 2. 对其进行分区，GPT 分区名 (PARTLABEL) 与架构文档 4.3 的命名规范一致
parted -s emmc_disk.img mklabel gpt
parted -s emmc_disk.img mkpart env     1MiB   2MiB   # 分区 1: env (U-Boot 冗余环境变量)
parted -s emmc_disk.img mkpart factory 2MiB   4MiB   # 分区 2: factory (出厂数据)
parted -s emmc_disk.img mkpart slot_a  4MiB   132MiB # 分区 3: slot_a (128MB)
parted -s emmc_disk.img mkpart slot_b  132MiB 260MiB # 分区 4: slot_b (128MB)
parted -s emmc_disk.img mkpart data    260MiB 511MiB # 分区 5: data (用户持久化区)

# 3. 将打好的 Octopus .ofi 固件写入 slot_a（4MiB 偏移 = 第 8192 个 512 字节扇区）
#    这里是虚拟 eMMC 块设备，可以用 dd；真实裸 NAND 严禁 dd，见 reliability 议题 6
dd if=octopus_fw_a.ofi of=emmc_disk.img seek=8192 bs=512 conv=notrunc
```

### 3.3 一键启动 QEMU 运行 U-Boot
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
**关键参数解析：**
- `-bios u-boot.bin`：让 QEMU 上电直接跳入编译好的通用 ARM64 U-Boot；
- `-device virtio-blk-device`：虚拟磁盘挂载为 VirtIO 设备（U-Boot 和 Linux 里均识别为磁盘）；
- `hostfwd=tcp::8080-:80`：**神奇端口映射！** 将 QEMU 里面的 80 端口（Hiawatha Web 服务）映射到宿主机的 `http://localhost:8080`。你在电脑浏览器里就能直接点开设备的 Web 管理页面！

### 3.4 在 QEMU 中调试 U-Boot 的杀手级优势：GDB 硬件断点
在真机上调试 U-Boot，通常只能靠 `printf` 打印大法；一旦内存搞坏或指针越界，板子直接静默卡死，非常痛苦。

**在 QEMU 里调试 U-Boot 可以享受“上帝视角”：**
1. 启动 QEMU 时追加参数：`-s -S`（`-s` 开启 1234 端口 GDB 服务，`-S` 表示上电第一行汇编暂停等待）；
2. 另起一个终端，启动 GDB 跨平台调试器：
   ```bash
   aarch64-linux-gnu-gdb u-boot
   (gdb) target remote localhost:1234
   (gdb) b do_ofi_boot          # 在你的 OFI 解析函数入口直接下断点！
   (gdb) continue
   ```
3. U-Boot 会精确停在你的 `cmd_ofi.c` 代码行上！你可以随时 `step` 单步执行、打印变量 `p *header`、检查 TLV 长度。**这是真机无论如何也给不了的极致调试体验！**

---

## 4. 真机测试怎么才能“方便又高效”？（规避反复烧录的噩梦）

当代码在 QEMU 中验证通过，要移植到真正的 RK3568 / SSD20X 开发板时，**千万不要每次改动都把整个固件重新烧录进 eMMC/Flash**，那样效率极低且容易磨损存储芯片。

工业级真机高效调试的 **“三大绝招”**：

### 绝招 1：U-Boot 网络加载（TFTP + NFS 网络根文件系统）
这是嵌入式驱动与底层调试的黄金法则：
```text
  开发板 (RK3568)                      开发机 (PC Linux / Ubuntu)
  [ U-Boot ] ────────── TFTP 协议 ──────────> [ TFTP 服务器 ]
  只固化在板上一次                               下载最新的 Image 内核
        │
        ▼ 启动内核
  [ Linux 内核 ] ─────── NFS 协议 ──────────> [ NFS 服务器目录 ]
                                               挂载 PC 上的 rootfs
```
- **工作机制**：板子上的 eMMC 只需要刷一次 U-Boot，之后内核和文件系统全放在你的电脑目录里；
- **巨大收益**：
  你在电脑上编译完 `octopus_app`，无需任何烧录动作，在板子的串口终端敲回车就能立刻运行最新的程序！改代码、编译、测试只需 5 秒钟！

### 绝招 2：自动化串口交互与回归测试（Python `pexpect` / `pytest`）
不要人工盯着串口敲命令测试，编写简单的 Python 脚本自动对串口收发：
```python
# test_reboot_cycle.py (自动化容灾测试脚本)
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
    raise TimeoutError(f"超时未等到: {keyword}")

# 自动化测试 100 次重启与 A/B 切换
for i in range(100):
    print(f"=== 第 {i} 次自动化重启测试 ===")
    ser.write(b"reboot\n")
    wait_for("U-Boot 2020.10")
    wait_for("OFI Header Valid")
    wait_for("Starting octopus_supervisor...")
    wait_for("Octopus application ready")
    print("启动健康，自检通过！")
```

### 绝招 3：智能排插 / 继电器实现“自动化强行掉电测试”
对于高可靠双备份固件体系，最怕的是：**固件写入到一半突然拔电源，或者开机到一半突然断电，设备会不会变砖？**
- 买一个几十块钱的可编程 USB 继电器或 Wi-Fi 智能排插；
- 编写脚本：发命令升级固件 -> 随机 sleep 2~8 秒 -> 继电器断电 -> 重新上电 -> 检查 U-Boot 能否自动识别故障并回滚到备份分区；
- 跑一个晚上自动化断电 500 次，第二天设备依然能正常联网工作，才算通过这一轮；NFR 要求的是 10,000 次。
- **断电点要覆盖关键窗口**：纯随机 sleep 很难恰好断在“ENV 写入中”“rename 与目录 fsync 之间”这类毫秒级窗口。让被测固件在关键步骤前后通过串口或 GPIO 打点，继电器脚本按打点触发断电，才能有针对性地覆盖；
- **每轮之后检查的不只是“能开机”**：还要检查配置是否完整（校验尾正确）、是否意外回退到了出厂配置、DATA 是否被重建、bootcount 与槽位状态是否符合预期；
- **长期拷机与 Flash 磨损**：连续运行数周，跟踪内存趋势（`show memory modules`）和 Flash 写入量（eMMC 的 `EXT_CSD` 寿命字段、UBI 的擦除计数），按写入速率推算产品寿命内是否会写穿。

> **注意**：绝招 1 的 NFS 根文件系统绕过了 `octopus_loader`、OverlayFS 与 A/B 流程，只适合调试应用逻辑。启动链与掉电相关的测试必须在真实存储上用正式固件跑。

---

## 5. 总结：测试平台搭建落地矩阵

| 开发阶段 | 推荐测试环境 | 目标与产出 |
| :--- | :--- | :--- |
| **阶段 1：核心业务库抽离**<br>(liboctopus_basics/json/octopus_cli) | **PC 本机 (x86_64)** | 在 Ubuntu/WSL2 上直接跑 gcc/cmake，秒级编译，跑通所有基础数据结构的单元测试。 |
| **阶段 2：OFI 引导与 A/B 仲裁逻辑**<br>(cmd_ofi.c / octopus_loader / octopus_supervisor) | **QEMU ARM64 (virt)** | 用虚拟硬盘跑通用 U-Boot，用 GDB 单步断点调通 OFI 提取、OverlayFS 挂载和 Web 响应。 |
| **阶段 3：RK3568 底层外设适配**<br>(DDR / PMIC / 网卡 / eMMC驱动) | **RK3568 真机 (TFTP/NFS)** | 调通网络与设备树，免去频繁烧写，实现快速交叉编译验证。 |
| **阶段 4：系统整体验收与可靠性拷机** | **RK3568 物理硬件 (自动化台架)** | 真实物理 eMMC 分区刷机、继电器异常掉电测试、长连接高并发压力测试。 |

这样一套**“PC 极速单元测试 + QEMU 架构仿真 + 真机网络联调 + 自动化掉电拷机”**的组合拳，不仅能让你摆脱频繁烧录变砖的痛苦，而且能大幅缩短项目研发周期！
