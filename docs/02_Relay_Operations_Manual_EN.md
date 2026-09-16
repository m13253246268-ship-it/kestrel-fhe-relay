[简体中文](02_同态加密推理接力操作手册.md) | **English**

# Homomorphic Encryption Inference Relay Operations Manual

This manual only covers how to do things; it does not open up any discussion of conclusions. The goal is that a relay runner or a verifier, once they have the repository, can complete the following in a fixed sequence:

1. File conversion
2. Starting the relay
3. Resuming the relay
4. Independent verification
5. Archiving and packaging

## 0. Path conventions (read this item first)

All relative paths in this document start from the **open-source repository root**:

```text
<reponame>/                       # open-source repository root, i.e. kestrel-llm
├── src/core/                     # RNS-CKKS engine (vllm_ntt.c / vllm_ckks.c / vllm_tp.c)
├── tools/drivers/                # layer-chain driver sources (t23_m3p.c / t23_chain.c)
├── tools/relay/                  # ← every script in this document lives here
├── .tmp_tok/                     # data and working directory (you must supply the ~7 GB data pack yourself)
└── results/                      # archive artifacts
```

Do not confuse the three kinds of paths:

| Category | Location | Notes |
|---|---|---|
| **Tool scripts** | `tools/relay/` | what this manual operates on; the scripts locate the repository root by themselves |
| **Driver/engine sources** | `tools/drivers/`, `src/core/` | used at compile time |
| **Data and artifacts** | `.tmp_tok/` | the engine hardcodes its data paths here; `.gitignore` already ignores it, so it never enters the repository |

> On "where to run": **all scripts run from the repository root**, but you do not have to `cd` there manually.
> A script locates the repository root by "walking up from its own directory until it finds a directory containing `src/`",
> so calling it from any directory with `-File <repo>/tools/relay/xxx.ps1` works correctly.

## 1. Main tools at a glance

| Tool | Purpose |
|---|---|
| `tools/relay/fix_crlf.sh` | Convert `CRLF` throughout the code tree to `LF` uniformly |
| `tools/relay/rerun5_lay_boot.ps1` | Run the `lay + boot` relay, with support for resuming from a checkpoint |
| `tools/relay/verify_layer.ps1` | Independent decryption plus numeric error check |
| `tools/relay/layver_run.ps1` | Numeric-equivalence acceptance (swap the input only, do not change the thread count) |
| `tools/relay/collect_results.ps1` | Archive the relay artifacts and logs |
| `tools/relay/pack_relay.ps1` | Pack the handoff artifacts into a zip package |
| `tools/relay/verify_relay.ps1` | Third-party verification of a received zip package |
| `tools/relay/watch_relay.ps1` | Unattended duty watching: record status on a timer, start runs automatically, archive and verify automatically |
| `tools/relay/_monitor.ps1` | Background progress monitoring (auxiliary) |

## 2. What to confirm before you start

You need at least the following files and directories:

- `tools/drivers/t23_m3p.c`
- `tools/drivers/t23_chain.c`
- `tools/relay/verify_layer.c`
- `src/core/vllm_ckks.c`
- `src/core/vllm_ntt.c`
- `src/core/vllm_tp.c`
- `.tmp_tok/chain/sk.bin`
- `.tmp_tok/l0/*`
- The plaintext reference `.tmp_tok/tail/l{L}_u2_ref.bin`

You also need the following to be directly callable on this machine:

- `gcc`
- `powershell`

## 3. File conversion: deal with CRLF first

If the code tree was exported from Windows and then copied to Linux or a board-side environment, do this step first.

What `tools/relay/fix_crlf.sh` does is convert `CRLF` to `LF` across the whole code tree, so that `.sh` does not report
`bad interpreter: /bin/bash^M`.

Usage:

```bash
bash tools/relay/fix_crlf.sh              # by default it processes the repository root
bash tools/relay/fix_crlf.sh <dir>        # process the specified directory
```

It converts **text files** only (`grep -I` skips binaries), and **never touches `.git/`**. After running it, check:

- whether `crlf_before` is greater than 0
- whether `crlf_after` becomes 0

## 4. Starting the relay

The main entry point is:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\rerun5_lay_boot.ps1
```

By default it:

- runs layers `0..4`
- uses `4` when `T23_NT` is not set
- `LayExe=.tmp_tok\t23lay.exe`
- `BootExe=.tmp_tok\t23boot.exe`
- `LogDir=.tmp_tok\relay_rerun`

If you want to specify the binaries and the log directory explicitly, it is recommended to run it like this:

```powershell
$env:T23_NT = "4"
powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\rerun5_lay_boot.ps1 `
  -From 0 -To 4 `
  -LayExe .tmp_tok\t23lay.exe `
  -BootExe .tmp_tok\t23boot.exe `
  -LogDir .tmp_tok\relay_rerun
