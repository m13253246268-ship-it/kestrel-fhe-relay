[English](SOURCE.en.md) | **简体中文**

# 代码版本锚点（SOURCE）

本仓**不复制任何源码**。接力用的驱动、引擎内核与脚本全部来自主仓 `kestrel-llm`，
本文件负责把「用哪个版本」钉死。

---

## 1. 主仓与版本锚点

| 远端 | 地址 | 备注 |
|---|---|---|
| **Gitee** | https://gitee.com/pei-xiaoguang/kestrel-llm | **本锚点的实测基准**（2026-09-16 复核可达） |
| GitHub | https://github.com/m13253246268-ship-it/kestrel-llm | 门面/镜像，状态见下方说明 |

**接力请使用这个 commit**（`master`，2026-09-16）：

```
4209982fb75bf8cda9ab452b8a40a7a24ea36fc7
```

核对方式与实测输出（2026-09-16，`origin` = Gitee）：

```
$ git ls-remote --heads origin master
4209982fb75bf8cda9ab452b8a40a7a24ea36fc7        refs/heads/master
```

> ✅ **Gitee 的 `master` 就是这个 commit**，且它已含接力所需的全部文件：`tools/relay/` 全套脚本、
> `tools/preproc/`（含 `_embed4.py` 与 `_silu_mode.py`）、`tools/drivers/`、`src/core/`。
> 已实测：从该 commit 导出 §3 的 6 个文件，逐项哈希与字节数**与表格完全一致**。
>
> ⚠️ **GitHub 侧未能实测**：复核时 GitHub 的 SSH 端不可达（连接挂起）。本机缓存的 `github/master`
> 指向 `2ed2e08b630154d9b2d2fff274ac5d2e1dfb8599`（该提交自述"合并到 GitHub 门面分支"），
> 与本锚点**互不为祖先**（共同祖先 `4f1ad70400205b8af112e5cd3c522a4d30d87df7`）——即 GitHub 上是一条
> **分叉的门面线**，且缓存可能已过期。
> **从 GitHub 克隆时请自行确认该 commit 是否存在。**
>
> 🔒 **无论从哪一端克隆，§3 的逐文件 SHA256 才是唯一实际依据**（commit 只用来定位）。

引擎版本：`CMakeLists.txt` → `project(vllm_kestrel VERSION 1.0.0 LANGUAGES C)`

---

## 2. 为什么本仓不复制源码

位级一致性是本链路唯一的硬约束：交接物要能被第三方**逐字节比对**（SHA256）。
如果本仓放一份源码副本，两份源码一旦漂移——哪怕只是优化顺序不同导致随机数消耗顺序变化——
产出的密文就会位级不一致，整个接力校验失去意义。

所以：**源码单点保留在主仓，本仓只钉版本。**

---

## 3. 版本锚点（逐文件 SHA256）

接力**必须**使用下列版本。克隆主仓后，先跑一遍校验再开始算：

> ⚠️ **哈希按 `LF` 形态计算**——也就是仓库里 blob 本身的字节（可用 `git cat-file -s <rev>:<路径>` 核对字节数）。
> Windows 上 `core.autocrlf=true` 检出的工作区是 `CRLF`，**直接 `Get-FileHash` 会得到完全不同的值**，
> 那不是文件被改过。所以下面的脚本**先把 CRLF 归一化成 LF 再算**，与操作系统、`autocrlf` 设置无关。

