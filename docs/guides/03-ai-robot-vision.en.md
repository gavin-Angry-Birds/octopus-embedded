# Guide to Deploying the Octopus Architecture in Embodied AI / Robotics and Edge AI Vision Scenarios

**English** | [简体中文](./03-ai-robot-vision.md)

> **Note**: This is an English translation. The Simplified Chinese version is authoritative; if the two differ, the Chinese version prevails.

> **Purpose**: Discusses the feasibility and engineering trade-offs of extending the Octopus **industrial gateway platform** to smart hardware, embodied robots, and edge AI vision gateways. Covers how to integrate the NPU, the recognition and reporting pipeline, interface evolution, and a rough compute budget on the RK3568.
>
> **Scope and Phase (read first)**
> - This document is a **product-oriented / long-term extension** discussion draft and is **not** on the platform's main path for phases 1–3 of the [ROADMAP](../../ROADMAP.md); phase 4 lists `ai_vision_daemon` only as an optional deliverable.
> - WebRTC / cloud streaming, millisecond-level motor closed loops, functional safety ownership, and similar topics remain listed in [Chapter 13 Open Issues](../architecture/00-platform-design.md#13-尚未覆盖的议题open-issues) of the architecture specification and **must not** be read as implemented capabilities.
> - The platform's "anti-brick" claims are constrained by architecture **§13.1 Phase 3 Blockers**; until the Blockers are resolved, this document's description of A/B OTA represents design direction only, not a frozen implementation specification.
> - NPU device nodes, vendor inference SDKs, board-level multimedia libraries, and the like mentioned in this document are all external dependencies **obtained and integrated separately by the device maker under the vendor's license**; **this repository does not include their source code or prebuilt packages**.

---

## 1. Scenario Positioning and Feasibility: Can This Architecture Power Robots?

**Core conclusion (provided the platform's Phase 3 Blockers are resolved and the implementation follows the specification):** Yes, it can. Compared with lab setups cobbled together on stock desktop Linux or Python scripts, Octopus has concrete architectural mechanisms for remote OTA rollback, root filesystem protection under power loss, process fault-domain isolation, and atomic configuration persistence. However, this is **not** a promise that it is "already validated in mass production and overwhelmingly superior"; real-time motion control and functional safety still require separate design (see Chapter 13).

### 1.1 The Critical Weak Points of Commercial Robots and Smart Hardware
Many developers build robots with ROS or Python on standard Ubuntu/Debian. The demos look perfect in the lab, but as soon as the product moves to commercial mass production, fatal pain points surface:
1. **Very high risk of bricking during remote OTA**: Robots run in users' homes or outdoors; an upgrade interrupted by power or network loss bricks the device so it cannot boot;
2. **Forced power-off / battery depletion corrupts the filesystem**: Robots frequently lose power when the battery runs out or the main power is cut abruptly, and a traditional read-write EXT4 filesystem is easily corrupted;
3. **An AI process crash takes down the whole machine**: NPU model inference and camera video streams are prone to sporadic out-of-bounds memory access or deadlocks, leaving the entire system hung;
4. **Calibration parameters are easily lost or corrupted**: Robot arm zero offsets, servo calibration angles, PID control parameters, and Wi-Fi credentials lack reliable atomic persistence management.

### 1.2 Four Decisive Advantages of the Octopus Architecture
- **OFI firmware + A/B dual partitions + boot counter**: Delivers true "anti-brick OTA"; if new firmware misbehaves, the device automatically rolls back to the old system;
- **Read-only SquashFS + in-memory OverlayFS**: Even if the robot's battery is yanked 1000 times, the underlying filesystem always stays read-only protected and boots intact;
- **Second-level crash self-healing by PID 1 octopus_supervisor**: When upper-layer vision or AI algorithms occasionally segfault, the platform "cerebellum" keeps the machine under stable control, and octopus_supervisor restarts the AI process within 1 second;
- **octopus_config text-based parameter persistence engine**: Servo offsets, user settings, and network parameters take effect on commit and are merged in the background and persisted atomically (usually within a few seconds), then replayed and restored automatically at boot. A power loss loses at most the last few seconds of not-yet-persisted changes; configuration already persisted will not be corrupted.

---

## 2. Core Architecture Pattern: Separating the Stable from the Dynamic, with Split Roles

In a robot system, the principle of **"separating the cerebellum (stable platform) from the cerebrum (AI vision)"** must be strictly enforced:

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │ Traditional fragile architecture (anti-pattern):                       │
 │  Camera capture, OpenCV, NPU models, motor drivers, and Web            │
 │  interaction all crammed into a single process                         │
 │  ==> Occasional NPU memory leak or inference timeout -> whole process  │
 │      crashes -> robot topples over and freezes                         │
 └────────────────────────────────────────────────────────────────────────┘
                                    VS
 ┌────────────────────────────────────────────────────────────────────────┐
 │ Octopus robust robot architecture (elegant pattern):                   │
 │  - Cerebrum (standalone AI/NPU process): focuses on heavy model        │
 │    inference; if it dies, octopus_supervisor restarts it in seconds    │
 │  - Cerebellum (Octopus main platform): focuses on chassis motors,      │
 │    power management, safety watchdog, OTA upgrades                     │
 └────────────────────────────────────────────────────────────────────────┘
```

1. **Octopus core main process (`/usr/bin/octopus_app`) — acts as the "steward"**:
   - Handles system network management, OTA upgrade arbitration, user parameter persistence (octopus_config), and battery level monitoring and alarms;
   - Sends targets (pose, velocity, motion sequences) to motion control but does not run the closed loop itself;
2. **Motion control (`motion_ctrl` real-time process or external MCU) — the real "cerebellum"**:
   - Handles millisecond-level closed loops such as motor PWM, servo inverse kinematics, and fall/collision protection;
   - Must not live inside `octopus_app`: the `octopus_app` main loop must handle RPC, configuration transactions, and logging; section 3.2 of the architecture document explicitly forbids heavy, time-consuming operations in it, and its scheduling latency cannot guarantee a 1ms-level closed loop. Putting it in the same process would also mean a single crash in the configuration module could leave the motors out of control;
   - A Linux-side implementation requires `SCHED_FIFO` + `mlockall()` + CPU isolation; for loop periods ≤ 1ms, a PREEMPT_RT kernel or handing the loop directly to an MCU is recommended;
   - **Safe state**: If the control process stops refreshing targets for longer than a configured timeout (e.g. 50ms), the motor driver or MCU must enter a safe state on its own (stop, cut torque, or hold), and must not rely on the Linux side to "recover after a restart";
3. **External standalone AI daemon (`/usr/bin/ai_vision_daemon`) — acts as the "eyes and cerebrum"**:
   - Runs as a standalone process with exclusive ownership of the NPU and camera drivers;
   - Handles video capture, object recognition (people/animals), and model inference;
   - Communicates with the platform cerebellum through the ultra-lightweight local Unix socket `/var/run/json_ipc.socket`.

---

## 3. Typical Requirement: Recognizing Animals/People with a Camera and Sending Results to the Cloud

This is a standard closed loop of **"edge perception (Edge AI) + local physical response + IoT cloud connectivity"**.

### 3.1 End-to-End Data Flow

```text
 [ 1. Physical capture ] Camera (MIPI-CSI / USB) captures raw video stream (NV12/YUV)
        │
        ▼ Hardware zero-copy (DMA-BUF / RGA hardware image crop and scale)
 [ 2. Edge inference ] RK3568 NPU runs a lightweight model (e.g. YOLOv8n-nano)
        │       Outputs recognition result: {"label": "cat", "confidence": 0.94, "bbox": [100, 120, 200, 300]}
        │
        ▼ Inter-process communication (Unix Domain Socket: /var/run/json_ipc.socket)
 [ 3. Local response ] Notify the Octopus main platform to react physically
        │       - Cerebellum control: the duck turns its head toward the cat and plays a sound
        │       - Status query: get current battery level, GPS/Wi-Fi signal strength
        │
        ▼ IoT protocol (MQTT / HTTPS)
 [ 4. Upload results ] Package event metadata (+ snapshot thumbnail) and push it to the cloud server
```

### 3.2 Multi-Process Collaboration Topology

```text
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │ External physical world and hardware                                                   │
 │   - Camera hardware (MIPI-CSI / USB)  - RK3568 NPU hardware   - Cloud IoT platform     │
 │                                                                 (MQTT/S3)              │
 └─────────────┬───────────────────────────────┬─────────────────────────────▲────────────┘
               │ Raw video frames              │ Hardware inference calls    │ JSON metadata/images
               ▼ /dev/video*                   ▼ NPU device node             │
 ┌─────────────────────────────────────────────────────────────┐             │
 │ [Standalone vision & AI process: /usr/bin/ai_vision_daemon] │             │
 │  1. V4L2 / Rockchip MPP capture + RGA hardware scaling      │             │
 │  2. Object detection via vendor NPU user-space runtime      │             │
 │     (recognize cats/dogs/people)                            │             │
 │  3. Target filtering (dedup, confidence threshold,          │             │
 │     ROI region detection)                                   │             │
 │  4. Trigger snapshot: encode a 640x480 JPEG thumbnail       │             │
 └──────────────┬──────────────────────────────────────────────┘             │
                │                                                            │
                │ 1. Result notification & platform control                  │ 2. Results uploaded
                │    (control commands and metadata)                         │    directly to cloud
                │    /var/run/json_ipc.socket (local Unix socket)            │    (or relayed by platform)
                ▼                                                            │
 ┌─────────────────────────────────────────────────────────────┐             │
 │ [Octopus core main process: /usr/bin/octopus_app]           │             │
 │ (device steward and cerebellum)                             │             │
 │  1. Local physical response:                                │             │
 │     - Send a "face toward the animal" target to motion_ctrl │             │
 │     - Play an audio cue / turn on the fill light            │             │
 │  2. Unified configuration management (octopus_config save): │             │
 │     - Cloud-pushed "recognition sensitivity" and "upload    │             │
 │       images or not" persisted across power loss            │             │
 │  3. System health guarding: monitor battery, network        │             │
 │     connectivity, feed the watchdog                         │             │
 └──────────────────────────────────────────────┬──────────────┘             │
                                                │ (Optional: upload via the   │
                                                │  built-in MQTT module)      │
                                                └────────────────────────────┘
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │ [PID 1: octopus_supervisor]: launches ai_vision_daemon and octopus_app uniformly,      │
 │ monitors for crashes, and self-heals within seconds                                    │
 └────────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Pitfalls to Avoid When Uploading to the Cloud
Sending results to the cloud usually involves two kinds of data: **"recognition text metadata"** and **"on-site snapshot images"**.

#### (1) Text Metadata over MQTT (Ultra-Lightweight, Low Latency)
When the NPU detects a target in the frame, publish a structured JSON message to the `/duck/ai/events` topic:
```json
{
  "device_id": "duck_88321",
  "timestamp": 1726000000,
  "event_type": "object_detected",
  "data": {
    "target": "cat",
    "confidence": 0.94,
    "box": [100, 120, 200, 300],
    "image_attached": true
  },
  "device_status": {
    "battery": 88,
    "wifi_rssi": -58
  }
}
```

#### (2) Uploading Snapshot Images (Never Stuff Base64 into MQTT!)
- **Wrong approach**: Converting the image to a Base64 string and stuffing it into the MQTT JSON causes severe memory bloat and easily stalls on weak networks;
- **Industrial best practice (direct upload to S3/OSS via presigned URL)**:
  1. After taking the snapshot, the AI process encodes it as JPEG and stores it in memory at `/tmp/snapshot.jpg`;
  2. After receiving the MQTT alert, the cloud issues a short-lived presigned upload URL (Presigned URL, e.g. Alibaba Cloud OSS / Tencent Cloud COS / AWS S3);
  3. The robot uses a lightweight HTTP client (such as `libcurl`) to PUT the binary stream directly to cloud object storage;
  4. The mobile app pulls the image directly from object storage within seconds, saving both robot CPU and bandwidth.

#### (3) Design for Weak and Lost Connectivity (Offline Local Cache)
- When the robot wanders into a dead spot with no Wi-Fi signal (such as under a bed or in a basement):
- After recognizing a target, `ai_vision_daemon` writes the event and image to `/data/spool/ai/`. This directory must have a capacity cap (e.g. 20MB) and an entry-count cap, discarding the oldest data when full, and the write rate of JPEG thumbnails must be throttled; otherwise, during a prolonged outage the camera will fill the DATA partition and wear out the Flash;
- Once connectivity is restored, the Octopus network module broadcasts a `network.online` event, triggering automatic **backfill upload of offline data**.

#### (4) Privacy and Compliance
Recognizing "people" and uploading snapshots constitutes processing personal information and must comply with regulations such as China's Personal Information Protection Law (PIPL) and the GDPR:
- By default only event metadata is uploaded; snapshot upload must be explicitly enabled by the user;
- Support on-device face blurring before upload, so original images never leave the device;
- Both the local cache and cloud storage must have clearly defined retention periods, and the local cache must be cleared on factory reset;
- The device must have a clearly visible "camera active" indicator (such as an LED); capture must never happen without the user's knowledge.

### 3.4 Delta OTA Upgrades for AI Models (Leveraging OFI Segmentation)
Vision models iterate very quickly (today a given version of the weights package, next week a `v2` trained for specific animals; the exact file extension depends on the export format of the vendor toolchain).
Using Octopus's **OFI segmentation mechanism**, the model files are packaged independently as **`model.squashfs`**, with two update methods (see section 8.5 of the architecture document):
- **Update with the slot**: The OTA downloads only the changed model segment; when writing the inactive slot, the remaining segments are copied from the current slot, taking effect after reboot. If a corrupted model prevents the system from passing the health gate, U-Boot's bootcount mechanism rolls back to the old slot (together with the old model);
- **Hot update**: The model package is signed separately and written to an inactive directory under `/data/models/`, and only `ai_vision_daemon` is restarted. This path has nothing to do with U-Boot; if the new model fails to load, the upgrade service switches the pointer back to the old directory.
- Note: U-Boot is only responsible for booting the kernel and **does not** load or check the model; whether the model can run inference correctly can only be judged by user-space health checks.

---

## 4. Interface Evolution: Meeting Modern AI and Streaming Interaction Needs

The four traditional management interfaces (WEB, `octopus_cli`, JSON-RPC, SNMP) are designed for Ethernet device configuration. In modern AI robot scenarios, they require deliberate **upgrading and restructuring**:

### 4.1 Reviewing and Trimming Traditional Interfaces
- **SNMP**: **Completely useless on consumer/commercial robots — remove it outright** (retain it only on specialized industrial inspection models);
- **octopus_cli**: Reduced to an engineering console for **internal R&D debugging, production-line motor calibration, and low-level fault diagnosis**;
- **JSON-RPC**: Remains the backbone for **basic configuration and status queries (Request-Response)**, but cannot handle streaming or high-bandwidth scenarios.

### 4.2 Four New Interfaces Required for Modern AI

```text
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │                   Complete interaction interface system for modern AI robots           │
 ├──────────────────────────────────────────┬─────────────────────────────────────────────┤
 │ Retained interfaces                      │ New streaming interfaces for AI / robots    │
 │ (stable config and platform control)     │                                             │
 ├──────────────────────────────────────────┼─────────────────────────────────────────────┤
 │  1. octopus_cli (serial/SSH)             │  ★ 5. WebSocket (bidirectional streaming)   │
 │     - Production-line calibration,       │     - Streaming LLM token output, real-time │
 │       low-level servo tuning, viewing    │       telemetry pushed at ms granularity    │
 │       kernel logs                        │                                             │
 │                                          │                                             │
 │  2. JSON-RPC (HTTP / local Unix socket)  │  ★ 6. WebRTC (ultra-low-latency A/V)        │
 │     - Parameter config (volume, Wi-Fi,   │     - Robot duck first-person live video    │
 │       battery queries)                   │       (FPV < 200ms)                         │
 │     - Control channel for the local AI   │     - Remote two-way voice intercom between │
 │       process to drive cerebellum motors │       phone and duck                        │
 │                                          │                                             │
 │  3. Web H5 (embedded lightweight         │  ★ 7. MQTT (cloud IoT and cloud-brain       │
 │     front end)                           │       commands)                             │
 │     - Provisioning page after scanning a │     - Connects to the cloud IoT platform,   │
 │       QR code to join the hotspot        │       receives remote wake-up               │
 │                                          │                                             │
 │  4. [Downgraded/trimmed] SNMP            │  ★ 8. DMA-BUF / shared memory (zero-copy    │
 │     - Removed on consumer models; only   │       local channel)                        │
 │       enabled on industrial inspection   │     - Camera frames go straight to the NPU, │
 │       models                             │       avoiding memory copy overhead         │
 └──────────────────────────────────────────┴─────────────────────────────────────────────┘
```

#### Interface 1: WebSocket (Solving Streaming "Typewriter" Interaction with Large Models)
- **Pain point**: Traditional JSON-RPC must wait for the entire response to be generated before returning it in one go; when conversing with a large model (e.g. integrating DeepSeek), the robot sits frozen for several seconds;
- **Solution**: Over a full-duplex, long-lived WebSocket connection, each time the cloud or local large model emits a token (one character), the robot outputs it in real time over WS, and text-to-speech (TTS) speaks it in a synchronized streaming fashion.

#### Interface 2: WebRTC (Solving First-Person Video and Two-Way Audio/Video Intercom)
- **Pain point**: When a user opens the mobile app to see "what the robot duck sees", traditional HTTP polling or RTSP has latency as high as 1–3 seconds and is extremely hard to get through intranet NAT;
- **Solution**: Introduce a WebRTC stack with end-to-end latency under **200 milliseconds** and built-in NAT traversal; it also supports low-latency two-way voice intercom between the phone and the robot, and virtual gamepad commands can be sent over a DataChannel.

#### Interface 3: MQTT (Solving Cloud Connectivity and Cloud-Brain Control)
- **Solution**: Integrate a lightweight MQTT client (based on the Eclipse Paho C library, roughly a few hundred KB). After boot and network connection, it maintains a long-lived connection to the cloud IoT platform, handling heartbeat keepalive, AI event reporting, and cloud control commands.

#### Interface 4: DMA-BUF / Shared Memory (Solving Zero-Copy Local Video Frame Transfer)
- **Pain point**: A single raw 1080P YUV/RGB frame is several megabytes; it must never be converted into a JSON string and sent over a socket, or the CPU will be saturated by memory copying;
- **Solution**: Use the standard Linux DMA-BUF (Direct Memory Access Buffer). The camera driver fills images directly into physical memory, processes pass only a file descriptor (fd) over a Unix socket, and the NPU hardware reads physical memory directly via DMA — **zero CPU copies, with overhead close to 0 milliseconds**.

---

## 5. The Complete Architecture Overview (AI Robot / Smart Vision Gateway Form Factor)

The system overview combining Octopus's high reliability with modern AI requirements is as follows:

```text
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │        External user interaction layer (mobile app / web page / remote / R&D terminal) │
 └─────────┬───────────────────────┬────────────────────────┬──────────────────────┬──────┘
           │ Phone live video      │ Phone provisioning /   │ R&D tuning           │ Cloud large-model
           │ (WebRTC)              │ commands (Web/WS)      │ (serial/SSH)         │ commands
           ▼                       ▼                        ▼                      ▼
    ┌─────────────┐         ┌──────────────┐         ┌──────────────┐       ┌─────────────┐
    │ WebRTC      │         │ Hiawatha Web │         │ octopus_cli  │       │ MQTT client │
    │ engine      │         │ WebSocket/RPC│         │ console      │       │ Cloud dialog│
    │ Video/audio │         │              │         │ Servo tuning │       │ relay       │
    │ streams     │         │              │         │ parameters   │       │             │
    └──────┬──────┘         └──────┬───────┘         └──────┬───────┘       └──────┬──────┘
           │ Zero-latency A/V      │ Control commands /     │                      │
           │                       │ streaming text         │                      │
           ▼                       ▼                        ▼                      ▼
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │   [Octopus core business platform: /usr/bin/octopus_app] (cerebellum and steward)      │
 │  - octopus_cli engine + JSON-RPC/WS engine + octopus_config text persistence           │
 │  - Robot steward module: sends motion targets to motion_ctrl (real-time process or     │
 │    MCU), battery and charging detection                                                │
 │  - System steward module: Wi-Fi provisioning, OFI dual-firmware A/B hot-standby OTA    │
 │    upgrade control                                                                     │
 └─────────────────────────────────────────▲──────────────────────────────────────────────┘
                                           │
                 /var/run/json_ipc.socket high-speed local long-lived connection (control commands)
                                           │
 ┌─────────────────────────────────────────▼──────────────────────────────────────────────┐
 │   [External AI business daemon: /usr/bin/ai_vision_daemon] (vision and cerebrum)       │
 │  - Kept alive by the octopus_supervisor process (auto-restarted on crash without       │
 │    affecting the platform cerebellum)                                                  │
 │  - Hardware acceleration: accesses the board-level accelerator through the vendor NPU  │
 │    user-space library for face tracking / object recognition                          │
 │  - Audio subsystem: microphone array VAD offline wake word, cloud TTS playback driver   │
 │  - Image pipeline: zero-copy connection to V4L2 camera nodes (/dev/video*) via DMA-BUF │
 └────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                 Uniformly monitored, kept alive, and dependency-orchestrated by PID 1 (octopus_supervisor)
 ┌─────────────────────────────────────────┴──────────────────────────────────────────────┐
 │                Base system and two-stage boot (initramfs / rootfs / OverlayFS)         │
 └────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. RK3568 Hardware Compute Budget and Resource Planning

For the Rockchip **RK3568**, the hardware specifications are more than sufficient for the requirements above:

| Hardware Core | Specification and Compute Metrics | Actual Workload Handled and Load Behavior |
| :--- | :--- | :--- |
| **CPU** | 4-core Cortex-A55 @ 2.0GHz | Runs the Octopus platform, MQTT communication, and Linux system scheduling; typical day-to-day utilization is **< 15%**. |
| **NPU** | **1.0 TOPS** (INT8/FP16 supported) | Runs lightweight object detection models such as **YOLOv8n / YOLOv5s-nano** (covering 80 common classes including people, cats, and dogs). Single-frame inference takes only **25–35ms**, sustaining **25–30 FPS real-time detection**. |
| **ISP / VPU** | 4K hardware decoding, RGA 2D hardware acceleration | Hardware handles camera image cropping, rotation, and color space conversion (YUV->RGB), **consuming no CPU compute at all**. |
| **Memory footprint** | 2GB or 4GB LPDDR4 recommended as standard | The Octopus platform uses about **20–30MB**; the AI runtime + model weights use about **80–120MB**; the system as a whole runs with ample headroom. |

---

## 7. Summary

1. **Excellent architectural fit**: By positioning Octopus as the "stable cerebellum and steward" and isolating AI inference in a "standalone, supervised cerebrum", the design combines industrial-grade resilience against hangs with the flexibility of modern AI vision;
2. **Clear implementation levers**: The lower layers reuse OFI segmentation, A/B partitions, octopus_config persistence, and Unix socket communication; the periphery only needs to add WebSocket, WebRTC, and MQTT to readily incubate highly reliable, commercial-grade smart robot products!
