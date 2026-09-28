[English](README.en.md) | **简体中文**

# 接力交接物归档 · Release 资产（三份）

本页说明本仓 **三份 Release 资产**的内容与校验方式。归档本体不放进 Git
（密文既压不动、也没必要进版本历史），而是作为本仓的 **Release 资产**分发；
打好的 zip 暂存在本地 `releases/`（该目录被 `.gitignore` 排除，不进版本历史）。

> **前两份是两条互不衔接的支线，不要读成一条链。**
> `L0-4` 是**链头**：从 0 层跑到 4 层，接力断点停在 `u4r112`；
> `L26-27-fin` 是**链尾**：从明文种子直接加密进 L26，**绕过 5–25 层**，跑到输出头 `fin`。
>
> **第三份是工具，不是结果**：预编译的 Windows x86-64 可执行文件，**跑前两份数据要用它**。
> 它不含任何密文，也不含数据包。

---

## 1. 资产总览

| 资产 | 文件名 | 大小 | 条目 | 解压根目录 |
|---|---|---|---|---|
| **链头 L0–4** | `kestrel-fhe-relay_L0-4_20260915.zip` | **158.75 MB**（下载体积；166,463,181 B；解压后 160.06 MB） | **105** | `L0-4/` |
| **链尾 L26–27 + `fin`** | `kestrel-fhe-relay_L26-27-fin_20260915.zip` | **64.99 MB**（下载体积；68,151,062 B；解压后 65.53 MB） | **46** | `L26-27-fin/` |
| **工具（Windows x86-64）** | `kestrel-fhe-relay_tools_win-x64_20260915.zip` | **0.53 MB**（559,724 B） | **7** | `tools-win-x64/` |

> **体积单位**：本页按惯例用 **MB 表示 MiB**（1 MB = 1,048,576 B，与 GitHub 的显示口径一致）；
> 三个资产都另附**精确字节数**，核对时以字节数为准。

ZIP 自身的 SHA256：

| 文件 | SHA256 |
|---|---|
| `kestrel-fhe-relay_L0-4_20260915.zip` | `b3a6c7af273c664854c2bdf8af4a045776cc5c9f204db90c9b2981de719bcbf7` |
| `kestrel-fhe-relay_L26-27-fin_20260915.zip` | `26d8a9f09741c507e8bd0e35810e18062775071dded5bdc0653aca2122c58402` |
| `kestrel-fhe-relay_tools_win-x64_20260915.zip` | `fa5bb5a349f777d9c30e2fc84edddfb43321dbe4670982b496b2816a3d31bd99` |

> 请先核对 ZIP 自身的 SHA256，再解压。哈希对不上就不要用。

**为什么三份都走 Releases**：GitHub 对**单文件**有 100 MB 硬限，`L0-4` 包 158.75 MB
**无法提交进仓库**；工具包只有 0.53 MB，但三份放同一处对外，取用最省事。

> ⚠️ **从 Gitee 镜像取用时注意**：Gitee 的限制作用在**附件**上（100 MB/文件），比 GitHub 紧。
> 三份里**只有链尾 `L26-27-fin` 与工具包**能上 Gitee，**链头 `L0-4` 只剩 GitHub 一途**。取舍与依据见 **§5.1**。

---

## 2. 包内是什么

```
L0-4/                                ← 链头：层 0–4（5 层）
├── chain/
│   ├── u{L}_{t}_{h}.ct        5 层 × 8 = 40 个   lay 输出（np = 16/20/15/14/15）
│   └── u{L}r112_{t}_{h}.ct    5 层 × 8 = 40 个   boot 刷新输出（np=112）← 交接物
├── logs/                      20 个：lay{L}.out/.err、boot{L}.out/.err（判据证据）
├── meta.txt                   机器 / 编译器 / 线程元数据
├── master.log                 10 跳时间线（逐跳 PASS/耗时）
├── timing.tsv                 逐层耗时汇总
├── manifest.sha256            包内全部文件的 SHA256（104 行，不含自身）
└── README.md                  归档详细说明：结果表 / 验证方法 / 边界声明

L26-27-fin/                          ← 链尾：层 26–27 + 输出头
├── chain/
│   ├── u26_{t}_{h}.ct / u27_{t}_{h}.ct         各 8 个   lay 输出（np=3）
│   └── u26r112 / u27r112_{t}_{h}.ct            各 8 个   boot 刷新输出（np=112）← 交接物
├── logs/                      12 个：t26 / boot26 / t27 / boot27 / fin 的 .out/.err
│                              ＋ master.log（五步时间线）＋ meta.txt
├── manifest.sha256            包内全部文件的 SHA256（45 行，不含自身）
└── README.md                  尾部归档说明：逐跳结果 / logits 与 top1 / 边界声明
```

