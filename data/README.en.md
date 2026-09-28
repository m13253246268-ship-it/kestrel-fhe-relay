[简体中文](README.md) | **English**

# Hand-off archive · Release assets (three)

This page describes the contents of this repository's **three Release assets** and how to verify them.
The archives themselves are not stored in Git (ciphertext neither compresses nor belongs in version
history); they are distributed as **Release assets**. The built zips are staged locally under `releases/`
(that directory is excluded by `.gitignore` and does not enter version history).

> **The first two are two branches that do not connect — do not read them as one chain.**
> `L0-4` is the **head**: layers 0 through 4, with the relay hand-off stopping at `u4r112`;
> `L26-27-fin` is the **tail**: encrypted directly from a plaintext seed into L26, **bypassing layers 5–25**,
> running through to the output head `fin`.
>
> **The third one is a tool, not a result**: prebuilt Windows x86-64 executables, **needed to run the first
> two**. It contains no ciphertext and no data package.

---

## 1. Asset overview

| Asset | Filename | Size | Entries | Root after extraction |
|---|---|---|---|---|
| **Head L0–4** | `kestrel-fhe-relay_L0-4_20260915.zip` | **158.75 MB** (as downloaded; 166,463,181 B; 160.06 MB extracted) | **105** | `L0-4/` |
| **Tail L26–27 + `fin`** | `kestrel-fhe-relay_L26-27-fin_20260915.zip` | **64.99 MB** (as downloaded; 68,151,062 B; 65.53 MB extracted) | **46** | `L26-27-fin/` |
| **Tools (Windows x86-64)** | `kestrel-fhe-relay_tools_win-x64_20260915.zip` | **0.53 MB** (559,724 B) | **7** | `tools-win-x64/` |

> **Size units**: this page follows the common convention of writing **MB for MiB** (1 MB = 1,048,576 B,
> matching how GitHub displays sizes); each of the three assets also carries its **exact byte count**, which
> is what you should check against.

SHA256 of the ZIPs themselves:

| File | SHA256 |
|---|---|
| `kestrel-fhe-relay_L0-4_20260915.zip` | `b3a6c7af273c664854c2bdf8af4a045776cc5c9f204db90c9b2981de719bcbf7` |
| `kestrel-fhe-relay_L26-27-fin_20260915.zip` | `26d8a9f09741c507e8bd0e35810e18062775071dded5bdc0653aca2122c58402` |
| `kestrel-fhe-relay_tools_win-x64_20260915.zip` | `fa5bb5a349f777d9c30e2fc84edddfb43321dbe4670982b496b2816a3d31bd99` |

> Verify the ZIP's own SHA256 first, then extract. If the hash does not match, do not use it.

**Why all three go through Releases**: GitHub enforces a hard **100 MB limit per file**, so the `L0-4`
package (158.75 MB) **cannot be committed to the repository**. The tools package is only 0.53 MB, but
publishing all three in one place is simply the least effort for anyone picking them up.

> ⚠️ **If you take these from the Gitee mirror instead**: Gitee's limit applies to **attachments**
> (100 MB per file) and is therefore tighter than GitHub's. Only the **tail `L26-27-fin` and the tools
> package** can be hosted there; the **head `L0-4` is GitHub-only** — but that copy is merely a way to
> **save about 9 hours**, because **the head can be reproduced locally** (see §5.1). Rationale in **§5.1**.

---

## 2. What is inside

