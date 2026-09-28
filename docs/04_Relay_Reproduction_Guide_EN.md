[简体中文](04_接力复现指南.md) | **English**

# 28-Layer Fully Homomorphic LLM Inference · Relay Reproduction Guide (Community Relay Edition)

> Audience: people who want to help **keep running** the RNS-CKKS fully-encrypted inference chain of Qwen3-VL-2B.
> This document only covers "how to keep computing", not the theory (the theory document lives in the main project at `资料/FHE同态加密推理技术介绍.md` and is **not shipped with this repository**).
> For the **tool inventory and per-parameter usage** see [`05_Tool_Reference_EN.md`](./05_Tool_Reference_EN.md).
> Version: 2026-09-14 · Author's self-tested environment: AMD Ryzen 7 9800X3D (8C/16T) · Windows 11 24H2 · MinGW-W64 gcc 16.1.0

---

## 0. One-page summary (TL;DR)

| Item | Content |
|---|---|
| Goal | On a single machine, CPU-only (no GPU), pure C11, zero third-party dependencies: run all 28 Transformer layers of Qwen3-VL-2B **entirely on ciphertexts** and output logits |
| Where we are now | **Layers 0–4 (5 layers) are done**: each layer's `lay` hop is `RESULT=PASS`, each layer's `boot` refresh hop is `BOOT=PASS` (this round was a rerun from the seed, see §8) |
| What is still missing | **Layers 5–25 on the chain** (the relay chain stops at `u4r112`). The tail 26–27 + `fin` has already been run through **separately, as another branch** (plaintext seed into L26, **bypassing layers 5–25**), see §8.1 |
| Per-layer cost | **lay ≈ 33 min + boot ≈ 72–81 min ≈ 1.8 h/layer** (measured on 9800X3D with 4 threads) |
| Why a relay is needed | The whole chain on one machine is ≈50 h of CPU-only compute; splitting it into "a few layers per person" gets it done within days |
| Per-layer hand-off artifact | The layer's output is **8 ciphertext files `u{L}r112_{t}_{h}.ct` (≈28 MB) + `sk.bin`**; whoever receives them can continue from `L+1` |
| Judging criteria | For `lay`, look for `RESULT=PASS` at the end of the log; for `boot`, look for `BOOT=PASS`. **Stage-level FAIL does not count**, see §7 |

---

## 0.1 The boundaries of what this project claims (**read this one first**)

We claim **exactly one thing** below; nothing else is treated as an established conclusion:

| Claim | Status |
|---|---|
| ✅ **The first 5 layers (layers 0–4) are reproducible and independently verifiable on CPU only**: layer by layer the `lay` criterion is `RESULT=PASS` and the `boot` refresh criterion is `BOOT=PASS` (each refresh returns to the full chain, `out np=2083`); the same source code is bit-identical coefficient by coefficient on x86-64 and aarch64 | **This repository is responsible for it** |
| ✅ **The tail (layers 26–27 + `fin`) also computes correctly**: encrypted directly from a plaintext seed into L26 (**bypassing layers 5–25**); all five steps `PASS`, and `fin`'s logits top1 matches the reference **4/4** | **This repository is responsible for it** (see §8.1) |
| ❌ "The full 28-layer chain runs" | **Not claimed** — that is the **community relay goal**, not an existing result |
| ❌ Cryptographic security / performance superiority | **Not claimed** (see §9) |

> So the remit of this guide is: **to explain the "the first 5 layers are reproducible and verifiable" claim thoroughly and make it solid**, and to hand out layers 5–27 as an open relay item.
> Any results from layer 6 onward that someone produces after taking over are **their contribution** and must not be backfilled into this project's existing conclusions.

---

## 1. What you need to prepare