> **两份包的顶层布局不同**：`L0-4` 把 `master.log` / `meta.txt` / `timing.tsv` 放在**包根**；
> `L26-27-fin` 把 `master.log` / `meta.txt` 放在 **`logs/` 里**（其包根只有 `chain/`、`logs/`、
> `manifest.sha256`、`README.md`）。上面的树按各自实际布局绘制，不是笔误。

约定：`t=0..3`（4 个 token）、`h=0..1`（2 个密文分量）；一层 = `lay`（前向）+ `boot`（自举刷新）两跳。

**链头的下一棒从 `u4r112` 起接 `lay5`。** 链尾不是从链头续接来的，见 §4.3。

---

## 3. 校验（从弱到强）

### ⓪ 先拿工具（不想自己装 gcc 的话）

工具包的用法见其包内 `README.md`。就位两步：

```powershell
New-Item -ItemType Directory -Force <仓库根>\.tmp_tok | Out-Null
Copy-Item tools-win-x64\*.exe <仓库根>\.tmp_tok\     # 然后所有命令都在【仓库根】执行
.tmp_tok\verify_layer.exe -L 4 -Ct u4                 # 冒烟：期望 VERIFY_PASS (8/8) max_err=1.3823e-02
```

自带 gcc 的话跳过这步，按主仓 `tools/relay/README.md` §2 自己编（**要位级一致就得照它的编译口径**）。

### ① 核完整性（秒级）

```powershell
# 1) 先核 ZIP 自身
(Get-FileHash kestrel-fhe-relay_L0-4_20260915.zip -Algorithm SHA256).Hash.ToLower()
# 期望 b3a6c7af273c664854c2bdf8af4a045776cc5c9f204db90c9b2981de719bcbf7
(Get-FileHash kestrel-fhe-relay_L26-27-fin_20260915.zip -Algorithm SHA256).Hash.ToLower()
# 期望 26d8a9f09741c507e8bd0e35810e18062775071dded5bdc0653aca2122c58402

# 2) 解压后逐行核包内文件（两个包同样做法）
Expand-Archive kestrel-fhe-relay_L0-4_20260915.zip -Destination .
Set-Location L0-4
Get-Content manifest.sha256 | ForEach-Object {
  if ($_ -match '^([0-9a-fA-F]{64})\s+(.+?)\s*$') {
    $want = $Matches[1].ToLower(); $rel = $Matches[2]
    $got = (Get-FileHash $rel -Algorithm SHA256).Hash.ToLower()
    if ($got -ne $want) { "MISMATCH $rel" }
  }
}
# 期望：无输出（L0-4 → 104/104 全匹配；L26-27-fin → 45/45 全匹配）
```

> **Linux 用户注意**：`manifest.sha256` 里的路径用的是**反斜杠**（`chain\u0_0_0.ct`）。
> 这是为了与接力工具链一致（`tools/relay/verify_relay.ps1` 的防夹带反查依赖这一口径），
> 但 `sha256sum -c` 不认反斜杠。先转成斜杠再校验：
> ```bash
> sed 's|\\|/|g' manifest.sha256 > manifest.posix.sha256 && sha256sum -c manifest.posix.sha256
> ```

### ② 判"算得对不对"（准确度校验）

需要**完整数据包**里的明文参考与私钥 `.tmp_tok/chain/sk.bin`
（数据包不随仓库分发，见本仓 README §5）。工具来自主仓 `tools/relay/`：

```powershell
# 链头 L0–4
powershell -NoProfile -ExecutionPolicy Bypass -File <主仓>\tools\relay\verify_layer.ps1 `
    -From 0 -To 4 -ChainDir <解压目录>\L0-4\chain -Sk .tmp_tok\chain\sk.bin

