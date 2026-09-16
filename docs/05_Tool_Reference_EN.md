[简体中文](05_工具说明.md) | **English**

# Tool Reference · RNS-CKKS Ciphertext Chain Toolchain

> Audience: relay runners (run layers), verifiers (review a received package), reproducers (reproduce from the seed).
> Companion docs: the workflow and the judging criteria are in [`04_Relay_Reproduction_Guide_EN.md`](./04_Relay_Reproduction_Guide_EN.md).
> Version: 2026-09-14 · **every script is always run from the repository root**.

**Tools live in the main repository, data lives in the working directory**

| Location | Purpose |
|---|---|
| `<main repo>/tools/relay/` | **Scripts and drivers (canonical copy)**: the run-layer / pack / verify-package / review scripts, plus the `t23_m3p.c` / `t23_chain.c` / `verify_layer.c` sources |
| `<main repo>/src/core/` | engine kernel `vllm_ntt.c` / `vllm_ckks.c` / `vllm_tp.c` (linked at compile time) |
| `.tmp_tok/` | **Working directory**: the scripts assume a `.tmp_tok/` under the cwd (where the data package is unpacked); ciphertexts and logs all land here |

---

## 0. Tool overview

| Tool | Purpose (one line) | Key parameters | Main artifacts | Exit code |
|---|---|---|---|---|
| `rerun5_lay_boot.ps1` | **Run layers**: `lay`+`boot` layer by layer, resumable | `-From -To` | `.tmp_tok/chain/*.ct`, `relay_rerun/{lay,boot}*.out`, `timing.tsv` | 0=done |
| `pack_relay.ps1` | **Pack**: bundle the results of a layer range into an uploadable zip | `-From -To -Id` | `.tmp_tok/relay_out/*.zip` (contains manifest + README) | 0=success |
| `verify_relay.ps1` | **Verify package**: check the zip's integrity + judging criteria + structure | `-Zip` | console PASS/FAIL detail | number of failed items |
| `verify_layer.exe` / `.ps1` | **Numeric check**: decrypt against the plaintext reference and judge "is the computation right" | `-L`/`-From -To` | `max\|err\|` + PASS/FAIL | failed layers/items |
| `_monitor.ps1` | Background progress monitoring (auxiliary) | none | `relay_rerun/_status.txt` | — |

Three binaries: `t23lay.exe` (np=112), `t23boot.exe` (np=2100), `verify_layer.exe` (np=112).

---

## 1. Prerequisite: build the three binaries

> **Would rather not install gcc**: this repository's **Releases** carry prebuilt Windows x86-64 versions of
> all three binaries (**statically linked, no MinGW needed**). Unzip and copy them into
> `<repo root>\.tmp_tok\` to **skip this whole section**; see [`data/README.en.md`](../data/README.en.md) §3 ⓪.
> On non-Windows platforms, build them with the commands below.
> Before building, run `New-Item -ItemType Directory -Force .tmp_tok, .tmp_tok\chain` — without `.tmp_tok` the
> linker reports `cannot open output file`, and without `.tmp_tok\chain` the driver reports `[FAIL] ct save`
> (the **drivers never create directories themselves**). If you use `rerun5_lay_boot.ps1` you can skip this:
> the script prepares both directories.

```powershell
# ① lay driver (intra-layer forward pass, modulus chain of 112 primes)
gcc -O2 -fopenmp -Wno-implicit-function-declaration `
    -I include -I include/core -I include/common `
    -DCKKS_N=2048 -DCKKS_NPRIMES=112 -DBB=32 -DGG=32 `
    src/core/vllm_ntt.c src/core/vllm_ckks.c src/core/vllm_tp.c tools/drivers/t23_m3p.c `
    -o .tmp_tok/t23lay.exe -lm

# ② boot driver (noise refresh, 2100 primes; a large stack is required)
gcc -O2 -fopenmp -Wno-implicit-function-declaration '-Wl,--stack,33554432' `
    -I include -I include/core -I include/common `
    -DCKKS_N=2048 -DCKKS_NPRIMES=2100 -DBB=32 -DGG=32 `
    src/core/vllm_ntt.c src/core/vllm_ckks.c src/core/vllm_tp.c tools/drivers/t23_chain.c `
    -o .tmp_tok/t23boot.exe -lm

