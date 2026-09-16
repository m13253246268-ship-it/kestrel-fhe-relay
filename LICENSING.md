# 许可说明 · fhe-relay / Licensing

Copyright (C) 2026 裴晓光 (Pei Xiaoguang) and contributors

> 本文件只说明**本仓（fhe-relay，协调仓）**各部分的许可。
> 主仓 `kestrel-llm` 的许可**不受本文件影响**，见其自身 `LICENSE` / `LICENSING.md`。

---

## 1. 本仓自身内容：MIT

**SPDX-License-Identifier: MIT**

本仓的文档、索引、清单（`manifest.sha256`），以及两份交接物归档
（`kestrel-fhe-relay_L0-4_*.zip`、`kestrel-fhe-relay_L26-27-fin_*.zip` —— 其中的密文、日志与包内 README）
均按 [MIT 许可](LICENSE) 发布：可自由使用、复制、修改、合并、发布、分发、再许可与销售，
只需保留版权声明与许可声明。

Everything in THIS repository — docs, indices, `manifest.sha256`, and the two hand-off
archives (the ciphertexts, logs and per-package README inside those zips) — is released under the
[MIT license](LICENSE): use, copy, modify, merge, publish, distribute, sublicense and sell freely,
provided the copyright and license notice are kept.

---

## 2. ⚠️ 例外：预编译二进制仍受 AGPL-3.0-or-later 约束

`kestrel-fhe-relay_tools_win-x64_*.zip` 内的 **`t23lay.exe` / `t23boot.exe` / `verify_layer.exe`**
是用**主仓 `kestrel-llm` 的 AGPL-3.0-or-later 源码编译出来的可执行文件**。
按 AGPL，这些二进制**不适用本仓的 MIT**，仍受 **AGPL-3.0-or-later** 约束。

- **对应源码（Corresponding Source）**：主仓 commit `4209982fb75bf8cda9ab452b8a40a7a24ea36fc7`，
  逐文件 SHA256 与获取方式见 [`SOURCE.md`](SOURCE.md)。
- 因此，**自行从主仓源码编译**得到的 `t23lay` / `t23boot` / `verify_layer` 同样是 AGPL 作品。
- 若你把这三个二进制并入自己的产品对外分发，需要承担 AGPL 的源码提供义务
  （AGPL §6：通过网络提供目标码时，须以同等方式免费提供对应源码）。

> 一句话：**本仓的文档与数据是 MIT；从主仓源码构建出来的二进制不是。**

The three executables inside `kestrel-fhe-relay_tools_win-x64_*.zip` are built from the
**AGPL-3.0-or-later** sources of the main repository `kestrel-llm`. They are therefore
**not** relicensed by this repository's MIT license and remain under **AGPL-3.0-or-later**;
the Corresponding Source is the pinned commit listed in [`SOURCE.md`](SOURCE.md).

---

## 3. 主仓 `kestrel-llm`：不受影响

接力需要克隆的主仓代码（`src/core/`、`tools/drivers/`、`tools/relay/`、`tools/preproc/`）
**仍按其自身许可发布**（该仓为 AGPL-3.0-or-later / 商业授权双许可，以该仓的
`LICENSE` 与 `LICENSING.md` 为准）。本仓的 MIT **不改变、也不覆盖**主仓的许可条款。

The main repository `kestrel-llm` keeps its own license (dual-licensed
AGPL-3.0-or-later / commercial, per that repository's own `LICENSE` and `LICENSING.md`).
This repository's MIT license does **not** alter or extend to it.

---

## 4. 第三方组件（各自遵循其原许可）

主仓内随附的第三方组件独立保留其原许可证，二者与 AGPL 兼容：

| 组件 | 位置 | 许可 |
|---|---|---|
| `stb_image.h` | `src/model/stb_image.h` | MIT License, Copyright (c) 2017 Sean Barrett |
| llama.cpp 派生 4x4 asm GEMM 内核 | `tools/kernels/llama_gemm_q4_0_4x4_asm.c`，以及 `src/model/vllm_safetensors.c` 中标注的 ggml_gemm / gemv / vec_dot 派生内核 | MIT License, Copyright (c) 2023–2026 The ggml authors |

版权声明与许可文本见各文件头。

---

## 5. 其他声明

1. 本许可**不转移**任何商标权、专利权。
2. 对本仓的贡献（Issue / PR）按"提交即同意以 **MIT** 条款发布"处理。
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
SPDX-License-Identifier: MIT
Copyright (C) 2026 裴晓光 and contributors
```

**注意**：不要把 `MIT` 标到主仓源码（`src/core/`、`tools/drivers/`、`tools/relay/*.c`、`tools/preproc/*.py`）上——
那些文件属于主仓，许可仍是 AGPL-3.0-or-later。
