[简体中文](SOURCE.md) | **English**

# Source version anchor (SOURCE)

This repository **does not copy any source code**. The relay drivers, the engine core and the scripts all come
from the main repository `kestrel-llm`; this file pins down *which version* to use.

---

## 1. Main repository and version anchor

| Remote | URL | Note |
|---|---|---|
| **Gitee** | https://gitee.com/pei-xiaoguang/kestrel-llm | **measured baseline for this anchor** (reachable, re-checked 2026-09-16) |
| GitHub | https://github.com/m13253246268-ship-it/kestrel-llm | facade/mirror — see the note below |

**Use this commit for the relay** (`master`, 2026-09-16):

```
4209982fb75bf8cda9ab452b8a40a7a24ea36fc7
```

How it was checked, with the actual output (2026-09-16, `origin` = Gitee):

```
$ git ls-remote --heads origin master
4209982fb75bf8cda9ab452b8a40a7a24ea36fc7        refs/heads/master
```

> ✅ **Gitee's `master` is exactly this commit**, and it already contains everything the relay needs:
> the full `tools/relay/` script set, `tools/preproc/` (including `_embed4.py` and `_silu_mode.py`),
> `tools/drivers/` and `src/core/`.
> Measured: exporting the six files of §3 from this commit yields hashes and byte counts that **match the
> table exactly**.
>
> ⚠️ **The GitHub side could not be measured**: GitHub's SSH endpoint was unreachable at check time (the
> connection hung). The locally cached `github/master` points at
> `2ed2e08b630154d9b2d2fff274ac5d2e1dfb8599` (whose own message says it "merges into the GitHub facade
> branch"), which is **not an ancestor and not a descendant** of this anchor (common ancestor
> `4f1ad70400205b8af112e5cd3c522a4d30d87df7`) — i.e. GitHub carries a **divergent facade line**, and the
> cache may be stale. **When cloning from GitHub, verify yourself that this commit exists.**
>
> 🔒 **Whichever remote you clone from, the per-file SHA256 values in §3 are the only real basis**
> (the commit is merely a locator).

Engine version: `CMakeLists.txt` → `project(vllm_kestrel VERSION 1.0.0 LANGUAGES C)`

---

## 2. Why this repository does not copy the sources

Bit-level consistency is the single hard constraint of this chain: a hand-off artefact must be
**byte-comparable by a third party** (SHA256). If this repository carried a second copy of the sources,
the two copies could drift — and even a difference as small as a changed optimisation order, which changes
the order in which random numbers are consumed, would make the resulting ciphertexts bit-inconsistent.
At that point the whole relay verification becomes meaningless.

Therefore: **the source lives in exactly one place, the main repository; this repository only pins the version.**

---

## 3. Version anchor (per-file SHA256)

The relay **must** use the versions below. After cloning the main repository, run this check before you start computing:

> ⚠️ **The hashes are computed over the `LF` form** — i.e. the exact bytes of the blob stored in the
> repository (cross-check the byte count with `git cat-file -s <rev>:<path>`). On Windows a checkout with
> `core.autocrlf=true` is `CRLF`, so **a plain `Get-FileHash` gives a completely different value** — that does
> not mean the file was modified. The script below therefore **normalises CRLF to LF first**, so it yields the
> same result on any OS and with any `autocrlf` setting.

