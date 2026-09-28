[简体中文](README.md) | **English**

# Kestrel FHE Ciphertext-Chain Relay · Call for Community Compute

> **This is a coordination repository**: it holds the technical documentation and the data index only
> (the outreach / announcement material is not part of it).
> **The code is not here** — the drivers and the engine sources live in one single place, the main
> `kestrel-llm` repository; the pinned version is in [`SOURCE.en.md`](SOURCE.en.md).
> **The data is not here** — two hand-off archives (head `L0-4` 158.75 MB, tail `L26-27-fin` 64.99 MB, both
> **as downloaded**) are
> attached to https://github.com/m13253246268-ship-it/kestrel-fhe-relay/releases ; see [`data/README.en.md`](data/README.en.md) for verification.
> **You do not have to build the tools either** — prebuilt Windows x86-64 binaries (0.53 MB, no MinGW
> needed) sit in the same Releases.
>
> **Gitee mirror**: this repository is mirrored at <https://gitee.com/pei-xiaoguang/fhe-relay>.
> There, the **tail archive `L26-27-fin` and the prebuilt tools** are also attached; the **head archive
> `L0-4` remains GitHub-only** — Gitee caps release attachments at **100 MB per file** (GitHub allows
> 2 GB per asset), and that package is 158.75 MB. It is deliberately **not split into parts**, because
> splitting would invalidate this repo's "single-zip SHA256" verification anchor.
> **The head is not a mandatory download, though**: it is a deterministic product and can be reproduced
> locally by running `lay0..boot4` from the seed (about 9 h) — see [`data/README.en.md`](data/README.en.md) §5.1.

---

## 1. What this is

A from-scratch **RNS-CKKS** engine in **pure C11 with zero third-party dependencies, pure CPU**
(no GPU), used to run Qwen3-VL-2B (28 layers) **entirely on ciphertexts** and produce logits:
input and intermediate activations stay encrypted, so the serving side never sees plaintext.

- Engine and driver sources: the main `kestrel-llm` repository (`src/core/vllm_ntt.c` / `vllm_ckks.c` /
  `vllm_tp.c` plus `tools/drivers/`)
- Current parameters: `n=2048`, `slots=1024`, `scale=2^60`; `lay` uses 112 RNS primes, `boot` uses 2100
- Verified: x86-64 and RK3588 (aarch64) agree **coefficient-by-coefficient, bit-exactly**

---

## 2. Where we actually are (honest version)

| Item | Status |
|---|---|
| Layers done | **Layers 0–4 (5 layers)**: every `lay` hop reports `RESULT=PASS`, every `boot` hop reports `BOOT=PASS` (refreshed back to the full chain, `out np=2083`) |
| Tail branch (a separate line, **not a continuation**) | **Layers 26–27 + the output head `fin`**: encrypted directly from a plaintext seed into L26, **bypassing layers 5–25**; all five steps `PASS`, and `fin`'s logits top1 matches the reference **4/4** (measured 2026-09-15, **16284 s ≈ 4.52 h** in total) |
| Not done on the chain | **Layers 5–27 (23 layers)** — the relay chain still stops at `u4r112`; the tail branch **cannot substitute** for continuing that segment on the chain |
| Per-hop cost (measured) | `lay` **1942–2008 s** (32.4–33.5 min); `boot` **4339–4891 s** (72.3–81.5 min) |
| Per-layer cost (measured) | about **105–115 min ≈ 1.8 h per layer** (`lay` + `boot`) |
| Layers 0–4 total (measured) | 10 hops, **32182 s ≈ 8.94 h** (5 `lay` hops + 5 `boot` hops) |
| Full-chain projection | **≈50 h** (28 × 1.8 h; a projection, **not a measurement**) |
| Can all 28 layers run through? | **Not yet verified** — that is exactly the problem this call is about |

> The per-hop ranges above come from this round's full seed rerun (2026-09-14 → 09-15); the raw data is in
> [`docs/03_Measured_Data_Appendix_EN.md`](docs/03_Measured_Data_Appendix_EN.md) §2. Hardware:
> AMD Ryzen 7 9800X3D (8C/16T), fixed at `T23_NT=4`, Windows 11 24H2, MinGW-W64 gcc 16.1.0.

Independent re-check with `verify_layer` (decrypt → decode → `max|err|` against the plaintext reference,
tolerance `3e-2`): **5/5 `ALL_LAYERS_VERIFY_PASS`** in both `-Mode lay` and `-Mode boot`.

---

## 3. Why a relay is needed

About 50 hours of pure-CPU compute on a single machine. Split into "a few layers per person", it finishes in days.
**Each layer's hand-off is only 8 ciphertext files `u{L}r112_{t}_{h}.ct` (about 28 MB)** — anyone who has them
can continue from `L+1`.

---

## 4. How to help (three levels of commitment, pick one)

