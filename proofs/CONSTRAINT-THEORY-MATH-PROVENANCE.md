# Provenance: Coq Proof Sketches from `constraint-theory-math`

**Date:** 2026-05-07 (source); imported 2026-07-05
**Origin:** `SuperInstance/constraint-theory-math`
**Files:**
- `proofs/XOR-ISOMORPHISM.v`
- `proofs/INTENT-HOLONOMY-DUALITY.v`

---

## Why These Two Files Are Here

The source repository (`SuperInstance/constraint-theory-math`) is mostly
prose and markdown-wrapped proof sketches. These two files are the **only
genuinely machine-checkable Coq** in that repo: actual `.v` source, closed
with `Qed.`, and containing **zero `Admitted.`**. Every other "proof" in
the source repo is markdown narrative, not Coq. That is the entire reason
they were selected and copied here — the rest was left behind.

## What Is Actually Proven vs. Open

This is the part that must not be smoothed over.

### `XOR-ISOMORPHISM.v` — ✅ GENUINELY COMPLETE

- **8 `Qed.`, 0 `Admitted.`, 0 `admit.`**
- Proves that symmetric difference (`sym_diff`) on finite sets forms an
  abelian group (identity, self-inverse, commutativity, associativity),
  and that the characteristic function `χ` is a homomorphism
  (`xor_bool (χ s1) (χ s2) = χ (s1 Δ s2)`).
- The file's own header note claims a full bijective-isomorphism result;
  the `Qed`'d content delivers the group axioms and the homomorphism
  lemma. Bijectivity of `χ` on a finite universe is stated in comments,
  not separately `Qed`'d as a theorem, but the substantive algebraic
  content is real and closed.
- Status: **bring-your-own-Coq verification**. `coqc` was not available
  on the source host or on this host, so the `Qed.`s have not been
  re-checked by a live kernel here. The counts are verifiable by
  `grep -c 'Qed\.\|Admitted\.'`.

### `INTENT-HOLONOMY-DUALITY.v` — ⚠️ HEADLINE THEOREM NOT PROVEN

- **5 `Qed.`, 0 `Admitted.` — but 7 `admit.` tactics, and the file is
  not compilable as-is.**
- The five `Qed.`s cover only small supporting lemmas
  (`inj_mono_preserves_lower`, `inj_lt`, and base-case fragments).
- The file's **main theorem** — that an injective monotone endomap on a
  finite total order is the identity, from which the intent/holonomy
  duality would follow — is **riddled with `admit.`** and additionally
  contains stray narrative prose mid-file ("Hmm, this is getting
  messy…", "OK this is getting really hairy…") that is not valid Coq.
  The kernel would reject this file.
- In the source repo's own `SOURCE-ERRATA.md` (item 5,
  "Intent-Holonomy (B)⟹(A) — UNPROVEN"), the author writes:

  > (A)⟹(B) partially proven. (B)⟹(A) requires fixed-point
  > strengthening that has not been shown. Internal confidence: 30%.
  > Fix: Mark as "one direction proven, converse open." Not "duality
  > proven."

- **That framing is preserved verbatim in intent:** one direction has
  partial supporting lemmas; the converse is open; the author's own
  confidence is 30%. This repo does **not** claim the duality is proven,
  and the README does not either.

## Bottom Line

| File | Real Coq? | Headline result | Honest status |
|------|-----------|-----------------|---------------|
| `XOR-ISOMORPHISM.v` | Yes | Abelian group + homomorphism | ✅ Qed-complete (kernel re-check pending — no `coqc` on host) |
| `INTENT-HOLONOMY-DUALITY.v` | Partial | Full duality | ⚠️ Supporting lemmas only; converse open; author confidence 30% |

These proofs are **adjacent theory** (constraint-set algebra and an
intent/holonomy conjecture), not proofs of the mixed-precision
soundness or the 3.17× number that are this repo's actual result. They
are included because they are real and honestly errata'd, not because
they enlarge what this repo measures.
