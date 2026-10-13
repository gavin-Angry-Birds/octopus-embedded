# Octopus Platform Technology Stack and Programming Language Selection Guidelines

**English** | [简体中文](./00-language-cpp-guidelines.md)

> **Note**: This is an English translation. The Simplified Chinese version is authoritative; if the two differ, the Chinese version prevails.

> **Purpose**: Addresses a core question raised by embedded developers and system architects — "Should the entire Octopus architecture be rewritten in pure C?" — with an in-depth technical analysis across **system safety, development efficiency, memory management, cross-language ecosystem, and long-term maintainability**. It draws clear language boundaries for each system layer and establishes **disciplined usage guidelines for industrial-grade embedded C++**.

---

## 1. Core Conclusion: Why Not Use Pure C Everywhere?

**Clear conclusion: Writing the entire Octopus platform in pure C is strongly discouraged.**

In embedded Linux system software design, "pure C fundamentalism" may look closer to the hardware, but when faced with "a highly reliable management plane, dynamic configuration reflection, and multi-threaded concurrency safety", it quickly turns into a maintenance disaster.

The golden rule for industrial-grade software should be:  
**"The closer to the hardware, the purer the C; lightweight C++ for the platform core; pure C at every external boundary; free choice of languages for upper-layer applications."**

---

## 2. Deep Dive: Two Major Disasters If the Core Platform Reverts to Pure C

One of the key technical moats that allows classic industrial-grade commercial network operating systems to support complex management planes reliably over the long term is the highly mature **C++ template metaprogramming and reflection system** in their user-space main program. Forcing a rewrite of the core platform in pure C would lead to two unbearable pain points:

### 2.1 Pain Point 1: Losing "Compile-Time Template Reflection" Turns the Management Plane into Code Hell

In the Octopus architecture, the system exposes a unified management plane (the octopus_cli command line, the Web UI, JSON-RPC / IPC, and octopus_config plain-text configuration save and restore).

#### With lightweight C++ (current state):
The developer only needs to define a plain business C struct and then write **a single line of template registration code**:
```cpp
// With just this one line, C++ templates use compile-time deduction to automatically handle:
// 1. Mounting into the JSON-RPC method tree with automatic parameter validation;
// 2. Automatic deserialization of Web front-end requests;
// 3. Exposure as a local IPC interface over a Unix Domain Socket;
// 4. octopus_config running-config text serialization for saving, and line-by-line replay parsing at boot.
StructReadWrite<my_device_cfg_t>(&ns_system, "config", get_cfg, set_cfg);
```
- **Mechanism**: The compiler automatically generates specialized read/write code for each data type at compile time — **zero runtime overhead and full type safety**.

#### If forced back to pure C:
C lacks generics, templates, and operator overloading. To implement the same full feature set, developers must:
1. **Hand-code cJSON parsing**: For every field of hundreds or thousands of structs, write `cJSON_GetObjectItem` calls, type checks, bounds checks, and error handling line by line;
2. **Hand-write serialization and deserialization**: Every configuration item needs its own string formatting for saving and line-by-line regex matching for replay;
3. **Or fall into "macro hell" (X-Macros / nested macros)**: Assemble complex function definitions from screens full of `##` and `#` macros, making compiler errors completely unreadable, breaking IDE symbol navigation and syntax highlighting, and making bugs extremely hard to track down.

> **Cost assessment**: Implementing the same integrated management plane in pure C would **inflate the code size by 3–5×**, and every field addition, removal, or change would require manual synchronized edits in five or six places, making typos and type mismatches that lead to segmentation faults extremely likely.

---

### 2.2 Pain Point 2: Lock and Resource Leaks Under High Concurrency (No RAII)

`octopus_app` is a long-running, resident multi-threaded main process (a Reactor main loop, 2–4 worker threads, a watchdog inspection thread, plus module threads for route event listening, hardware status polling, and so on). It involves a huge number of mutexes, file descriptors (FDs), and temporary dynamic buffers.

#### With lightweight C++ (current state):
Make full use of **RAII (Resource Acquisition Is Initialization)**:
```cpp
void handle_port_event(int port_no) {
    // Locks automatically on scope entry; no matter where we later return or break, leaving the scope always unlocks!
    std::lock_guard<std::mutex> lock(g_port_mutex);

    if (!is_valid(port_no)) {
        return; // Safe exit, the lock is released automatically
    }
    // Business logic...
}
```

