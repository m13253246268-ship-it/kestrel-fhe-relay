[简体中文](00_已完成层状态说明.md) | **English**

# Status of Completed Layers (Layers 0–4)

> Data as of **2026-09-15 08:45 (local time)**
> Machine: AMD Ryzen 7 9800X3D (8C/16T) · Windows 11 24H2 · MinGW-W64 gcc 16.1.0 · `T23_NT=4`
> Parameters: `n=2048`, `slots=1024`, `scale=2^60`, layer chain `LCHAIN=112` 60-bit primes; `boot` refreshes back to `out np=2083`

---

## 0. One-line conclusion

**All 10 hops of layers 0–4 are finished, and both the process criterion and the numeric criterion pass. This is the first time this chain has run end to end.**

The handoff artifact `u4r112` is complete at 8/8 and is ready for layer 5.

| Item | Status |
|---|---|
| Process criterion for all 10 hops (`RESULT=PASS` / `BOOT=PASS`) | ✅ **10/10 passed** |
| Did `boot` really return to the full chain (`out np=2083` × 8)? | ✅ Yes, every layer |
| Independent numeric check (`verify_layer`, `lay` outputs) | ✅ **5/5 ALL_LAYERS_VERIFY_PASS** |
| Independent numeric check (`verify_layer`, `boot` refreshed outputs) | ✅ **5/5 ALL_LAYERS_VERIFY_PASS** |
| Bit-level comparison (`u0` vs reference lineage) | ✅ **same=8 diff=0 missing=0** |
| Archive | ✅ 104 files + `manifest.sha256` |
| Effective compute time (10 hops) | **32182 s ≈ 8.94 h** |
| `NT=8` numeric equivalence (corrected run) | ✅ **passes**: bytes differ, decrypted values agree (see §4.1) |

---

## 1. This round: full rerun from the seed (log dir `.tmp_tok/relay_gh/`)

| Hop | Criterion | Wall clock | Completed at |
|---|---|---|---|
| `lay0` | `RESULT=PASS` | 1962 s (32.7 min) | 09-14 15:16:11 |
| `boot0` | `BOOT=PASS` | 4377 s (73.0 min) | 09-14 16:29:08 |
| `lay1` | `RESULT=PASS` | 2008 s (33.5 min) | 09-14 17:02:35 |
| `boot1` | `BOOT=PASS` | 4891 s (81.5 min) | 09-15 03:10:24 |
| `lay2` | `RESULT=PASS` | 1942 s (32.4 min) | 09-15 03:42:46 |
| `boot2` | `BOOT=PASS` | 4350 s (72.5 min) | 09-15 04:55:17 |
| `lay3` | `RESULT=PASS` | 1949 s (32.5 min) | 09-15 05:27:46 |
| `boot3` | `BOOT=PASS` | 4339 s (72.3 min) | 09-15 06:40:05 |
| `lay4` | `RESULT=PASS` | 1972 s (32.9 min) | 09-15 07:12:57 |
| `boot4` | `BOOT=PASS` | **4392 s (73.2 min)** | **09-15 08:26:10** |

- **Effective compute time: 32182 s ≈ 8.94 h.**
- The wall-clock span is longer than that (09-14 14:43 → 09-15 08:26) because the run was interrupted once and
  the watcher resumed it automatically at **01:48:53** (it skipped the already-complete `lay0/boot0/lay1` and
  continued from `boot1`). That resumed invocation alone took **23837 s ≈ 6.62 h** of wall clock.
- Every `lay` hop lands in **32–34 min**, every `boot` hop in **72–82 min**, i.e. **≈ 1.6–1.9 h per layer**.

`boot` stage profile (each refresh really does return to the full chain, `out np=2083`):

| Hop | coeff_to_slot | slot_to_coeff | sin_fold(re+im) | all others | total |
|---|---|---|---|---|---|
| `boot0` | 1699.54 s | 1701.60 s | 863.20 s | 32.45 s | 4296.79 s |
| `boot1` | 1921.41 s | 1901.37 s | 953.72 s | 34.62 s | 4811.12 s |
| `boot2` | 1696.82 s | 1684.42 s | 858.53 s | 31.84 s | 4271.61 s |
| `boot3` | 1693.72 s | 1680.53 s | 857.48 s | 31.35 s | 4263.08 s |
| `boot4` | 1717.49 s | 1700.39 s | 862.28 s | 32.20 s | 4312.36 s |