# 链尾 L26 / L27（boot 输出；参考 tail\l26_u2_ref.bin、tail\l27_u2_ref.bin）
powershell -NoProfile -ExecutionPolicy Bypass -File <主仓>\tools\relay\verify_layer.ps1 `
    -From 26 -To 26 -Mode boot -ChainDir <解压目录>\L26-27-fin\chain -Sk .tmp_tok\chain\sk.bin
.tmp_tok\verify_layer.exe -L 27 -Ct u27r112 -Dir <解压目录>\L26-27-fin\chain -Tol 3e-2
```

`verify_layer` 是**独立于驱动判据的代码路径**（自行解密 / 解码 / 对照明文参考），所以它能作为第三方复核。
期望：逐层 `VERIFY_PASS`，`max|err|` 见各包内 `README.md`。

**链尾要额外看输出头**：`fin` 的 `[F:logits t?]` 四行 top1，见包内 `README.md` §5.2。

### ③ 最强：位级交叉复算

换台机器（不同 ISA 更好）用同样输入重跑同一层，比对密文：

```powershell
.tmp_tok\verify_layer.exe -BitA u0 -BitB u0_independent -L 0     # 链头
.tmp_tok\verify_layer.exe -BitA u26 -BitB u26_independent -L 26  # 链尾（重跑 t26）
```

确定性流程下相同输入 → 相同密文。两方独立跑出的 SHA256 若一致，该层结果就**不依赖任何一方的说法**。
前提是**线程数一致**（见 §4.1）。

---

## 4. 已知事实与边界（不要误读）

### 4.1 两份资产共同适用

1. **`manifest.sha256` 是重新生成的**（两份都是）。归档 `README.md` 在归档后被修订（把本地工作区路径
   改成发布口径、并补上判据说明），清单随之重算，因此现在是**全匹配**（`L0-4` 104/104、`L26-27-fin` 45/45）；
   **除 `README.md` 外的其余文件与归档时生成的原清单逐字节相同**，密文本身**未做任何改动**。
2. **`BOOT=PASS` 不等于数值通过**。它只说明刷新流程走完并刷回满链 `out np=2083`；数值要看
   `verify_layer` 的 `max|err|`。
3. **线程数是结果变量**：`T23_NT=4` 与 `T23_NT=8` 的密文**位级不一致**（自举掩码随线程的随机数消费
   历史变化），两份归档均固定 **4 线程**。你要做位级比对，必须同样用 4 线程。

### 4.2 链头 `L0-4` 专属

1. **误差余量不宽松**：层 2 的 `max|err| = 2.8372e-02`，对 `3e-2` 容差只剩约 **5.4%**。
2. 本归档只覆盖**层 0–4**。「28 层全链」是接力目标，**不是**已有结论。

### 4.3 链尾 `L26-27-fin` 专属

1. **是尾部支线，不是整链**。它从明文种子 `tail/l26_u1.bin` 直接加密进 L26，**绕过 5–25 层**；
   接力链仍停在 `u4r112`。它证明的是「**尾部两层 + 输出头算得对**」，**不是**「整条链从 0 层通到 27 层」。
2. **`t27` 的层输出有 2/8 超容差**（最大 `7.969e-02`，超 `3e-2` 容差 2.66 倍）。它没有让最终 top1
   出错，但**不代表精度健康**。
3. **早期（2026-09-03，96 素数链）也跑过末层 + `fin` 并出过 top1 基线，但那份结果不作数**——
   它基于因果掩码 bug 修复前的参考系，与本文档的参考**不是同一代**，不可与本文结果互相印证。

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

**注意：`fin` 阶段的门与上面不是同一套。** `C/D/U2` 由 `T23_E2EMODE` 控制（不设该变量时计入 `fails`），
而 `fin`（phase 5）的 `[F:xnorm]` / `[F:top1]` 走的是**硬编码 `&rf`**——无论是否设置 `T23_E2EMODE`，
都不计入 `fails`。**两套门不要混着读。**