#### If forced back to pure C:
C has no destructors, so developers must rely on rigorous manual review and cascading `goto` jumps:
```c
int handle_port_event(int port_no) {
    pthread_mutex_lock(&g_port_mutex);

    if (!is_valid(port_no)) {
        pthread_mutex_unlock(&g_port_mutex); // Very easy to forget to unlock!
        return -1;
    }

    int fd = open("/dev/xxx", O_RDWR);
    if (fd < 0) {
        goto out_unlock; // Jumping to the wrong label causes a deadlock or handle leak
    }

    // More branches...
out_unlock:
    pthread_mutex_unlock(&g_port_mutex);
    return ret;
}
```
In industrial and automotive equipment that must run continuously without failure for months or even years, **missing a single unlock in multi-branch pure C logic causes a deadlock that hangs the whole device and effectively bricks it; missing a single `close(fd)` exhausts file handles**.

---

## 3. Octopus Global Technology Stack and Language Selection Matrix

Following the principle of **"C++ for cohesion, pure C for boundaries and low levels"**, each Octopus system layer has clearly defined language responsibilities:

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │ 4. Application & Extension Layer:                                      │
 │    - Web front end: HTML5 + modern JavaScript (Vue / native light JS)  │
 │    - Standalone AI vision process (ai_vision_daemon): C++ (C++14/17,   │
 │      optionally links vendor NPU/RGA libraries)                        │
 │    - Customer standalone business processes: any language             │
 │      (Python / Go / Rust / pure C)                                     │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 3. Core Platform Main Process (octopus_app):                           │
 │    - Business module pool (In-Tree Modules): lightweight C++ (C++11/14)│
 │    - Management plane engine (octopus_cli / JSON-RPC / octopus_config):│
 │      lightweight C++ (C++11/14)                                        │
 │    - Core common library (liboctopus_basics): lightweight C++          │
 │      (OSAL / reflection / logging)                                     │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 2. External SDK & Middleware Interfaces (Boundary Layer):              │
 │    - liboctopus_client SDK: pure C (C99, zero third-party deps,        │
 │      extern "C")                                                       │
 │    - Service supervision (octopus_supervisor / octopus_loader):        │
 │      pure C (C99)                                                      │
 ├────────────────────────────────────────────────────────────────────────┤
 │ 1. Hardware Drivers, Boot & OS Layer (OS & Hardware Layer):            │
 │    - Board HAL (liboctopus_board): pure C (C99)                        │
 │    - Switch chip drivers (switch chip SDK / board switch adaptation    │
 │      layer): pure C (C99)                                              │
 │    - Bootloader & Linux kernel (U-Boot / cmd_ofi.c / Kernel):          │
 │      pure C (C99)                                                      │
 └────────────────────────────────────────────────────────────────────────┘
