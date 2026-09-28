# 许可说明 · fhe-relay / Licensing

Copyright (C) 2026 裴晓光 (Pei Xiaoguang) and contributors

> 本文件只说明**本仓（fhe-relay，协调仓）**各部分的许可。
> 主仓 `kestrel-llm` 的许可**不受本文件影响**，见其自身 `LICENSE` / `LICENSING.md`。

---

## 1. 本仓自身内容：Apache-2.0

**SPDX-License-Identifier: Apache-2.0**

本仓的文档、索引、清单（`manifest.sha256`），以及两份交接物归档
（`kestrel-fhe-relay_L0-4_*.zip`、`kestrel-fhe-relay_L26-27-fin_*.zip` —— 其中的密文、日志与包内 README）
均按 [Apache License 2.0](LICENSE) 发布：可自由使用、复制、修改、合并、发布、分发、再许可与销售
（**含闭源商用**），只需保留版权与许可声明、并标注你修改过的文件（第 4 条）；第 3 条另含**明示专利授权**。

Everything in THIS repository — docs, indices, `manifest.sha256`, and the two hand-off
archives (the ciphertexts, logs and per-package README inside those zips) — is released under the
[Apache License 2.0](LICENSE): use, copy, modify, merge, publish, distribute, sublicense and sell
freely, **including in closed-source commercial products**, provided the copyright and licence
notices are kept and modified files are marked (section 4). Section 3 also grants an express
patent licence.

---

## 2. 预编译二进制：跟随主仓，**旧包仍是 AGPL**

`kestrel-fhe-relay_tools_win-x64_*.zip` 内的 **`t23lay.exe` / `t23boot.exe` / `verify_layer.exe`**
是用**主仓 `kestrel-llm` 的源码**编译出来的可执行文件，因此它们**不是本仓的原创作品**，
许可**跟随主仓在构建当时的许可**：

| 版本 | 许可 |
|---|---|
| **2026-09 已上传的那份 zip**（由主仓 commit `4209982…` 构建，当时主仓为 AGPL 双许可） | **AGPL-3.0-or-later**。已下载者永久保有该授权，**不可撤回** |
| **本次许可变更之后**从主仓**重新构建**的工具 | **Apache-2.0**（与主仓现行许可一致） |

- 本仓的 Apache-2.0 **不覆盖**旧二进制。
- 旧包对应源码：主仓 commit `4209982fb75bf8cda9ab452b8a40a7a24ea36fc7`（**AGPL 时期**），
  逐文件 SHA256 与获取方式见 [`SOURCE.md`](SOURCE.md)。
- 一句话：**文档与数据看本文件；二进制看它编译时主仓的许可。**

The three executables inside `kestrel-fhe-relay_tools_win-x64_*.zip` are built from the **main
repository** sources and therefore follow **the main repository's licence as of build time**:

| Version | Licence |
|---|---|
| **The zip already uploaded in 2026-09** (built from main-repo commit `4209982…`, when the main repo was dual-licensed AGPL) | **AGPL-3.0-or-later**; recipients keep that grant permanently — it **cannot be revoked** |
| Tools **rebuilt after this licence change** | **Apache-2.0** (matching the main repo's current licence) |

---

## 3. 主仓 `kestrel-llm`：现行 Apache-2.0，历史版本 AGPL

接力需要克隆的主仓代码（`src/core/`、`tools/drivers/`、`tools/relay/`、`tools/preproc/`）
以**该仓自身的 `LICENSE` / `LICENSING.md`** 为准：**自 2026-09 本次变更起为 Apache-2.0**；
**变更之前**的 commit（含 [`SOURCE.md`](SOURCE.md) 锚定的 `4209982…`）**仍为 AGPL-3.0-or-later**。

本仓的 Apache-2.0 **不改变、也不覆盖**主仓的许可条款。

The main repository `kestrel-llm` is governed by **that repository's own `LICENSE` /
`LICENSING.md`**: it is **Apache-2.0 from this change onward (2026-09)**, while **commits before
the change** (including the `4209982…` pinned in [`SOURCE.md`](SOURCE.md)) **remain
AGPL-3.0-or-later**. This repository's Apache-2.0 licence **does not alter or extend to it**.

---

## 4. 第三方组件（各自遵循其原许可）

主仓内随附的第三方组件独立保留其原许可证，二者与 Apache-2.0 兼容：

| 组件 | 位置 | 许可 |
|---|---|---|
| `stb_image.h` | `src/model/stb_image.h` | MIT License, Copyright (c) 2017 Sean Barrett |
| llama.cpp 派生 4x4 asm GEMM 内核 | `tools/kernels/llama_gemm_q4_0_4x4_asm.c`，以及 `src/model/vllm_safetensors.c` 中标注的 ggml_gemm / gemv / vec_dot 派生内核 | MIT License, Copyright (c) 2023–2026 The ggml authors |

版权声明与许可文本见各文件头。

---

## 5. 其他声明

1. 本许可**不转移**任何商标权；专利权按 Apache-2.0 第 3 条授予。
2. 对本仓的贡献（Issue / PR）按 **Apache-2.0 第 5 条**处理：提交即同意以 **Apache-2.0** 条款授权。
3. **模型权重、数据集与转换产物不在本仓分发**；如需使用，请遵循其原始来源的许可
   （例如 Qwen 系列遵循 Qwen 社区许可）。
4. 安全漏洞请按主仓 `kestrel-llm` 中 `SECURITY.md` 的流程私下报告，**不要**直接开公开 Issue。
5. **本仓不宣称密码学安全强度**（当前参数为机制验证级，远低于 128-bit）。该边界声明属于交付内容的一部分，
   见 [`README.md`](README.md) §8 与 [`docs/04_接力复现指南.md`](docs/04_接力复现指南.md) §9；
   换成宽松许可**不改变**这一点，反而意味着使用方**须自行承担**误用风险。

---

## 附：文件头建议附注（SPDX）

本仓的脚本/文本如需加注，建议：

```
SPDX-License-Identifier: Apache-2.0
Copyright (C) 2026 裴晓光 and contributors
```

**注意**：不要把 `Apache-2.0` 标到主仓源码（`src/core/`、`tools/drivers/`、`tools/relay/*.c`、`tools/preproc/*.py`）上——
那些文件属于主仓，许可以主仓自己的 `LICENSE` 为准。
