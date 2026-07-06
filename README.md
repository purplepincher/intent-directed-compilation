# Intent-Directed Compilation

Using semantic criticality ("stakes") to choose instruction-level precision
for constraint checking: low-stakes constraints are checked in 8-bit
arithmetic (64 per AVX-512 register) instead of 32-bit (16 per register),
with no change to the comparison result for in-range values.

## Quick start

**C benchmarks (AVX-512, where the real numbers come from).** AVX-512
hardware is required to *run* these — the published results were measured
on an AMD Ryzen AI 9 HX 370. On a non-AVX-512 host the binaries compile
and link cleanly but `SIGILL` at runtime.

```bash
gcc -O3 -mavx512f -mavx512bw -mavx512dq -o avx512_soa src/avx512_soa_benchmark.c
./avx512_soa
```

The multicore variant adds `-pthread`:

```bash
gcc -O3 -mavx512f -mavx512bw -mavx512dq -pthread -o avx512_mc src/avx512_multicore.c
```

`src/avx512_soa_benchmark.c`, `src/avx512_multicore.c`, and
`src/neon_fallback.c` each carry their compile line in a header comment.
`src/e2e_pipeline_benchmark.c` and `src/rars_imu_proof.c` build with the
same AVX-512 flags but do not state a compile line in their headers.

**ARM NEON fallback.** Cross-compile for aarch64, or build the scalar
fallback on x86 for a syntax/link smoke test (no NEON instructions are
emitted in the scalar path):

```bash
aarch64-linux-gnu-gcc -O3 -march=armv8-a+simd -o neon_check src/neon_fallback.c
# x86 smoke test:
gcc -O3 -DSCALAR_FALLBACK -o neon_check src/neon_fallback.c
```

**Python validation.** This is a pure-Python differential check (INT8/INT16
vs INT32). It needs the `polyformalism-a2a` package on the import path:

```bash
python3 benchmarks/benchmark_intent_compilation.py
```

Note: Python simulates the narrower types in software — it cannot emit
`VPCMPD`, so it validates correctness, not the SIMD throughput. The
throughput numbers all come from the C benchmarks above.

**Coq proof.** No toolchain is bundled. To type-check the Qed-complete
file:

```bash
coqc proofs/XOR-ISOMORPHISM.v   # 8 Qed, 0 Admitted
```

