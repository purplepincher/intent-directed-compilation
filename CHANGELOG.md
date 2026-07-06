# Changelog

All notable changes to this project are documented in this file. Format
follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

This is a research artifact (AVX-512 mixed-precision constraint-checking
benchmarks, a Python differential validator, two imported Coq proof files,
an IEEE-style writeup) with no package manifest and no registry publication
— versioning here tracks the repo's own history, not a distributed package.

## [Unreleased]

### Fixed

- README restructured to lead with Quick Start ahead of any narrative
  framing, and tightened for accuracy against the actual benchmark
  numbers and proof files.

### Added

- `proofs/CONSTRAINT-THEORY-MATH-PROVENANCE.md`: states plainly that
  `XOR-ISOMORPHISM.v` is Qed-complete (8 Qed, 0 Admitted) but
  `INTENT-HOLONOMY-DUALITY.v`'s headline theorem is **not** proven (5
  Qed, 7 `admit.`) — the source author's own 30% confidence assessment
  is preserved verbatim rather than smoothed over.

## [0.1.0] - 2026-05-08

Initial standalone extraction from the `polyformalism-thinking` sketchbook
project, with CI/CD added the same day. C benchmarks (AVX-512 SoA/AoS
mixed-precision kernels, NEON fallback), a Python differential validator,
and the paper draft (`paper/PAPER.md`) trace to this point.
