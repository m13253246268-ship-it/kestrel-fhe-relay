[简体中文](03_实测数据附录.md) | **English**

# Measured Data Appendix (Layers 0–4 + tail branch L26/27 + `fin`)

> This file is the data source for documents `00`/`01`. Every number here is measured; nothing is extrapolated.
> **Run**: full seed rerun · log dir `.tmp_tok/relay_gh/` · 2026-09-14 14:43 → 2026-09-15 08:26
> **Archive**: two **Release assets** of this repository — head `L0-4` (104 files + `manifest.sha256`) and
> tail `L26-27-fin` (45 files + `manifest.sha256`); distributed via Releases since 2026-09-15, no longer
> directories in the repo — see [`data/README.en.md`](../data/README.en.md)
> **Machine**: AMD Ryzen 7 9800X3D (8C/16T) · Windows 11 24H2 · MinGW-W64 gcc 16.1.0 · `T23_NT=4`

---

## 1. Runtime environment and compile settings

From `.tmp_tok/relay_gh/meta.txt`:

```text
host       : CHINAMI-JDEGI3P
os         : Microsoft Windows NT 10.0.26100.0
cpu        : AMD Ryzen 7 9800X3D 8-Core Processor
cores_phys : 8  cores_log : 16
threads    : T23_NT=4
cc         : gcc.exe (MinGW-W64 x86_64-ucrt-posix-seh, built by Brecht Sanders, r2) 16.1.0
compile    : lay CKKS_NPRIMES=112; boot CKKS_NPRIMES=2100; n=2048; scale=2^60
judge      : lay -> RESULT=PASS; boot -> BOOT=PASS
```

---

## 2. Hop-by-hop wall clock (10 hops)

| Hop | Criterion | Wall (s) | Wall (min) | Completed at |
|---|---|---|---|---|
| `lay0` | `RESULT=PASS` | 1962 | 32.7 | 09-14 15:16:11 |
| `boot0` | `BOOT=PASS` | 4377 | 73.0 | 09-14 16:29:08 |
| `lay1` | `RESULT=PASS` | 2008 | 33.5 | 09-14 17:02:35 |
| `boot1` | `BOOT=PASS` | 4891 | 81.5 | 09-15 03:10:24 |
| `lay2` | `RESULT=PASS` | 1942 | 32.4 | 09-15 03:42:46 |
| `boot2` | `BOOT=PASS` | 4350 | 72.5 | 09-15 04:55:17 |
| `lay3` | `RESULT=PASS` | 1949 | 32.5 | 09-15 05:27:46 |
| `boot3` | `BOOT=PASS` | 4339 | 72.3 | 09-15 06:40:05 |
| `lay4` | `RESULT=PASS` | 1972 | 32.9 | 09-15 07:12:57 |
| `boot4` | `BOOT=PASS` | 4392 | 73.2 | 09-15 08:26:10 |
| **Total** | **10/10** | **32182** | **536.4 (≈8.94 h)** | — |

Notes:

- `lay` spans 1942–2008 s (range 66 s); `boot` spans 4339–4891 s (range 552 s).
- The resumed segment recorded in `master.log`: `rerun END total wall=23837s` (≈6.62 h, 01:48:53 → 08:26:10).
- The run was interrupted once (idle gap 09-14 17:02:35 → 09-15 01:48:53), so the total wall-clock span exceeds the effective compute time.

---

## 3. `boot` stage profile (5 hops)

Units: seconds.