- [ ] **Claim 1 layer** (≈1.8 h) — the easiest way to take part
- [ ] **Claim 3 layers** (≈5.5 h)
- [ ] **Claim a whole segment** (e.g. 8–10, or 25–27 + `fin`)

**Steps** (full version in [`docs/04_Relay_Reproduction_Guide_EN.md`](docs/04_Relay_Reproduction_Guide_EN.md)):

1. Prepare an x86-64 or ARM64 machine with ≥4 logical cores and ≥8 GB RAM, plus gcc;
2. Clone the main `kestrel-llm` repository (**at the version pinned in** [`SOURCE.en.md`](SOURCE.en.md)) and
   prepare the data package (about 7 GB);
3. Build `t23lay` / `t23boot` as described in the guide, §2;
4. Run the layers you claimed:
   ```powershell
   powershell -NoProfile -ExecutionPolicy Bypass -File tools\relay\rerun5_lay_boot.ps1 -From 5 -To 7
   ```
5. Pack and submit with one command: `pack_relay.ps1 -From 5 -To 7 -Id <your-id>`, then host the resulting
   zip on **your own Release or cloud drive** and reply in the issue with the **download link plus the
   zip's SHA256** (1 layer ≈ 28 MB);
   ⚠️ **do not attach the zip itself** — GitHub caps issue/discussion attachments at **25 MB** (a one-layer
   package is 28 MB), and intermediaries such as email re-encode bytes, which breaks every entry in
   `manifest.sha256`. **A link is the byte-faithful way to hand it over**;
6. The next hand takes over at `L+1`; **anyone** can check a received package with a single command
   `verify_relay.ps1 -Zip <package>` (integrity + judging criteria + structure, exit code 0/1);
7. **To check whether the numbers are actually right**: use `verify_layer.ps1 -From L -To L` to decrypt
   and compare element-wise against the plaintext reference (a separate code path, it does not reuse the
   driver's own verdict); for critical layers, have a second person recompute independently and then compare
   byte-by-byte with `verify_layer.exe -BitA/-BitB`.

> **To claim layers, please open an issue** titled `[claim] layer N`, so the schedule stays visible.

---

## 5. Where the code and the data live

| Content | Location |
|---|---|
| Chain drivers `t23_m3p.c` / `t23_chain.c` | main repo, `tools/drivers/` |
| Relay scripts (run / pack / verify / accuracy check) | main repo, `tools/relay/` |
| Engine sources `vllm_ntt.c` / `vllm_ckks.c` / `vllm_tp.c` | main repo, `src/core/` |
| Data-preprocessing scripts | main repo, `tools/preproc/` |
| **Head archive `L0-4` (158.75 MB / 105 files)** | this repo's https://github.com/m13253246268-ship-it/kestrel-fhe-relay/releases — **GitHub only** (exceeds Gitee's 100 MB attachment cap), see [`data/README.en.md`](data/README.en.md) |
| **Tail archive `L26-27-fin` (64.99 MB / 46 files)** | GitHub as above; **also on Gitee** at <https://gitee.com/pei-xiaoguang/fhe-relay/releases> . **It is a tail branch that bypasses layers 5–25** — do not read it as the whole chain |
| **Prebuilt tools (Windows x86-64, 0.53 MB / 7 files)** | GitHub as above; **also on Gitee** at <https://gitee.com/pei-xiaoguang/fhe-relay/releases> . `t23lay` / `t23boot` / `verify_layer`, statically linked, no MinGW required. **Note: these three binaries remain AGPL-3.0-or-later** — see §10 |
| Full run data package (about 7 GB: `sk.bin` + weights + plaintext references) | **not shipped with the repository**; ask in an issue, or build it with the main repo's `tools/preproc/` |

> Size units: this repository follows the common convention of writing **MB for MiB** (1 MB = 1,048,576 B,
> matching how GitHub displays sizes); the **exact byte counts** of the three release assets are in
> [`data/README.en.md`](data/README.en.md) §1.

`sk.bin` (the secret key) must come from the same source as the weights/references — **do not mix them**.

---

## 6. Judging criteria (read this first to avoid misreading the logs)

The logs contain both segment-level `FAIL` and trailing `RESULT=PASS` — **this is not a contradiction**:
segments A/B/C/D are compared against a plaintext reference that has *not* drifted, while a ciphertext chain
necessarily accumulates drift between layers. **The layer-output verdicts (`RESULT=PASS` / `BOOT=PASS`) are
the criteria.**

Two more easy-to-get-wrong points:

1. **Neither `RESULT=PASS` nor `BOOT=PASS` means the numbers passed.** They only say the flow completed with
   no hard error (`BOOT=PASS` additionally confirms the refresh really did return to the full chain,
   `out np=2083`). The relay script sets `T23_E2EMODE=1`, which downgrades mid-pipeline deviations to
   report-only (marked `[E2E:ref-deviate-ok]`), so **a shallow layer on the wrong SiLU path still prints
   PASS** (measured: layer 0 with `silu0.bin` missing gives `max|err|` = 4.2e-2 ~ 8.5e-2 yet still reports
   `RESULT=PASS (0)`). **Every layer must additionally be checked with `verify_layer.ps1` for `max|err|`**,
   see the guide §7.