### 1.1 Hardware
- x86-64 or ARM64 (RK3588 has been verified to be **bit-identical coefficient by coefficient** with x86, difference 0 / ~1e-16);
- ≥4 logical cores recommended (default `T23_NT=4`; 8 threads give higher absolute throughput but also higher power draw);
- ≥8 GB of available memory (`lay` stage measured working set ≈1.0 GB; the `boot` stage's modulus chain `np=2100` is larger, so leave more headroom);
- Disk: **data package ≈7 GB + model 4.26 GB (only if you generate the data yourself) + ≈28 MB of artifacts per layer**; leave **≥20 GB** free.

### 1.2 Toolchain
- A `gcc` that can compile C11 + OpenMP (MinGW-W64 or Linux gcc both work); the whole chain only depends on `libc + libm`;
- No cryptographic library needs to be installed.

### 1.3 Data package (critical, about 7 GB)
The source code **does not include** the weights or the reference data. Two routes, pick either:

| Route | How | Cost |
|---|---|---|
| **A. Generate it yourself (fully reproducible)** | Download `Modl/Qwen3-VL-2B-Instruct/` (`model.safetensors` 4.26 GB + `vocab.json`) and install `numpy` + `safetensors`, then run, from the repository root, the steps in the main repo's `tools/preproc/README.md` §3: `_export_weights.py` → **`_embed4.py`** → **`_silu_mode.py`** → `_full_layers.py` → `_tail_ref.py` → `_sin_fit.py` / `_recip_fit.py` → `_adap_scale.py` | one Python preprocessing pass |
| **B. Ask the author for a ready-made package** | leave a note in an Issue (we reply with a **download link**, never as an attachment — 7 GB is far past any attachment cap) | transferring 7 GB |

> ⚠️ **The order is not optional — two files fail silently when missing**:
> `_embed4.py` (lay0's plaintext input) and `_silu_mode.py` (the per-layer SiLU path switch) produce files
> the driver **needs but does not complain about** — without them the driver **still prints `RESULT=PASS`
> while the numbers are wrong**.
> Measured: layer 0 with `silu0.bin` missing gives `max|err|` = 4.2e-2 – 8.5e-2 (above the 3e-2 tolerance);
> with it, back to 1.27e-3.
> **So a criteria log is not a substitute for numeric re-checking**: after every layer, re-check `max|err|`
> independently with `tools/relay/verify_layer.ps1` (or `verify_layer.exe`) — see §5.6 and §7.

> **The one former gap — `l0/embed4.bin` — is now closed.** It is lay0's plaintext input (the embedding
> lookup for the 4 tokens) and used to have no generator, so it could only be shipped with the data
> package. It is now produced by **`tools/preproc/_embed4.py`**, which defaults to
> `--ids 100,101,102,103` (exactly the four ids the published data uses); the output is 32768 B and
> **byte-identical** to the `embed4.bin` in the data package (SHA256 prefix `c945c8fe92cc0595`, verified
> by re-running it).
>
> `sk.bin` can also be generated by the driver itself: **create `.tmp_tok/chain/` first**, then run
> `T23_PHASE=l0 T23_KEYONLY=1 .tmp_tok/t23lay.exe` and look for `KEYONLY: sk.bin saved`.
> That directory step is not optional — the driver uses a bare `fopen` and **never creates directories**;
> without it you get `[FAIL] sk save` (and the same applies when running layers: `[FAIL] ct save`).

```
.tmp_tok/
├── chain/sk.bin                 # secret key (2 KB) — unique to the whole chain, must come from the same source as the weights/reference
├── chain/u0r112_{t}_{h}.ct      # starting ciphertext seed (8 files), or your u{L-1}r112 when you take over
├── tail/w0 .. w27/              # per-layer weights (the .bin files read by gcc, 9 per layer)
├── tail/l{L}_*.bin              # per-layer plaintext reference (for verification)
├── tail/scale{L}.bin            # per-layer adaptive scale (residual folding divisor)
├── tail/silu{L}.bin             # SiLU approximation mode switch
├── tail/ln{L}_e1.bin / ln{L}_p1.bin / m2c{L}_e.bin / m2c{L}_p.bin
├── l0/  l1w/  l1r/              # weights and references specific to layer0 / layer1
└── ...
```

> **Data self-check (confirm before running)**: `.tmp_tok/chain/sk.bin`, the input ciphertexts for your leg, `tail/w{L}/` (9 files) and `tail/scale{L}.bin` must all exist. If one is missing, the run will error out and exit straight away during the load stage.
>
> **Also check these two separately — they do not error out when missing, but the numbers come out wrong**:
> `tail/silu{L}.bin` (one per layer, 8 B; produced by `_silu_mode.py` in §1.3) and `l0/embed4.bin`
> (used by `lay0` only; produced by `_embed4.py`). **Do not start after checking only the four items above.**

(The tree above is the data package's layout relative to the repository root; the engine hard-codes the `.tmp_tok/` prefix.)

### 1.4 Would rather not install gcc? Use the prebuilt binaries and skip the build

This repository's **Releases** carry prebuilt Windows x86-64 binaries (`t23lay.exe` / `t23boot.exe` /
`verify_layer.exe`, **statically linked, no MinGW needed**). Unzip, put them in place, and you can
**skip the build in §2**:

```powershell
cd <repo root>
New-Item -ItemType Directory -Force .tmp_tok, .tmp_tok\chain | Out-Null
Copy-Item <extracted dir>\tools-win-x64\*.exe .tmp_tok\
```

Once the data is in place, smoke-test with it (seconds, no layer run needed):

```powershell
.tmp_tok\verify_layer.exe -L 4 -Ct u4     # expect RESULT: VERIFY_PASS (8/8)  max_err=1.3823e-02
```

Details in [`data/README.en.md`](../data/README.en.md) §3 ⓪. On **non-Windows platforms**
(Linux / macOS / aarch64) build it yourself per §2.

---

## 2. Build (two commands, both verified)

Source locations: `src/core/{vllm_ntt.c, vllm_ckks.c, vllm_tp.c}` + the drivers `tools/drivers/{t23_m3p.c, t23_chain.c}` (**both ship with the repository**; `.tmp_tok/` holds only data and artifacts).

> **Create the working directory first**: `mkdir -p .tmp_tok .tmp_tok/chain .tmp_tok/relay` (PowerShell:
> `New-Item -ItemType Directory -Force .tmp_tok, .tmp_tok\chain, .tmp_tok\relay | Out-Null`).
> `.tmp_tok/chain/` is where ciphertexts are written and the **drivers never create it** (without it you get
> `[FAIL] ct save` / `[FAIL] sk save`); `.tmp_tok/relay/` holds the logs of the manual runs in §4.2/§4.3 and
> **redirection does not create directories either** (see the measured note in §4.2); both `-o` targets also
> write into `.tmp_tok/`, and if the directory is missing the linker fails immediately with
> `cannot open output file .tmp_tok/t23lay.exe`. Naming the output `t23lay` is enough — MinGW gcc appends
> `.exe` itself.
> (If you use the one-key script in §4.4 you can skip this — the script prepares the directories it needs itself.)

```bash
# lay hop (intra-layer forward): modulus chain of 112 primes
gcc -O2 -fopenmp -Wno-implicit-function-declaration \
    -I include -I include/core -I include/common \
    -DCKKS_N=2048 -DCKKS_NPRIMES=112 -DBB=32 -DGG=32 \
    src/core/vllm_ntt.c src/core/vllm_ckks.c src/core/vllm_tp.c tools/drivers/t23_m3p.c \
    -o .tmp_tok/t23lay -lm

# boot hop (noise refresh): modulus chain of 2100 primes
gcc -O2 -fopenmp -Wno-implicit-function-declaration '-Wl,--stack,33554432' \
    -I include -I include/core -I include/common \
    -DCKKS_N=2048 -DCKKS_NPRIMES=2100 -DBB=32 -DGG=32 \
    src/core/vllm_ntt.c src/core/vllm_ckks.c src/core/vllm_tp.c tools/drivers/t23_chain.c \
    -o .tmp_tok/t23boot -lm
```

**Compile-time traps (we hit every one of them)**

1. `-I` must be given for `include`, `include/core` and `include/common` at the same time (`vllm_tp.c` references `vllm_platform.h` in `common/`); passing only `include/core` gives `vllm_platform.h: No such file`.
2. On Windows, the comma inside `-Wl,--stack,...` is treated by PowerShell as an argument separator → **the whole thing must be quoted** as `'-Wl,--stack,33554432'`, or run it through `cmd /c`.
3. `boot` uses `np=2100`, and the MinGW default stack overflows (`0xC00000FD`) → add `-Wl,--stack,33554432`.

---

## 3. Ciphertext chain protocol: one layer = two hops

Every layer `L` requires **two hops**, both landing in `.tmp_tok/chain/`; each hop produces **8 `.ct` files** (4 tokens × 2 ciphertext components):

```
seed             : chain/u0r112_{t}_{h}.ct            (starting seed, produced by encrypting the plaintext starting point and refreshing it)
lay{L}  (forward)   : u{L-1}r112_{t}_{h}.ct  ──▶  u{L}_{t}_{h}.ct         criterion RESULT=PASS
boot{L} (refresh)   : u{L}_{t}_{h}.ct        ──▶  u{L}r112_{t}_{h}.ct     criterion BOOT=PASS
```

- `lay` cryptographically finishes this layer's RMSNorm → QKV → Attention → O → SwiGLU MLP → residual, and outputs the ciphertext `u{L}` whose "noise is nearly used up";
- `boot` performs one full bootstrapping (ModRaise → CoeffToSlot → EvalMod/sin folding → SlotToCoeff), **refreshing the noise budget back to the full chain**, and outputs `u{L}r112`, which the next layer can keep consuming on the deep chain.

> Layer numbers start at 0: the records currently held in this repository are **lay0..lay4** (layers 0..4), and the hand-off point is `u4r112`. The person taking over starts at **lay5**.

---

## 4. Running one layer: the full command template

### 4.1 Environment variables (the driver uses them to choose its behaviour)

| Variable | Role | Example value |
|---|---|---|
| `T23_PHASE` | Phase | `lay` / `boot` / `fin` |
| `T23_LAY` | Intermediate layer number (`lay` only) | `5` |
| `T23_BI` / `T23_BO` | boot input/output prefixes (`boot` only) | `u5` / `u5r` |
| `T23_E2EMODE` | End-to-end acceptance mode (ignores stage-level FAILs caused by reference deviation) | `1` |
| `T23_NT` | Thread count | `4` |

### 4.2 Windows / PowerShell (verified on this machine)

```powershell
cd <repo root>

# the log directory must exist first: cmd's > redirection, like the drivers, never creates directories
New-Item -ItemType Directory -Force .tmp_tok\relay | Out-Null

# ---- layer 5's lay hop ----
$env:T23_PHASE="lay"; $env:T23_LAY="5"; $env:T23_E2EMODE="1"; $env:T23_NT="4"
# use cmd for redirection: guarantees the log lands on disk byte for byte, not transcoded by PowerShell
cmd /c "`.tmp_tok\t23lay.exe` > `.tmp_tok\relay\lay5.out` 2> `.tmp_tok\relay\lay5.err`"
Select-String -Path .tmp_tok\relay\lay5.out -Pattern 'RESULT=PASS'   # a match = pass

# ---- layer 5's boot hop (refresh) ----
$env:T23_PHASE="boot"; $env:T23_BI="u5"; $env:T23_BO="u5r"
cmd /c "`.tmp_tok\t23boot.exe` > `.tmp_tok\relay\boot5.out` 2> `.tmp_tok\relay\boot5.err`"
Select-String -Path .tmp_tok\relay\boot5.out -Pattern 'BOOT=PASS'   # a match = pass
```

> ⚠️ **If `.tmp_tok\relay` does not exist the redirection above fails silently** (measured, not theoretical):
> cmd only prints `The system cannot find the path specified.` to stderr, **no log file is created at all,
> and `%ERRORLEVEL%` still stays 0**; only the following `Select-String` then reports a missing path — which
> is easily misread as "this layer crashed".
> Likewise, if `.tmp_tok/chain/` is missing the driver reports `[FAIL] ct save` (already noted in §2).
> **Create both directories up front.**

### 4.3 Linux / macOS / RK3588 (bash)

```bash
cd <repo root>                                # must be the repo root: driver paths are hard-coded to .tmp_tok/...
mkdir -p .tmp_tok/chain .tmp_tok/relay        # the log directory is not created for you either
T23_PHASE=lay  T23_LAY=5 T23_E2EMODE=1 T23_NT=4 .tmp_tok/t23lay  > .tmp_tok/relay/lay5.out  2> .tmp_tok/relay/lay5.err
T23_PHASE=boot T23_BI=u5 T23_BO=u5r    T23_NT=4 .tmp_tok/t23boot > .tmp_tok/relay/boot5.out 2> .tmp_tok/relay/boot5.err
grep -q RESULT=PASS .tmp_tok/relay/lay5.out   && echo "lay5 PASS"
grep -q BOOT=PASS   .tmp_tok/relay/boot5.out  && echo "boot5 PASS"
```

> The binaries live under `.tmp_tok/` (the `-o` targets of §2), not at `./t23lay` in the repo root;
> and **do not `cd .tmp_tok` before running** — the driver writes its data paths as `.tmp_tok/...`, so a
> different working directory makes it look for `.tmp_tok/.tmp_tok/...` and every load fails.

### 4.4 One-shot script (recommended)

The repository ships `tools/relay/rerun5_lay_boot.ps1`, which can take a layer range, build automatically, resume from where it stopped, and write `timing.tsv`:

```powershell
# run layers 5..9 (lay+boot for each); layers whose artifacts already exist are skipped automatically
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\rerun5_lay_boot.ps1 -From 5 -To 9
```

The logs and summary land in `.tmp_tok/relay_rerun/`: `master.log` (timeline), `timing.tsv` (per-layer wall clock/status), `lay{L}.out`/`boot{L}.out` (raw logs), `meta.txt` (machine metadata).

> **Log-directory convention**: the one-shot script writes `.tmp_tok/relay_rerun/`, while the manual runs in
> §4.2/§4.3 write `.tmp_tok/relay/` — the two are not interchangeable, so do not look/`grep` in the wrong one.

> The script follows three engineering conventions that you must observe too, and that you should copy if you write your own script:
> ① if a `.ps1` contains Chinese comments, it **must be saved as UTF-8 with BOM** (otherwise PowerShell 5.1 reads it as ANSI and reports "unexpected token }");
> ② redirect subprocess output with `cmd /c "... > out 2> err"`, never with a PowerShell pipe (which turns UTF-8 Chinese into mojibake);
> ③ the criterion must be taken from the **layer output**, see §7.

---

## 5. Relay protocol: how to hand your result to the next person

### 5.1 Deciding "this layer is finished"
Once these 8 files are all present under `.tmp_tok/chain/`, the layer's refresh is considered complete:

```
u{L}r112_0_0.ct  u{L}r112_0_1.ct
u{L}r112_1_0.ct  u{L}r112_1_1.ct
u{L}r112_2_0.ct  u{L}r112_2_1.ct
u{L}r112_3_0.ct  u{L}r112_3_1.ct
```

### 5.2 Handoff bundle

Pack the things below into **one zip**, host it on **your own Release or cloud drive**, and reply in the
claim issue with the **download link plus that zip's SHA256**:

| File | Description | Size |
|---|---|---|
| `u{L}r112_{t}_{h}.ct` ×8 | **Core**: the input ciphertexts for the next leg | ≈28 MB |
| `sk.bin` | Secret key (identical across the whole chain; omit it if the other side already has it) | 2 KB |
| `lay{L}.out` `boot{L}.out` | Criterion evidence (containing `RESULT=PASS` / `BOOT=PASS`); manual runs keep them in `.tmp_tok/relay/`, the one-shot script in `.tmp_tok/relay_rerun/` | a few hundred KB |
| `timing.tsv` `meta.txt` | Wall clock and machine metadata | KB-scale |

Naming suggestion: `relay_L{L}_{yourname}_{date}.zip` (**in practice just generate it in one command with `pack_relay.ps1`, see §5.5**).

> ⚠️ **Do not attach the zip itself to an issue or discussion.** Two hard reasons:
> 1. **It will not fit**: GitHub caps attachments in issues / PRs / discussions at **10 MB for images and
>    GIFs and 25 MB for anything else**, while a one-layer package is **≈28 MB**;
> 2. **Even if it fit, it should not be trusted**: every piece of evidence in this chain rests on
>    **byte-level hashes** (`manifest.sha256`, `verify_relay.ps1`, `verify_layer -BitA/-BitB`), and
>    intermediaries such as email re-encode or normalise bytes — after which every `manifest.sha256` entry
>    mismatches and **you cannot tell "the package was tampered with" from "the relay changed one byte"**.
>
> **A link is byte-faithful**: the package that `verify_relay.ps1` downloads must match the SHA256 posted in
> the claim issue, which also means the contributor cannot swap it afterwards.

> **The next person** drops `u{L}r112_*.ct` back into `.tmp_tok/chain/` and continues with `-From $($L+1)`.

### 5.3 Typical split (for reference when claiming)

| Leg | Layer range | Estimated single-machine time |
|---|---|---|
| ✅ Done | 0 – 4 | ≈9 h (measured 8.94 h in this round's rerun, see §8) |
| Leg 2 | 5 – 7 | ≈5.5 h |
| Leg 3 | 8 – 10 | ≈5.5 h |
| … | … | … |
| Last leg | 25 – 27 + `fin` | ≈6 h |

> You can also claim just 1 layer (≈1.8 h); the hand-off bundle is just as small, and **any amount of compute helps**.

### 5.4 How others **independently verify** these 5 layers (the concrete practice of "technically verifiable")

You do not need to trust our verbal conclusion; run the three levels of checks below — failing any single one of them means something is wrong:

**Check ①: criterion self-check (seconds)**
```powershell
# the lay / boot criterion must hit for every layer; any layer without a match is a real failure
foreach ($L in 0..4) {
  "lay$L  : " + ((Select-String .tmp_tok\relay_rerun\lay$L.out  -Pattern 'RESULT=PASS') -ne $null)
  "boot$L : " + ((Select-String .tmp_tok\relay_rerun\boot$L.out -Pattern 'BOOT=PASS')   -ne $null)
}
# whether the refresh really returns to the full chain (should be 2083 for every token/component of every layer)
Select-String .tmp_tok\relay_rerun\boot*.out -Pattern 'out np=2083' | Measure-Object
```

**Check ②: hand-off bundle integrity and criteria (pack + verify loop, see §5.5)**
```powershell
# pack your layer range into an uploadable bundle (auto-generates the in-bundle SHA256 manifest and README)
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\pack_relay.ps1 -From 0 -To 4 -Id <yourname>
# after anyone receives the zip, one command checks integrity + criteria + structure
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\verify_relay.ps1 -Zip <package path>
```
> Hand-off artifacts are **deterministic products**: same source data + same source binaries + same layer order → byte-identical. A matching manifest = what you got is the same artifact, not replaced or truncated.

**Check ③: cross-architecture bit-level consistency (optional, strongest)**
On another machine with a **different ISA** (e.g. ARM64/RK3588), rerun `lay0`,
decode `u0_{t}_{h}.ct` with the same `sk.bin`, and the decrypted values should be **coefficient-for-coefficient identical** to the x86 result (measured difference 0 / ~1e-16).
If this check passes, it shows the conclusion is independent of floating-point rounding order and of the hardware.

**All three pass ⇒ "the first 5 layers are reproducible and technically verifiable" holds**; checks ① and ② can be done by anyone on their own machine.

### 5.5 Pack and upload, and third-party verification (two scripts, one command each)

**Relay participant: pack** (the bundle automatically contains `manifest.sha256` and `README.txt`)
```powershell
# after finishing L5..7, fill in your own layer range and ID
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\pack_relay.ps1 -From 5 -To 7 -Id yourname
# output: .tmp_tok\relay_out\relay_L5-7_yourname_<timestamp>.zip
```
Bundle structure:

```
relay_L5-7_yourname_<ts>.zip
└── relay_L5-7_yourname_<ts>/
    ├── chain/    u{L}r112_{t}_{h}.ct ×8 × per layer     ← core hand-off artifact
    ├── logs/     lay{L}.out/.err  boot{L}.out/.err ← criterion evidence
    ├── timing.tsv  meta.txt  master.log
    ├── sk.bin                                      ← secret key shared by the whole chain
    ├── manifest.sha256                             ← SHA256 of every file in the bundle
    └── README.txt                                  ← layer range / claim boundaries / verification and continuation instructions
```
> Size reference: 1 layer ≈28 MB, 4 layers ≈111 MB (measured). `sk.bin` is the single secret key of the whole chain; `-NoKey` excludes it (the receiver must then already hold the same `sk.bin`).

**Any third party: verify** (no need to trust the packer)
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\verify_relay.ps1 -Zip <package path>
# exit code 0 = everything passed; 1 = at least one check failed
```
The script performs three kinds of check:

| # | Check | Description |
|---|---|---|
| ① | **Integrity** | Every file in the bundle matches `manifest.sha256` byte for byte; and it **also checks in reverse "whether any smuggled file is not covered by the manifest"** |
| ② | **Criteria** | `logs/lay*.out` contains `RESULT=PASS`; `logs/boot*.out` contains `BOOT=PASS` and `out np=2083` exactly 8 times per layer (4 tokens × 2 components) |
| ③ | **Structure** | 8 `u{L}r112_*.ct` per layer, each file ≈3.5 MB (`np=112`) |

Self-test sample from this repository (packing the real layers 0–3 data of `_bak_20260914` and then verifying it):

```
[1] integrity   [PASS] all files match manifest -- checked=51 mismatched=0
                [PASS] no unlisted files (no smuggling) -- 0 extra
[2] criteria    [PASS] lay criterion lay0..lay3
                [PASS] boot criterion boot0..boot3 -- BOOT=PASS=True refreshes=8
[3] structure   [PASS] ciphertexts present -- count=32   [PASS] each ct ~3.5MB
RESULT: ALL_CHECKS_PASS
```

**Trust model (must be stated clearly; do not overclaim)**
- ✅ The SHA256 manifest can prove: **the bundle was not tampered with, was not truncated, and carries no extra smuggled files**; the criterion logs and the ciphertexts are artifacts from the **same batch**.
- ⚠️ It **cannot** prove "the computation itself is correct" — the criterion logs could in theory be forged.
- 🔒 **The only way to prevent forgery is cross-recomputation**: because the pipeline is deterministic, **if two people independently rerun the same layer, the SHA256 of the produced `u{L}r112_*.ct` must be byte-identical**. So the real verification stance is:
  1. run ①②③ first (seconds; filters out corrupt/wrong bundles);
  2. for key layers (e.g. L10, L27) **find a second person to recompute independently**, and compare the SHA256 of the hand-off artifacts;
  3. if the two agree ⇒ that layer's result is trustworthy, and it does not depend on anyone's "statement".

> This is the extra value of a "relay" over "running it all on one machine": **every layer can naturally be reproduced independently and falsified by cross-checking**.

### 5.6 Accuracy verification: the `verify_layer` tool (asks directly "is the computation right?")

§5.5 only answered "was the bundle modified"; it **did not** answer "is the computation right". `verify_layer` fills that gap.

**Principle**: decrypt the layer's ciphertext with `sk.bin` → decode into real slots → compare element by element against the reference `tail/l{L}_u2_ref.bin` produced by the **float32 plaintext forward pass**, and compute `max|err|`.
This is a **code path independent of the driver's PASS logic** (it implements decryption/decoding/comparison itself), so it can serve as third-party review.

**Usage**
```powershell
# ① accuracy: verify the lay output u{L} (default tolerance 3e-2)
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\verify_layer.ps1 -From 0 -To 4
# ② accuracy: verify the boot refresh output u{L}r112
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\verify_layer.ps1 -From 0 -To 3 -Mode boot
# ③ bit-level cross-recomputation: two people run the same layer independently and compare byte for byte (anti-forgery)
.tmp_tok\verify_layer.exe -BitA u5 -BitB u5_other -L 5
# the underlying program can also be invoked for a single item
.tmp_tok\verify_layer.exe -L 4 -Dir .tmp_tok/chain -Sk .tmp_tok/chain/sk.bin -Tol 3e-2
```
Dependency: the plaintext reference `tail/l{L}_u2_ref.bin` from the data package (8192 float32 = 4 tokens × 2048 dimensions).

**Calibration of the tool itself** (validated against the **previous** round's real data; matches layer by layer — the difference against this round's rerun in §8 is in the 4th–5th significant digit, which is normal across rounds)

| Layer | `verify_layer` measured max\|err\| | Corresponding value in the driver log `.tmp_tok/relay/lay{L}.out` | Match |
|---|---|---|---|
| 0 | 1.2712e-03 | 1.271e-03 | ✅ |
| 1 | 3.1467e-03 | 3.147e-03 | ✅ |
| 2 | 2.8366e-02 | 2.837e-02 | ✅ |
| 3 | 1.2284e-02 | 1.228e-02 | ✅ |
| 4 | 1.3820e-02 | 1.382e-02 | ✅ |

> All 5 layers agree (4–5 significant digits), showing the tool is equivalent to the driver; conclusion: **the first 5 layers are `ALL_LAYERS_VERIFY_PASS`**.

**⚠️ One finding that must be recorded (the margin is not generous)**
- Layer 2's `t1 h1` component has `max|err| = 2.837e-02` against a tolerance of 3e-2 — **only 5.8% margin left**;
- Layer 2 also has two further points of the same order of magnitude, 1.93e-02 / 1.52e-02;
- Pattern: **the error is systematically concentrated in the `h=1` component** (the second half of the dimensions, 1024–2047), while `h=0` is usually an order of magnitude smaller.
- Implication: **the first 5 layers pass, but the error budget is not generous**; if deeper layers keep accumulating, the tolerance may be breached — this is exactly the metric to watch closely once the relay reaches layer 6 and beyond (record the `max|err|` trend of `h=1` for every layer).

---

## 6. Directory quick reference: what is input and what is output

```
.tmp_tok/
├── chain/
│   ├── sk.bin                  [input] secret key
│   ├── u{L-1}r112_{t}_{h}.ct   [input] the input ciphertexts for your leg
│   ├── u{L}_{t}_{h}.ct         [output] lay output
│   └── u{L}r112_{t}_{h}.ct     [output] boot output (= what you hand to the next leg)
├── tail/w{L}/                  [input] layer L weights
├── tail/l{L}_*.bin             [input] layer L plaintext reference
├── relay_rerun/                [output] logs / timing.tsv / meta.txt
└── t23lay(.exe) t23boot(.exe)  [output] binaries
```

---

## 7. How to read the criteria (**the easiest place to misjudge — be sure to read this first**)

Once it is running, the log will show `PASS` together with a large number of `FAIL`s, for example:

```
[A:xnorm t0] max|err|=2.404e-01 FAIL
[C:attn  t0] max|err|=4.941e-01 FAIL
[U2 t0 h0]   max|err|=3.980e-03 PASS [E2E:ref-deviate-ok]
RESULT=PASS (0)
```

**This is not a contradiction, it is a matter of convention** — the reasoning and the quantified basis are in "Why a stage can FAIL while the final verdict is still PASS" at the end of this section. The project's convention is:

1. **The layer-output criterion is the criterion**: whether a layer passes is decided only by the `RESULT=PASS` (lay) or `BOOT=PASS` (boot) at the **end** of the log:
   - `grep RESULT=PASS .tmp_tok/relay/lay{L}.out` has a match (with the §4.4 one-shot script the directory is `.tmp_tok/relay_rerun/`);
   - `grep BOOT=PASS .tmp_tok/relay/boot{L}.out` has a match (and the log shows `out np=2083`, proving it really did refresh back to the full chain).
2. There is also a `fold` count that changes with the layer number (lay0/1 = 16, lay2 = 128, lay3/4 = 256); it moves in step with the number of stage-level FAIL lines — this is part of the reference construction, not an error.
3. **But `RESULT=PASS` only means "the flow completed with no hard error" — it does not mean the numbers passed.**
   The relay script sets `T23_E2EMODE=1`, which downgrades intermediate deviations to "reported only, not counted
   in `fails`" (tagged `[E2E:ref-deviate-ok]` in the log) — so **a shallow layer running the wrong SiLU path
   still prints PASS** (measured: `[U2 t3 h0] max|err|=4.460e-02 FAIL [E2E:ref-deviate-ok]` followed by
   `RESULT=PASS (0)`). **Every layer must be independently re-checked with `verify_layer` (§5.6) for `max|err|`
   before it counts.**

> In one sentence: **only the `RESULT=PASS` / `BOOT=PASS` at the end of a line counts; ignore the stage-level FAILs.** Conversely, if the layer-output criterion fails, that is a real problem — please post the full log back to the Issue.

**Why a stage can FAIL while the final verdict is still PASS** — the two are not measuring the same thing:

| Step | Explanation |
|---|---|
| Why the stage FAILs | L26/27 use a **real C-fold** (`g_causal` + real C folding); the intermediate `C/D/U2` values deviating from the true values is **expected**, not a defect |
| Magnitude of the expected deviation | Pre-quantified by the planning script `_cfold_plan.py`: `\|Δao\| ~ 1.4 / 0.8` — the same order as the observed `[C:attn] 8.5~12` and `[D:o] 14~29` |
| Why the final verdict is still PASS | The same plan defines the **normalized criterion**: `E2E logits drift 0.14 < margin 2.03` — the internal deviation shrinks to 0.14 at the logits, while the top1 decision margin is 2.03 |
| Where acceptance moves to | In this mode `C/D/U2` are **report-only (not counted in `fails`)**; acceptance is transferred to the **E2E logits gate** |

**One line**: a stage FAIL is a difference in an intermediate quantity's *convention* (expected and pre-quantified); the final PASS is the *end-to-end logits-gate verdict* — they are not the same ruler, and reading them together creates an illusion of self-contradiction.

Source of authority: the E2E-mode comment in the driver source `tools/drivers/t23_m3p.c` (around lines 1678-1680).

---

## 8. This round's (2026-09-14 → 09-15) measured record of rerunning the first 5 layers from the seed

- Machine: AMD Ryzen 7 9800X3D (8C/16T) · Windows 11 24H2 · MinGW-W64 gcc 16.1.0 · `T23_NT=4`
- Reference (previous round, `relay/master.log`) hop-by-hop wall clock:

| Hop | Wall clock | Hop | Wall clock |
|---|---|---|---|
| lay0 | 1992 s (33.2 min) | boot0 | 3873 s (64.6 min) |
| lay1 | 1926 s (32.1 min) | boot1 | 3890 s (64.8 min) |
| lay2 | 2039 s (34.0 min) | boot2 | 3978 s (66.3 min) |
| lay3 | 2025 s (33.8 min) | boot3 | 4337 s (72.3 min) |
| lay4 | 1857 s (31.0 min) | — | — |

- This round's rerun log: `.tmp_tok/relay_gh/master.log` and `timing.tsv` (**complete, all 10 hops PASS**).

| Layer | lay | boot | Hand-off artifact |
|---|---|---|---|
| 0 | 1962 s (32.7 min) | 4377 s (73.0 min) | `u0r112` |
| 1 | 2008 s (33.5 min) | 4891 s (81.5 min) | `u1r112` |
| 2 | 1942 s (32.4 min) | 4350 s (72.5 min) | `u2r112` |
| 3 | 1949 s (32.5 min) | 4339 s (72.3 min) | `u3r112` |
| 4 | 1972 s (32.9 min) | 4392 s (73.2 min) | `u4r112` ← **the next leg starts here** |
| **Total** | **9833 s** | **22349 s** | 10 hops **32182 s ≈ 8.94 h** |

> The table is generated directly from `relay_rerun/timing.tsv`; for the full hop-by-hop record (including the `boot` stage profile and the per-layer `max|err|`) see
> [`03_Measured_Data_Appendix_EN.md`](./03_Measured_Data_Appendix_EN.md) §2–§4.

### 8.1 Tail branch (2026-09-15): layers 26–27 + `fin`

**This is not a continuation of the chain** — it encrypts the plaintext seed `tail/l26_u1.bin` directly into
L26, **bypassing layers 5–25**, in order to validate "the last two layers + the output head" on their own.

| Step | Wall | Result |
|---|---|---|
| `t26` | 2079 s | `RESULT=PASS`, U2 8/8 PASS (4.471e-03 ~ 2.299e-02) |
| `boot26` | 4469 s | `BOOT=PASS`, 8× `out np=2083` |
| `t27` | 2068 s | `RESULT=PASS`, U2 **2/8 over threshold** (3.822e-02 / 7.969e-02) |
| `boot27` | 4489 s | `BOOT=PASS`, 8× `out np=2083` |
| `fin` | 3178 s | `RESULT=PASS (0; ref-deviate 8)`, **top1 4/4 correct** (all `120555`) |
| **Total** | **16284 s ≈ 4.52 h** | — |

Archiving and independent re-check: see [`03_Measured_Data_Appendix_EN.md`](./03_Measured_Data_Appendix_EN.md) §9.2 and its "Tail measurement" section.

---

## 9. Honest boundary statement (please be aware of this before joining the relay)

These are facts that were written into the paper draft from the very beginning and that cannot be concealed:

1. **No claim of cryptographic security strength**: the current parameters (`n=2048`, an extremely long modulus chain) are at the **mechanism/correctness verification level**, far below 128-bit; upgrading to 128-bit would require `n ≥ 16384` or equivalent parameters. What the relay reproduces is "**can it compute correctly**", not "**can it resist attack**".
2. **The randomness is a prototype RNG**; the interface already reserves a place for injecting a CSPRNG, but none is wired in at present.
3. **The weights are plaintext constants** (HE plaintexts preprocessed offline); what this chain protects is the **inference input and intermediate activations**, not the model weights. Understand "fully encrypted" as "input and activations fully encrypted".
4. **Performance is not a selling point**: single-token latency is hours, which suits offline batch processing / auditable private inference, not interactive generation. The cost of FHE is overhead on the order of 10^5–10^7.
5. **The full 28-layer chain has not yet been run to completion**: every mention of "28 layers" in this document is an **extrapolated ETA (≈50 h)**, not a measured value.

---

## 10. How to report results / claim work

1. Claim a layer range (e.g. `5–7`) in the Issue, to avoid collisions;
2. when done, host the hand-off zip on **your own Release or cloud drive** and reply in the Issue with the
   **download link plus the zip's SHA256**, attaching `timing.tsv` and the criterion logs (**logs are only
   KB-scale, so pasting them inline is fine; do not attach the zip** — see §5.2);
3. if the layer-output criterion FAILs, do **not** delete the logs — post `lay{L}.out` / `lay{L}.err` in full (keep the original bytes; do not re-save through Notepad, which re-encodes them);
4. once the hand-off bundle is confirmed received by the next leg, you can free your own `.ct` reservation.

**Verification and registry (maintainer side, publicly reproducible)**: anyone downloads the package from the
link and runs one command

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools/relay\verify_relay.ps1 -Zip <downloaded package>
```

(exit code 0 = integrity + criteria + structure all pass), then `verify_layer.ps1 -From L -To L` for
`max|err|`. Results are recorded in an index table in the **first post of the claim issue**, one row per layer:

| layer | zip SHA256 | contributor | verifier | date | criterion | `verify_layer` max\|err\| | notes |
|---|---|---|---|---|---|---|---|
| 5 | `<64 hex>` | `<ID>` | `<ID>` | `<date>` | `RESULT=PASS` / `BOOT=PASS` | `<value>` | — |

> **The registry is an endorsement, not a gate**: verifiability always stays on the public side — anyone can
> download the same package, run the same command, and reach the same conclusion **without going through us**.
> That is exactly what §5.5 means by "depending on no one's word".

---

## Appendix A · FAQ

**Q: `lay` finished without `RESULT=PASS`, but with `CHAIN: dumped uL_*`?**
A: That means the dump succeeded but verification did not pass; it is a real FAIL, so post the log back to the Issue.

**Q: Can I run only `lay` without `boot`?**
A: No. The noise budget of `lay`'s output `u{L}` is nearly exhausted, and it must be refreshed back to the full chain by `boot` (`out np=2083`) before it can be fed to the next layer; otherwise the next layer suffers **noise breakdown**.

**Q: `boot`'s `in np=` differs per layer (e.g. 15/16/20) — is that normal?**
A: Yes, it is normal. That is the remainder of the modulus chain consumed by the previous `lay` hop, and it depends on that layer's operator depth. As long as `out np=2083`, the refresh succeeded.

**Q: The Chinese text in the log turns into mojibake?**
A: Redirect with `cmd /c "... > file"` (do not go through a PowerShell pipe); save `.ps1` files as UTF-8 with BOM.

**Q: Can it run on ARM64/RK3588?**
A: Yes. It has been verified to be **bit-identical coefficient by coefficient** with x86, so a cross-architecture relay is possible (but RK3588 is slower, about 40 min for the t0 stage; evaluate that for yourself).

**Q: If I continue on a different machine, will the precision change?**
A: No. Bit-level consistency means the result is decoupled from the hardware rounding order and is reproducible across machines.

---

## Appendix B · Publication credential: hand-off bundle SHA256

Every hand-off bundle is generated by `pack_relay.ps1` and carries its own `manifest.sha256`; **the credential published externally is the SHA256 of the zip itself**.
After downloading, collaborators first check the zip's SHA256, then run `verify_relay.ps1` for the byte-for-byte check inside the bundle.

| Hand-off bundle | Layer range | zip SHA256 |
|---|---|---|
| `kestrel-fhe-relay_L0-4_20260915.zip` (this repo's Releases) | 0 – 4 | `b3a6c7af273c664854c2bdf8af4a045776cc5c9f204db90c9b2981de719bcbf7` |
| `kestrel-fhe-relay_L26-27-fin_20260915.zip` (this repo's Releases, **tail branch**) | 26 – 27 + `fin` | `26d8a9f09741c507e8bd0e35810e18062775071dded5bdc0653aca2122c58402` |

> After finishing your leg, generate your own bundle with
> `tools/relay/pack_relay.ps1 -From <a> -To <b> -Id <ID>`; the packing log prints `sha256 : ...` directly —
> copy that value into this table (or just post it in the Issue).
> Full verification of the two published assets is in [`../data/README.en.md`](../data/README.en.md).

---

*This guide is updated on a rolling basis as the chain progresses; for the latest progress see `.tmp_tok/relay_rerun/master.log`.*