```
L0-4/                                <- head: layers 0-4 (5 layers)
├── chain/
│   ├── u{L}_{t}_{h}.ct        5 layers × 8 = 40   lay output (np = 16/20/15/14/15)
│   └── u{L}r112_{t}_{h}.ct    5 layers × 8 = 40   boot refresh output (np=112)  <- hand-off artefact
├── logs/                      20 files: lay{L}.out/.err, boot{L}.out/.err (criterion evidence)
├── meta.txt                   machine / compiler / thread metadata
├── master.log                 timeline of the 10 hops (per-hop PASS/wall clock)
├── timing.tsv                 per-layer wall-clock summary
├── manifest.sha256            SHA256 of every file in the package (104 lines, excluding itself)
└── README.md                  detailed archive notes: results table / verification / boundaries

L26-27-fin/                          <- tail: layers 26-27 + output head
├── chain/
│   ├── u26_{t}_{h}.ct / u27_{t}_{h}.ct         8 each   lay output (np=3)
│   └── u26r112 / u27r112_{t}_{h}.ct            8 each   boot refresh output (np=112)  <- hand-off artefact
├── logs/                      12 files: t26 / boot26 / t27 / boot27 / fin .out/.err
│                              plus master.log (five-step timeline) and meta.txt
├── manifest.sha256            SHA256 of every file in the package (45 lines, excluding itself)
└── README.md                  tail archive notes: per-hop results / logits and top1 / boundaries

tools-win-x64/                      <- tools (not results): needed to run the two packages above
├── t23lay.exe                 584,907 B   lay intra-layer forward (CKKS_NPRIMES=112)
├── t23boot.exe                569,720 B   boot bootstrap refresh (CKKS_NPRIMES=2100, 32 MB stack baked in)
├── verify_layer.exe           214,536 B   independent numeric re-check (decrypt -> decode -> max|err|)
├── manifest.sha256            6 lines (the three files above + both READMEs + meta.txt; LF endings)
├── README.md / README.en.md   usage, reproduce-from-source commands, verification record, boundaries
└── meta.txt                   build environment, full compile commands, source anchors, binary SHA256
```

> The three executables are **statically linked**: they depend only on `KERNEL32.dll` and the UCRT
> (`api-ms-win-crt-*`, shipped with Windows 10+), so **MinGW is not required**. Usage and where to place
> them are in the package's own `README.md` (key points: put the executables in `<repo root>\.tmp_tok\`,
> and **run every command from the repo root** — the engine hard-codes its data path to `.tmp_tok/`,
> resolved relative to the current directory).

> **The two packages do not have the same top-level layout**: `L0-4` keeps `master.log` / `meta.txt` /
> `timing.tsv` at the **package root**, while `L26-27-fin` keeps `master.log` / `meta.txt` **inside `logs/`**
> (its package root holds only `chain/`, `logs/`, `manifest.sha256`, `README.md`). The trees above follow each
> package's actual layout — this is not a typo.

Convention: `t=0..3` (4 tokens), `h=0..1` (2 ciphertext components); one layer = `lay` (forward) + `boot` (bootstrap refresh).

**The next hand for the head starts from `u4r112` and continues with `lay5`.** The tail is not a continuation of the
head — see §4.3.

---

## 3. Verification (weakest to strongest)

### ⓪ Get the tools first (if you would rather not install gcc)

Usage is in the tools package's own `README.md`. Two steps to put it in place:

```powershell
New-Item -ItemType Directory -Force <repo root>\.tmp_tok | Out-Null
Copy-Item tools-win-x64\*.exe <repo root>\.tmp_tok\    # then run everything from the REPO ROOT
.tmp_tok\verify_layer.exe -L 4 -Ct u4                  # smoke test: expect VERIFY_PASS (8/8) max_err=1.3823e-02
```

If you do have gcc, skip this and build it yourself per the main repo's `tools/relay/README.md` §2
(**bit-level consistency requires that exact compile setup**).

### ① Check integrity (a few seconds)

```powershell
# 1) verify the ZIPs themselves first
(Get-FileHash kestrel-fhe-relay_L0-4_20260915.zip -Algorithm SHA256).Hash.ToLower()
# expected b3a6c7af273c664854c2bdf8af4a045776cc5c9f204db90c9b2981de719bcbf7
(Get-FileHash kestrel-fhe-relay_L26-27-fin_20260915.zip -Algorithm SHA256).Hash.ToLower()
# expected 26d8a9f09741c507e8bd0e35810e18062775071dded5bdc0653aca2122c58402

# 2) extract, then check every file listed in the manifest (same procedure for both packages)
Expand-Archive kestrel-fhe-relay_L0-4_20260915.zip -Destination .
Set-Location L0-4
Get-Content manifest.sha256 | ForEach-Object {
  if ($_ -match '^([0-9a-fA-F]{64})\s+(.+?)\s*$') {
    $want = $Matches[1].ToLower(); $rel = $Matches[2]
    $got = (Get-FileHash $rel -Algorithm SHA256).Hash.ToLower()
    if ($got -ne $want) { "MISMATCH $rel" }
  }
}
# expected: no output (L0-4 -> 104/104 match; L26-27-fin -> 45/45 match)
```