| Hop | modraise | coeff_to_slot | rotate_k | conj_extract | sin_fold_re | sin_fold_im | restore_merge | slot_to_coeff | total |
|---|---|---|---|---|---|---|---|---|---|
| `boot0` | 0.55 | 1699.54 | 12.35 | 7.75 | 430.65 | 432.55 | 11.80 | 1701.60 | 4296.79 |
| `boot1` | 0.60 | 1921.41 | 13.54 | 8.47 | 477.56 | 476.16 | 12.01 | 1901.37 | 4811.12 |
| `boot2` | 0.49 | 1696.82 | 12.49 | 7.65 | 428.78 | 429.75 | 11.20 | 1684.42 | 4271.61 |
| `boot3` | 0.52 | 1693.72 | 12.14 | 7.63 | 429.01 | 428.47 | 11.06 | 1680.53 | 4263.08 |
| `boot4` | 0.48 | 1717.49 | 12.61 | 7.75 | 431.74 | 430.54 | 11.35 | 1700.39 | 4312.36 |
| **mean** | **0.53** | **1745.80** | **12.63** | **7.85** | **439.55** | **439.49** | **11.48** | **1733.66** | **4390.99** |

Share of the total (by mean):

| Stages | Share |
|---|---|
| `coeff_to_slot` + `slot_to_coeff` | **≈ 79.2%** (about 29 min each) |
| `sin_fold_re` + `sin_fold_im` | ≈ 20.0% (about 7.3 min each) |
| the other 5 stages combined | ≈ 0.7% (about 32 s total) |

> Note: `total` is slightly larger than the wall clock in `boot{n}.out`; the difference is process start/stop and I/O overhead.

---

## 4. `verify_layer` per-layer numbers (tolerance `3e-2`)

### 4.1 `-Mode lay` (validating `u{L}`)

```text
mode=lay  layers=0..4  chain=.tmp_tok\chain  sk=.tmp_tok\chain\sk.bin  tol=0.03
layer 0  PASS     max|err|=1.2712e-03
layer 1  PASS     max|err|=3.1467e-03
layer 2  PASS     max|err|=2.8372e-02
layer 3  PASS     max|err|=1.2285e-02
layer 4  PASS     max|err|=1.3823e-02
RESULT: ALL_LAYERS_VERIFY_PASS  (5 layers, mode=lay, tol=0.03)
```

### 4.2 `-Mode boot` (validating `u{L}r112`)

```text
mode=boot  layers=0..4  chain=.tmp_tok\chain  sk=.tmp_tok\chain\sk.bin  tol=0.03
layer 0  PASS     max|err|=1.2706e-03
layer 1  PASS     max|err|=3.1460e-03
layer 2  PASS     max|err|=2.8372e-02
layer 3  PASS     max|err|=1.2285e-02
layer 4  PASS     max|err|=1.3823e-02
RESULT: ALL_LAYERS_VERIFY_PASS  (5 layers, mode=boot, tol=0.03)
```

### 4.3 Margin

| Layer | max\|err\| | Margin against `3e-2` |
|---|---|---|
| 0 | 1.2712e-03 | 95.8% |
| 1 | 3.1467e-03 | 89.5% |
| **2** | **2.8372e-02** | **5.4% (tightest)** |
| 3 | 1.2285e-02 | 59.0% |
| 4 | 1.3823e-02 | 53.9% |

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

## 5. Input-level decrypt comparison: NT=4 lineage vs NT=8 output (`u0r112`)

Same `sk.bin`, same plaintext reference `.tmp_tok/tail/l0_u2_ref.bin`, `tol=3e-2`.

| token/component | NT=4 lineage `max\|err\|` | NT=8 output `max\|err\|` | Diff |
|---|---|---|---|
| t0 h0 | 3.2984e-04 | 3.2984e-04 | 0 |
| t0 h1 | 7.9285e-04 | 7.9285e-04 | 0 |
| t1 h0 | 9.8160e-04 | 9.8160e-04 | 0 |
| t1 h1 | 9.6113e-04 | 9.6113e-04 | 0 |
| t2 h0 | 1.2307e-03 | 1.2307e-03 | 0 |
| t2 h1 | 1.2706e-03 | 1.2706e-03 | 0 |
| t3 h0 | 8.7015e-04 | 8.7015e-04 | 0 |
| t3 h1 | 9.7758e-04 | 9.7758e-04 | 0 |
| **overall** | **PASS 8/8, max_err=1.2706e-03** | **PASS 8/8, max_err=1.2706e-03** | **0** |

At the byte level (SHA256, showing the two ciphertext sets differ):

