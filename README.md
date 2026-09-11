# mips

Recon on MIPS (a GlobalFoundries company) as an llama.cpp/ggml target, and the first landed work
against it. MIPS is RISC-V now — the legacy MIPS ISA is not part of this.

## Contents

- `patches/0001-riscv-zba-zbb-mips-p8700.patch` — the working patch against llama.cpp
  `a1f96d4`: declares the missing `GGML_RV_ZBA`, adds `GGML_RV_ZBB`, adds a P8700 toolchain file
  and a QEMU-tested CI job, and documents the RISC-V build flags for the first time.
- `zbb-mnemonics.txt`, `zbb-by-function.txt` — retained disassembly evidence from the verification
  build (counts per mnemonic, and per hosting function).

## What MIPS is, as of 2026-09

Two unrelated hardware lineages under one name, after GlobalFoundries closed its **$455M acquisition
of the Synopsys ARC Processor IP business on 2026-06-01** and folded it into MIPS.

| | ARC NPX6 | MIPS S8200 |
|---|---|---|
| Type | fixed-function NPU | RISC-V cores + tightly-coupled AI engines |
| Compute | 1,024–98,304 MACs; 250 TOPS @1.3 GHz 5nm; 3,500 TOPS multi-instance | "tens to hundreds of TOPS" via coherent cluster tiling |
| Dtypes | INT4/8/16, optional FP16/BF16, OCP MX microscaling | vector + matrix, specifics undisclosed |
| Toolchain | MetaWare MX (NN compiler + runtime) | MLIR/IREE + Atlas Explorer |
| Status | in customer silicon | **sampling; reference silicon 2027** |

CPU IP, which determines the fallback path:

| Core | ISA | Vector |
|---|---|---|
| P8700 | `RV64GC_Zba_Zbb` + Xmipscmov/Xmipslsp/Xmipscbop, 4-issue OoO, 1–2-way SMT | **none** |
| I8500 / I8600 | data movement subsystems | — |
| ARC-V RHX-100V / 105V | 32-bit real-time | optional RVV (rv32 — llama.cpp is rv64-only) |

Correction to a claim that circulates in search results: **MetaWare MX does not mention llama.cpp or
GGUF.** Traced to source — neither the `arc-npx6` nor the `arc-metaware-mx` page contains the string.
The search engines are conflating llama.cpp's own README blurb. Do not plan against it.

## Verified: no hardware needed

**Upstream QEMU already models the P8700.** `TYPE_RISCV_CPU_MIPS_P8700` (`-cpu mips-p8700`),
`MIPS_VENDOR_ID 0x127`, and the three vendor extensions `xmipscmov` / `xmipslsp` / `xmipscbop` are in
`qemu/qemu` master. There is also a `boston-aia` board model (up to 64 harts, MIPS CPS/GCR/CPC, AIA
PLIC/CLINT), so full-system Linux is emulable. Contributed by MIPS engineers over 15 revisions on
qemu-devel. LLVM documents all three extensions as non-experimental, targeting the P8700.

MIPS upstreams its own enablement into QEMU, LLVM, GCC, and the kernel. Nothing in the LLM inference
stack knows MIPS exists. That gap is the opening.

## Done: the Zba/Zbb patch

Found while scoping P8700 support: **`GGML_RV_ZBA` is read at `ggml/src/ggml-cpu/CMakeLists.txt:490`
but never declared as an option**, so it is silently off for everyone except the SpacemiT CI job that
passes it explicitly. `GGML_RV_ZBB` did not exist at all. On a core with no vector unit the scalar
path is the only path, so this is the whole performance surface on a P8700.

Confirmed by configuring against unpatched master with `CMAKE_SYSTEM_PROCESSOR` forced to riscv64:

```
master defaults:   -march=rv64gcv_zfh_zvfh_zicbop_zihintpause
patched defaults:  -march=rv64gcv_zfh_zvfh_zicbop_zihintpause_zba_zbb
patched, P8700:    -march=rv64gc_zihintpause_zba_zbb
```

### Measured

Real cross-build, GCC 14.2 (`riscv64-linux-gnu-gcc-14`), `libggml-cpu.so`:

| Build | Zbb instrs | total instrs | size |
|---|---|---|---|
| `GGML_RV_ZBB=ON` | **694** | 129,093 | 649,512 B |
| control (ZBA+ZBB off) | **0** | 131,568 | 653,624 B |

