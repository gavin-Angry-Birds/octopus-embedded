# Contributing to Octopus Embedded

Thank you for your interest in **Octopus Embedded**. During the **design phase**, we especially welcome architecture feedback, RFC discussions, and documentation improvements.

**English** | 中文摘要见文末

---

## Language policy (source of truth)

- **Simplified Chinese** (`README.zh-CN.md` and the body of docs under `docs/`) is the **authoritative** source for architecture intent.
- Keep English entry points (`README.md`, `SECURITY.md`, `CHANGELOG.md`, and English sections of bilingual files) aligned in the same PR when you change meaning.
- Translations that only improve English wording without changing Chinese intent are welcome.

Security reports: [SECURITY.md](SECURITY.md). Do not paste exploit details into public issues.

---

## What to contribute now

| Type | Examples |
|------|----------|
| **Design RFC** | Propose changes to watchdog policy, IPC layout, module lifecycle |
| **Documentation** | Fix typos, clarify diagrams, translate sections to English |
| **Issues** | Report inconsistencies between docs, ask architecture questions |
| **Logo / brand** | Follow [docs/project/branding.md](docs/project/branding.md); submit SVG via PR |

**Not accepted yet:** Large code dumps without prior RFC alignment (code phase starts Phase 1).

---

## RFC process (design phase)

1. Open a **GitHub Discussion** (category: RFC) or an **Issue** with title `[RFC] Short summary`.
2. Describe problem, proposed change, alternatives, and impact on existing docs.
3. Maintainers will label `rfc` / `accepted` / `declined` after review.
4. Accepted RFCs → update relevant files under `docs/` via pull request.

---

## Pull requests

1. Fork the repository and create a branch: `docs/fix-watchdog-wording` or `rfc/shm-channel-id`.
2. Keep PRs focused — one topic per PR.
3. For documentation only: no build required; ensure Markdown renders correctly.
4. Sign your commits (optional but appreciated).

### Commit message style

```
docs: clarify lock_watchdog deadlock timeout in topic 7
rfc: propose octopus_shm channel_id allocation
fix: typo in README.zh-CN.md
```

---

## Code of conduct

Be respectful and constructive. See [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).

---

## Licensing

By contributing, you agree that your contributions are licensed under the [Apache License 2.0](LICENSE).

Exception: patches that land in GPL projects (U-Boot, Linux kernel drivers) are licensed under GPL-2.0-or-later (U-Boot) or GPL-2.0 (kernel) and must carry the matching SPDX identifier. See chapter 12 of the [architecture specification](docs/architecture/00-platform-design.md).

Do **not** submit proprietary/closed-source code, vendor prebuilt binaries (NPU runtimes, boot firmware packs, etc.), or confidential third-party materials. Architecture docs may name vendor SDKs only as **out-of-tree integration boundaries**.

---

## 中文摘要

- **权威源**：架构意图以简体中文（`README.zh-CN.md` 与 `docs/` 正文）为准；改结论时同一 PR 内对齐英文入口。
- **设计阶段**欢迎：架构 RFC、文档修正、Issue 讨论。
- 较大改动请先开 Discussion / `[RFC]` Issue，合并前需维护者确认。
- 安全漏洞与可利用细节见 [SECURITY.md](SECURITY.md)，不要发在公开 Issue。
- **禁止**提交任何第三方专有/闭源源码、厂商预编译包（NPU 运行时、启动固件包等）或保密材料。文档中提及厂商 SDK 仅作**仓库外集成边界**说明。
- 贡献内容遵循 Apache 2.0 许可证；进入 U-Boot / Linux 内核的补丁例外，分别使用 GPL-2.0-or-later / GPL-2.0。