| File | NT=4 lineage | NT=8 output |
|---|---|---|
| `u0r112_0_0.ct` | `37f0857d871538f2953d07558d174de29618f8734a3c9eee6f0f6b74bd1ce1d3` | `2854fb5c6a76764d2ee71b7b91195f93d8e911476aaa054cd046735949e3c98a` |
| `u0r112_0_1.ct` | `7f6f43a68ec38cbb07ed1d279c2e6facf85056c1c1a2f82cc898851324d09552` | `13d039cbb0ed6b71e223df752aeb27374906cecca97d4eaad91758883e337918` |
| `u0r112_1_0.ct` | `c958522bf81c57cb613b0fa5aea67175f4890b7ebfd3f50909d92b0214a5a665` | `eb3dfa197fee3871173b58fb07ea778dc4bca1ff5855d93d6fd60e274c4a15bf` |
| `u0r112_1_1.ct` | `08b48b82ca9f8f0ccb8da3a2bba821591b9da167aba67da010a0925fa049022a` | `4fb5182eb466b25e1e7f5c2087867c889fa913d3dc784bb701a22ad29d34454d` |
| `u0r112_2_0.ct` | `7e872fe50bfb8c950f5d25699f7a4224d005b4e86f812fd4b39f6949774360ab` | `26f179a200f76a52b3ceb957847c336aa4fad0278f2d7f7b1dc5f005ef07957f` |
| `u0r112_2_1.ct` | `aabbed0f99aa719cec5433361cf875486ce62dee4d6aa176ede613f3fc991782` | `253935ad26d40f1a01935a218c3bc7a939ff012adfc12e820cbff955ddebc88d` |
| `u0r112_3_0.ct` | `e0ebd87f82a419d7ec8b5b80409d9d8667e97fb960dcf650da424c977b54ce55` | `64ecbb8b66658827e143a0e767cfc7c7ba62be9424927cd7d9c4ccad055d65f8` |
| `u0r112_3_1.ct` | `058f38b84fd020363a65429f3af82a35da515a26c796dcc255ff9860092c4ace` | `330456c2cf7efab9560d4225afbbb180b6a314b95fc5eb2252ca5b89eee56c8e` |
| **same/diff** | — | **8/8 all different** |

---

## 6. Downstream experiment with the alternative input (`layver_run.ps1 -TargetLayer 1`)

After injecting `.tmp_tok\chain\_n8nt8_out\u0r112*.ct` (the NT=8 output), `lay1` was run:

```text
inject   = .tmp_tok\chain\_n8nt8_out\u0r112*.ct
run      = lay1  (T23_NT=4, T23_E2EMODE=1)
baseline = .tmp_tok\relay_gh\lay1.out
lay1 rc=0 wall=2118s  RESULT=PASS  at 09:13:17
  NEW  (input from .tmp_tok\chain\_n8nt8_out): count=40  max=1.917E-002  min=1.294E-003  mean=8.431E-003
  BASE (.tmp_tok\relay_gh\lay1.out):           count=40  max=1.917E-002  min=1.294E-003  mean=8.431E-003
```

Post-experiment site check: live `chain` vs the 08:33 archive → **`same=80 diff=0`**.

---

## 7. On-chain ciphertext SHA256 (80 files, taken from the archived `manifest.sha256`)

Header form: `u{L}_{t}_{h}.ct` and `u{L}r112_{t}_{h}.ct`, with `t=0..3`, `h=0..1`.

### Layer 0