> Takeaway: **≈ 79%** of `boot` time sits in `coeff_to_slot` + `slot_to_coeff` (about 29 min each).
> The two `sin_fold` stages add up to about 14 min, and `modraise / rotate_k / conj_extract /
> restore_merge` together amount to only tens of seconds.

---

## 2. The previous round (log dir `.tmp_tok/relay/`, 2026-09-14)

Same machine, same `T23_NT=4`, but **only 9 hops completed**:

| Hop | Wall clock | Hop | Wall clock |
|---|---|---|---|
| `lay0` | 1992 s | `boot0` | 3873 s |
| `lay1` | 1926 s | `boot1` | 3890 s |
| `lay2` | 2039 s | `boot2` | 3978 s |
| `lay3` | 2025 s | `boot3` | 4337 s |
| `lay4` | 1857 s | `boot4` | **started but never finished** |

**Key difference**: the previous round's `boot4` started but never completed, so its `u4r112` was incomplete.
This round reran the entire chain from the seed precisely to close that gap — and it now is closed.

Comparing `boot` cost across the two rounds: this round 4339–4891 s vs the previous 3873–4337 s —
this round is 1%–26% slower (`boot1` the worst). That matches "the same machine drifts by roughly 10%
between sessions"; it is not a change of code or of the judging criterion.

---

## 3. Artifacts and where they live

| What | Path | State |
|---|---|---|
| Ciphertexts on the chain (lay/boot outputs) | `.tmp_tok/chain/u{L}_{t}_{h}.ct`, `u{L}r112_{t}_{h}.ct` | **`u0..u4` and `u0r112..u4r112` all 8/8** |
| Hop-by-hop timeline | `.tmp_tok/relay_gh/master.log` | complete (contains `rerun END`) |
| Per-layer timing summary | `.tmp_tok/relay_gh/timing.tsv` | written |
| Machine/compiler metadata | `.tmp_tok/relay_gh/meta.txt` | present |
| Raw criterion logs | `.tmp_tok/relay_gh/{lay,boot}{0..4}.out/.err` | present |
| Archived bundle | Release asset (104 files + `manifest.sha256`), documented in [`data/README.en.md`](../data/README.en.md) | ✅ produced |

Convention: `t=0..3` (4 tokens) × `h=0..1` (2 ciphertext components) = 8 `.ct` files per hop.
A `u{L}` ciphertext has burned down to `np≈15–20`; `u{L}r112` is refreshed back to `np=112`
(3670028 bytes ≈ 3.5 MB per file).