| 文件 | 作用 | 字节（LF） | SHA256（LF 形态） |
|---|---|---|---|
| `src/core/vllm_ntt.c` | NTT / RNS 内核 | 20655 | `0d95e49a3a8bc3782fc6dcf65cce3dfc4801cb4e42fd461aa8963840fde371c1` |
| `src/core/vllm_ckks.c` | CKKS 加解密 / 自举 | 71105 | `fa275ede9e3dad4bff25b7ef7dea92225cf2afcdedb63f02abcc1b4fca08ea1a` |
| `src/core/vllm_tp.c` | 自研线程池 | 18695 | `18ff77ed5fbfc8208552b6cb58508e254b15776fe33b8c3125dfadb846df813c` |
| `tools/drivers/t23_m3p.c` | `lay` 层内前向驱动 | 114668 | `e849103140d6c962c3f7c8c37df62015bbbea4bc6ec88ea3ff2986297b390fa3` |
| `tools/drivers/t23_chain.c` | `boot` 自举刷新驱动 | 89483 | `543ba8cb64160cbef5018af0c0c697abe1d8cb57dd39a0f4c21d973a48bb7c8b` |
| `tools/relay/verify_layer.c` | 独立数值复核（不依赖驱动判据） | 8274 | `e762f296d604a303b5538e3f9551151758be1711b507d47bfde8d86d35a5b24a` |

校验（PowerShell，任意平台/任意 `autocrlf` 都得到同一结果）：

```powershell
$files = "src/core/vllm_ntt.c","src/core/vllm_ckks.c","src/core/vllm_tp.c",
         "tools/drivers/t23_m3p.c","tools/drivers/t23_chain.c","tools/relay/verify_layer.c"
foreach ($f in $files) {
  $p = (Get-Item $f).FullName                        # 相对路径按 PowerShell 当前位置解析
  $s = [Text.Encoding]::UTF8.GetString([IO.File]::ReadAllBytes($p))
  $s = $s.Replace("`r`n", "`n")                      # CRLF -> LF 归一化
  $b = [Text.Encoding]::UTF8.GetBytes($s)
  $h = [BitConverter]::ToString([Security.Cryptography.SHA256]::Create().ComputeHash($b)) -replace '-',''
  "{0}  {1}  {2}" -f $h.ToLower(), $b.Length, $f
}
```

Linux / macOS（工作区本来就是 LF，可直接算）：

```bash
sha256sum src/core/vllm_ntt.c src/core/vllm_ckks.c src/core/vllm_tp.c \
          tools/drivers/t23_m3p.c tools/drivers/t23_chain.c tools/relay/verify_layer.c
```

**任何一个对不上，先停下来**——在 Issue 里贴出你算到的哈希，并**注明你的 `core.autocrlf` 设置**，不要直接开跑。

---

## 4. 接力相关的目录（都在主仓）

| 路径 | 内容 |
|---|---|
| `src/core/vllm_ntt.c` / `vllm_ckks.c` / `vllm_tp.c` | 引擎内核（编译时链接） |
| `tools/drivers/t23_m3p.c` | `lay` 驱动（`CKKS_NPRIMES=112`） |
| `tools/drivers/t23_chain.c` | `boot` 驱动（`CKKS_NPRIMES=2100`，需大栈） |
| `tools/relay/` | 跑层 / 打包 / 验包 / 数值复核脚本，见其 `README.md` |
| `tools/preproc/` | 数据预处理脚本（自行生成约 7 GB 数据包时用） |

编译命令与逐参数说明：主仓 `tools/relay/README.md` §2，或本仓
[`docs/02_同态加密推理接力操作手册.md`](docs/02_同态加密推理接力操作手册.md)。

---

## 5. 已知的、必须遵守的运行约束

1. **`T23_NT=4`**：`vllm_ckks.c` 的 RNG 在 `_Thread_local` 上，线程数变化会让密文位级不一致。
   接力统一 4 线程，否则交接物无法逐字节比对。
2. **数据路径硬编码在 `.tmp_tok/`**：引擎把工作目录写死在那里，脚本内部会自行定位仓库根。
3. **`.ps1` 的编码是交付物的一部分**：含中文的脚本必须存为 **UTF-8 with BOM，且只能有一个 BOM**。
   双 BOM 会让 PowerShell 5.1 解析失败，而日志文件里仍有内容（内容是报错），看起来「像跑过了」。
4. **`verify_layer.ps1 -TargetLayer` 必须 ≥1**：`lay0` 走明文种子加密分支，不读任何链上密文，
   传 0 会得到一次空转（脚本已加硬保护，传 0 直接 `exit 2`）。