| File | SHA256 | File | SHA256 |
|---|---|---|---|
| `u0_0_0.ct` | `c3c1d48419f3590ec8f34c88ae553ecd4d4f063a962fd916517accbca8c9897b` | `u0r112_0_0.ct` | `37f0857d871538f2953d07558d174de29618f8734a3c9eee6f0f6b74bd1ce1d3` |
| `u0_0_1.ct` | `83c1f8709c8692dafac6b0dff0328e1a015c395d1fb93d86a5931780b675f0e8` | `u0r112_0_1.ct` | `7f6f43a68ec38cbb07ed1d279c2e6facf85056c1c1a2f82cc898851324d09552` |
| `u0_1_0.ct` | `1b7bc0b7df2888d359d3414e7d5926bf673a8b140f60d8af813d31e06cdbf4bb` | `u0r112_1_0.ct` | `c958522bf81c57cb613b0fa5aea67175f4890b7ebfd3f50909d92b0214a5a665` |
| `u0_1_1.ct` | `0a737dc3eca78fbddc8356345812f92823a55b4c4dd7774996c32dba5434eb25` | `u0r112_1_1.ct` | `08b48b82ca9f8f0ccb8da3a2bba821591b9da167aba67da010a0925fa049022a` |
| `u0_2_0.ct` | `abbdf1acb65f9f3b6c60f1ccb26950ee041a4e586124d7a7e5495cecc8cb37f5` | `u0r112_2_0.ct` | `7e872fe50bfb8c950f5d25699f7a4224d005b4e86f812fd4b39f6949774360ab` |
| `u0_2_1.ct` | `0c960fa53dfba2fc554522d70c04466b3bce79c497e11e23fa511f05a3e52f9b` | `u0r112_2_1.ct` | `aabbed0f99aa719cec5433361cf875486ce62dee4d6aa176ede613f3fc991782` |
| `u0_3_0.ct` | `dc1d0e1bc1e9c118070512a0cdd2ab1795e3b8c9b37b8b6f6cc987ad4cc6843e` | `u0r112_3_0.ct` | `e0ebd87f82a419d7ec8b5b80409d9d8667e97fb960dcf650da424c977b54ce55` |
| `u0_3_1.ct` | `eb068bb905950667395ef2dc8afe2e614c5dc0524406b7ce378f40c258d0f907` | `u0r112_3_1.ct` | `058f38b84fd020363a65429f3af82a35da515a26c796dcc255ff9860092c4ace` |

### Layer 1

| File | SHA256 | File | SHA256 |
|---|---|---|---|
| `u1_0_0.ct` | `7cbe3ac0d7bcca6db6157aed6a636274f3cb0ad2eb03d9ba4346555a76419ba8` | `u1r112_0_0.ct` | `328c962b2858375e12cd1d123275aeda09119b4304cc924c064a8f67b7ba5ba8` |
| `u1_0_1.ct` | `06133b0fcdb977fcfa5c1d7b9d5ca27b113ca9ea12fd91654454ef099b539295` | `u1r112_0_1.ct` | `6e3dc5ee27b1fa3fda1388a2cad796dec0d5c211a20869df92faaebe64561796` |
| `u1_1_0.ct` | `f0411b944a846632779f362cf68bd340ab322836130a226d4a4907694822f3dc` | `u1r112_1_0.ct` | `a3ae289898ead557b96322d5825f775e502a8f353d65452bdd9718e53a19e996` |
| `u1_1_1.ct` | `99df325836bfbc375c9a48f319dcc33bbde7e6d780d8619b0c24570e2a190828` | `u1r112_1_1.ct` | `c31e4db1c2fd2ddac946931c6deda07f5116caa5106b139a57e575ad9f151513` |
| `u1_2_0.ct` | `3188020af5f79eec83314a5088283cf8e57268278453241ac7a9780e126aa248` | `u1r112_2_0.ct` | `41c4493e442bbf8dfb674e8638c1dc1febfb76f33b38643ac9922934db844a6e` |
| `u1_2_1.ct` | `02cd4fd58d170c97f5a5c1c7b3c0c97e50b20bd258f2d9c9b9a39187ea48b8f2` | `u1r112_2_1.ct` | `7a62b3c18192785283b9bc37369a178ec05c7d8084bab39eb9aa5336e3f5c5db` |
| `u1_3_0.ct` | `1b72aeb30681bf70359d82156afe2edc8fb63b1d95209c7aa7382a3c715ae5ff` | `u1r112_3_0.ct` | `a74cf8e01d7fa990703b5635efec6659a159c34356981fd7868601219ff4afb8` |
| `u1_3_1.ct` | `548ec9a0ba48be1a547d0c56e5a56c39728b413541def8aac631d2ae6780e1aa` | `u1r112_3_1.ct` | `f7af19fca5bb5cea6f355f9d3196affab4b53b2f44d9465a4235b018479f279d` |

