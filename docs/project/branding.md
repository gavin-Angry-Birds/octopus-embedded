# Octopus Embedded — Brand & Logo Guidelines

**English** | [中文](#中文)

---

## Official naming

| Role | Name |
|------|------|
| **Full brand name** | Octopus Embedded |
| **Short name** | Octopus (in prose only; always disambiguate in titles) |
| **Chinese** | Octopus 嵌入式平台 / Octopus 通用嵌入式底座 |
| **GitHub repository** | `octopus-embedded` |
| **GitHub path** | [`gavin-Angry-Birds/octopus-embedded`](https://github.com/gavin-Angry-Birds/octopus-embedded) |
| **API / library prefix** | `octopus_*`, `liboctopus_*` |
| **Main daemon (future)** | `octopus_app` |
| **PID 1 supervisor (future)** | `octopus_supervisor` (`/sbin/octopus_supervisor`) |
| **Initramfs loader (future)** | `octopus_loader` (`/sbin/octopus_loader`) |
| **Firmware image format** | **OFI** (Octopus Firmware Image), file extension `.ofi` |
| **U-Boot firmware driver (future)** | `cmd_ofi.c` — commands `ofi load`, `ofi boot` |
| **Host pack tool (future)** | `ofi_pack` |
| **CLI engine (future)** | `octopus_cli` (`liboctopus_cli`), command DSL `.ocli`, host compiler `ocli_tool` |
| **Config persistence (future)** | `octopus_config` (`liboctopus_config`) — text running-config save and replay |

### Do not use

- “Octopus Deploy” or any DevOps/deployment positioning
- Bare “Octopus” as the only title on marketing pages without “Embedded”
- Logos or colors that resemble [Octopus Deploy](https://octopus.com)

---

## Tagline (optional)

- **EN:** *One platform, many SoCs.*
- **CN:** *一套底座，多芯通用。*

---

## Logo specification

### Wordmark layout

```text
  Octopus          ← Primary line: bold, clean sans-serif
  Embedded         ← Secondary line: lighter weight, smaller size
```

### Visual metaphor (icon)

Abstract geometry preferred — **not** a cartoon octopus:

- **Center:** core daemon / platform hub (hexagon or circle)
- **Radiating arms:** modules, SoC targets, data-plane channels
- **Style:** flat, minimal strokes; readable at 16×16 (favicon)

### Colors (draft palette)

| Role | Hex | Usage |
|------|-----|-------|
| Primary | `#0D9488` | Teal — reliability, embedded |
| Secondary | `#64748B` | Slate — text, secondary lines |
| Dark BG | `#0F172A` | README dark mode / social banner |
| Light BG | `#F8FAFC` | Documents, light theme |

### Deliverables (when logo is ready)

Place under `assets/`:

```
assets/
├── logo.svg           Full wordmark + icon (light background)
├── logo-dark.svg      For dark backgrounds
├── icon-512.png       GitHub social preview / app icon
└── favicon.ico        16×16 / 32×32
```

---

## README usage

- Default homepage: [README.md](../README.md) (English)
- Chinese: [README.zh-CN.md](../README.zh-CN.md)
- Link line: `**English** | [简体中文](README.zh-CN.md)`

---

## Disclaimer (required in README)

> Octopus Embedded is an independent open-source project. It is not affiliated with Octopus Deploy or any other "Octopus" trademark holder.

---

## Trademark note

“Octopus” is used descriptively with the qualifier **Embedded** to reduce confusion with existing software brands. Before commercial productization, conduct trademark searches in target jurisdictions (e.g. US Class 9, CN Class 9/42) and consider registering **OCTOPUS EMBEDDED** as a composite mark.

---

## 中文

### 正式命名

- **品牌全称：** Octopus Embedded  
- **简称：** Octopus（正文可用，标题需带 Embedded）  
- **仓库名：** `octopus-embedded`  
- **GitHub 路径：** [`gavin-Angry-Birds/octopus-embedded`](https://github.com/gavin-Angry-Birds/octopus-embedded)  
- **PID 1 监护器：** `octopus_supervisor`（二进制 `/sbin/octopus_supervisor`）  
- **Initramfs 引导：** `octopus_loader`（二进制 `/sbin/octopus_loader`）  
- **固件镜像格式：** **OFI**（Octopus Firmware Image），扩展名 `.ofi`  
- **U-Boot 驱动 / 打包工具：** `cmd_ofi.c`、`ofi_pack`  
- **命令行引擎：** `octopus_cli`（库 `liboctopus_cli`，语法文件 `.ocli`，编译器 `ocli_tool`）  
- **配置持久化：** `octopus_config`（库 `liboctopus_config`）  

### Logo 要求

- 字标两行：Octopus + Embedded  
- 图标：抽象几何（中心核心 + 多触手/多节点），避免写实蓝色章鱼  
- 主色建议：`#0D9488` 青绿 + `#64748B` 灰  

### 禁止

- 与 Octopus Deploy 相似的视觉或 DevOps 话术  
- 在本仓库提交任何第三方专有/闭源源码或厂商预编译二进制（含 NPU 运行时、启动固件包等）  
