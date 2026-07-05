# Intent-Directed Compilation

Using semantic criticality ("stakes") to drive instruction-level precision for constraint checking.

## The Idea

Not all constraints carry equal weight. A water temperature sensor (informational) doesn't need the same verification as a depth sensor (life-critical). We classify constraints by stakes and use narrower integer types for lower-stakes checks, processing 64 per AVX-512 register instead of 16.

## Key Results (Measured, rdtsc, AMD Ryzen AI 9 HX 370)

| Metric | Value |
|--------|-------|
| SoA mixed speedup | **3.17×** (5-run mean) |
| INT8 raw | **4.58×** (exceeds 4.0× theoretical) |
| Differential mismatches | **0 / 100M** |
| Break-even reuses | **8** |
| Steady-state speedup | **12×** at 10K reuses |

## Structure

- `paper/` — IEEE conference paper + research documentation
- `benchmarks/` — Real hardware numbers, cycle-accurate
- `proofs/` — Formal proof notes (INT8 soundness, XOR equivalence, dim H⁰=9) and the Coq proof sketches below
- `src/` — C benchmarks, Python validation, ARM NEON fallback

## Proven Theorems

1. **INT8 Soundness:** Cast is identity on [-127, 127] → comparison preserved
2. **XOR Dual-Path:** Sign-bit XOR = adding 2³¹, bijective order isomorphism
3. **dim H⁰ = 9:** Global sections of trivial GL(9) bundle on tree = 9

## Coq Proof Sketches (adjacent theory, imported)

Two `.v` files imported from `SuperInstance/constraint-theory-math` live in
`proofs/`, with full provenance in [`proofs/CONSTRAINT-THEORY-MATH-PROVENANCE.md`](proofs/CONSTRAINT-THEORY-MATH-PROVENANCE.md).

- [`proofs/XOR-ISOMORPHISM.v`](proofs/XOR-ISOMORPHISM.v) — **genuinely
  Qed-complete** (8 `Qed.`, 0 `Admitted.`): symmetric difference on finite
  sets forms an abelian group and the characteristic function is a
  homomorphism.
- [`proofs/INTENT-HOLONOMY-DUALITY.v`](proofs/INTENT-HOLONOMY-DUALITY.v) —
  **headline theorem NOT proven.** Only small supporting lemmas are
  `Qed`'d; the main duality is riddled with `admit.` and would not pass
  the kernel. The source author's own assessment (30% confidence, "one
  direction proven, converse open") is preserved in the provenance note —
  this repo does **not** claim the duality is proven.

These are adjacent constraint-theory material, not proofs of the 3.17×
result or of mixed-precision soundness. `coqc` was unavailable on both
the source and import hosts, so the `Qed.`s have not been re-checked by
a live kernel here.

## Bugs Found by Adversarial Testing

- INT8 overflow wrapping (4.9% mismatch) → fixed with range validation
- Dual-path subtraction overflow → fixed with XOR (6% faster than broken code)

## Quick Start

**C benchmarks (AVX-512).** Compile and link, then run on AVX-512 hardware:

```bash
gcc -O3 -mavx512f -mavx512bw -mavx512dq -o avx512_soa src/avx512_soa_benchmark.c -lm
./avx512_soa
```

The same flags build `src/avx512_multicore.c`, `src/e2e_pipeline_benchmark.c`,
and `src/rars_imu_proof.c`. Each file's header comment names its compile line.
**AVX-512 hardware is required to *run* them** (the published 3.17× was
measured on an AMD Ryzen AI 9 HX 370). On a non-AVX-512 host the binaries
compile and link cleanly but will `SIGILL` at runtime.

**ARM NEON fallback.** Cross-compile for aarch64, or build the scalar
fallback on x86 for a syntax/link smoke test:

```bash
aarch64-linux-gnu-gcc -O3 -march=armv8-a+simd -o neon_check src/neon_fallback.c
# x86 smoke test (no NEON instructions emitted):
gcc -O3 -DSCALAR_FALLBACK -o neon_check src/neon_fallback.c
```

**Python validation.** Requires the `polyformalism-a2a` package on the
import path (see `benchmarks/benchmark_intent_compilation.py`):

```bash
python3 benchmarks/benchmark_intent_compilation.py
```

**Coq proofs.** No toolchain bundled. To type-check the Qed-complete file:

```bash
coqc proofs/XOR-ISOMORPHISM.v   # 8 Qed, 0 Admitted
```

`proofs/INTENT-HOLONOMY-DUALITY.v` is **not** expected to compile — see the
provenance note.

## License

MIT — see [LICENSE](LICENSE).
