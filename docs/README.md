# Octopus Embedded — Documentation Index

**English** | [中文](#中文)

Welcome to the **Octopus Embedded** documentation. This repository is in the **architecture discussion** phase. [00-platform-design](./architecture/00-platform-design.md) is the draft to review. Other documents are supporting notes. Source code will follow the [ROADMAP](../ROADMAP.md).

### Language policy (source of truth)

- **Simplified Chinese** under `docs/` (and `README.zh-CN.md`) is the **authoritative** wording for architecture intent and RFCs.
- English text in this index, `README.md`, `SECURITY.md`, and `CHANGELOG.md` is kept in sync for entry points; it must not silently diverge from the Chinese specs.
- When translating or editing, update the Chinese document first (or in the same PR), then align English.

### What is frozen vs draft

| Material | Status |
|----------|--------|
| Layering / HAL boundaries in `architecture/00-platform-design.md` | Draft under RFC; treat as the discussion baseline |
| **Chapter 13.1 Phase 3 blockers** (single-valid-slot window, bootcount classification, Rescue Mode, OFI license split, …) | **Not decided** — must converge before `cmd_ofi.c` / loader / firmware_worker coding |
| `guides/03-ai-robot-vision.md` | Product / future extension notes — **not** on the Phase 1–3 critical path |
| NFR numbers, QEMU green checks | Do not read as “already proven on all four storage topologies” |

Security reporting: see [SECURITY.md](../SECURITY.md).

---

## Recommended reading order

### 1. Start here

| # | Document | Time | Purpose |
|---|----------|------|---------|
| 1 | [architecture/00-platform-design.md](./architecture/00-platform-design.md) | ~15 min | Architecture draft: layers, boundaries, open questions |
| 2 | [reliability/00-critical-topics.md](./reliability/00-critical-topics.md) | ~30 min | Supporting notes: watchdog, OOM, storage, diagnostics |

### 2. Deep dives (as needed)

| Document | When to read |
|----------|--------------|
| [guides/00-language-cpp-guidelines.md](./guides/00-language-cpp-guidelines.md) | Before writing C/C++ code |
| [guides/02-switch-hal-drivers.md](./guides/02-switch-hal-drivers.md) | HAL boundaries, optional switch ASIC plugin |
| [guides/01-qemu-test-strategy.md](./guides/01-qemu-test-strategy.md) | Test pyramid before CI setup |
| [guides/03-ai-robot-vision.md](./guides/03-ai-robot-vision.md) | Robotics / NPU / dual-brain design |
| [project/branding.md](./project/branding.md) | Logo, naming, trademark disclaimer |

---

## Document map

```
docs/
├── README.md                          ← You are here
├── architecture/                      ★ Core architecture (frozen boundaries)
│   └── 00-platform-design.md          Architecture draft
├── reliability/                       Production readiness
│   └── 00-critical-topics.md          Supporting notes (×7)
├── guides/                            Deep dives (read as needed)
│   ├── 00-language-cpp-guidelines.md
│   ├── 01-qemu-test-strategy.md
│   ├── 02-switch-hal-drivers.md
│   └── 03-ai-robot-vision.md
└── project/                           Brand & process
    └── branding.md
```

**Convention:** `NN-slug.md` — `00` = entry point of that folder, numbers fix reading order. New docs go into the folder that matches their concern; don't drop files flat into `docs/`.

---

## Terminology note

Design documents may use **Octopus** as shorthand. The official product name is **Octopus Embedded** (see [project/branding.md](./project/branding.md)).

All documents in this repository describe **original Octopus Embedded designs** only. This repository does **not** include or distribute any third-party proprietary/closed-source source code or prebuilt binaries. Mentions of vendor SDKs, NPU runtimes, or boot firmware are **integration-boundary notes** only; those materials stay outside this repo and must be obtained under the vendor’s own license.

---

## 中文

### 语言与权威源

- **`docs/` 下的简体中文正文**以及根目录 `README.zh-CN.md` 是架构意图与 RFC 的**权威源**。
- 英文入口（`README.md`、本索引英文章节、`SECURITY.md`、`CHANGELOG.md`）须与中文对齐，不得另写一套结论。
- 修改规范时优先改中文（或同一 PR 内中英同时改）。

### 草案与未决

| 材料 | 状态 |
|------|------|
| `architecture/00-platform-design.md` 中的分层与 HAL 边界 | 讨论基线（草案 / RFC） |
| **第 13.1 节 Phase 3 前 Blockers**（单一有效槽窗口、bootcount 分类、救援模式、OFI 许可拆分等） | **尚未定稿** — 编写 `cmd_ofi.c` / loader / firmware_worker 之前必须收敛 |
| `guides/03-ai-robot-vision.md` | 产品向 / 远期扩展，**不在**阶段 1–3 主路径 |
| NFR 数字、QEMU 绿灯 | 不要理解成「四类存储拓扑均已验证」 |

漏洞与安全设计缺陷报告见 [SECURITY.md](../SECURITY.md)。

### 推荐阅读顺序

1. **[architecture/00-platform-design.md](./architecture/00-platform-design.md)** — 架构设计草案（先读这篇，参与讨论；重点看第 13 / 13.1 章未决项）
2. **[reliability/00-critical-topics.md](./reliability/00-critical-topics.md)** — 配套讨论稿（看门狗、存储、诊断等）
3. 其余 `guides/` 下 HAL、语言、测试、AI 文档同样是配套材料，不作为已冻结的实现承诺

### 目录约定

- `architecture/`：核心架构与分层边界（先收敛这部分）
- `reliability/`：量产关键议题（看门狗、OOM、存储、诊断）
- `guides/`：专题深挖（按需阅读）
- `project/`：品牌与流程
- 文件命名 `NN-主题.md`，`00` 为每目录入口；新文档请放入对应目录，不要直接堆在 `docs/` 根下

### 术语说明

文档中「Octopus」为简称，正式产品名为 **Octopus Embedded**。本仓库文档均为 Octopus Embedded 原创设计；**不包含、不分发**任何第三方专有/闭源源码或预编译包。文中提及的厂商 SDK、NPU 运行时、启动固件等仅说明集成边界，须由设备方按厂商许可自行获取。