# ③ standalone verifier (np=112 is enough to cover the lay output and the refreshed ciphertexts)
gcc -O2 -fopenmp -Wno-implicit-function-declaration `
    -I include -I include/core -I include/common `
    -DCKKS_N=2048 -DCKKS_NPRIMES=112 -DBB=32 -DGG=32 `
    src/core/vllm_ntt.c src/core/vllm_ckks.c src/core/vllm_tp.c tools/relay/verify_layer.c `
    -o .tmp_tok/verify_layer.exe -lm
```

**Build pitfalls (all of them have bitten us)**
1. `-I` must be given for `include`, `include/core` and `include/common` at the same time (`vllm_tp.c` depends on `common/vllm_platform.h`).
2. Under PowerShell the **commas in `-Wl,--stack,...` are treated as argument separators** → quote the whole thing: `'-Wl,--stack,33554432'`.
3. `boot` (np=2100) must have a larger stack, otherwise MinGW dies with a `0xC00000FD` stack overflow.
4. A `.ps1` containing Chinese comments must be saved as **UTF-8 with BOM**, otherwise PowerShell 5.1 reads it as ANSI and reports "unexpected token }".

---

## 2. `rerun5_lay_boot.ps1` — run layers (resumable)

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\rerun5_lay_boot.ps1 -From 5 -To 7
```

| Parameter | Default | Description |
|---|---|---|
| `-From` | 0 | first layer number (inclusive) |
| `-To` | 4 | last layer number (inclusive) |
| `$env:T23_NT` | 4 | thread count (an environment variable, not a parameter) |

**Behaviour**
- For each layer it runs `lay` (`u{L-1}r112 → u{L}`) and then `boot` (`u{L} → u{L}r112`);
- **resumable**: if a hop's 8 output `.ct` files are all present it is skipped; after an interruption, rerunning the same command just continues from there;
- if any hop fails it aborts and records that (it will not carry on with a bad result).

**Artifacts**

| Path | Content |
|---|---|
| `.tmp_tok/chain/u{L}_{t}_{h}.ct` | lay output (8 per layer) |
| `.tmp_tok/chain/u{L}r112_{t}_{h}.ct` | **boot output = the hand-off passed to the next runner** |
| `.tmp_tok/relay_rerun/master.log` | hop-by-hop timeline (with PASS/FAIL and elapsed time) |
| `.tmp_tok/relay_rerun/lay{L}.out` / `boot{L}.out` | raw logs (the evidence for the judging criteria) |
| `.tmp_tok/relay_rerun/timing.tsv` | per-layer timing summary |
| `.tmp_tok/relay_rerun/meta.txt` | machine/compiler/thread metadata |

**Judging criteria**: for `lay`, look for `RESULT=PASS` at the end of the log; for `boot`, look for `BOOT=PASS` together with `out np=2083` (8 times per layer). **Stage-level A/B/C/D FAIL lines do not count** (see the guide §7).

---

## 3. `pack_relay.ps1` — pack the hand-off

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\pack_relay.ps1 -From 5 -To 7 -Id yourname
```

| Parameter | Required | Default | Description |
|---|---|---|---|
| `-From` | ✅ | — | first layer number |
| `-To` | ✅ | — | last layer number |
| `-Id` | | `anon` | packer identifier (goes into the package name and the README) |
| `-LogDir` | | `.tmp_tok\relay_rerun` | log directory |
| `-ChainDir` | | `.tmp_tok\chain` | ciphertext directory |
| `-NoKey` | | off | exclude `sk.bin` (the receiver must already hold the same private key) |

**Package layout**

```
relay_L5-7_yourname_<timestamp>/
├── chain/    u{L}r112_{t}_{h}.ct ×8 × per layer      ← the core hand-off
├── logs/     lay{L}.out/.err  boot{L}.out/.err  ← the evidence for the judging criteria
├── timing.tsv  meta.txt  master.log
├── sk.bin                                        ← the private key shared along the whole chain
├── manifest.sha256                               ← the SHA256 of every file in the package
└── README.txt                                    ← layer range / claim boundary / verification and resumption
```

**Size**: 1 layer ≈28 MB; 4 layers ≈111 MB (measured). The output prints the zip's SHA256, which serves as the credential for public release.

---

## 4. `verify_relay.ps1` — verify a hand-off package

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\verify_relay.ps1 -Zip .tmp_tok\relay_out\relay_L5-7_x_20260914-1200.zip
```

| Parameter | Required | Description |
|---|---|---|
| `-Zip` | ✅ | path of the package to verify |
| `-Keep` | | keep the unpacked directory (by default it is cleaned up automatically) |

**Three kinds of check**