365 `min`, 99 `sext.h`, 99 `maxu`, 71 `sext.b`, 23 `max`, 22 `zext.h`, 7 `andn`, 5 `rori`,
2 `minu`, 1 `clz`. They land in the scalar quant hot paths — `ggml_vec_dot_q5_K_q8_K_generic` (32),
`ggml_vec_dot_q4_K_q8_K_generic` (32), `ggml_vec_dot_q6_K_q8_K_generic` (16),
`ggml_vec_dot_q3_K_q8_K_generic` (16), `ggml_gemv_q4_0_4x4_q8_0` (16) — plus elementwise forwards.

Correctness under QEMU 8.2 (`-cpu rv64,v=false,zba=true,zbb=true`): `test-quantize-fns` pass (all
types incl. q4_K/q5_K/q6_K/mxfp4), `test-rope` pass (rel err 0.000000), `test-barrier` pass.

The CI guard was rehearsed both directions: PASS at 694 on the patched build, correctly FAILS at 0 on
the control.

### Not quotable

The 2,475-instruction reduction covers **Zba and Zbb together** — the control had both off. Only the
694 figure is Zbb-specific.

**There is no throughput number and cannot be one yet.** QEMU measures correctness, not speed, and
there is no P8700 silicon. The defensible claim is: *the compiler now emits 694 Zbb instructions in
the scalar quant hot paths where it previously emitted none, and results remain numerically correct.*
Nothing about tok/s.

Two other limits: QEMU 8.2 cannot do `-cpu mips-p8700` (needs ≥10), so the CI job spells out the
equivalent ISA; and `test-backend-ops` is unusable here — its default mode prints `Skipping CPU
backend` when CPU is the only device, and `grad` mode did not finish in 10 minutes under emulation.

### Upstreaming

Split into two PRs. The Zba/Zbb fix stands alone and also fixes the SpacemiT toolchain file (K1 is
RVA22, it has Zbb, the file omitted it) — reviewers with K1 hardware can validate it, so it merges on
its own evidence without anyone trusting claims about unreleased MIPS silicon. The P8700 toolchain +
CI job is the MIPS-specific half and reviews more easily once the first has landed.

Defence for defaulting both ON: the existing defaults already demand `V` + `Zfh` + `Zvfh`, and `Zvfh`
is not even in the RVA22 baseline while `Zba`/`Zbb` are. Any core satisfying today's defaults has both.

## Next, once a board is here

The intended board is a **SpacemiT K3** (Pico-ITX SBC $299+, K3-CoM260 SoM $309+; same silicon as
Milk-V Jupiter 2 and Banana Pi BPI-SM10). It is the closest commercially available analogue to the
S8200: 8× X100 RISC-V app cores (RVV 1.0, VLEN 256) + 8× A100 AI cores (RVV VLEN 1024) + IME 2.0
matrix unit at VLEN 1024/2048, 60 TOPS INT4, RVA23. A K1 board (BPI-F3 / Milk-V Jupiter, from $60,
X60 VLEN 256) exercises the vector path but has no IME 2.0.

In order:

1. **Benchmark the Zbb patch for real.** The one thing emulation cannot supply. Build with and
   without `GGML_RV_ZBB` on the K3's scalar path and get a tok/s delta. Converts the PR's static
   instruction count into a performance claim.
2. **Baseline `-DGGML_RVV=ON -DGGML_CPU_RISCV64_SPACEMIT=ON`.** Exercises the code shape a MIPS
   backend would take: custom buffer type in `ggml-cpu/spacemit/ime.cpp`, weight repack, TCM
   management in `spine_tcm.h`, hand-written matrix microkernels. Read these on hardware, not on
   paper.
3. **Close the IME test gap.** `ime1_kernels.cpp` + `ime2_kernels.cpp` — 6,795 lines — have **zero**
   automated coverage: the SpacemiT CI job is build-only (`LLAMA_BUILD_TESTS=OFF`, no test step) and
   the riscv CI runs on native runners that never touch IME. A board makes this testable; a CI job
   built on it is a second upstream contribution and buys standing with the RISC-V maintainers.
4. **Sweep VLEN sensitivity.** `quants.c` has `_vl128`/`_vl256`/`_vl512`/`_vl1024` variants of
   `q2_K`/`q3_K`. The S8200's VLEN is undisclosed, so knowing where these break or underperform
   de-risks it directly. Partially doable in QEMU first (`-cpu rv64,v=true,vlen=N`).