> **Note for Linux users**: the paths inside `manifest.sha256` use **backslashes** (`chain\u0_0_0.ct`).
> That is to stay consistent with the relay toolchain (`tools/relay/verify_relay.ps1`'s anti-smuggling
> reverse lookup depends on this form), but `sha256sum -c` does not accept backslashes. Convert first:
> ```bash
> sed 's|\\|/|g' manifest.sha256 > manifest.posix.sha256 && sha256sum -c manifest.posix.sha256
> ```

### ② Check "is it computed correctly" (accuracy check)

This needs the plaintext references and the secret key `.tmp_tok/chain/sk.bin` from the **full data package**
(that package is not shipped with the repository, see this repo's README §5). The tool comes from the main
repository's `tools/relay/`:

```powershell
# head L0-4
powershell -NoProfile -ExecutionPolicy Bypass -File <main repo>\tools\relay\verify_layer.ps1 `
    -From 0 -To 4 -ChainDir <extracted dir>\L0-4\chain -Sk .tmp_tok\chain\sk.bin

# tail L26 / L27 (boot output; references tail\l26_u2_ref.bin, tail\l27_u2_ref.bin)
powershell -NoProfile -ExecutionPolicy Bypass -File <main repo>\tools\relay\verify_layer.ps1 `
    -From 26 -To 26 -Mode boot -ChainDir <extracted dir>\L26-27-fin\chain -Sk .tmp_tok\chain\sk.bin
.tmp_tok\verify_layer.exe -L 27 -Ct u27r112 -Dir <extracted dir>\L26-27-fin\chain -Tol 3e-2
```

`verify_layer` is a **code path independent of the driver's own verdict** (it decrypts, decodes and compares
against the plaintext reference itself), which is what makes it usable as a third-party re-check.
Expected: `VERIFY_PASS` on every layer; the `max|err|` values are in each package's `README.md`.

**For the tail, also read the output head**: the four `[F:logits t?]` top1 lines, see the package's `README.md` §5.2.

### ③ Strongest: bit-level cross-recomputation

Rerun the same layer on another machine (a different ISA is even better) with the same input and compare the
ciphertexts:

```powershell
.tmp_tok\verify_layer.exe -BitA u0 -BitB u0_independent -L 0     # head
.tmp_tok\verify_layer.exe -BitA u26 -BitB u26_independent -L 26  # tail (rerun t26)
```

Under a deterministic pipeline, identical input → identical ciphertext. If two parties independently obtain the
same SHA256, that layer's result **does not depend on either party's word**. The prerequisite is an
**identical thread count** (see §4.1).

---

## 4. Known facts and boundaries (do not misread)

### 4.1 Applies to both assets

1. **`manifest.sha256` was regenerated** (for both). The archive `README.md` was revised after archiving (local
   working-tree paths were changed to release-facing ones, and the judging-criteria notes were added), so the
   manifest was recomputed accordingly and everything now matches (`L0-4` 104/104, `L26-27-fin` 45/45);
   **all files other than `README.md` are byte-identical to the original manifest produced at archive time**,
   and the ciphertexts themselves were **not changed at all**.
2. **`BOOT=PASS` does not mean the numbers passed.** It only says the refresh flow completed and returned to the
   full chain (`out np=2083`); numerical quality is what `verify_layer`'s `max|err|` tells you.
3. **The thread count is a result variable**: `T23_NT=4` and `T23_NT=8` produce **bit-different** ciphertexts
   (the bootstrap mask depends on the thread-dependent RNG consumption history). Both archives are fixed at
   **4 threads**. If you want to do a bit-level comparison, you must also use 4 threads.

### 4.2 Specific to the head `L0-4`

1. **The error margin is not generous**: layer 2 has `max|err| = 2.8372e-02`, leaving only about **5.4%** of
   margin against the `3e-2` tolerance.
2. This archive covers **layers 0–4 only**. "The full 28-layer chain" is the relay goal, **not** an existing result.