| # | Check | Verdict |
|---|---|---|
| ① | Integrity | every file in the package has a SHA256 matching `manifest.sha256`; **and the reverse check for files not listed in the manifest being smuggled in** |
| ② | Judging criteria | `logs/lay*.out` contains `RESULT=PASS`; `logs/boot*.out` contains `BOOT=PASS` and `out np=2083` exactly 8 times |
| ③ | Structure | 8 `u{L}r112_*.ct` per layer, ≈3.5 MB per file (np=112) |

**Sample output** (self-test in this repository)

```
[1] integrity   [PASS] all files match manifest -- checked=51 mismatched=0
                [PASS] no unlisted files (no smuggling) -- 0 extra
[2] criteria    [PASS] lay criterion lay0..lay3
                [PASS] boot criterion boot0..boot3 -- BOOT=PASS=True refreshes=8
[3] structure   [PASS] ciphertexts present -- count=32   [PASS] each ct ~3.5MB
RESULT: ALL_CHECKS_PASS
```

**Trust boundary (must be stated explicitly)**
- ✅ What it can prove: the package has **not been tampered with / truncated / smuggled into**, and the ciphertexts and the criterion logs are products of the same run.
- ⚠️ What it **cannot** prove: that "the computation itself is right" — in theory the criterion logs can be forged. To judge correctness use §5; to defend against forgery use §5's **bit-level cross-recomputation**.

---

## 5. `verify_layer` — judging "is the computation right" (accuracy check)

### 5.1 How it works

It decrypts that layer's ciphertext with `sk.bin` → decodes it into real-valued slots → compares element by element against the reference `tail/l{L}_u2_ref.bin` produced by a **float32 plaintext forward pass**, and computes `max|err|`.
This is a **code path independent of the driver's PASS criterion** (it implements decryption/decoding/comparison itself), so it can serve as a third-party review.

> Premise: FHE has **no key-free general correctness verification**; "the computation is right" can only be established by "decrypt and compare against a plaintext reference". The package therefore must contain `sk.bin`, and the verifier needs the reference `tail/l{L}_u2_ref.bin` from the data package.

### 5.2 Usage (batch wrapper)

```powershell
# ① verify the lay output u{L} (default tolerance 3e-2)
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\verify_layer.ps1 -From 0 -To 4
# ② verify the boot-refreshed output u{L}r112
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\verify_layer.ps1 -From 0 -To 3 -Mode boot
```

| Parameter | Default | Description |
|---|---|---|
| `-From` / `-To` | 0 / 4 | layer range |
| `-Mode` | `lay` | `lay` = check `u{L}`; `boot` = check `u{L}r112` |
| `-ChainDir` | `.tmp_tok\chain` | ciphertext directory |
| `-Sk` | `<ChainDir>\sk.bin` | private-key path |
| `-Tol` | `3e-2` | decision tolerance |
| `-Exe` | `.tmp_tok\verify_layer.exe` | underlying program (built automatically if missing) |

### 5.3 Usage (the underlying program, including bit-level cross-recomputation)

```powershell
# single-item accuracy
.tmp_tok\verify_layer.exe -L 4 -Dir .tmp_tok/chain -Sk .tmp_tok/chain/sk.bin -Tol 3e-2

# bit-level cross-recomputation (anti-forgery): two people run the same layer independently and compare byte by byte
.tmp_tok\verify_layer.exe -BitA u5 -BitB u5_other -L 5
```

| Parameter | Default | Description |
|---|---|---|
| `-L` | 4 | layer number |
| `-Ct` | `u{L}` | ciphertext prefix (e.g. `u4`, `u4r112`) |
| `-Ref` | `.tmp_tok/tail/l{L}_u2_ref.bin` | plaintext reference (8192 float32 values) |
| `-Tol` | `3e-2` | tolerance |
| `-Scale` | 1.0 | reference scaling factor |
| `-Dir` | `.tmp_tok/chain` | ciphertext directory |
| `-Sk` | `.tmp_tok/chain/sk.bin` | private-key path |
| `-BitA` / `-BitB` | — | the two prefixes for the bit-level comparison (must be given as a pair) |

### 5.4 Calibration of the tool itself (against the previous round's real data, matching layer by layer)

| Layer | `verify_layer` measured `max\|err\|` | Driver log `.tmp_tok/relay/lay{L}.out` | Agree |
|---|---|---|---|
| 0 | 1.2712e-03 | 1.271e-03 | ✅ |
| 1 | 3.1467e-03 | 3.147e-03 | ✅ |
| 2 | 2.8366e-02 | 2.837e-02 | ✅ |
| 3 | 1.2284e-02 | 1.228e-02 | ✅ |
| 4 | 1.3820e-02 | 1.382e-02 | ✅ |