`proofs/INTENT-HOLONOMY-DUALITY.v` is **not** expected to compile — see
[Proofs and their status](#proofs-and-their-status).

## What you get when you run it

Measured on an AMD Ryzen AI 9 HX 370, single-threaded, `rdtsc`, AVX-512
F/DQ/BW/VNNI/BF16. Full run log and reproducibility table in
[`benchmarks/REAL-NUMBERS.md`](benchmarks/REAL-NUMBERS.md).

| Metric | Value |
|--------|-------|
| SoA mixed-precision speedup | **3.17×** (5-run mean; range 2.72×–3.52×) |
| INT8 raw throughput | **4.58×** over INT32 (theoretical packing gain 4.0×) |
| Differential mismatches | **0** across the checked-in suite (see below) |
| Break-even (constraint reuses) | **8** |
| Steady-state speedup | **12×** at 10,000 reuses |

The "0 mismatches" figure is the sum of the differential tests that
actually run: 39,049 hardware `VPCMPD` checks, 8,323,200 Python
exhaustive INT8 checks over [-127, 127], and 10,000,000 Python random
INT16 checks. (The C SoA benchmark itself processes 10,000,000
constraints.) Earlier drafts cited "100M"; that number is not produced
by any checked-in code and has been removed.

Layout matters more than the algorithm: the same kernel in
array-of-structs (AoS) layout runs **slower** than the INT32 baseline
(0.42×) because of scatter/gather. The 3.17× requires
struct-of-arrays (SoA) layout.

## How it works

1. **Classify by stakes.** Each constraint carries a continuous stakes
   value `s ∈ [0,1]` (0 = informational, 1 = life-critical). Stakes map
   to a precision class.
2. **Pack by precision class.** Constraints of the same class are stored
   contiguously (SoA), so a single AVX-512 register holds 64 INT8, 32
   INT16, or 16 INT32 constraints.
3. **Check in packed form.** `VPCMPD` does 16 INT32 comparisons per
   instruction; on INT8 data the same instruction does 64. Low-stakes
   constraints cost ¼ the register bandwidth.
4. **Escalate the few that matter.** Safety-critical constraints get a
   dual-path INT32 check (redundant comparison) instead of a cheaper one.

The reference autonomous-vehicle mix (75% advisory / 15% operational /
8% technical / 2% safety-critical) is the workload behind the 3.17×
number. A different mix gives a different number; the harmonic-mean
throughput formula in `benchmarks/REAL-NUMBERS.md` predicts it.

### Precision classes

| Stakes `s` and value range `r = hi − lo` | Class | Bits | Per AVX-512 register |
|---|---|---|---|
| `s ≤ 0.25` and `r ≤ 127` | INT8 | 8 | 64 |
| `s ≤ 0.50` and `r ≤ 32000` | INT16 | 16 | 32 |
| `s ≤ 0.75` | INT32 | 32 | 16 |
| otherwise | INT32 dual | 64 (effective) | 16, checked twice |

## Bugs found by adversarial review, and fixed

Four independent models reviewed the kernel. Two real bugs surfaced,
both documented in `benchmarks/REAL-NUMBERS.md` and fixed in the checked-in
code:

- **INT8 overflow wrapping** (4.9% mismatch on out-of-range values) →
  fixed with range validation before truncation.
- **Dual-path subtraction overflow** → fixed by replacing the
  subtraction-based check with a sign-bit XOR, which is also ~6% faster
  than the broken code.

The same review pass **disproved** the original throughput formula (it
used an arithmetic mean; the correct formula is the harmonic mean
`G = 4 / (a + 2b + 4c + 8d)`). The corrected formula is what the numbers
above rest on.

## Proofs and their status

The proof artifacts are not all of the same strength. Stated precisely:

- **[`proofs/XOR-ISOMORPHISM.v`](proofs/XOR-ISOMORPHISM.v)** — Qed-complete
  Coq (8 `Qed.`, 0 `Admitted.`): symmetric difference on finite sets
  forms an abelian group and the characteristic function is a
  homomorphism. This is the one kernel-checkable artifact in the repo.
  `coqc` was not available on the import host, so the `Qed`s have not
  been re-run by a live kernel here — the counts are from reading the
  file.
- **[`proofs/INTENT-HOLONOMY-DUALITY.v`](proofs/INTENT-HOLONOMY-DUALITY.v)**
  — **headline theorem not proven.** 5 `Qed.`, 7 `admit.`; the main
  duality would not pass the Coq kernel. The source author's own
  assessment (30% confidence, "one direction proven, converse open") is
  preserved verbatim in
  [`proofs/CONSTRAINT-THEORY-MATH-PROVENANCE.md`](proofs/CONSTRAINT-THEORY-MATH-PROVENANCE.md).
  This repo does **not** claim the duality is proven.
- **INT8 soundness** ([`proofs/FORMAL-PROOF-MIXED-PRECISION.md`](proofs/FORMAL-PROOF-MIXED-PRECISION.md),
  theorem T1/T2) and **dim H⁰ = 9**
  ([`proofs/PROOF-DIM-H0-EQUALS-9.md`](proofs/PROOF-DIM-H0-EQUALS-9.md))
  — prose proofs produced by `deepseek-reasoner`, not machine-checked.
  The arguments are standard and correct (in-range values are exactly
  representable, so b-bit and 32-bit comparisons agree; global sections
  of a trivial rank-9 bundle on a tree have dimension 9), but they are
  markdown, not Coq. The same DeepSeek pass that produced them
  **disproved** theorem T4 (the throughput formula) — see above.

The two `.v` files were imported from `SuperInstance/constraint-theory-math`
and are adjacent constraint-theory material. They are **not** proofs of
the 3.17× result or of mixed-precision soundness.

## Constraints and limitations

- **AVX-512 is required to reproduce the throughput numbers.** The
  binaries link on any x86-64 machine but `SIGILL` without AVX-512. There
  is no SIMD speedup to measure in the Python validation path.
- **The result is layout-dependent.** AoS layout is slower than the
  baseline; only SoA gives the 3.17×. Anything that defeats the SoA
  layout (per-constraint branching, scattered data) defeats the speedup.
- **It only pays off with reuse.** The end-to-end pipeline (classify →
  SoA-convert → check) is *slower* than scalar for a one-shot check
  (0.14×). It breaks even at ~8 reuses of the same constraint set and
  reaches 12× at 10,000. A stream of fresh, never-reused constraints
  will not benefit.
- **There is no automated test suite gating this repo.** The GitHub
  Actions workflow runs `python -m pytest ... || true`, so CI is green
  regardless of outcome, and there are no pytest tests to run in any
  case — only the benchmark scripts above. The numbers in this README
  come from manual hardware runs logged in
  [`benchmarks/REAL-NUMBERS.md`](benchmarks/REAL-NUMBERS.md), not from CI.
- **INT8 soundness holds only for in-range values.** Out-of-range values
  are why the range-validation bug fix above exists; the narrower types
  are not a drop-in replacement for INT32 on unbounded inputs.
- **Only the XOR isomorphism is machine-checkable here**, and even that
  has not been re-run with a live `coqc` in this repo (none installed).
  The headline duality in the imported `.v` file is open.

## Repository layout

- `src/` — C benchmarks: `avx512_soa_benchmark.c` (the 3.17× source),
  `avx512_multicore.c`, `e2e_pipeline_benchmark.c`, `rars_imu_proof.c`,
  and `neon_fallback.c` (ARM).
- `benchmarks/` — `benchmark_intent_compilation.py` (Python differential
  check) and `REAL-NUMBERS.md` (the full measured-results log).
- `proofs/` — the two `.v` files, their provenance, and the prose proof
  notes above.
- `paper/` — a paper written in IEEE conference (IEEEtran) format
  ([`PAPER.md`](paper/PAPER.md)), plus [`RESEARCH.md`](paper/RESEARCH.md)
  and [`VALIDATION.md`](paper/VALIDATION.md). This is a document written
  in IEEEtran style, not a paper submitted to or accepted by any IEEE
  conference.

## License

MIT — see [LICENSE](LICENSE).