### 4.3 Specific to the tail `L26-27-fin`

1. **It is a tail branch, not the whole chain.** It encrypts directly from the plaintext seed `tail/l26_u1.bin`
   into L26, **bypassing layers 5–25**; the relay chain still stops at `u4r112`. What it demonstrates is that
   "**the last two layers plus the output head compute correctly**", **not** that "the whole chain runs from
   layer 0 through layer 27".
2. **`t27`'s layer output has 2/8 values over tolerance** (max `7.969e-02`, 2.66× the `3e-2` tolerance). It did
   not make the final top1 wrong, but it **does not mean the precision is healthy**.
3. **An earlier run (2026-09-03, 96-prime chain) also covered the last layers + `fin` and produced a top1
   baseline, but that result does not count** — it was based on the reference system from before the causal-mask
   bug fix and is **not the same generation** as the references here; it cannot be used to corroborate these
   results.

### Why a stage can FAIL while the final verdict is still PASS

Logs will show both `PASS` and a large number of `FAIL` — **this is not a contradiction**. The two are not
measuring the same thing:

| Step | Explanation |
|---|---|
| Why the stage FAILs | L26/27 use a **real C-fold** (`g_causal` + real C folding); the intermediate `C/D/U2` values deviating from the true values is **expected**, not a defect |
| Magnitude of the expected deviation | Pre-quantified by the planning script `_cfold_plan.py`: `\|Δao\| ~ 1.4 / 0.8` — the same order as the observed `[C:attn] 8.5~12` and `[D:o] 14~29` |
| Why the final verdict is still PASS | The same plan defines the **normalized criterion**: `E2E logits drift 0.14 < margin 2.03` — the internal deviation shrinks to 0.14 at the logits, while the top1 decision margin is 2.03 |
| Where acceptance moves to | In this mode `C/D/U2` are **report-only (not counted in `fails`)**; acceptance is transferred to the **E2E logits gate** |

**One line**: a stage FAIL is a difference in an intermediate quantity's *convention* (expected and pre-quantified); the final PASS is the *end-to-end logits-gate verdict* — they are not the same ruler, and reading them together creates an illusion of self-contradiction.

Source of authority: the E2E-mode comment in the driver source `tools/drivers/t23_m3p.c` (around lines 1678-1680).

**Note: the gate in the `fin` stage is not the same one described above.** `C/D/U2` are controlled by
`T23_E2EMODE` (when the variable is unset they count towards `fails`), whereas the `[F:xnorm]` / `[F:top1]` checks
in `fin` (phase 5) go through a **hard-coded `&rf`** — whether or not `T23_E2EMODE` is set, they never count
towards `fails`. **Do not mix the two gates when reading logs.**

Empirical confirmation: the tail `fin` log reads `RESULT=PASS (0; ref-deviate 8)` — `rf = 8` is exactly the
8 `[F:xnorm]` entries (2 components × 4 tokens), i.e. **the number of top1 mismatches is 0**; all four
`[F:logits t?]` top1 values match the reference exactly (all `120555`). Details in `L26-27-fin/README.md` §5.

### 4.4 Specific to the tools package

1. **x86-64 Windows (PE) only.** It will not run directly on Linux / macOS, nor on aarch64 (RK3588).
   For non-Windows platforms, build it yourself per the main repo's `tools/relay/README.md` §2
   (the engine only needs `libc + libm`).
2. **`-static` is packaging only and does not change the numbers**; but after **changing the compiler
   version** or **dropping `-O2`**, whether the result is still bit-identical to the archive
   **cannot be assumed** — run the read-path check in §3 first (the tools package's `README.md` §5 has
   the recomputation record).
3. The tools package **contains no data**: weights, plaintext references and `sk.bin` (about 7 GB) are not
   shipped with any of the assets.

---

## 5. Suggestions for uploading the Release