```

### Detailed Selection Rationale

| Layer / Module | Language | Core Considerations and Technical Rationale |
| :--- | :--- | :--- |
| **Bootloader & boot** (`cmd_ofi.c` / U-Boot patches) | **Pure C (C99)** | In the U-Boot bare-metal environment and memory-constrained boot stages, strictly standard C is mandatory. |
| **Linux kernel & device drivers** (kernel modules, DTS device trees) | **Pure C (C89/C99)** | The official standard language of the Linux kernel; directly manipulates physical hardware registers. |
| **Board HAL** (`liboctopus_board`) | **Pure C (C99)** | Declares a minimal C header; responsible for reading the MAC and SN, driving LEDs, and handling the factory reset button, providing the most stable C ABI for all applications. |
| **Chip hardware pass-through library** (switch chip SDK / board switch adaptation layer) | **Pure C (C99)** | User-space C library supplied by the switch chip vendor; interacts with physical registers via UIO and kernel `mmap`. |
| **External SDK** (`liboctopus_client`) | **Pure C (C99)** | **A critical architectural boundary**: the delivered headers and shared libraries are implemented entirely in pure C (zero C++ dependency, zero STL dependency), so external processes written in Python/Go/Rust/pure C can bind to them seamlessly and directly. |
| **Core support library** (`liboctopus_basics`) | **Lightweight C++ (C++11/14)** | Provides OSAL thread/lock wrappers, `StructReadWrite` template reflection, and the leveled Trace logging system. |
| **Core management plane middleware** (octopus_cli / JSON / octopus_config) | **Lightweight C++ (C++11/14)** | Hosts global `JSON_RPC_ROOT` method tree routing, plain-text configuration serialization, and replay logic. |
| **Core main process entry & modules** (`octopus_app`) | **Lightweight C++ (C++11/14)** | `main.cxx` lifecycle broadcasting and business component scheduling, balancing high performance with code reuse. |
| **Initramfs root filesystem bootstrap** (`octopus_loader`) | **Pure C (C99)** | PID 1 during the initramfs stage; mounts OverlayFS on top of the SquashFS layer, then after `switch_root` execs `octopus_supervisor` (`pivot_root` cannot be used from initramfs). Statically linked, with no dependency on the C++ runtime. |
| **PID 1 process supervisor** (`octopus_supervisor`) | **Pure C (C99)** | Consistent with section 5.2 of the architecture document: < 2000 lines of pure C, single-threaded, and no dynamic memory allocation after startup. If PID 1 crashes the kernel panics, so neither the C++ runtime nor the STL is introduced. |
| **Standalone AI vision daemon** (`ai_vision_daemon`) | **C++ (C++14/17)** | May optionally link OpenCV, board-level RGA, and NPU user-space SDKs supplied separately by chip vendors under license (not included in this repository); such SDKs mostly expose C++ interfaces, hence C++ for this process. |
| **Web front-end management UI** | **HTML5 + native JS / Vue** | Runs in the client browser and submits JSON-RPC via HTTP POST; the back end has no PHP/Python dynamic environment at all. |
| **Host build tools** (`ocli_tool`, `ofi_pack`) | **C++ / Python / Shell** | Run on the build machine (PC/server) for offline syntax compilation and OFI image packaging. |

---

## 4. Disciplined Usage Guidelines for Embedded C++

When using C++ in the industrial and embedded domain, **you must never indulge in modern C++ features as freely as on x86 servers**. Instead, treat it as **"industrial-grade C with strong type safety, compile-time templates, and RAII"** (C with Classes and Templates).

When compiling and writing code for `octopus-core`, the following **five guidelines** must be strictly observed.

**Language standard baseline**: `octopus-core` uses **C++14** as its baseline. C++17 features (`std::optional`, `if constexpr`, etc.) are allowed only after the cross toolchains for all target SoCs are confirmed to be GCC ≥ 8, and the switch must be made uniformly in the top-level CMake; individual modules may not raise the standard on their own.

**Scope**: The following guidelines govern `octopus-core` (`octopus_app` and `liboctopus_*`). Standalone processes such as `ai_vision_daemon` that link third-party C++ components relying on exceptions and RTTI (such as OpenCV or vendor NPU user-space libraries) may leave `-fno-exceptions` / `-fno-rtti` disabled, but these libraries must not be linked into `octopus_app`. Vendor libraries must be obtained by the device maker under the vendor's license and **must not be committed to this repository**.

### Guideline 1: Mandatory Disabling of RTTI and Heavyweight Exceptions
- **Enforced via CMake compile options**:
  ```cmake
  add_compile_options(-fno-rtti -fno-exceptions)
  ```
- **Technical rationale**:
  - **RTTI (Run-Time Type Information)** generates a `type_info` data structure for every class, enlarging the read-only data segment and adding dynamic overhead;
  - **Exceptions** introduce large exception-handling tables (`.eh_frame`), break deterministic real-time response times, and add the uncertain risks of stack unwinding.
- **Error handling convention**:
  All modules uniformly propagate errors using strongly typed status code enums (such as `octopus_rc_t`) or result structs (`std::optional` once C++17 is enabled); throwing exceptions is forbidden.
- **Note on memory allocation failure under `-fno-exceptions`**: When STL containers or `new` fail to allocate, they cannot throw `std::bad_alloc`, and the process aborts immediately. This is consistent with the Octopus fault model (crash dump → supervisor restart), but it means "out of memory" is not a recoverable error. Foreseeable large allocations (such as receiving firmware chunks or importing large configurations) should use `new (std::nothrow)` or pre-allocated buffers with explicit checks.

### Guideline 2: Bloated Third-Party C++ Libraries Are Strictly Forbidden
- **Heavyweight frameworks such as Boost are strictly forbidden**: Flash/RAM resources on embedded devices are extremely precious; pulling in heavy libraries can bloat binaries by tens of megabytes and create complex, tangled dependencies;
- **Platform self-reliance**: Basic threads, locks, and message queues are implemented by Octopus's in-house `liboctopus_basics` (OSAL) directly on top of Linux POSIX interfaces; lightweight containers use only basic STL (such as `std::vector`, `std::string`, `std::map`).

### Guideline 3: Strict Control of Dynamic Heap Allocation (Eliminate Memory Fragmentation)
- **Zero dynamic allocation on the data plane**: On critical real-time data paths (such as network packet TX/RX, motor angle control loops, and sensor sampling timers), **calling `new` / `malloc` in runtime loops is strictly forbidden**;
- **Prefer pre-allocation**: Use static arrays, fixed-size ring buffers, or fixed-block memory pools to eliminate heap fragmentation caused by long-term operation.

### Guideline 4: Prefer Flat Structures; Complex Multiple-Inheritance Trees Are Strictly Forbidden
- Avoid deep class inheritance and complex virtual base classes. Overly deep vtable hierarchies add indirect function-pointer jump overhead and hurt CPU instruction cache (I-Cache) locality;
- Prefer **object composition** and **struct mapping (POD structs)**.

### Guideline 5: Well-Defined Boundaries for RAII Usage
- All mutexes must be wrapped with `std::lock_guard` or `std::unique_lock`; any manual `pthread_mutex_unlock` is forbidden;
- All dynamically acquired temporary file descriptors (FDs) and sockets must be wrapped in lifetime guard classes, guaranteeing they are `close()`d when leaving scope.

---

## 5. The Golden Architectural Divide: "C++ for Cohesion, Pure C for Boundaries"

The most elegant aspect of the Octopus architecture is its boundary handling — **internal efficiency and external compatibility coexist perfectly**:

```text
  ┌────────────────────────────────────────────────────────┐
  │                 [ Inside octopus-core ]                │
  │  - Uses modern lightweight C++ (C++11/14)              │
  │  - Enjoys drastic code reduction from StructReadWrite  │
  │    template-based automatic reflection                 │
  │  - Enjoys lock safety and zero leaks from RAII         │
  │  - Modules call each other via in-memory pointers,     │
  │    for maximum efficiency                              │
  └───────────────────────────┬────────────────────────────┘
                              │
                              │ Crossing a process boundary or board boundary
                              ▼
  ══════════════════════════════════════════════════════════
           [ Golden Divide: strictly extern "C" ]
      (Headers and shared libs: liboctopus_client.h, board_hal.h)
  ══════════════════════════════════════════════════════════
                              │
                              │ Pure C ABI, no C++ name mangling whatsoever
                              ▼
  ┌────────────────────────────────────────────────────────┐
  │        [ External Ecosystem & Customer Business ]      │
  │  - Customers call into the platform directly from      │
  │    Python via ctypes / cffi                            │
  │  - Customers write standalone Go / Rust / pure C       │
  │    processes that connect directly to the IPC socket   │
  │  - Board hardware drivers need not care about          │
  │    user-space logic; they only conform to the C API    │
  └────────────────────────────────────────────────────────┘