2. **The accuracy margin is thin.** The tightest layer so far (layer 2) has `max|err| = 2.8372e-02`, only about
   5.4% of margin against the `3e-2` tolerance. After taking over, please record the per-layer `max|err|`
   trend and report any regression.

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

## 7. The one and only thing we claim

**Layers 0–4 are reproducible and independently verifiable on pure CPU**: every `lay` hop `RESULT=PASS`,
every `boot` hop `BOOT=PASS`, with a **SHA256 manifest** of the hand-off artefacts for byte-level checking.

**The tail (layers 26–27 + `fin`) also computes correctly**: encrypted directly from a plaintext seed into L26,
all five steps `PASS`, and `fin`'s logits top1 matches the reference **4/4**. But that is **another branch**
that **bypasses layers 5–25**; it does **not** mean "the whole chain runs from layer 0 through layer 27".

"The full 28-layer chain" is the **relay goal**, **not** an existing result of this project. Any result for
layer 6 and beyond produced by someone who takes over the relay is **their contribution**, and must not be
back-filled as an existing result of this project.

---

## 8. Explicit boundary statements (please do not quote them as something else)

1. **No cryptographic security claim**: the current parameters are **mechanism-validation grade**, far below
   128-bit (which needs `n ≥ 16384` or equivalent);
2. The RNG is a prototype; a CSPRNG interface is reserved but not yet wired in;
3. **The weights are plaintext constants** — what this chain protects is the **input and the intermediate
   activations**;
4. **Performance is not a selling point** — single-token latency is hours; offline batch only;
5. Layer indices beyond 5 are **extrapolated**, not measured;
6. The **thread count is a result variable**: `T23_NT=4` and `T23_NT=8` produce **bit-different** ciphertexts
   (the RNG offsets with scheduling). For the relay, **use `T23_NT=4` uniformly**, otherwise the hand-off
   artefacts cannot be compared byte-by-byte.

---

## 9. Document index

| Document | Content |
|---|---|
| [`SOURCE.en.md`](SOURCE.en.md) | **Code version anchor**: source SHA256, the bit-exactness constraint, and why this repo does not copy the sources |
| [`data/README.en.md`](data/README.en.md) | The three Release assets (head `L0-4` / tail `L26-27-fin` / tools `tools-win-x64`): what they contain and how to verify them |
| [`docs/00_Status_of_Completed_Layers_EN.md`](docs/00_Status_of_Completed_Layers_EN.md) | Per-item status of the completed layers (incl. an encoding-pitfall lesson) |
| [`docs/02_Relay_Operations_Manual_EN.md`](docs/02_Relay_Operations_Manual_EN.md) | Tool-by-tool operating manual: parameters and step-by-step procedures |
| [`docs/03_Measured_Data_Appendix_EN.md`](docs/03_Measured_Data_Appendix_EN.md) | **All raw measured data** (per-hop wall clock, `boot` section profile, `max|err|`, SHA256 of 80 ciphertexts) |
| [`docs/04_Relay_Reproduction_Guide_EN.md`](docs/04_Relay_Reproduction_Guide_EN.md) | Full reproduction steps, from zero to one completed layer |
| [`docs/05_Tool_Reference_EN.md`](docs/05_Tool_Reference_EN.md) | Tool inventory and per-parameter reference |

Every document above has a Chinese counterpart in the same directory — see [`README.md`](README.md) §9.

> The numbering skips `01` and `06`: those two are **outreach / announcement material** (a discussion
> summary and the five-platform relay post) rather than technical deliverables, and are **not part of this
> repository**.

---

## 10. Contact and licence

**Contact**: please use **Issues** first (claim a layer / request the data package / report a problem).

**Licence (this repository)**: **MIT** — this repository's docs, indices, manifests and the two hand-off
archives may be freely used, copied, modified, distributed, sublicensed and sold, provided the copyright
and licence notice are kept. See [`LICENSE`](LICENSE).

**Two exceptions — do not misread them** (details in [`LICENSING.md`](LICENSING.md)):

1. The **`t23lay.exe` / `t23boot.exe` / `verify_layer.exe`** inside
   `releases/kestrel-fhe-relay_tools_win-x64_*.zip` are built from the **AGPL-3.0-or-later sources** of the
   main repository and therefore **remain under AGPL-3.0-or-later**; this repository's MIT does not
   relicense them (the Corresponding Source is the commit pinned in [`SOURCE.en.md`](SOURCE.en.md)).
2. **The main repository `kestrel-llm`'s code** stays under its own licence
   (dual-licensed AGPL-3.0-or-later / commercial); this repository's MIT **does not alter or extend to it**.
