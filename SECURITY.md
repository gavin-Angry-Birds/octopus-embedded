# Security Policy

**English** | [简体中文](#简体中文)

This repository is currently in the **architecture draft** phase (`v0.1.0-design`). It ships design documents only; application source code begins in Phase 1. Even so, we accept reports about security-relevant design flaws (boot/OTA assumptions, credential handling, threat-model gaps) and, once code lands, vulnerabilities in Octopus Embedded software.

## Supported versions

| Version / tag | Supported |
|---------------|-----------|
| `v0.1.0-design` (docs only) | Design-flaw reports yes |
| Unreleased Phase 1+ code | Will be listed here when first tagged |

## How to report a vulnerability

**Do not** open a public GitHub Issue for vulnerabilities or for details that could help an attacker.

Prefer one of these channels (in order):

1. **GitHub Security Advisories** (preferred once the public repository is live)  
   Open a private advisory:  
   `https://github.com/gavin-Angry-Birds/octopus-embedded/security/advisories/new`  
   If the repository is moved or renamed, use **Security → Advisories → New draft security advisory** on the current repository.
2. If private advisories are not yet enabled, open a **minimal** public Issue titled `[SECURITY]` that contains **only** a request for a private channel (no exploit details, no secrets). Maintainers will follow up privately.

Please include, when applicable:

- Affected document section or (later) component / commit
- Impact (brick, remote code execution, credential leak, downgrade, etc.)
- Reproduction or threat scenario
- Whether a fix or mitigation is already known

## Response targets

These are targets for the design phase; they may be tightened once production firmware ships.

| Step | Target |
|------|--------|
| Initial acknowledgement | within **5 business days** |
| Triage (severity / scope) | within **15 business days** |
| Public disclosure coordination | case-by-case; we prefer coordinated disclosure after a fix or documented mitigation exists |

We may request a CVE for code-level issues once releases exist. Design-only issues are usually tracked as RFCs / architecture updates without a CVE.

## Scope

**In scope**

- Flaws in Octopus Embedded design or (later) code that weaken anti-brick OTA, Secure Boot assumptions, credential storage, or management-plane auth
- Leaked secrets accidentally committed to this repository
- Clear violations of the stated threat model in `docs/`

**Out of scope**

- Vulnerabilities solely in third-party SoC BSPs, closed NPU runtimes, or upstream U-Boot/Linux that are not introduced by Octopus patches
- Theoretical issues with no credible impact on the stated device classes
- Social-engineering reports against individual contributors

## Safe harbor

We will not pursue legal action against researchers who:

- Make a good-faith effort to avoid privacy violations, data destruction, and service disruption
- Do not access data that is not their own beyond what is needed to demonstrate the issue
- Report promptly and keep details private until we coordinate disclosure

## Conduct reports

Reports about community conduct (see [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)) are **not** security vulnerabilities. Use the enforcement contact described in that file.

---

## 简体中文

本仓库当前处于**架构草案**阶段（`v0.1.0-design`），以设计文档为主；应用源码自阶段 1 起编写。即便如此，我们仍接受与安全相关的设计缺陷报告（引导 / OTA 假设、凭证存放、威胁模型缺口等）；代码落地后亦接受软件漏洞报告。

### 如何报告

**请勿**在公开 Issue 中贴出可被利用的细节。

1. **优先**：GitHub 私有安全公告  
   `https://github.com/gavin-Angry-Birds/octopus-embedded/security/advisories/new`  
   （若仓库迁移或改名，请在对应仓库的 **Security → Advisories** 中新建。）
2. 若尚未开通私有公告：开一个**仅含对接请求**的 Issue，标题 `[SECURITY]`，不要写利用细节；维护者会私下跟进。

### 响应时限（设计阶段目标）

| 步骤 | 目标 |
|------|------|
| 首次确认收到 | **5** 个工作日内 |
| 分级与影响范围 | **15** 个工作日内 |
| 公开披露协调 | 视情况；优先在修复或文档化缓解措施之后协调披露 |

### 行为准则相关投诉

社区行为问题见 [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)，**不要**当作漏洞走上述安全渠道。
