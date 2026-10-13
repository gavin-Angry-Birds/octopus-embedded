# Octopus Embedded — Documentation Index

**English** | [中文](#中文)

Welcome to the **Octopus Embedded** documentation. This repository is in the **architecture discussion** phase. Source code follows the [ROADMAP](../ROADMAP.md).

### Language policy

| Role | Rule |
|------|------|
| **Authoritative intent** | Simplified Chinese bodies (`NN-slug.md`, `README.zh-CN.md`) |
| **English twin** | Same stem with `.en.md` (e.g. `00-platform-design.en.md`) |
| **Entry / legal** | `README.md`, `SECURITY.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md` are English-first or bilingual |
| **Edits** | Change meaning in Chinese first (or same PR), then align the `.en.md` twin |

Every Chinese design document below has an English counterpart. If the two diverge, **Chinese wins** for architecture intent until an RFC updates both.

### What is frozen vs draft

| Material | Status |
|----------|--------|
| Layering / HAL boundaries | Draft under RFC — discussion baseline |
| **Chapter 13.1 Phase 3 blockers** | **Not decided** — must converge before boot/OTA coding |
| AI / robotics guide | Future / product extension — not on Phase 1–3 critical path |

Security: [SECURITY.md](../SECURITY.md).

---

## Recommended reading order

| # | Chinese (authoritative) | English twin | Purpose |
|---|-------------------------|--------------|---------|
| 1 | [architecture/00-platform-design.md](./architecture/00-platform-design.md) | [`.en.md`](./architecture/00-platform-design.en.md) | Architecture draft, RFCs, open issues |
| 2 | [reliability/00-critical-topics.md](./reliability/00-critical-topics.md) | [`.en.md`](./reliability/00-critical-topics.en.md) | Watchdog, OOM, storage, diagnostics |

### Deep dives

| Chinese | English | When |
|---------|---------|------|
| [00-language-cpp-guidelines.md](./guides/00-language-cpp-guidelines.md) | [`.en.md`](./guides/00-language-cpp-guidelines.en.md) | Before C/C++ coding |
| [01-qemu-test-strategy.md](./guides/01-qemu-test-strategy.md) | [`.en.md`](./guides/01-qemu-test-strategy.en.md) | Test pyramid / CI |
| [02-switch-hal-drivers.md](./guides/02-switch-hal-drivers.md) | [`.en.md`](./guides/02-switch-hal-drivers.en.md) | HAL / optional switch offload |
| [03-ai-robot-vision.md](./guides/03-ai-robot-vision.md) | [`.en.md`](./guides/03-ai-robot-vision.en.md) | Vision / robotics (later) |
| [project/branding.md](./project/branding.md) | (bilingual in one file) | Naming / logo |

---

## Document map

```
docs/
├── README.md
├── architecture/
│   ├── 00-platform-design.md       ← Chinese
│   └── 00-platform-design.en.md    ← English twin
├── reliability/
│   ├── 00-critical-topics.md
│   └── 00-critical-topics.en.md
├── guides/
│   ├── 00-… / 01-… / 02-… / 03-….md
│   └── 00-… / 01-… / 02-… / 03-….en.md
└── project/
    └── branding.md                 ← bilingual
```

**Convention:** `NN-slug.md` = Chinese; `NN-slug.en.md` = English twin. Folder READMEs stay short indexes.

---

## Terminology

Official name: **Octopus Embedded**. Shorthand “Octopus” is fine in prose.

This repository holds **original architecture documentation only**. It does not include or distribute third-party proprietary/closed-source code or prebuilt binaries. Vendor SDKs / NPU runtimes / boot firmware are out-of-tree integration boundaries.

---

## 中文

### 语言与对照

| 角色 | 规则 |
|------|------|
| **权威意图** | `docs/` 下简体中文正文（`NN-slug.md`）与根目录 `README.zh-CN.md` |
| **英文对照** | 同名 `NN-slug.en.md` |
| **修改** | 改结论时先改中文（或同一 PR 改两边） |

下列设计文档均已提供英文对照；若中英冲突，**架构意图以中文为准**。

### 推荐阅读

1. [architecture/00-platform-design.md](./architecture/00-platform-design.md)（[English](./architecture/00-platform-design.en.md)）— 架构草案与 RFC  
2. [reliability/00-critical-topics.md](./reliability/00-critical-topics.md)（[English](./reliability/00-critical-topics.en.md)）— 量产关键议题  
3. `guides/` 下语言 / 测试 / 交换 / 视觉专题（均有 `.en.md`）

### 目录约定

- `architecture/`、`reliability/`、`guides/`：中文正文 + `.en.md` 对照  
- `project/branding.md`：单文件中英双语  
- 命名 `NN-主题.md`；新文档放入对应目录，并同时提供英文 twin  