### Layer 2

| File | SHA256 | File | SHA256 |
|---|---|---|---|
| `u2_0_0.ct` | `5b6acd2fbb2817f7d15428814370c97c14bf4d0b46351f384e5ed6290d120a72` | `u2r112_0_0.ct` | `e5b750df287a1be8b7bdbe34a44aa833d375fe2998c98f46e9d10dab866d5730` |
| `u2_0_1.ct` | `6790c2109a0e7d52a3ea65eb51319459eb06d7887221274e94276ec67aa81a5c` | `u2r112_0_1.ct` | `4c3a169584af5ff17c5a088aa18210b310f3088414e4948eb2c7fa10f2a33980` |
| `u2_1_0.ct` | `6df641c564d50343c8a34c280bf5fc3673ca48d85edcf8fe54d069b4515a94d7` | `u2r112_1_0.ct` | `d29b9b68a0ffc8d8340bb86708181c5bc50a53065bae774a41c87a01417a7348` |
| `u2_1_1.ct` | `08c251471cd5281e5dbc4b6fc09468ebb78929b216a9f092bc4da212b7d46585` | `u2r112_1_1.ct` | `367daa7970404c885cda171f3ff3999adee3b5569f11813d1e11226dd39828d6` |
| `u2_2_0.ct` | `4b1e6644ab2627058dc0a4ff63889458f0a93d0a4b4b6d2cc8868ea29dbd92c7` | `u2r112_2_0.ct` | `f062aaf89405388f3abf68bb1814fcbb5e63c284da40fd8124d041aa663ba07e` |
| `u2_2_1.ct` | `bdd373a80dd17e65a565bbd2b033a6e0928c2bbb0753e13fff2c35f3e8a6505e` | `u2r112_2_1.ct` | `467efe19f2684da9d902a3175d0a522c1b31bd886d134fafb5203ae7e1dd6f12` |
| `u2_3_0.ct` | `e1046ef4932958de85f7e9be722a462d01c34e3cd9896b19cb580f26d58eaec2` | `u2r112_3_0.ct` | `1f4df1ebb7d3db07d171ba20c24171121db646ef672907bbb1078f2df1eaf19e` |
| `u2_3_1.ct` | `9342d0086010e12bfe0e6c612239085e1d3441c23d5f78c896c98374d86454f7` | `u2r112_3_1.ct` | `322f0d4f27a57d819c2bac6f2dab5c4916e9b76ab3ae7b230c49e30ca2a10389` |

### Layer 3

| File | SHA256 | File | SHA256 |
|---|---|---|---|
| `u3_0_0.ct` | `2637274ee00985e5cd2577f68524129dd25f51efc4ac142ba967bbb41ed3d98f` | `u3r112_0_0.ct` | `161695aa0cc8e615edec86108ee30d7bd790bd9ee1bee30fa6acf9e3fe86d9c8` |
| `u3_0_1.ct` | `7f2300884106f4e32e1f367c2c4a968495841ff6e92c9727cfae7745149c8149` | `u3r112_0_1.ct` | `da7bba6c99e2ea7d0ef4eb869d6ffaee4831850c1602f493ba0477cf9ed9858a` |
| `u3_1_0.ct` | `af82e32d4ffd2eafecb67138605bcb2d33ad62b7ca29ccebab9465af4bf345df` | `u3r112_1_0.ct` | `37e526a0a7c5534ee560f2c64549421f74b68c1c39f579dac7ca1e5ee50b62b8` |
| `u3_1_1.ct` | `c95f8b1b8d5060c214465a1751c727c3078cb8ec1b28b3432d96e0f30bc89a1c` | `u3r112_1_1.ct` | `8128c7fb90436470dad5808bcb8e777f32a7bd48071b0f5c22563ae354620a16` |
| `u3_2_0.ct` | `19db40033850ba6a393eaec0f95ad4b6c14d41244f5ce100937b1264d75eaf1e` | `u3r112_2_0.ct` | `9e206dcddac32374703bf9068a1f5a8fcbb95bc90486eaff679390ae72b07554` |
| `u3_2_1.ct` | `04ca3de9b9c383ca976247e26f60e6b7049d3fe519689f0617ea0056f7c0a7eb` | `u3r112_2_1.ct` | `23209e0671200f3009e46c902d019a33536ae055a4c5f615a1ee86cdf4d64412` |
| `u3_3_0.ct` | `87999059af3741ac74612c47899da67004ba6af632f0aa49d8d33780ad2cc242` | `u3r112_3_0.ct` | `5a849eabe1b9818338b4470194b9d9cc7ae289af67c2f703e38ba7835981f681` |
| `u3_3_1.ct` | `7e0f4b125a9ad650e3b373639de3d8bb1102f96b67e6c834747be0e5dd627792` | `u3r112_3_1.ct` | `b26ba706c1670c4d320ac72ccb54a056f51273780c0e686d6c4ad8832edf7752` |