Board-independent, worth starting before it arrives:

- **TCG plugin profiling of llama.cpp decode.** The published IME-vs-AME workload analysis
  (RISC-V SIG-Vector's Vector-Matrix Profiler) is CNN-only. Nobody has characterised the instruction
  mix of autoregressive decode. Stock QEMU, no extension work. It is simultaneously what the AME TG
  is arguing over this quarter and what MIPS needs to size the S8200's engines — arriving with it
  means contributing to their design rather than asking for their documentation.
- **`vmadot` in QEMU.** Corrects an earlier assumption: `spacemit-com/qemu` branch
  `v10.1.2-xsmtame-v0.6` (QEMU 10.1.2, May 2026) *does* implement a SpacemiT matrix extension —
  `xsmtame.decode`, `xsmtame_helper.c`, `xsmtsfu.*` — but that is the **AME** ISA (`mmacc`, `mmov`,
  tile registers). llama.cpp's kernels use the **IME** ISA: mnemonic counts are 112 `vmadot`,
  48 `vmadotsu.hp`, 40 `vmadotsu`, 32 `vmadotu.hp`, 24 `vmadotu` in `ime2_kernels.cpp`, 16 `vmadot`
  in `ime1`, plus `vpack.vv`/`vupack.vv` — and **zero** `mm*` instructions. So the AME fork does not
  cover them. The uncovered surface is small and well-documented (public spec at
  `spacemit-com/riscv-ime-extension-spec`, encodings in binutils/LLVM, `xsmtame_helper.c` as a
  vendor reference in the same tree). This is the direct rehearsal for MIPS's engines.
- **Read the AME TG.** RISC-V Insider membership is free and technical groups are publicly visible
  read-only; posting needs paid membership. The TG was polling members on required data types as of
  2026-03 — that decision determines whether GGUF quant layouts map onto a standard matrix extension
  or fight it. Three competing efforts (IME, AME, VMEX), none ratified.

## Access paths

No self-serve route to S8200 exists. No public SDK, no Atlas Explorer trial, no cloud instance.

- `mips.com/contact` — the only public entry. Frame as inference-runtime enablement, not IP
  licensing; the ask is ISA docs + toolchain, plus Atlas Explorer for pre-silicon bring-up.
- `atlasportal.mips.com` — customer/vendor/partner gated; follows from the above, does not precede it.
- CES (LVCC West Hall Booth 3253 in 2026) and RISC-V Summit — materially faster than the web form.

**The gating question**, to settle before writing any MIPS-specific code: *are the S8200's AI engines
reachable as instructions from the RISC-V application core, or a separate device behind a command
queue?* Instructions → clone the `spacemit/` directory shape, weeks. Command queue → a full
`ggml_backend_reg` plus a device-side kernel corpus (the ET tree is ~50 kernels / 17k lines). Their
"add custom instructions" messaging hints at the former; hints are not a spec.

**NDA trap.** Both working precedents exist *because* the ISA is public — SpacemiT's `xsmtvdot` went
through binutils review, ET's Programmer's Reference Manual is Apache-2.0 on GitHub. llama.cpp is
MIT. Taking S8200 docs under NDA may make the resulting backend unpublishable, leaving a private fork
against a fast-moving tree forever. Negotiate publication rights for the encodings *before* signing.

## Adjacent, free, real hardware

`aifoundry-org/et-platform` (Apache-2.0) ships `sw-sysemu`, a software simulator for the Esperanto
ET-SoC-1 — build llama.cpp's in-tree ET backend with `-DGGML_ET=ON -DGGML_ET_SYSEMU=ON`, no hardware.
`aifoundry-org/et-man` (Apache-2.0) carries the Programmer's Reference Manual and datasheet.

GWDG hosts **4 compute nodes × 8 ET-SoC-1 cards**, open to external researchers evaluating
edge/small-model deployment — request at `hpc-support@gwdg.de`. Free access to a real manycore RISC-V
accelerator (1000+ cores, vector + tensor units, 32 GB LPDDR4X, ~40 W per card). Esperanto closed
July 2025; AINekko acquired the IP and open-sourced the stack, so this is effectively the only route
to the silicon. The ET backend is the template for the S8200-as-offload-device case.
