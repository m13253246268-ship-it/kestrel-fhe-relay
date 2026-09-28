[English](README.en.md) | **简体中文**

# Kestrel FHE 密文推理链接力 · 社区算力征集

> **本仓是协调仓**：只放技术文档与数据索引（对外征集的帖子材料不在这里）。
> **代码不在这里**——驱动与引擎源码单点保留在主仓 `kestrel-llm`，版本锚点见 [`SOURCE.md`](SOURCE.md)。
> **数据不在这里**——两份交接物归档（链头 `L0-4` 158.75 MB、链尾 `L26-27-fin` 64.99 MB，均为**下载体积**）挂在
> https://github.com/m13253246268-ship-it/kestrel-fhe-relay/releases ，校验方式见 [`data/README.md`](data/README.md)。
>
> **Gitee 镜像**：本仓在 <https://gitee.com/pei-xiaoguang/fhe-relay> 同步一份。
> 其中**链尾 `L26-27-fin` 与预编译工具**同时挂在 Gitee Releases；**链头 `L0-4` 只在 GitHub** ——
> Gitee 的发行版附件有 **100 MB 单文件上限**（GitHub 是 2 GB/资产），该包 158.75 MB 超限，
> 且**不拆分**（拆卷会使本仓那个"单一 zip 的 SHA256"验证锚点失效）。取舍理由见 [`data/README.md`](data/README.md) §1。

---

## 1. 我们在做什么

用**纯 C11、零第三方依赖、纯 CPU**（无 GPU）从零实现的 RNS-CKKS 引擎，把 Qwen3-VL-2B（28 层）
**全程在密文上**跑完并输出 logits：输入与中间激活全密文，服务方看不到明文。

- 引擎与驱动源码：主仓 `kestrel-llm`（`src/core/vllm_ntt.c` / `vllm_ckks.c` / `vllm_tp.c` + `tools/drivers/`）
- 当前参数：`n=2048`、`slots=1024`、`scale=2^60`；`lay` 用 112 个 RNS 素数、`boot` 用 2100 个素数
- 已验证：x86-64 与 RK3588(aarch64) **逐系数位级一致**

---

## 2. 现在到哪了（诚实版）

| 指标 | 状态 |
|---|---|
| 已完成层 | **第 0–4 层（5 层）**：每层 `lay` → `RESULT=PASS`，每层 `boot` → `BOOT=PASS`（刷新回满链 `out np=2083`） |
| 链尾支线（另一条线，**不是续接**） | **层 26–27 + 输出头 `fin`**：从明文种子直接加密进 L26、**绕过 5–25 层**，五步全 `PASS`，`fin` 的 logits top1 **4/4** 与参考一致（2026-09-15 实测，合计 **16284 s ≈ 4.52 h**） |
| 链上未完成 | **第 5–27 层（23 层）**——接力链仍停在 `u4r112`；链尾支线**不能替代**这一段在链上的续接 |
| 单跳成本（实测） | `lay` **1942–2008 s**（32.4–33.5 min）；`boot` **4339–4891 s**（72.3–81.5 min） |
| 单层成本（实测） | 约 **105–115 min ≈ 1.8 h/层**（`lay` + `boot`） |
| 0–4 层实测合计 | 10 跳 **32182 s ≈ 8.94 h**（`lay` 5 跳 + `boot` 5 跳） |
| 全链外推 | **≈50 h**（28 × 1.8 h；projection，**不是实测值**） |
| 28 层全链能否跑通 | **尚未验证**——这正是本次求助要解决的问题 |

> 上面单跳区间取自本轮（2026-09-14 → 09-15）从种子重跑的逐跳墙钟，原始数据见
> [`docs/03_实测数据附录.md`](docs/03_实测数据附录.md) §2。硬件：AMD Ryzen 7 9800X3D (8C/16T)，
> 固定 `T23_NT=4`，Windows 11 24H2，MinGW-W64 gcc 16.1.0。