```

## 5. What the relay script actually does

The internal flow of `rerun5_lay_boot.ps1` is quite fixed:

1. If `t23lay*.exe` / `t23boot*.exe` do not exist, first compile them from the sources in `tools/drivers/`
2. Write `meta.txt`
3. Execute layer by layer from `From..To`
4. For each layer, run `lay` first, then `boot`
5. If the 8 `.ct` files of a hop are already complete, skip it automatically
6. Finally write `timing.tsv`

The judging criteria are:

- the `lay` log contains `RESULT=PASS`
- the `boot` log contains `BOOT=PASS` (and `out np=2083`, confirming the refresh really returned to the full chain)

Note: **both only mean the flow completed with no hard error — they do not mean the numbers are correct.**
The script sets `T23_E2EMODE=1`, which downgrades mid-pipeline deviations to report-only (marked
`[E2E:ref-deviate-ok]`), so **a shallow layer on the wrong SiLU path still prints PASS** (measured: layer 0
with `silu0.bin` missing gives `max|err|` = 4.2e-2 ~ 8.5e-2 yet still reports `RESULT=PASS (0)`).
**To judge the numbers you must additionally run `verify_layer.ps1` and look at `max|err|`**
(see [`05_Tool_Reference_EN.md`](./05_Tool_Reference_EN.md) §5).

**Why a stage can FAIL while the final verdict is still PASS** — the two are not measuring the same thing:

| Step | Explanation |
|---|---|
| Why the stage FAILs | L26/27 use a **real C-fold** (`g_causal` + real C folding); the intermediate `C/D/U2` values deviating from the true values is **expected**, not a defect |
| Magnitude of the expected deviation | Pre-quantified by the planning script `_cfold_plan.py`: `\|Δao\| ~ 1.4 / 0.8` — the same order as the observed `[C:attn] 8.5~12` and `[D:o] 14~29` |
| Why the final verdict is still PASS | The same plan defines the **normalized criterion**: `E2E logits drift 0.14 < margin 2.03` — the internal deviation shrinks to 0.14 at the logits, while the top1 decision margin is 2.03 |
| Where acceptance moves to | In this mode `C/D/U2` are **report-only (not counted in `fails`)**; acceptance is transferred to the **E2E logits gate** |

**One line**: a stage FAIL is a difference in an intermediate quantity's *convention* (expected and pre-quantified); the final PASS is the *end-to-end logits-gate verdict* — they are not the same ruler, and reading them together creates an illusion of self-contradiction.

Source of authority: the E2E-mode comment in the driver source `tools/drivers/t23_m3p.c` (around lines 1678-1680).

## 6. How to resume the relay

This script supports resuming from a checkpoint by design.

For example:

- if `lay0/boot0/lay1` have all completed
- but `boot1` was interrupted

then when you run the same command again, it automatically skips the steps that are already fully on disk and continues only from the missing hop.

After it continues, the key logs are in:

- `.tmp_tok\relay_rerun\master.log`
- `.tmp_tok\relay_rerun\lay*.out`
- `.tmp_tok\relay_rerun\boot*.out`

## 7. How to monitor the relay

The most direct way is to look at these files:

- `.tmp_tok\relay_rerun\master.log`
- `.tmp_tok\relay_state\watch_relay_log.txt`
- `.tmp_tok\relay_state\watch_relay_decision.txt`

If `watch_relay.ps1` is enabled, it will:

1. Write the status once every 15 minutes
2. Read `.tmp_tok\relay_state\layver_result.txt`
3. Automatically start the relay when the criterion passes
4. Automatically do archiving, verification and bit-level comparison after the relay ends

To start the watcher in the background manually:

```powershell
Start-Process powershell -ArgumentList @(
  '-NoProfile',
  '-ExecutionPolicy','Bypass',
  '-File','tools\relay\watch_relay.ps1',
  '-From','0','-To','4',
  '-RefDir','.tmp_tok\chain_ref'
) -WorkingDirectory <repo root> -WindowStyle Hidden
```

Or use the lightweight progress monitor:

```powershell
Start-Process powershell -ArgumentList '-NoProfile','-File','tools\relay\_monitor.ps1' -WindowStyle Hidden
Get-Content .tmp_tok\relay_rerun\_status.txt     # refreshed automatically every minute
```

## 8. How to do numeric-equivalence acceptance

If the question you need to answer is:

> "Although some boot output is bit-level different, is the subsequent lay still numerically equivalent?"

then you have to run `layver_run.ps1`; you cannot just look at `BOOT=PASS`, nor compare for byte equality.

Usage:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\layver_run.ps1 `
  -TargetLayer 1 `
  -AltDir .tmp_tok\chain\_alt_in `
  -BaselineLog .tmp_tok\relay_rerun\lay1.out `
  -NT 4
```

What it does is:

1. Back up the current `u{L-1}r112*.ct` and `u{L}_*.ct`
2. Overwrite the chain directory with the alternative `u{L-1}r112*.ct` from `-AltDir`
3. Run `lay{L}` with `T23_E2EMODE=1` and `T23_NT=-NT`
4. Compute the statistics of the new `max|err|`
5. Compare against the baseline log
6. Finally restore the original state

The result file is in:

```text
.tmp_tok\relay_state\layver_result.txt
```

Pass criteria:

- `RESULT=PASS`
- and `NEW max|err|` is in the same order of magnitude / within tolerance as the baseline