```

1. **Internally (In-Tree)**:
   Internal developers gain a tenfold boost in development and maintenance efficiency through C++ templates. Simply defining a struct automatically yields the complete Web/CLI/octopus_config management plane, reducing code size by more than 70%;
2. **Externally (Out-of-Tree & HAL)**:
   The SDK (`liboctopus_client`) and board interface (`liboctopus_board`) exposed to customers are **the purest C99 interfaces**. External developers do not need to install any C++ runtime and are unaffected by C++ compiler versions or standard library ABI changes; they can integrate with the Octopus platform in minutes using any modern language.

---

## 6. Summary

| Dimension | Pure C Approach | Octopus's Chosen "Lightweight C++ Core + Pure C Boundary" Approach |
| :--- | :--- | :--- |
| **Code size and maintenance cost** | Extremely high (tens of thousands of lines of hand-written cJSON parsing and serialization boilerplate) | **Extremely low (one line of C++ template auto-generates the full Web/CLI/octopus_config loop)** |
| **Long-term high-concurrency safety** | Fragile (relies on manual checks of goto-based unlocking; missed unlocks easily cause deadlocks) | **Extremely high (RAII automatic destruction handles lock/unlock, eliminating deadlocks at the language level)** |
| **Firmware and memory footprint** | Minimal | **Minimal (with RTTI and exceptions disabled, size increase vs. pure C is < 5%)** |
| **Cross-language SDK ecosystem** | Good | **Excellent (the external boundary consists entirely of pure C headers, with zero barriers to cross-language calls)** |
| **Modern AI / vision integration** | Difficult (mainstream algorithm / board-level acceleration SDKs are almost all wrapped in C++) | **Feasible (standalone processes can interface with the C++ APIs of OpenCV, RGA, and vendor NPU user-space SDKs)** |

Therefore, **"keeping lightweight C++ for the platform core and template reflection, while using pure C to solidify low-level drivers and the external SDK"** is the single best choice that balances peak performance, high reliability, development efficiency, and an open ecosystem.