All 5 layers match (4–5 significant digits) ⇒ the tool and the driver are equivalent; the conclusion is **`ALL_LAYERS_VERIFY_PASS`**.

### 5.5 ⚠️ Known finding: the error margin is not generous

- The `t1 h1` component of layer 2 has `max|err| = 2.837e-02` against a tolerance of `3e-2` — **only 5.8% of margin left**; layer 2 also has 1.93e-02 / 1.52e-02.
- Pattern: **the error is systematically concentrated in the `h=1` component** (dimension 1024–2047); `h=0` is usually an order of magnitude smaller.
- Implication: passing for the first 5 layers holds, but **deeper layers may become tight**. From layer 6 of the relay onward, the `h=1` `max|err|` trend should be recorded layer by layer.
- **Remaining trust gap**: the reference `tail/l{L}_u2_ref.bin` currently comes from the author's plaintext pipeline; to make even the reference independent, one has to recompute the float32 forward pass from the original weights with numpy and replace that file.

---

## 6. `_monitor.ps1` — progress monitoring (auxiliary)

```powershell
Start-Process powershell -ArgumentList '-NoProfile','-File','tools\relay\_monitor.ps1' -WindowStyle Hidden
Get-Content .tmp_tok\relay_rerun\_status.txt     # refreshed automatically every minute
```
Output: the current process's CPU/memory, the `chain/*.ct` count, the contents of `master.log`, the latest criterion line per layer, and the tail of the current log. Useful for long runs when you do not want to keep a terminal open watching the screen.

---

## 7. Typical workflows for the three roles

**Relay runner (run N layers → submit)**
```powershell
# 1) confirm the inputs are in place: u{L-1}r112_*.ct in chain/ and tail/w{L}/ present
# 2) run
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\rerun5_lay_boot.ps1 -From 5 -To 7
# 3) self-check accuracy (optional, needs the data package reference)
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\verify_layer.ps1 -From 5 -To 7
# 4) pack and submit
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\pack_relay.ps1 -From 5 -To 7 -Id yourname
```

**Verifier (review a received package)**
```powershell
# 1) first check the zip's SHA256 (the value published by the releaser)
Get-FileHash .tmp_tok\relay_out\relay_L5-7_x.zip -Algorithm SHA256
# 2) byte by byte inside the package + judging criteria + structure
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\verify_relay.ps1 -Zip <package>
# 3) with the data package you can go one step further and judge "is the computation right"
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\verify_layer.ps1 -From 5 -To 7
# 4) anti-forgery for key layers: a second person recomputes independently, then the two are compared at the bit level
.tmp_tok\verify_layer.exe -BitA u7 -BitB u7_independent -L 7
```

**Reproducer (reproduce L0–4 from the seed)**
```powershell
# prerequisite: back up and empty chain/'s *.ct (keeping sk.bin)
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\rerun5_lay_boot.ps1 -From 0 -To 4
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\verify_layer.ps1 -From 0 -To 4
```

---

## 8. General conventions and pitfalls

| Item | Convention |
|---|---|
| Working directory | every script uses the **repository root** as its cwd (the scripts `Set-Location` to the root internally) |
| Encoding | a `.ps1` containing Chinese is saved as **UTF-8 with BOM**; subprocess output is redirected with `cmd /c "... > out 2> err"` (to avoid the PowerShell pipeline mangling UTF-8 Chinese into mojibake) |
| Judging criteria | the layer output's `RESULT=PASS` / `BOOT=PASS` is the criterion (stage-level A/B/C/D `FAIL` lines are artifacts of the reference convention and are not a conclusion); **the criterion is not numerical correctness** — that needs `verify_layer` in §5 |
| Resumption semantics | a hop whose 8 output `.ct` files are all present is treated as complete and skipped |
| Hand-off | 8 `u{L}r112_{t}_{h}.ct` per layer (≈28 MB/layer) + `sk.bin` |
| Determinism | same-origin data + same-origin binaries + same layer order ⇒ byte-identical (which supports cross-machine, cross-ISA cross-recomputation) |

---

*The tools are updated continuously as the chain progresses; when usage changes, please update this file and [`04_Relay_Reproduction_Guide_EN.md`](./04_Relay_Reproduction_Guide_EN.md) in step.*