| Item | Head | Tail | Tools |
|---|---|---|---|
| Tag | `L0-4` (or `relay-L0-4`) | `L26-27-fin` (or `relay-tail`) | `tools-win-x64` (or fold into either tag above) |
| Title | `Layer 0–4 hand-off artefacts (L0–L4)` | `Tail layers 26–27 + fin (ciphertext logits)` | `Prebuilt x86-64 toolchain (Windows)` |
| Attachment | `kestrel-fhe-relay_L0-4_20260915.zip` (158.75 MB) | `kestrel-fhe-relay_L26-27-fin_20260915.zip` (64.99 MB) | `kestrel-fhe-relay_tools_win-x64_20260915.zip` (0.53 MB) |
| Description | Put the ZIP's SHA256 in it, consistent with §1 | Same; **and state explicitly that it is a tail branch that bypasses layers 5–25** | Same; **and note it is x86-64 Windows only, fixed at 4 threads** |

Each attachment is well within GitHub's 2 GB per-asset limit, but note §1: **the 100 MB repository hard
limit only constrains "committing to the repository", not Release attachments.** The tools package is only
0.53 MB and could in principle be committed — it still goes through Releases so that all three assets are
picked up in one place without undermining the "sources live in exactly one place" rule.

### 5.1 Gitee mirror: only two of the three fit (rationale)

This repository is mirrored at <https://gitee.com/pei-xiaoguang/fhe-relay>. Gitee's quotas differ from
GitHub's. Per the [Gitee quota page](https://gitee.com/help/articles/4283) (community edition / personal):

> Attachment capacity: **maximum 100 MB per attachment**; 1 GB total per repository.

That is an **attachment** cap — unlike GitHub, where the 100 MB figure applies only to committing into
the repository while releases allow 2 GB per asset. The consequence:

| Asset | Size | GitHub Releases | Gitee Releases |
|---|---|---|---|
| Head `L0-4` | 158.75 MB | ✅ (2 GB per asset) | ❌ **exceeds the 100 MB attachment cap** |
| Tail `L26-27-fin` | 64.99 MB | ✅ | ✅ |
| Tools `tools_win-x64` | 0.53 MB | ✅ | ✅ |

**Why the head archive is not split into <100 MB parts**: this repo's verification procedure is "check the
SHA256 of *this one zip*, then extract" (§1 table and the guide's Appendix B both anchor on a single zip
hash). Splitting changes the hash, which would force the documentation to become "per-part hashes + a
merge step" — turning a 6-step flow into 8 steps and adding a new "did the merge succeed" failure mode.
Trading verifiability for one cross-site download is a bad trade.

**A missing head archive does not mean the relay is out of reach: the head can be reproduced locally.**
This needs stating, or the situation reads as "Gitee readers cannot relay at all". The head `L0-4` is not a
mandatory download — it is a **deterministic product**: generate the data set with the main repo's
`tools/preproc/` (about 7 GB, **not shipped here** — see §1 and the main repo's
`tools/preproc/README.md`), then run `lay0..boot4` from the seed exactly as in the guide
[`../docs/04_Relay_Reproduction_Guide_EN.md`](../docs/04_Relay_Reproduction_Guide_EN.md), and **you obtain
the same `u4r112`** (the first 5 layers / 10 hops take about 9 h — see that guide's §5.3; cross-architecture
bit-identity is its §5.4 check ③).

So the two ways to obtain the head relate as follows:

| Route | Cost | Note |
|---|---|---|
| Download `L0-4` from GitHub | one 158.75 MB download | **Saves time**: skips data generation (a Python preprocessing pass) and the ≈9 h of layers 0–4 |
| Reproduce the head locally | ≈9 h of CPU + ≈7 GB of data (self-generated) | **Requires no large download at all**, and is consistent with this repo's "reproducible and verifiable" claim |

> **This repository deliberately does not distribute large data**: the ≈7 GB data set (including
> `embed.bin` at 1.24 GB and the per-layer weights under `tail/w{L}/`) is **in no Release and is not going
> to be** — it is generated from the public model (`Qwen3-VL-2B-Instruct`, ≈4.26 GB) by the main repo's
> `tools/preproc/`. The head zip contains **no weights**, only ciphertexts and logs; so the "do not ship the
> weights" principle has always held here, and the Gitee attachment cap merely removes one download channel
> for **that one ready-made copy of the head**.
>
> (The head zip's SHA256 in §1 remains the authority regardless of the channel: a file downloaded from
> GitHub and one you reproduced yourself must both match the same value.)