| File | Role | Bytes (LF) | SHA256 (LF form) |
|---|---|---|---|
| `src/core/vllm_ntt.c` | NTT / RNS core | 20655 | `0d95e49a3a8bc3782fc6dcf65cce3dfc4801cb4e42fd461aa8963840fde371c1` |
| `src/core/vllm_ckks.c` | CKKS encrypt/decrypt / bootstrap | 71105 | `fa275ede9e3dad4bff25b7ef7dea92225cf2afcdedb63f02abcc1b4fca08ea1a` |
| `src/core/vllm_tp.c` | in-house thread pool | 18695 | `18ff77ed5fbfc8208552b6cb58508e254b15776fe33b8c3125dfadb846df813c` |
| `tools/drivers/t23_m3p.c` | `lay` (in-layer forward) driver | 114668 | `e849103140d6c962c3f7c8c37df62015bbbea4bc6ec88ea3ff2986297b390fa3` |
| `tools/drivers/t23_chain.c` | `boot` (bootstrap refresh) driver | 89483 | `543ba8cb64160cbef5018af0c0c697abe1d8cb57dd39a0f4c21d973a48bb7c8b` |
| `tools/relay/verify_layer.c` | independent numerical re-check (does not rely on the driver's verdict) | 8274 | `e762f296d604a303b5538e3f9551151758be1711b507d47bfde8d86d35a5b24a` |

Check (PowerShell; identical result on any platform and with any `autocrlf`):

```powershell
$files = "src/core/vllm_ntt.c","src/core/vllm_ckks.c","src/core/vllm_tp.c",
         "tools/drivers/t23_m3p.c","tools/drivers/t23_chain.c","tools/relay/verify_layer.c"
foreach ($f in $files) {
  $p = (Get-Item $f).FullName                        # resolve the relative path against PowerShell's location
  $s = [Text.Encoding]::UTF8.GetString([IO.File]::ReadAllBytes($p))
  $s = $s.Replace("`r`n", "`n")                      # normalise CRLF -> LF
  $b = [Text.Encoding]::UTF8.GetBytes($s)
  $h = [BitConverter]::ToString([Security.Cryptography.SHA256]::Create().ComputeHash($b)) -replace '-',''
  "{0}  {1}  {2}" -f $h.ToLower(), $b.Length, $f
}
```

Linux / macOS (the worktree is already LF, so hash it directly):

```bash
sha256sum src/core/vllm_ntt.c src/core/vllm_ckks.c src/core/vllm_tp.c \
          tools/drivers/t23_m3p.c tools/drivers/t23_chain.c tools/relay/verify_layer.c
```

**If any hash does not match, stop** — post the hash you computed in an issue, **and state your
`core.autocrlf` setting**, instead of starting to compute.

---

## 4. Relay-related directories (all in the main repository)

| Path | Content |
|---|---|
| `src/core/vllm_ntt.c` / `vllm_ckks.c` / `vllm_tp.c` | engine core (linked at compile time) |
| `tools/drivers/t23_m3p.c` | `lay` driver (`CKKS_NPRIMES=112`) |
| `tools/drivers/t23_chain.c` | `boot` driver (`CKKS_NPRIMES=2100`, large stack required) |
| `tools/relay/` | run / pack / verify / numerical re-check scripts, see its `README.md` |
| `tools/preproc/` | data-preprocessing scripts (use these if you want to build the ~7 GB data package yourself) |

Build commands and per-parameter reference: main repository `tools/relay/README.md` §2, or this repository's
[`docs/02_Relay_Operations_Manual_EN.md`](docs/02_Relay_Operations_Manual_EN.md).

---

## 5. Known operational constraints you must respect

1. **`T23_NT=4`**: the RNG in `vllm_ckks.c` is `_Thread_local`; changing the thread count makes the
   ciphertexts bit-inconsistent. The relay uses 4 threads uniformly — otherwise hand-off artefacts cannot
   be compared byte-by-byte.
2. **The data path is hard-coded to `.tmp_tok/`**: the engine has that working directory baked in, and the
   scripts locate the repository root themselves.
3. **The encoding of a `.ps1` file is part of the deliverable**: a script containing Chinese must be saved as
   **UTF-8 with BOM, and with exactly one BOM**. A double BOM makes PowerShell 5.1 fail to parse it, while the
   log file still gets content (the content being the error) — so it *looks* as if it ran.
4. **`verify_layer.ps1 -TargetLayer` must be ≥ 1**: `lay0` takes the plaintext-seed encryption branch and does
   not read any on-chain ciphertext, so passing 0 gives you an idle run (the script now hard-guards this and
   exits with 2 instead).