> **⚠️ `-TargetLayer` must be ≥ 1.**
> `lay0` takes the "encrypt the plaintext seed" branch and **reads no ciphertext from the chain at all**; injecting the alternative `u0r112` and then running `lay0`
> verifies nothing at all (the statistics on both sides are necessarily identical). To accept `u0r112`, you have to run `lay1`.

## 9. How to do independent verification

`verify_layer.ps1` is a third-party criterion; it does not reuse the driver's own PASS criterion.

### 9.1 Checking the lay outputs

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\verify_layer.ps1 `
  -From 0 -To 4 -Mode lay
```

### 9.2 Checking the boot outputs

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\verify_layer.ps1 `
  -From 0 -To 4 -Mode boot
```

Default tolerance:

```text
Tol = 3e-2
```

Return code meaning:

- `0`: everything passed
- non-`0`: the number of failing layers

## 10. How to archive the results

After the relay finishes, use:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\collect_results.ps1 `
  -From 0 -To 4 `
  -LogDir .tmp_tok\relay_rerun `
  -Dest results\L0-4
```

It archives:

- `chain/u{L}_*.ct`
- `chain/u{L}r112_*.ct`
- `logs/lay{L}.out/.err`
- `logs/boot{L}.out/.err`
- `meta.txt`
- `timing.tsv`
- `master.log`
- `manifest.sha256`

## 11. How to package it for the next relay runner

If you want to hand a layer range to someone else to continue, use:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\pack_relay.ps1 `
  -From 0 -To 4 -Id yourname
```

Default output:

```text
.tmp_tok\relay_out\relay_L0-4_yourname_<timestamp>.zip
```

The package contains:

- `chain/`: the core handoff ciphertexts `u{L}r112_*.ct`
- `logs/`
- `timing.tsv`
- `meta.txt`
- `master.log`
- `manifest.sha256`
- `README.txt`
- `sk.bin` (included by default)

If you do not want to include the private key:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\pack_relay.ps1 `
  -From 0 -To 4 -Id yourname -NoKey
```

## 12. How to verify a received handoff package

After receiving the zip, use:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\verify_relay.ps1 `
  -Zip <package path>
```

It checks three things:

1. whether `manifest.sha256` matches the files inside the package one by one (and, in reverse, whether any files not listed were smuggled in)
2. whether `lay*.out` contains `RESULT=PASS`
3. whether `boot*.out` contains `BOOT=PASS` and enough `out np=2083`

Return codes:

- `0`: everything passed
- `1`: at least one check failed

## 13. Recommended standard operating order

It is recommended to always follow the order below; it is the least likely to miss a step:

1. If there was a cross-platform export, run `tools/relay/fix_crlf.sh` first
2. Confirm that `.tmp_tok/chain/sk.bin`, `.tmp_tok/l0/*` and the plaintext reference are all present
3. Set `T23_NT`
4. Run `tools/relay/rerun5_lay_boot.ps1`
5. Run `tools/relay/verify_layer.ps1 -Mode lay`
6. Run `tools/relay/verify_layer.ps1 -Mode boot`
7. Run `tools/relay/collect_results.ps1`
8. If a handoff is needed, then run `tools/relay/pack_relay.ps1`
9. After the receiver gets the package, they run `tools/relay/verify_relay.ps1`

## 14. Common misconceptions

### 14.1 Misconception: `BOOT=PASS` means the numbers passed

No.  
`BOOT=PASS` only means the boot driver itself did not fail at `load / refresh / save`.

### 14.2 Misconception: different thread counts but PASS means bit-identical

Neither is this true.  
This round has already shown that:

- `NT=4` and `NT=8` can both PASS
- but the output `.ct` files can still be bit-level different

### 14.3 Misconception: keeping only the `.ct` files is enough

It is not enough.  
A relay also needs at least:

- the corresponding logs
- `meta.txt`
- `timing.tsv`
- `manifest.sha256`

Otherwise nobody else can independently judge how far you actually got and what criterion you used.

### 14.4 Misconception: after injecting an alternative input, you can pick any layer to run and still accept equivalence

No.  
The layer number is bound to the input it reads: `lay{L}` reads `u{L-1}r112`. Picking the wrong layer (for example taking `u0r112` to run `lay0`)
yields a false pass in which "both sides are exactly the same".

## 15. Minimal command list

If you want to memorize the fewest commands, this set is enough:

```powershell
$env:T23_NT = "4"

powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\rerun5_lay_boot.ps1 `
  -From 0 -To 4 `
  -LayExe .tmp_tok\t23lay.exe `
  -BootExe .tmp_tok\t23boot.exe `
  -LogDir .tmp_tok\relay_rerun

powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\verify_layer.ps1 -From 0 -To 4 -Mode lay
powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\verify_layer.ps1 -From 0 -To 4 -Mode boot

powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\collect_results.ps1 `
  -From 0 -To 4 -LogDir .tmp_tok\relay_rerun -Dest results\L0-4

powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\pack_relay.ps1 `
  -From 0 -To 4 -Id yourname
```