`verify_layer` 独立复核（解密 → 解码 → 对明文参考算 `max|err|`，容差 `3e-2`）：
`-Mode lay` 与 `-Mode boot` 各 **5/5 `ALL_LAYERS_VERIFY_PASS`**。

---

## 3. 为什么需要接力

单机 ≈50 小时纯 CPU 计算。切成「每人几层」就能几天内出结果。
**每层交接物只有 8 个密文文件 `u{L}r112_{t}_{h}.ct`（约 28 MB）**，别人拿到即可从 `L+1` 继续。

---

## 4. 怎么帮（三层参与门槛，任选）

- [ ] **认领 1 层**（≈1.8 h）：最简单的参与方式
- [ ] **认领 3 层**（≈5.5 h）
- [ ] **认领一整段**（如 8–10 / 25–27 + `fin`）

**步骤**（详细版见 [`docs/04_接力复现指南.md`](docs/04_接力复现指南.md)）：

1. 准备一台 ≥4 逻辑核、≥8 GB 内存的 x86-64 或 ARM64 机器 + gcc；
2. 克隆主仓 `kestrel-llm`（**必须取 [`SOURCE.md`](SOURCE.md) 锚定的版本**），并准备数据包（约 7 GB）；
3. 编译出 `t23lay` / `t23boot`（按指南 §2）；**不想装 gcc 就直接用 Releases 里的预编译三件套**——
   解压后把 `t23lay.exe` / `t23boot.exe` / `verify_layer.exe` 放进 `<仓库根>\.tmp_tok\`，
   见 [`data/README.md`](data/README.md) §3 ⓪；
4. 跑指定层：
   ```powershell
   powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\rerun5_lay_boot.ps1 -From 5 -To 7
   ```
5. 一条命令打包并提交：`pack_relay.ps1 -From 5 -To 7 -Id <你的ID>`，把生成的 zip 贴回本仓 Issue（1 层 ≈28 MB）；
6. 下一棒从 `L+1` 继续；**任何人**拿到包后可用 `verify_relay.ps1 -Zip <包>` **一条命令**校验
   （完整性 + 判据 + 结构，退出码 0/1）；
7. **想查「到底算得对不对」**：用 `verify_layer.ps1 -From L -To L` 解密密文与明文参考逐元素对照
   （独立代码路径，非复用驱动判据）；对关键层可让第二人独立复算，再用 `verify_layer.exe -BitA/-BitB`
   逐字节比对。

> **认领请开 Issue**，标题写 `[claim] layer N`，便于统计与排期。

---

## 5. 代码与数据在哪

| 内容 | 位置 |
|---|---|
| 层链驱动 `t23_m3p.c` / `t23_chain.c` | 主仓 `tools/drivers/` |
| 接力脚本（跑层/打包/验包/准确度校验） | 主仓 `tools/relay/` |
| 引擎源码 `vllm_ntt.c` / `vllm_ckks.c` / `vllm_tp.c` | 主仓 `src/core/` |
| 数据预处理脚本 | 主仓 `tools/preproc/` |
| **链头归档 `L0-4`（158.75 MB / 105 文件）** | 本仓 https://github.com/m13253246268-ship-it/kestrel-fhe-relay/releases —— **只在 GitHub**（超 Gitee 附件 100 MB 上限），见 [`data/README.md`](data/README.md) |
| **链尾归档 `L26-27-fin`（64.99 MB / 46 文件）** | GitHub 同上；**Gitee 亦有** <https://gitee.com/pei-xiaoguang/fhe-relay/releases> 。**是尾部支线，绕过 5–25 层**，不要读成整链 |
| **预编译工具（Windows x86-64，0.53 MB / 7 文件）** | GitHub 同上；**Gitee 亦有** <https://gitee.com/pei-xiaoguang/fhe-relay/releases> 。`t23lay` / `t23boot` / `verify_layer`，静态链接，免装 MinGW。**注意：这三个二进制仍是 AGPL-3.0-or-later**，见 §10 |
| 运行数据包（约 7 GB，`sk.bin` + 权重 + 明文参考） | **不随仓库分发**；可在 Issue 下索取，或用主仓 `tools/preproc/` 自行转换 |

> 体积单位：本仓按惯例用 **MB 表示 MiB**（1 MB = 1,048,576 B，与 GitHub 的显示口径一致）；
> 三个 Release 资产的**精确字节数**见 [`data/README.md`](data/README.md) §1。

`sk.bin`（私钥）必须与权重/参考同源，**请勿混用**。

---

## 6. 判据（避免误判，务必先读）

日志里会同时出现段级 `FAIL` 和行尾 `RESULT=PASS` —— **这不是矛盾**：
段级 A/B/C/D 是拿「输入未偏离的明文参考」做对照，密文链上层间必然带偏离；
**层输出判据（`RESULT=PASS` / `BOOT=PASS`）才是判据**。

另外两点容易误判的地方：

1. **`RESULT=PASS` / `BOOT=PASS` 都不等于数值通过**。它们只说明流程走完、没有硬错误
   （`BOOT=PASS` 另说明确实刷回了满链 `out np=2083`）。接力脚本开了 `T23_E2EMODE=1`，
   会把中间量偏差降级成"仅报告"（日志里标 `[E2E:ref-deviate-ok]`），所以**浅层跑错 SiLU 路径也照样打 PASS**
   （实测：层 0 缺 `silu0.bin` 时 `max|err|` = 4.2e-2 ~ 8.5e-2，仍报 `RESULT=PASS (0)`）。
   **每层都必须用 `verify_layer.ps1` 独立看 `max|err|` 才算数**，见指南 §7。
2. **精度余量不大**。当前逐层 `max|err|` 最紧的一层（层 2）为 `2.8372e-02`，对 `3e-2` 容差
   余量仅约 5.4%。接手后请按指南记录每层 `max|err|` 趋势，出现退化及时回报。

### 段级 FAIL 但最终 PASS 的原因

跑起来后日志里会同时出现 `PASS` 和大量 `FAIL`，**这不是矛盾**。原因是两者量的不是同一件事：

| 环节 | 说明 |
|---|---|
| 段级为何 FAIL | L26/27 使用**真实 C 折叠**（`g_causal` + 实 C 折叠），中间量 `C/D/U2` 偏离真值**属预期**，不是缺陷 |
| 预期偏差的量级 | 规划脚本 `_cfold_plan.py` 事先量化：`\|Δao\| ~ 1.4 / 0.8`（与实测 `[C:attn] 8.5~12`、`[D:o] 14~29` 同量级） |
| 为何最终仍 PASS | 同一份规划给出**归一化判据**：`E2E logits drift 0.14 < margin 2.03`——中间偏差传到 logits 只剩 0.14，而 top1 的判决余量是 2.03 |
| 判定移交到哪 | 该模式下 `C/D/U2` **仅报告 err、不计入 `fails`**，验收移交 **E2E logits 门** |

**一句话**：段级 FAIL 是中间量的**口径差异**（可预期、已量化），最终 PASS 是**端到端 logits 门的判决**——两者不是同一把尺子，混着看就会得出"自相矛盾"的错觉。

依据出处：驱动源码 `tools/drivers/t23_m3p.c` 的 E2E 模式注释（约 1678-1680 行）。

---

## 7. 我们只主张这一件事

**前 5 层（层 0–4）在纯 CPU 上可复现、可独立验证**：逐层 `RESULT=PASS` / `BOOT=PASS`，
并提供交接物的 **SHA256 清单**供逐字节核对。

**链尾（层 26–27 + `fin`）也算得对**：从明文种子直接加密进 L26，五步全 `PASS`，
`fin` 的 logits top1 **4/4** 与参考一致。但这是**另一条支线**、**绕过 5–25 层**，
**不等于**「整条链从 0 层通到 27 层」。

「28 层全链」是**接力目标**，**不是**本项目已有结论。任何人接手后跑出的第 6 层及以后的结果，
属于**他的贡献**，不能倒填成本项目已有结论。

---

## 8. 明确的边界声明（请不要引用成别的意思）

1. **不宣称密码学安全强度**：当前参数是**机制验证级**，远低于 128-bit（需 `n ≥ 16384` 或等效参数）；
2. 随机数为原型 RNG，CSPRNG 接口已预留但未接入；
3. **权重是明文常量**，本链保护的是**输入与中间激活**；
4. **性能不是卖点**，单 token 级延迟数小时，只适合离线批处理；
5. 5 层以外的层号为**外推**，未实测；
6. 线程数是**结果变量**：`T23_NT=4` 与 `T23_NT=8` 产出的密文**位级不一致**（随机数按调度偏移）。
   接力请**统一使用 `T23_NT=4`**，否则交接物无法逐字节比对。

---

## 9. 文档索引

| 文档 | 内容 |
|---|---|
| [`SOURCE.md`](SOURCE.md) | **代码版本锚点**：源码 SHA256、位级一致性约束、为什么本仓不复制源码 |
| [`data/README.md`](data/README.md) | 三份 Release 资产（链头 `L0-4` / 链尾 `L26-27-fin` / 工具 `tools-win-x64`）的说明与校验流程 |
| [`docs/00_已完成层状态说明.md`](docs/00_已完成层状态说明.md) | 已跑完层逐项状态（含编码陷阱教训） |
| [`docs/02_同态加密推理接力操作手册.md`](docs/02_同态加密推理接力操作手册.md) | 工具逐个参数与操作步骤 |
| [`docs/03_实测数据附录.md`](docs/03_实测数据附录.md) | **全部原始实测数据**（逐跳墙钟、分段 profile、`max|err|`、80 个密文 SHA256） |
| [`docs/04_接力复现指南.md`](docs/04_接力复现指南.md) | 从零到跑通一层的完整复现步骤 |
| [`docs/05_工具说明.md`](docs/05_工具说明.md) | 工具清单与逐参数用法 |

英文版（以上每份都有对应）：[`SOURCE.en.md`](SOURCE.en.md)、[`data/README.en.md`](data/README.en.md)、
[`docs/00_Status_of_Completed_Layers_EN.md`](docs/00_Status_of_Completed_Layers_EN.md)、
[`docs/02_Relay_Operations_Manual_EN.md`](docs/02_Relay_Operations_Manual_EN.md)、
[`docs/03_Measured_Data_Appendix_EN.md`](docs/03_Measured_Data_Appendix_EN.md)、
[`docs/04_Relay_Reproduction_Guide_EN.md`](docs/04_Relay_Reproduction_Guide_EN.md)、
[`docs/05_Tool_Reference_EN.md`](docs/05_Tool_Reference_EN.md)。

> 目录编号在 `01` 与 `06` 处留空：这两份是**征集/发布用的帖子材料**（讨论整理、五平台接力帖），
> 属于运营材料而非技术交付物，**未包含在本仓**。

---

## 10. 联系与许可

**联系**：请优先在 **Issues** 区留言（认领层号 / 索取数据 / 报告问题）。

**许可（本仓）**：**MIT** —— 本仓的文档、索引、清单与两份交接物归档可自由使用、复制、修改、分发、再许可与销售，只需保留版权与许可声明，见 [`LICENSE`](LICENSE)。

**两处例外，请勿误读**（详见 [`LICENSING.md`](LICENSING.md)）：

1. `releases/` 中 `kestrel-fhe-relay_tools_win-x64_*.zip` 里的 **`t23lay.exe` / `t23boot.exe` / `verify_layer.exe`** 是用**主仓 AGPL-3.0-or-later 源码**编译出来的，**仍受 AGPL-3.0-or-later 约束**，不受本仓 MIT 的重新授权（对应源码见 [`SOURCE.md`](SOURCE.md) 锚定的 commit）。
2. **主仓 `kestrel-llm` 的代码**仍按其自身许可发布（AGPL-3.0-or-later / 商业授权双许可）；本仓的 MIT **不改变、也不覆盖**它。