### Layer 4

| File | SHA256 | File | SHA256 |
|---|---|---|---|
| `u4_0_0.ct` | `ecb4fdec8638d17aded9c0d66e503f31a41d4d3e9b8f8c78ad49758e20dc1f1e` | `u4r112_0_0.ct` | `c7cced0054653d2aec3b19c78c3291d0a1648768fa73fe6148d3e86c5e39e944` |
| `u4_0_1.ct` | `2c4f9f39510c945877ba1e57e7d9a8e92c1182c7439530fadf6e2c65615b4ee0` | `u4r112_0_1.ct` | `c584c714b561c03d959dc1f73319fdddedf589d034436322be2c282ca03bc7da` |
| `u4_1_0.ct` | `83eeac316eb5bf2b3c6cdb7b7b531a0f1e6dc3ab5082093bcc824860551532a2` | `u4r112_1_0.ct` | `79815a4c1cf2e643a4b1c4d84742253784ec6144a52351cfa81ad1ab42bcd0a1` |
| `u4_1_1.ct` | `9f6da56094efec526e485cc95f7252ad70a61c1c3b368a65894c0d9f2e498e37` | `u4r112_1_1.ct` | `e175a6d4b78f3a1832c38e593dc3ed286ae5ca4653aaba82527b5a6e9ffee479` |
| `u4_2_0.ct` | `58f5983b1dd645538ea8f1919b93d78b7e104cd8ddb63078bf7c329e7157c4fb` | `u4r112_2_0.ct` | `ea204239341579a8aa978ee29057ebefbed893041e774866c3d6d53d32d6ae0a` |
| `u4_2_1.ct` | `69c75c506ae5516a37bf17f00d0c7a6c65a81608226edf1a16b29346ec1dfbe1` | `u4r112_2_1.ct` | `e3220561de6c14917e782ee1d784371e1de874f3350a9ea69d5f9507e2410690` |
| `u4_3_0.ct` | `4912488f46131a952c6508346779c8667f696538ffee0f9d7cb0f33b31264fac` | `u4r112_3_0.ct` | `47a75d67b54f871f1014b7a48eed399ea79936be732cba32dfced6f5f16b5301` |
| `u4_3_1.ct` | `50d1e99851a505ed7c6224e667af14137f79773943702b04073592cf554f41dd` | `u4r112_3_1.ct` | `e52e4615c60943a7a3b2984f54b46c77606fe5bd23035219cd0160f49477e4cb` |

---

## 8. File sizes

| Prefix | Bytes | Modulus chain |
|---|---|---|
| `u0` | 524300 | np=16 (32768 B per prime × 16) |
| `u1` | 655372 | np=20 |
| `u2` | 491532 | np=15 |
| `u3` | 458764 | np=14 |
| `u4` | 491532 | np=15 |
| `u0r112` … `u4r112` | 3670028 | np=112 (32768 B per prime × 112 + 12 B header) |