实测印证：链尾 `fin` 的日志是 `RESULT=PASS (0; ref-deviate 8)`——`rf = 8` 恰好等于 8 条 `[F:xnorm]`
（2 分量 × 4 token），即 **top1 不匹配次数为 0**；四行 `[F:logits t?]` 的 top1 与参考完全一致
（均为 `120555`）。细节见 `L26-27-fin/README.md` §5。

### 4.4 工具包专属

1. **只覆盖 x86-64 Windows（PE）**。Linux / macOS 不能直接跑，aarch64（RK3588）也不能；
   非 Windows 平台按主仓 `tools/relay/README.md` §2 自行编译（引擎只依赖 `libc + libm`）。
2. **`-static` 只是打包方式，不改变数值**；但**换了编译器版本**或**去掉 `-O2`** 后，
   是否仍与归档位级一致**不能默认成立**——先按 §3 的读路径对一遍（工具包 `README.md` §5 有复算记录）。
3. 工具包**不含数据**：权重、明文参考、`sk.bin`（约 7 GB）不随任何一份资产分发。

---

## 5. 上传 Release 时的建议

| 项 | 链头 | 链尾 | 工具 |
|---|---|---|---|
| Tag | `L0-4`（或 `relay-L0-4`） | `L26-27-fin`（或 `relay-tail`） | `tools-win-x64`（或并入上面任一 Tag） |
| 标题 | `Layer 0–4 hand-off artefacts (L0–L4)` | `Tail layers 26–27 + fin (ciphertext logits)` | `Prebuilt x86-64 toolchain (Windows)` |
| 附件 | `kestrel-fhe-relay_L0-4_20260915.zip`（158.75 MB） | `kestrel-fhe-relay_L26-27-fin_20260915.zip`（64.99 MB） | `kestrel-fhe-relay_tools_win-x64_20260915.zip`（0.53 MB） |
| 正文 | 贴上对应 ZIP 的 SHA256，与 §1 保持一致 | 同左；**并注明"尾部支线、绕过 5–25 层"** | 同左；**并注明"只覆盖 x86-64 Windows、固定 4 线程口径"** |

三个附件各自都在 GitHub 单资产 2 GB 上限内，但见 §1：**单文件 100 MB 的仓库硬限只约束"提交进仓库"，
不约束 Release 附件**。工具包只有 0.53 MB，其实也可以直接提交进仓库——之所以仍然走 Releases，
是为了让三份资产在同一处取用，且**源码单点保留在主仓**这条约束不被打折。

### 5.1 Gitee 镜像：只有两份能上去（取舍与依据）

本仓在 <https://gitee.com/pei-xiaoguang/fhe-relay> 有镜像。Gitee 的配额与 GitHub **不同**，
按 [Gitee 产品配额说明](https://gitee.com/help/articles/4283)（社区版 / 个人用户）：

> 附件容量：**附件单文件大小上限为 100 MB**；单仓库附件总容量 1 GB。

这条是**附件**上限（不是"提交进仓库"的上限），所以与 GitHub 的可传范围不一样：

| 资产 | 大小 | GitHub Releases | Gitee Releases |
|---|---|---|---|
| 链头 `L0-4` | 158.75 MB | ✅（2 GB/资产） | ❌ **超 100 MB 附件上限** |
| 链尾 `L26-27-fin` | 64.99 MB | ✅ | ✅ |
| 工具 `tools_win-x64` | 0.53 MB | ✅ | ✅ |

**为什么不把链头拆成两卷传上去**：本仓的验证流程是"先核**这一个 zip** 的 SHA256，再解压"（§1 表、
指南附录 B 都以单一 zip 的 SHA256 为锚）。拆卷会改变哈希，必须把文档改成"逐卷哈希 + 合并步骤"，
把一个 6 步流程变成 8 步、并新增一个"合并对不对"的失败面——**为省一次跨站下载而牺牲可验证性，不划算**。

**代价，必须写明**：Gitee 读者**拿不到链头 `L0-4`**，而接力下一棒的输入 `u4r112` 正在那份包里。
所以**要在 Gitee 上真正接一棒，仍须从 GitHub 取链头**。链尾与工具在 Gitee 可取，仅能省一部分流量。
（链头 zip 的 SHA256 仍以 §1 表为准，与获取渠道无关——从哪个站下都要对得上。）