> Note: the archive originally landed in `arxiv\results\L0-4\`. Since 2026-09-15 it is distributed as a
> **Release asset** of this repository instead of a directory in the repo, so see
> [`data/README.en.md`](../data/README.en.md)
> (all 104 files verified byte-for-byte against `manifest.sha256`).
> Its `README.md` is still an **unfilled template**; every number it needs is collected in
> [`03_Measured_Data_Appendix_EN.md`](./03_Measured_Data_Appendix_EN.md) (§2–§9).

---

## 4. Verification status: what is closed and what is not

| Item | Method | Status |
|---|---|---|
| Process criterion | `lay` → `RESULT=PASS`; `boot` → `BOOT=PASS` + `out np=2083` | ✅ **10/10 passed** |
| Numeric criterion (`lay` outputs) | `verify_layer.ps1 -Mode lay`: independent decrypt → decode → `max\|err\|` vs the plaintext reference | ✅ **5/5 PASS** |
| Numeric criterion (`boot` refreshed outputs) | `verify_layer.ps1 -Mode boot` (same, on `u{L}r112`) | ✅ **5/5 PASS** |
| Bit-level comparison | per-file SHA256 against a reference lineage | ✅ `same=8 diff=0 missing=0` |
| Numeric equivalence of the NT=8 refreshed output | `layver_run.ps1 -TargetLayer 1` plus a direct input-level decrypt comparison | ✅ **passes (see §4.1)** |

**Per-layer numeric results** (tolerance `3e-2`):

| Layer | `lay` output `max\|err\|` | `boot` refreshed output `max\|err\|` | Margin |
|---|---|---|---|
| 0 | 1.2712e-03 | 1.2706e-03 | comfortable |
| 1 | 3.1467e-03 | 3.1460e-03 | comfortable |
| 2 | **2.8372e-02** | **2.8372e-02** | **only 5.4%** |
| 3 | 1.2285e-02 | 1.2285e-02 | moderate |
| 4 | 1.3823e-02 | 1.3823e-02 | moderate |

> Result: `ALL_LAYERS_VERIFY_PASS` (5/5 in each mode).
> But **layer 2 has only 5.4% margin left**, which is a real problem: as deeper layers accumulate error the
> tolerance may be breached. That is exactly why, from layer 6 onward, the `h=1` component's `max|err|`
> trend must be tracked layer by layer.

### 4.1 NT=4 vs NT=8 numeric equivalence (corrected run — passes)

This was the last open item, and it is now closed. **Conclusion: not bit-identical, but numerically equivalent.**

**Evidence 1: decrypting the two inputs directly (the strongest evidence)**

Both `u0r112` versions were decrypted with the same `sk.bin` and the same plaintext reference
`.tmp_tok/tail/l0_u2_ref.bin`:

| token/component | NT=4 lineage `max\|err\|` | NT=8 output `max\|err\|` |
|---|---|---|
| t0 h0 | 3.2984e-04 | 3.2984e-04 |
| t0 h1 | 7.9285e-04 | 7.9285e-04 |
| t1 h0 | 9.8160e-04 | 9.8160e-04 |
| t1 h1 | 9.6113e-04 | 9.6113e-04 |
| t2 h0 | 1.2307e-03 | 1.2307e-03 |
| t2 h1 | 1.2706e-03 | 1.2706e-03 |
| t3 h0 | 8.7015e-04 | 8.7015e-04 |
| t3 h1 | 9.7758e-04 | 9.7758e-04 |
| **overall** | **PASS 8/8, max_err=1.2706e-03** | **PASS 8/8, max_err=1.2706e-03** |

At the same time, the **SHA256 of all 8 ciphertext pairs differ** (about 98.9% differing bytes).

In other words: **two ciphertexts with completely different bytes decrypt back to the same real slots.**
This matches the root cause in §2.4 — the thread count only changes the bootstrap masks (the ciphertext
representation), not the underlying plaintext.

**Evidence 2: a downstream layer consuming the alternative input**

`layver_run.ps1 -TargetLayer 1` injected the NT=8 `u0r112` into the chain and ran `lay1`:

- `RESULT=PASS`, `wall=2118s`;
- the 40 `max|err|` statistics are **identical digit for digit** to the NT=4 baseline:
  `max=1.917E-002`, `min=1.294E-003`, `mean=8.431E-003`.

**Boundaries that must be stated**

- This equivalence holds **within the tolerance convention and across one `lay1` hop**. It is not a proof
  that all 28 layers are equivalent; noise accumulation over many hops still has to be watched.
- Evidence 1 alone already supports the conclusion and does not depend on whether the injection was read;
  evidence 2 serves as a downstream cross-check.

### 4.2 Criterion boundaries that must be stated

- `BOOT=PASS` **only means the boot flow (load / refresh / save) did not fail. It does not mean the numbers are right.**
- Stage-level `max|err| FAIL` lines in groups `A/B/C/D` **are not the criterion**: they compare against a
  plaintext reference from a *non-drifting* input, whereas a real ciphertext chain necessarily carries drift
  from the previous layer. Normalization pulls the magnitude back, which is why the final stage is tagged
  `[E2E:ref-deviate-ok]`.
- So "10/10 process pass" answers "it runs", and "5/5 numeric pass" answers "it is right".
  **Both** are now in hand — but the conclusion still only covers layers 0–4.

---

## 5.1 Another trap that has to be recorded: a "double BOM" made the automatic verification fail silently

When the watcher did its closing two steps (archive → independent verification), the archive succeeded but
**both `verify_layer` invocations failed**, with a PowerShell parse error
(`赋值表达式无效 … verify_layer.ps1:19`).

The root cause was not the script logic but **file encoding**:

- `verify_layer.ps1` began with `EF BB BF EF BB BF` — **two BOMs**;
- PowerShell 5.1 consumes only the first BOM, so the second becomes a U+FEFF in the body that derails the
  parse of the `param(...)` block;
- the script never even reached syntax-valid state, so `verify_layer` never actually ran.

**What makes this class of failure dangerous is that it looks like it ran**: the log file has content
(the content being an error). Without reading it line by line, it is easy to mistake it for "verification done".

Remediation:

1. Normalize every relevant `.ps1` to **exactly one BOM** (pure-ASCII scripts get none).
2. Change the validation to **mimic PowerShell 5.1's real read path** (decode as UTF-8 when a BOM is present,
   otherwise decode as ANSI/GBK) and then try to build a scriptblock. This is far stricter than
   `Parser::ParseFile`, which still accepts a double BOM — so the earlier "8/8 parse OK" check was unreliable.
3. After the fix and a rerun, both `lay` and `boot` modes produced `ALL_LAYERS_VERIFY_PASS`.

**Lesson**: a tool's encoding is part of the deliverable. A `.ps1` containing Chinese must be UTF-8 with BOM,
and there must be exactly one BOM; otherwise it fails silently on someone else's machine.

---

## 5. A correction that has to be recorded: the first `layver` experiment was a no-op

An earlier "numeric equivalence check" concluded that "the `u0r112` refreshed at `NT=8` is numerically
equivalent to the `NT=4` baseline". **That conclusion does not hold and is withdrawn.** The wrong layer was chosen:

- In the driver, `lay{L}` reads `chain/u{L-1}r112`, and **only `lay >= 1` takes the "read ciphertext from the chain" branch**.
- `lay0` takes the "encrypt the plaintext seed" branch and **never opens any ciphertext from the chain**.
- So when the alternative `u0r112` was injected into the chain dir and `T23_LAY=0` was run, that ciphertext was
  never read. The log shows freshly created ciphertexts (`[rms A] in xa np=112 xb np=112`) and no load of `u0r112`.
- "NEW and BASE statistics match exactly" therefore only means both runs were the same computation.

**The correct approach**: to check `u0r112`, run `lay1`. The corrected experiment is queued: once archiving and
numeric verification finish, it will run automatically (feed the `NT=8` `u0r112` into `lay1`, then compare
`max|err|` against the `NT=4` lineage's `lay1` baseline).

A guard is now built into the tool: `tools/relay/layver_run.ps1` requires `-TargetLayer >= 1` and exits with an
error if you pass 0.

---

## 6. What can and cannot be stated publicly

**Can be stated**

- **Layers 0–4 have now run end to end from the seed**: the `lay` / `boot` criteria pass hop by hop, and every
  `boot` really does return to the full modulus chain (`out np=2083` × 8).
- **The numbers pass independent verification too**: `verify_layer` reports 5/5 `ALL_LAYERS_VERIFY_PASS` in both
  `lay` and `boot` modes (tolerance `3e-2`), and `u0` is byte-identical to the reference lineage (`same=8 diff=0`).
- **NT=8 and NT=4 are not bit-identical but are numerically equivalent**: the SHA256 of all 8 `u0r112` pairs differ,
  yet after decryption the per-element `max|err|` values are identical (`max_err=1.2706e-03`, 8/8 PASS), and `lay1`
  consuming the alternative input also returns `RESULT=PASS` with digit-identical statistics.
- Measured per-layer cost: `lay` ≈ 33 min, `boot` ≈ 72–82 min, i.e. **≈ 1.6–1.9 h per layer** (4 threads, CPU only);
  5 layers / 10 hops total **≈ 8.9 h**.
- The bulk of that time sits in `coeff_to_slot` and `slot_to_coeff` (**79.2%** combined); the other
  20.0% is the two `sin_fold` stages, and the remaining five stages total only 0.7%
  (see [`03_Measured_Data_Appendix_EN.md`](./03_Measured_Data_Appendix_EN.md) §3).

**Cannot be stated**

- ❌ "The full 28-layer chain runs" — only the first 5 layers exist; the rest is extrapolation.
- ❌ "Error margin is comfortable" — layer 2 has only 5.4% margin, and whether deeper layers still pass is untested.
- ❌ "The whole chain is numerically equivalent" — the equivalence shown holds only within the tolerance convention
  and across one `lay1` hop.
- ❌ No claim of cryptographic security strength or of performance superiority: the current parameters are
  mechanism/correctness-verification level; the weights are plaintext constants, and what this chain protects is
  the inference input and intermediate activations.

---

## 7. Next steps

1. ✅ `boot4` finished → `u4r112` at 8/8 → layers 0–4 complete for the first time.
2. ✅ Archive (104 files + manifest), independent numeric verification (lay/boot 5/5 each), bit-level comparison (8/8 identical).
3. ✅ The corrected `lay1` numeric-equivalence experiment passes (`RESULT=PASS`, statistics digit-identical to the baseline).
4. ✅ The template tables in the archive's `README.md` have been filled from
   [`03_Measured_Data_Appendix_EN.md`](./03_Measured_Data_Appendix_EN.md) (per-layer criteria/wall clock/`max|err|`,
   cross-build bit-level comparison, cost structure, boundary statements).
5. TODO: from layer 5 onward, record the `h=1` component's `max|err|` trend every layer (layer 2's margin is only 5.4%).
6. TODO: the cross-build (repository release build vs the local `src/` development tree) bit-level comparison for the handoff artifact `u0r112` has not been done;
   it requires rerunning `boot0` once (≈73 min).