> The size of `u{L}` is exactly its remaining prime count, matching `in np=` in the `boot` logs.

---

## 9. Archive inventory summary (Release asset, see [`data/README.en.md`](../data/README.en.md))

| Category | Count | Notes |
|---|---|---|
| `chain\*.ct` | 80 | 5 layers × 2 prefixes × 8 |
| `logs\lay*.out` / `lay*.err` | 10 | the 5 `.err` files are identical (19 B) |
| `logs\boot*.out` / `boot*.err` | 10 | criterion evidence |
| `master.log` / `meta.txt` / `timing.tsv` | 3 | timeline, machine metadata, per-layer summary |
| `README.md` | 1 | **still an unfilled template** |
| `manifest.sha256` | 1 | SHA256 list of the files above |
| **Total** | **104 + 1** | `collect_results.ps1` exit code 0, `warnings=0` |

SHA256 of the non-ciphertext files:

| File | SHA256 |
|---|---|
| `master.log` | `eb27c54b66dbeb3e13bfc5036ddf5cb9aba0dbe5e80dddf3e70688f581b470f7` |
| `meta.txt` | `0bd0d7b3620be9b71e1f559c22366914fcbf741ae52cc9dff99f1dcea3f8e383` |
| `timing.tsv` | `b4ef7862cc8a2c7d64f8362552f641758726a2bcc0144853ddf36f9ad972f76a` |
| `README.md` | archived value `f15f9c18774ab9692eff3ebccd4d83fe3224c563f867bb512af34a55427afa47`; **changed after revision**, current value `dc3792edb0e5c27e48f8708b9b3b86687e434e08097fbebde71834f9fbf4ce03` (that manifest line has been regenerated accordingly) |
| `logs\boot4.out` | `03a7a41fcfb0b853439c9ff73d756d893958e29d6a657f8a0bf435bfd76f7d3d` |
| `logs\lay4.out` | `4c4c36ac5a22d66eb3bfaa376602af15f46aadd4f511dc484c96bf81ff2b9a74` |
| `logs\lay{0..4}.err` | `43585c1a96323665e2ccb675fdca9d4e6f8bdac8fff360d7e39848b6b4fa2a0a` (all 5 identical) |

`timing.tsv` in full (as seen by the resumed segment, hence `SKIP` for layers 0/1):

```text
layer	lay_s	lay_st	boot_s	boot_st	boot_total_s	cts_ok
0		SKIP		SKIP		u0+u0r112
1		SKIP	4891	PASS	4811.12	u1+u1r112
2	1942	PASS	4350	PASS	4271.61	u2+u2r112
3	1949	PASS	4339	PASS	4263.08	u3+u3r112
4	1972	PASS	4392	PASS	4312.36	u4+u4r112
```

### 9.2 Tail archive `L26-27-fin` (the other Release asset)

| Category | Count | Notes |
|---|---|---|
| `chain\u26_*.ct` / `u27_*.ct` | 16 | 2 layers × 8, `lay` output (np=3) |
| `chain\u26r112_*.ct` / `u27r112_*.ct` | 16 | 2 layers × 8, `boot` refresh output (np=112)  <- hand-off artefact |
| `logs\{t26,boot26,t27,boot27,fin}.out` / `.err` | 10 | criterion evidence |
| `logs\master.log` / `logs\meta.txt` | 2 | five-step timeline, machine and exe/source SHA256 |
| `README.md` | 1 | tail archive notes |
| `manifest.sha256` | 1 | SHA256 list of the files above (45 lines) |
| **Total** | **45 + 1** | in-package self-check **45/45 match** |

> ⚠️ **Note**: this one is **not** a continuation of the chain in §2–§8; it is a **branch encrypted directly
> from a plaintext seed into L26**, **bypassing layers 5–25**. What it demonstrates is that "the last two
> layers plus the output head compute correctly", **not** that "the whole chain runs from layer 0 through
> layer 27".

---

## Tail (L26/27 + `fin`) measurement: large stage-level deviations, top1 all correct

The mechanism above is not a paper argument — it has been closed empirically.

- **Date**: 2026-09-15
- **Chain**: r112 chain + **same-generation** references (`tail/l26_u1.bin`, `l26/l27_u2_ref.bin`, `xnorm_f.bin`, `logits.bin`, all generated 2026-09-04)
- **Method**: encrypt the plaintext seed `tail/l26_u1.bin` directly into L26 (phase 3 takes the plaintext-encrypt branch), **bypassing layers 5–25**

| Step | Wall | Result |
|---|---|---|
| `t26` | 2079 s | U2 **8/8 PASS** (4.471e-03 ~ 2.299e-02) |
| `boot26` | 4469 s | `BOOT=PASS`, 8× `out np=2083`, profile total 4393.50 s |
| `t27` | 2068 s | U2 **2/8 over threshold** (t1h1 = 3.822e-02, t3h1 = 7.969e-02) |
| `boot27` | 4489 s | `BOOT=PASS`, 8× `out np=2083`, profile total 4410.70 s |
| `fin` | 3178 s (RMSNorm 75.1 + lm_head 3102.4) | `RESULT=PASS (0; ref-deviate 8)`, **top1 4/4 correct** |
| **Total** | **16284 s ≈ 4.52 h** | — |

`fin` logits, per token:

```
[F:logits t0] max|err|=0.4376 (vocab 7267)  | top1=120555 ct=16.592 ref=120555 | margin ct=2.114 ref=2.027
[F:logits t1] max|err|=1.294  (vocab 47516) | top1=120555 ct=15.791 ref=120555 | margin ct=1.388 ref=1.619
[F:logits t2] max|err|=1.247  (vocab 60163) | top1=120555 ct=15.964 ref=120555 | margin ct=1.917 ref=2.280
[F:logits t3] max|err|=1.319  (vocab 63019) | top1=120555 ct=16.228 ref=120555 | margin ct=1.418 ref=2.307
```

`[F:xnorm]` per component (all 8 FAIL):

```
[F:xnorm  t0] max|err|=2.505e-01   [F:xnorm1 t0] max|err|=2.918e-01
[F:xnorm  t1] max|err|=8.771e-01   [F:xnorm1 t1] max|err|=8.420e-01
[F:xnorm  t2] max|err|=9.291e-01   [F:xnorm1 t2] max|err|=1.116e+00
[F:xnorm  t3] max|err|=9.247e-01   [F:xnorm1 t3] max|err|=7.709e-01
```

**How to read it**: `ref-deviate 8` is **exactly** the 8 `[F:xnorm]` FAILs (2 components × 4 tokens).
Since a top1 mismatch also does `rf++`, **`rf = 8` by itself proves the top1 mismatch count is 0** —
and indeed there is no `[F:top1] MISMATCH` line in the log at all.

This is the empirical closure of "**large intermediate deviation, normalized criterion still PASS, and
top1 correct**": every internal deviation (`[C:attn]` 8.5~12, `[D:o]` 14~29, `[F:xnorm]` 0.25~1.12,
`[F:logits]` 0.44~1.32) was absorbed by the normalized criterion, and **the argmax matched the
reference for all 4 tokens** (all `120555`).

> Boundary: this section validates the **last two layers plus the output head** given the true L26
> input. It does **not** mean layers 5–25 have been connected. The relay chain still stops at `u4r112`.

---

## 10. Data boundaries

- The data covers **layers 0–4 only** (§2–§8) **plus a separate tail branch (layers 26–27 + `fin`)** (the
  "Tail measurement" section); layers 5–25 on the chain are not included. "The full 28-layer chain" is an
  extrapolation and is outside the scope of this data.
- The `verify_layer` tolerance is fixed at `3e-2`; layer 2 has only 5.4% margin and is currently the tightest layer.
- The thread-count-related bit divergence (`NT=4` vs `NT=8`) is documented in §5; the conclusion is **different bytes, identical values**.
