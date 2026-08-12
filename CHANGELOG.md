# Changelog

All notable changes to BSP will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.4] - 2026-08-12 — `fx_div` rewritten, and the rest of the audit backlog

1.2.3's audit verified only the top two findings per lens; twelve were carried
unverified. Working through them found that **`fx_div` — the division primitive
under every ray cast and point-to-segment distance — was wrong by more than 1 %
on 95 % of divisions with a dividend past 32767**, worst case **241× off**.

**This release changes numeric output for valid input.** That is deliberate:
the old outputs were incorrect. The blast radius is `fx_div` itself and its two
callers, `bsp_ray_cast` and `bsp_point_seg_dist` (and `bsp_nearest_seg` /
`bsp_point_in_subsector` through the latter). `aabb`, `tree` and `frustum` are
bit-identical — confirmed per-module with the whole-API differential.

**Assertions 136 → 168.**

### Fixed — `fx_div` was wrong on ordinary input

The 1.1.0–1.2.3 body scaled **both** operands down by 8 bits whenever
`|a| > 32767`, then divided:

```
if (fx_abs(aa) > 32767) { aa = asr(aa,8); bb = asr(bb,8);
                          if (bb == 0) { return FX_MAX; } }
return (aa << 16) / bb;
```

Three separate defects, all on valid input, all measured:

1. **The scale-down triggered ~2^32 times too eagerly.** `aa << 16` only
   overflows i64 at `|a| >= 2^47`, but the guard fired at 32767 — so almost
   every division with a dividend above ~0.5 in 16.16 threw away 8 bits of
   *both* operands for no reason. This is the bulk of the error: measured over
   290,835 divisions whose true quotient is representable, the old code was
   **worse than 1 % on 276,028 of them (95 %)**, with a worst case of
   **241,103 parts per thousand — 241× off**. The new code is **exact on every
   one** (worst-case relative error 0).
2. **`asr` is FLOOR, so scaling the divisor was sign-asymmetric.** A positive
   divisor of 1..255 floored to **0** and returned the divide-by-zero sentinel
   for a perfectly representable quotient — `fx_div(100000, 255)` gave
   `FX_MAX` instead of `25700392`. A negative divisor of −1..−255 floored to
   **−1**, so *every* divisor in that range produced the **same** answer:
   `fx_div(100000, -1)` and `fx_div(100000, -255)` both returned `-25559040`,
   up to 256× off.
3. **The sentinel was always `+FX_MAX` regardless of sign — a sign inversion.**
   `fx_div(-100000, 100)` returned `+2147483647` where the true quotient is
   `-65536000`. Reachable through `bsp_point_seg_dist`, whose projection term
   `fx_div(dot, len_sq)` takes a signed `dot`: the wrong-signed result then hit
   `fx_clamp(t, 0, FX_ONE)` and pinned the projection to the *far* endpoint.

Now: scale only when the shift would actually overflow, scale the divisor by
**magnitude** so both signs behave identically, and saturate to `±FX_MAX` with
the sign of the true quotient. Divide-by-zero saturates by sign too. Bounds are
compared directly rather than through `fx_abs`, because `fx_abs(INT64_MIN)` is
itself negative.

The **small-dividend path (`|a| <= 32767`) is bit-identical** — 200,000
randomised control samples, 0 mismatches — so this is strictly a fix to inputs
that were already being computed wrongly. No performance cost: `fx_div` 17 ns,
`ray_cast` 59 ns, `point_seg_dist` 89 ns, all unchanged.

### Fixed — remaining hardening backlog

- **`blockmap_cell_seg` range-checked `idx` against the constant cap, not the
  cell's live count.** 1.2.3 added `idx < BM_CELL_MAX_SEGS`, which keeps the
  read inside the cell but still hands back whatever sits in its **unused**
  slots — a stale index from an earlier `blockmap_init`, or uninitialised
  memory. It now checks against `load64(cell)`, the live count.
- **`blockmap_init` reported success while zeroing nothing.** With `cols` or
  `rows` <= 0 the zeroing loop ran zero times and it still returned `buf`, so
  the caller received a "valid" blockmap whose cell counts had never been
  initialised. It now returns **0** for unusable dimensions, and caps them so
  `cols * rows` cannot overflow i64 and wrap negative (which would skip the
  zeroing the same way).
- **Child references above 16 bits — the high-side twin of the negative-child
  hole closed in 1.2.3.** Bit 15 is the subsector flag, so `0x18005` passed
  `bsp_is_subsector` and `bsp_subsector_idx` masked it to **subsector 5**.
  Rejected now. Likewise `node_count > 32768` is rejected outright: such a tree
  cannot be addressed by a 15-bit index at all, and the root index itself would
  have bit 15 set and be read as a leaf (`node_count` 40000 previously resolved
  to subsector 7231).

### Quality gates (on Cyrius 6.5.20)

- **Tests**: **168 passed, 0 failed** (was 136), with two new groups —
  `fx_div correctness (1.2.4)` and `malformed-input hardening (1.2.4)`.
- **Mutation-tested against 1.2.3**: the new assertions produce **17 failures**
  when run against the 1.2.3 source, and pass here. One assertion was
  *rewritten* after it turned out not to discriminate — `0x18000` masks to
  subsector 0, which is also what the fix returns, so it passed either way;
  it now uses `0x18005`, which masks to 5.
- **`fx_div` differential vs the 1.2.3 body**, both compiled into one binary:
  500,000 adversarial samples characterised, plus a **200,000-sample control on
  `|a| <= 32767` with 0 mismatches**, and a 290,835-sample accuracy comparison
  against the exact quotient (old worst error 241,103 ppt, new 0).
- **Whole-API differential, per module**: `fixed`, `isect` and `query` change
  as intended; **`aabb`, `tree` and `frustum` are bit-identical**.
- **Fuzz**: `cyrius fuzz` 3/3, plus **3,000,000 iterations** (3 harnesses ×
  5 seeds × 200K), 0 failures.
- **`cyrius audit`**: fmt clean, lint clean, docs complete, tests 168/0,
  bench 1/0. **`cyrius vet`: 8 deps, 0 untrusted, 0 missing.**

## [1.2.3] - 2026-08-12 — Cyrius 6.5.20, native `asr()`, and a full audit

Three pieces of work: a **free** toolchain pin move, the first payoff from having
crossed into the 6.4.x band (native `>>>`), and a **full codebase audit**.

The audit found **five correctness defects** — a bounding box that has been
silently wrong for any coordinate past 32767.99 world units since the library
existed, a tree walk that hangs forever on a cyclic tree, a containment query
that reports a point as inside an *empty* set, an accessor that SIGSEGVs on the
value its own sibling is documented to return, and a spatial index that drops
segments with no way for the caller to find out — plus **two defects in the fuzz
harnesses themselves**, which is why none of the above had been caught. Every
harness had been drawing correlated random numbers, and the one that should have
caught the bounding-box bug could not reach it and would not have asserted it.

**Assertions 103 → 136. Undocumented public functions 33 → 0.**

### Fixed — audit findings

- **`aabb_init` sentinels were ±2^31 in a 64-bit library — silent wrong
  containment.** Coordinates are 16.16 fixed-point in **i64**, so any world
  coordinate above 32767.99 produces a value LARGER than the seed and
  `aabb_add_point`'s comparisons never fire. Measured: a box grown from the
  single point `(40000, 40000)` came out **7232 world units wide** and reported
  `(35000, 35000)` — a point 5000 units away — as **inside** it. No crash, no
  assertion: silently wrong containment, and therefore silently wrong frustum
  culling, since `tree.cyr` hands out node bboxes for direct use with `aabb_*`.

  This is the **same defect class as the `bsp_nearest_seg` sentinel fixed in
  1.2.0**, and the library's own test suite already exercised world coordinates
  of 40000/60000 in the query group — it was asserting a range `aabb_init`
  could not represent. Seeds are now **±2^50**, named `AABB_EMPTY`.

  The magnitude is bounded from **both** sides, which is the whole subtlety:
  - **Not INT64_MIN/INT64_MAX.** `aabb_width` computes `RIGHT - LEFT`, and
    `INT64_MIN - INT64_MAX` wraps to `+1`, so a fresh box would report width 1
    while still satisfying `fuzz_aabb`'s `width >= 0` invariant.
  - **Not 2^61 either** — the first magnitude tried, and it *overflows
    `frustum_test_aabb`*. That function computes `lx = asr(LEFT - vx, 4)` and
    then `fx_mul(lx, lnx)` where `lnx` reaches 2^12 for a unit normal, so the
    product must stay under 2^63 and the sentinel under **2^55**. Measured at
    2^61: the raw product came out as exactly **0** (the low 64 bits of 2^69),
    which would silently flip a corner's half-plane sign and cull or un-cull a
    fresh box arbitrarily. At 2^50 that same product is 2^58 — well inside
    range, with 32× headroom under the ceiling.

  A fresh box therefore still reports width 0, and `aabb_center_x` on one now
  returns exactly **0**; the asymmetric ±2^31 pair returned **−1**.

  **Documented consequence**: `aabb_init` now states its supported range —
  `|coord| < 2^50` fixed (~1.7e10 world units). It cannot simply be raised,
  for the frustum reason above, and 16.16 has only ~32768 world units of useful
  precision regardless. The `fuzz_aabb` coordinate bands were set to straddle
  the *old* 2^31 seed while staying under the new one, so the harness tests
  inside the contract rather than past it.

  *Behaviour change*: correct results now replace incorrect ones for
  coordinates above 32767.99 world units. DOOM-range consumers (cyrius-doom)
  were never in that range; kiran, aethersafha and phylax are not DOOM-bounded.

- **`bsp_point_in_subsector` reported a point as INSIDE an empty set, and
  `bsp_nearest_seg` returned a valid-looking index into nothing.** Both tested
  `seg_count == 0` rather than `<= 0`, so a **negative** count skipped the loop
  and fell through to the success path: `bsp_point_in_subsector(_, _, segs, -1)`
  returned **1** (inside), and `bsp_nearest_seg(_, _, segs, -1)` returned **0**.
  The containment one inverts in the more dangerous direction, since callers
  branch on it for collision. Both now use `<= 0`. Checked and deliberately left
  alone: `bsp_count_in_aabb` already returns 0 for a negative count, and
  `bsp_check_sight` returns 1 ("visible"), the conservative direction for a
  sight test — both are pinned by new assertions.

- **`bsp_find_subsector` could hang forever, read out of bounds, or return a
  garbage subsector.** `node_count` was passed but used only to pick the root —
  nothing validated the child indices being followed. Three distinct failures,
  all reachable from map data a consumer did not author:
  1. **A cycle hung the process.** Proven with a 2-node cycle: it spins
     indefinitely. The new test for this *hangs* the suite on 1.2.2 rather than
     failing it — verified, exit 124 on a 45 s timeout.
  2. **A child index past the array read arbitrary memory.**
  3. **A NEGATIVE child was silently accepted as a LEAF.** Two's complement
     sets bit 15, so `bsp_is_subsector(-5)` is 1 and `bsp_subsector_idx`
     returned `-5 & 0x7FFF` = **32763**. No hang and no crash here — the caller
     then indexed *its* subsector array with 32763. This one had to be rejected
     where the child is read, because the loop condition had already classified
     it as a leaf.

  The walk is now bounded by `node_count` and range-checked. Valid trees are
  unaffected; `find_subsector_3node` measures 45 ns, unchanged.

- **`blockmap_cell_seg` took SIGSEGV on the value its own sibling returns.**
  `blockmap_get_cell` is documented to return **0** for an out-of-range cell.
  `blockmap_cell_count` already tolerated that 0; `blockmap_cell_seg`
  dereferenced it — so the documented `get_cell` → `cell_seg` sequence crashed
  on any out-of-range coordinate (reproduced: exit 139). It now returns −1, and
  range-checks `idx` as well: an unchecked `idx` read into the *next* cell's
  data and returned a segment that is not in this cell.

- **`blockmap` silently discarded segments with no way to detect it.** A cell
  holds at most `BM_CELL_MAX_SEGS` (16); past that `blockmap_add` rejects, and
  `blockmap_insert_seg` ignored the rejection and always returned **0**. Dense
  geometry hits this with entirely valid input — the segment is simply absent
  from the index, so `blockmap_query_point` never reports it and the caller's
  narrowphase never tests it. Measured on a random-geometry probe: **~3,800
  dropped inserts per 2,000-iteration run, all previously invisible.**
  `blockmap_insert_seg` now returns the **number of cells that rejected** the
  insert (0 = fully indexed). The old return was a constant, so reading it is
  new information rather than a changed contract.

### Fixed — the fuzz harnesses were systematically under-sampling

- **All three harnesses returned the raw LCG state, so `% n` correlated with
  the call index.** A power-of-two LCG has period 2^(k+1) in bit k — bit 0
  simply **alternates**. Instrumented `fuzz_aabb`: **every `% 10` draw came out
  EVEN across 50,000 iterations**, so `np == 1` was *unreachable* and a
  single-point box was never once constructed. `_fz_rand` now mixes the high
  bits down (`x ^= x >> 31; x ^= x >> 17;` — `>>` being Cyrius's LOGICAL shift,
  which is what a bit-mixer wants).

- **`fuzz_aabb` could not reach the bug it should have caught, twice over.**
  Its coordinates were `fx_from_int((r % 2000) - 1000)` — a maximum of ~65.5M
  fixed, about **32× below** the old ±2^31 sentinel — and its only real
  invariant was `width >= 0`, which a *corrupted* box satisfies. Coordinates now
  span three bands (DOOM-scale, ~100k world units, and ±2^45 fixed — above the
  old sentinel, below the new one) and the harness asserts the property that
  actually pins them: **every point added to a box must be contained in it**,
  plus a single-point box having zero width and height.

  **Mutation-tested, which is the only thing that makes it a gate:** the new
  harness is **RED on the 1.2.2 source** (exit 1, `FAIL: single-point box has
  nonzero width`, on all three seeds tried) and **GREEN on 1.2.3**. The old
  harness was green on both.

### Fixed — two benchmarks were measuring the wrong thing

Both were found while re-checking a performance claim, and both had been
producing numbers quoted in previous CHANGELOG entries.

- **`check_sight_8walls` measured ONE wall, not eight.** The walls are vertical
  lines at x = 50, 150 … 750 and the ray ran horizontally at y = 500 from x = 0
  to x = 1000 — so wall 0 blocked it and `bsp_check_sight` returned at the first
  iteration. The ray now stops at x = 40, reaching no wall, so all 8 are tested:
  the true worst case the name claims. **The honest number is ~290 ns, not the
  ~51 ns previously reported.**
- **`bench_blockmap_query` called `alloc(128)` inside the timed body**, so it
  measured the allocator on every iteration. The buffer is now allocated once in
  `bench_setup`. **The honest number is ~165 ns, not the ~305 ns previously
  reported.**

### Changed — performance

- **`bsp_check_sight`: ~290 ns → ~245 ns (−15 %).** The sight ray is fixed
  across the loop, so its pre-shifted deltas are loop-invariant, but
  `bsp_seg_intersect` recomputed `asr(tx - sx, 8)` / `asr(ty - sy, 8)` once per
  wall. They are now hoisted and passed to an internal `_seg_intersect_pre`.
  `bsp_seg_intersect` deliberately keeps its own inline copy of the body rather
  than delegating — routing it through the helper costs the primitive a call it
  cannot afford (seg_intersect_hit 52 → 59 ns, miss 39 → 46 ns, measured).

  **This reverses a call made earlier in this same release.** The hoist was
  first implemented, measured as worthless, and reverted — on the broken
  1-wall benchmark above, where hoisting work out of a loop that runs once
  obviously saves nothing. Re-measured on the corrected 8-wall benchmark it is
  a clear win. Recorded because the failure mode generalises: *an optimisation
  measured against a broken benchmark was rejected for the wrong reason.*

- **`blockmap_query_point`: no measurable change** — reported honestly, since
  the earlier "305 → 252 ns (−17 %)" claim in this release was an artifact of
  the allocator being inside the timed body. Measured pre/post on the corrected
  benchmark: **165–175 ns before, 165–169 ns after.** The change is kept as an
  instruction-count reduction with no regression, not as a speedup: the inner
  loop called `blockmap_cell_count` and `blockmap_cell_seg` per segment, but
  `cell` is provably non-zero there and `i < n <= BM_CELL_MAX_SEGS`, so their
  guards could never fire — the loads are now direct, dropping two calls and
  five branches per segment. And on reaching `out_max` the loop kept scanning
  every remaining cell to store nothing; it now returns immediately, which
  cannot change the returned count.

### Added — documentation

- **Every public function is now documented: 33 undocumented → 0**
  (`cyrius audit` docs stage reports `ok: docs complete`). `cyrius doc`
  associates a comment with a function only when it is **directly adjacent** —
  a single blank line between a banner comment and its `fn` made it invisible,
  which is most of why the count was so high. Documentation is **provably
  codegen-inert**: the compiled binary is byte-identical (same SHA256) before
  and after the doc pass.
- Contracts that were previously implicit are now stated: `fx_to_int` floors
  toward negative infinity; `fx_mul` is not associative across a chain;
  `fx_div` returns `FX_MAX` on divide-by-zero and cannot be distinguished from
  a real `FX_MAX`; `_seg_check` requires a non-zero denominator (a zero one
  falls through both branches and reports parallel segments as intersecting);
  `blockmap_cell_x`/`_y` return unclamped indices; the `bsp_node_*` accessors
  do no bounds checking; `blockmap_insert_seg` indexes by bounding box, not by
  exact rasterisation.
- `programs/test_bsp.cyr` documented and its stale build line fixed — it told
  readers to pipe through `cc2`, a command as long-dead as the `cyrb` reference
  fixed in 1.2.2. Its header now states plainly that it is a hand-run smoke test
  that always exits 0 and gates nothing; `tests/bsp.tcyr` is the real suite.

### Changed — toolchain and `asr()`

The pin move costs **nothing** — the 6.5.19 and 6.5.20 binaries are
**byte-identical** (same SHA256, 118,176 B, same 433 fns / 81,549 B dead-code
accounting) — and `asr()` becomes the first real payoff from having crossed into
the 6.4.x/6.5.x band in 1.2.2.

- **Cyrius pin 6.5.19 → 6.5.20** (`cyrius.cyml`). **Zero growth tax and zero
  codegen change**: fresh-tree builds at both pins produce bit-identical
  binaries. This is the cheapest pin move in the project's history, and stands in
  deliberate contrast to 1.2.2's +19,936 B.

  6.5.20's headline fix is a **silent miscompile of `switch` / `match`**: a case
  body that exited by anything other than `return` fell through a jump table that
  the v5.6.27 regalloc NOP-harvest compactor had shifted by +4 bytes per
  preceding body — wrong answer with no diagnostic, or SIGSEGV. **BSP is not
  exposed**: it contains no `switch`, no `match`, and no `#derive` (grep-verified
  across `src/`, `tests/`, `benches/`, `fuzz/`). The pin is taken for currency
  and for the toolchain-drift warning, not to fix anything here.

- **`asr()` reimplemented on the native `>>>` operator.** Cyrius gained `>>>` —
  an arithmetic, sign-preserving right shift — in **v6.4.46**. BSP was pinned to
  6.3.5 until yesterday's 1.2.2, so every release from 1.1.0 through 1.2.2
  synthesised the negative path by hand:

  ```
  if (val >= 0) { return val >> bits; }
  var pos = 0 - val;
  return 0 - ((pos + (1 << bits) - 1) >> bits);
  ```

  That is now `return val >>> bits;` — a branch, a negate, an add, a shift and a
  second negate collapsed into a single `sar`. **Emitted code shrinks 81,549 →
  81,421 B (−128 B)** for a function called from **43 sites across 4 modules**
  and sitting under every `fx_mul` / `fx_to_int` in the renderer's hot path.

  `asr()` is **kept as a function, not removed** — it is public API that
  consumers call, and it remains the safe spelling. `>>>` sits on the **TERM**
  precedence tier alongside `* / %`, binding *tighter* than `+ - & | ^`, so
  `a + b >>> 16` means `a + (b >>> 16)`. Any direct use over a sum must
  parenthesise; `asr()` has no such trap. Bypassing the call in `fx_mul` /
  `fx_to_int` / `fx_div` was measured and produced **no further size win**, so it
  was not taken.

  **This is a pure internal substitution — no API, no behaviour, no semantics
  change.** `asr()` remains FLOOR division by 2^bits (the 1.2.1 RC-F2 fix), which
  is exactly what `sar` does for negative values.

- **README** now states the 6.5.20 pin and the `1.2.3` consumer tag.

### Quality gates (on Cyrius 6.5.20)

- **Tests**: **136 passed, 0 failed** (was 103) — including the seven explicit
  `asr` floor assertions (`asr(-3,1) = -2`, `asr(-9,2) = -3`,
  `fx_to_int(-0.5) = -1`) that pin the semantics the `>>>` substitution had to
  preserve, and a new **malformed-input hardening** group of 27 assertions
  covering every audit fix above.
- **The hardening group is mutation-tested against 1.2.2**: run there, the suite
  prints the group header and then **hangs** (exit 124 on a 45 s timeout) at the
  cyclic-tree assertion — the failure mode being pinned. The remaining
  assertions fail on 1.2.2 rather than passing vacuously.
- **`asr()` ≡ `>>>` differential**: **600,576 cases, 0 mismatches**. Every shift
  width 0–31 against 18 pathological values (0, ±1, ±3, ±65535, ±65536,
  ±2147483647, ±2ᶟ²,  −(2⁶³−1)), a 300K random sweep over ±2e9, and a further
  300K over the exact `x * y >>> 16` shape `fx_mul` produces. All four shift
  widths BSP actually uses (1, 4, 8, 16) are literals inside that range.
- **Whole-API differential, 1.2.2 source vs 1.2.3 source**: one probe calling
  **every public function** — all 32 of them — folding results into six
  per-module checksums, compiled *unchanged* against both trees.
  **6 seeds × 100,000 iterations = 600,000 per build: every checksum identical,
  zero divergence.** Corpus spans sub-precision, normal-world, ±2e9 extreme and
  degenerate/tiny coordinate bands — i.e. everything *below* the old ±2^31
  `aabb_init` sentinel, which is precisely the range where the audit fixes are
  required to change nothing. Above it, behaviour intentionally changes from
  wrong to correct.
- **Blockmap differential** (added because the API probe did not cover it):
  query results — `blockmap_query_point` including the `out_max` truncation
  path, `get_cell`, `cell_count` over the whole grid plus out-of-range
  coordinates — are **bit-identical across all 6 seeds**, while the same runs
  surface **~3,800 previously-invisible dropped inserts** each.
- **Fuzz**: `cyrius fuzz` 3/3 pass (25K standard gate), plus a stress run of
  **3,000,000 iterations** (3 harnesses × 5 seeds × 200K) with the corrected
  PRNG — which reaches generator states no previous run could — **0 failures**.
  The deep runs earned their keep: at 200K they initially failed on every seed,
  correctly reporting that the first sentinel magnitude tried (2^61) did not
  dominate the coordinate band being generated.
- **Benches**: 13/13, and **two of them were fixed before being trusted** (see
  above). `check_sight_8walls` **~290 → ~245 ns (−15 %)** is the one repeatable
  performance win. `blockmap_query` ~165 ns before and after — no change.
  `fx_mul` 11 ns, `find_subsector_3node` 45 ns (unchanged despite the new
  bounds checks), `seg_intersect_hit` 52 ns. Everything except check_sight is
  inside run-to-run noise and should be read as *no regression*, not as a win.
- **`cyrius audit`**: fmt clean, lint clean (0 warnings, 0 untracked deferrals),
  **docs `ok: docs complete`**, tests 130/0, bench 1/0. **`cyrius vet`: 8 deps,
  0 untrusted, 0 missing.** `cyrius check dist/bsp.cyr`: ok.
- **Bundle**: `cyrius distlib` regenerated `dist/bsp.cyr` (v1.2.3). Re-running
  `distlib` reproduces it byte-for-byte.

## [1.2.2] - 2026-08-11 — Cyrius 6.5.19 catch-up

Toolchain catch-up release. **The Cyrius pin moves 6.3.5 → 6.5.19**, crossing the
6.4.x band (86 patches) and the 6.5.x band (19 patches) — roughly 130 toolchain
releases. **No geometry source changed**, and behaviour is proven identical to
6.3.5 (see Quality gates). It carries the largest growth-tax in this project's
history: **98,240 → 118,176 B (+19,936 B, +20.3 %)**, all of it in the
prelude/stdlib, none of it in BSP's own emitted code.

Alongside the pin, this release closes three pieces of drift that had gone
unnoticed because nothing was checking for them: the fuzz harnesses had a file
extension `cyrius fuzz` no longer discovers (so CI had never run them), two
vendored stdlib modules were never being re-synced, and `cyrius vet` was
reporting a phantom dependency.

### Changed

- **Cyrius pin 6.3.5 → 6.5.19** (`cyrius.cyml`). Clears the `toolchain drift`
  warning that every local build had been printing (`cyrius.cyml pins 6.3.5 but
  cycc is 6.5.19`).
- **Vendored stdlib re-synced** from the 6.5.19 snapshot via `cyrius lib sync`.
  **13 of the 14 declared modules changed** between 6.3.5 and 6.5.19 (only
  `str.cyr` is byte-identical); `alloc.cyr` alone grew 26,485 → 42,247 B.
- **Fuzz harnesses renamed `fuzz/*.cyr` → `fuzz/*.fcyr`.** `cyrius fuzz`
  discovers `fuzz/*.fcyr`, so with the old names it reported *"No fuzz harnesses
  found"* and exited **clean** — the project's most productive gate had silently
  been a no-op. This is long-standing drift, **not** something the pin move
  introduced: 6.3.5 expected `.fcyr` too, so `cyrius fuzz` had been finding
  nothing here for at least the whole 6.3.x line. Only the documented manual
  path (`cyrius build fuzz/fuzz_intersect.cyr …`) ever actually ran them.
  All three now auto-discover and pass. Harness contents are unchanged; this is
  a pure rename, and `.fcyr` matches the ecosystem convention (abaco, agnodrm,
  agnosai, ai-hwaccel).
- **`benches/bsp.bcyr` numbers are meaningful again.** The 6.5.19 bench harness
  measures the clock-read floor (~1.3 µs here) and subtracts it from every
  sample. Previously every op reported a near-uniform 1.34–1.75 µs that was
  almost entirely timer overhead; the same 13 benchmarks now report true per-op
  cost. No code changed — only what the harness reports.

### Added

- **`atomic` and `result` declared in `[deps] stdlib`.** Both are transitive-only
  requirements (`alloc` → `atomic`, `io` → `result`) and so were invisible to
  `cyrius lib sync`, which syncs the *declared* set. The copies in `lib/` were
  therefore never refreshed — the vendored `atomic.cyr` predated even the 6.3.5
  pin the project was sitting on. They now sync with everything else, and they
  correctly appear in the `dist/bsp.deps` sidecar consumers read.
- **`dist/*.deps` gitignored.** 6.5.19's `distlib` emits a `dist/bsp.deps`
  sidecar derived from `[deps] stdlib`. Those are BSP's *harness* dependencies —
  the bundle itself is pure geometry and imports none of them — so shipping the
  sidecar would make a consumer's `cyrius deps` demand leaves it does not need.
  The file is generated locally and not committed.
- **CI now runs the fuzz harnesses** (`cyrius fuzz`, asserting `0 failed`).
  They had never run in CI.
- **CI now checks the `dist/bsp.cyr` version header** against `VERSION`. A stale
  bundle header is a shipped-wrong-version bug for every consumer that vendors
  the file, and nothing caught it before.

### Fixed

- **`cyrius vet` phantom dependency.** `src/lib.cyr`'s usage comment contained
  the literal text `include "bsp/src/lib.cyr"`; `vet` parses that pattern even
  inside a comment and reported `MISSING bsp/src/lib.cyr` — a self-referential
  dep that never existed. Reworded the comment: `vet` is now
  **8 deps, 0 untrusted, 0 missing** (was 9 deps, 1 missing).
- **Stale documentation.** `README.md` told readers to build with `cyrb build`,
  a command that has not existed for several major versions. `CLAUDE.md`
  claimed a `lib/sakshi.cyr` that is not present, and quoted 74/79/94 assertion
  counts and a `v1.2.0` status against an actual 103. The CI test job was still
  named *"Test (79 assertions)"*.

### Quality gates (on Cyrius 6.5.19)

- **Cross-toolchain differential**: a probe exercising **every public function**
  across all 8 modules — including degenerate, sub-precision, and ±2e9 inputs —
  was compiled *unchanged* under 6.3.5 and under 6.5.19 and run over
  **3,000,000 iterations (6 seeds × 500K)**. Per-module and total checksums are
  **byte-identical, zero divergence**. This is what licenses the "no behaviour
  change" claim; the pin move was not assumed safe.
- **Tests**: 103/103 pass, unchanged from 1.2.1 (17 groups).
- **Fuzz**: standard 25K gate (10K intersect + 10K aabb + 5K blockmap) clean,
  plus **600K extra stress** (400K intersect across 2 seeds, 100K aabb,
  100K blockmap) — all clean.
- **Benches**: 13/13, now `fx_mul` 12 ns → `blockmap_query` 311 ns per op.
- **Static gates**: `cyrius lint` 0 warnings, `cyrius fmt --check` clean,
  `cyrius deny` 0 violations, `cyrius vet` 0 missing, `cyrius audit` passes,
  `cyrius check dist/bsp.cyr` ok.
- **Binary (standalone bsp)**: **98,240 → 118,176 B (+19,936 B, +20.3 %)** —
  the largest pin-move growth-tax this project has taken. `CYRIUS_DCE=1` is
  identical at 118,176 B (DCE NOPs in place; it does not shrink the file).
  Directly comparable to the 98,208 B recorded for 1.2.0.

  **Measurement method matters here** — the naive way to measure this produces a
  number that is wrong by 26 KB. `cyrius build` **auto-vendors the stdlib into
  `lib/`**, and the snapshot it vendors is chosen by the **`cyrius =` pin in
  `cyrius.cyml`**, not by which `cycc` you invoke. So a second build in a tree
  that already has a `lib/` reuses whatever vintage got vendored first, and
  measuring old-vs-new by swapping compilers in one working tree silently
  compares a *mixed* configuration that never exists in practice. Every figure
  above was taken in a **fresh tree per build** (`git archive <tag>` into a
  clean dir, no `lib/`). Ablation confirms the pin is the whole story: reverting
  only the `cyrius =` line in the 1.2.2 tree gives 97,248 B, while reverting the
  `[deps] stdlib` change, the `dist/bsp.deps` file, or the fuzz rename each
  leaves 118,176 B untouched.

  **Attribution** — the growth is prelude/stdlib widening, not BSP codegen. The
  dead-code accounting moves from **384 unreachable fns / 69,598 B** to
  **433 / 81,549 B**; BSP contributes 8 modules either way. Of the total,
  **+896 B is the 6.3.12 W^X change** — the binary now has two
  permission-separated `PT_LOAD` segments instead of one (`readelf -l`: 2 vs 1;
  `CYRIUS_WX=0` builds to 117,280 B). That 896 B buys a non-writable text
  segment and is worth paying.
- **`dist/bsp.cyr`** regenerated at 871 lines; only the version header line
  differs from 1.2.1, since no module source changed.

## [1.2.1] - 2026-07-12 — `asr()` is now FLOOR, not round-toward-zero

Bug-fix release. Reported by the cyrius-doom 0.33.6 audit (RC-F2). **The Cyrius pin
is unchanged at 6.3.5.**

### Fixed

- **`asr(val, bits)` rounded negatives toward ZERO instead of flooring.** The old
  negative path returned `-((-val) >> bits)`, so e.g. `asr(-3, 1)` gave `-1` where a
  true arithmetic (sign-preserving) shift — what C's `>>` on signed ints does, and
  what DOOM's fixed-point + coordinate math assume — floors to `-2`. Every downstream
  `fx_to_int` / `fx_mul` on a negative operand inherited the truncation, which in the
  cyrius-doom renderer mis-wrapped flats by one texel over **negative world
  coordinates** and doubled the texel band straddling a world axis. Now floors:
  negative `val` returns `-((|val| + 2^bits - 1) >> bits)`. Positive values are
  unchanged, so all-positive call sites (the common case) are byte-identical.
  +9 `asr`/`fx_to_int` floor assertions (`tests/bsp.tcyr`), suite **94 → 103**;
  geometry tests (point-side, blockmap, frustum, traverse) unchanged and green.



Deep audit / optimization release — the first **source-change** release since
1.1.0 (1.1.1–1.1.5 were packaging and Cyrius-pin moves). **The Cyrius pin is
unchanged at 6.3.5; this is a pure source-quality release.** A multi-agent
audit fanned out across five dimensions (overflow/correctness, redundant
computation, refactor/dedup, dead-code/API, precision/numerics); every
candidate was adversarially verified for semantic equivalence before landing.
12 findings applied (1 rejected as unreachable-but-harmless). The one
behavioral change is a bug fix with a new regression test; every performance
change is proven **bit-identical** to 1.1.5.

### Fixed

- **`bsp_nearest_seg` far-query correctness bug** — the running-minimum was
  seeded at `2147483647` (2³¹−1 ≈ 32768.0 world units), but
  `bsp_point_seg_dist` returns DOOM approximate-distance values up to
  ~1.5·2³² (~6.4e9) at 16.16 scale. When the nearest segment lay farther than
  ~32768 world units, `dist < best_dist` never fired and the function silently
  returned segment index **0** instead of the true nearest. Re-seeded with
  `INT64_MAX` (`9223372036854775807`); `FX_MAX` is unusable here because it
  *equals* the buggy sentinel. Reproduced empirically (query (0,0): seg0 @
  60000 won over the nearer seg1 @ 40000) and guarded by a new regression
  assertion. DOOM-scale maps span ±32768 units, so cross-map nearest queries
  hit this routinely.

### Changed — performance (instruction-count reductions, behavior-identical)

Proven bit-identical to 1.1.5 over **500,000 random-input differential
iterations** (old-vs-new for `fx_div` / `bsp_seg_intersect` / `bsp_ray_cast` /
`frustum_test_aabb`, zero divergence). `asr` is a pure deterministic
sign-branch+shift, and `fx_mul` rounds each product independently, so every
hoist below preserves the exact result.

- **`bsp_seg_intersect`** — pre-shift each of the 6 deltas with `asr(...,8)`
  once instead of twice per use: **12 `asr` → 6**. On the per-wall hot path of
  `bsp_check_sight`.
- **`bsp_ray_cast`** — same hoist on `dx`/`dy`/`sdx`/`sdy`/`abx`/`aby`:
  **12 `asr` → 6**.
- **`frustum_test_aabb`** — each half-plane has only 4 distinct corner
  products shared across its 4 corner tests; hoisted them: **16 `fx_mul` → 8**
  (largest single op reduction).
- **`bsp_point_on_side`** — compute the node base (`node_idx * BSP_NODE_SIZE`)
  once and `load64` the fields directly, replacing 4 accessor calls that each
  recomputed the multiply. Hot path under `bsp_find_subsector`.
- **`blockmap_query_point`** — the cell coordinates are provably in range
  after clamping, so the per-cell `blockmap_get_cell` call is inlined to a
  direct address computation, dropping a function call, two redundant
  `cols`/`rows` reloads, four bounds checks, and an always-true `cell != 0`
  guard per visited cell.
- **`bsp_count_in_aabb`** — hoist `entities + i*16` into a base local (matches
  the `base`-once pattern already used by the file's other per-element loops).

### Changed — refactor / clarity (behavior-identical)

- **`fx_div`** — unified the two mutually-exclusive scale-down branches
  (`aa > 32767` and `aa < -32767`) into a single `fx_abs(aa) > 32767` test
  (relies on the existing intra-module forward reference, same mechanism as
  `_approx_dist` in intersect.cyr). −276 B of unreachable-fn footprint.
- **`bsp_bbox_visible`** — removed 4 dead `load64` calls; the loaded AABB
  fields were never read before the unconditional `return 1`. Signature
  preserved (params kept) for API stability and the future cull implementation.
- **`bsp_point_side`** — replaced the terse overflow comment with a quantified
  note on the >>8 precision/overflow tradeoff and why the `asr 4` alternative
  must not be adopted without re-running tests + fuzz (it moves the
  side-classification boundary).
- **`blockmap_query_point` header** — rewritten to state the actual broadphase
  contract: AABB-window candidate collection, **no** exact-radius narrowing,
  per-cell duplicates expected; caller dedupes/narrows in narrowphase.

### Added

- **Test coverage for `query.cyr` and `frustum.cyr`** (both previously
  untested). New **"spatial queries"** group (`bsp_nearest_seg` incl. the
  far-query regression, `bsp_count_in_aabb`, `bsp_point_in_subsector`) and
  **"view frustum"** group (`frustum_test_point` + `frustum_test_aabb` golden
  values, locking in the product hoist). **79 → 94 assertions.**

### Quality gates (on Cyrius 6.3.5)

- **Tests**: 94/94 pass (was 79; +15).
- **Differential**: 500,000 random iters, OLD(1.1.5) == NEW for every
  refactored numeric fn — zero divergence (the address-hoist refactors are
  covered by the unit tests + fuzz harnesses).
- **Benches**: 13/13 still sub-microsecond (min 488–907 ns; per-op averages
  remain harness-dominated, no regression attributable to the changes).
- **Fuzz**: standard 25K gate (10K intersect + 10K aabb + 5K blockmap) **plus**
  300K extra stress (200K intersect across 2 seeds, 50K aabb, 50K blockmap) —
  all clean.
- **Binary (standalone bsp)**: 98,560 → **98,208 B (−352 B, −0.36 %)** — a
  genuine shrink (not a pin-move growth-tax) from removed dead loads, halved
  `asr`/`fx_mul` counts, and the unified `fx_div` branch. `CYRIUS_DCE=1` build
  identical at 98,208 B.
- **`dist/bsp.cyr`** regenerated from the optimized sources (849 → 890 lines;
  growth is the added explanatory comments). The unchanged `aabb` and `tree`
  sections are byte-identical to 1.1.5, confirming bundle-format fidelity.

### Roadmap note

This release reprioritizes the v1.2.x arc: the originally-planned 1.2.0 `: i64`
annotation sweep (parse-only, zero-codegen) is deferred to 1.2.1, since the
deep audit surfaced real correctness and performance wins worth shipping first.

## [1.1.5] - 2026-06-29

### Changed

- **Cyrius 6.3.5** — toolchain pin bump from 6.2.11. Pure pin move, no
  source changes. Crosses the 6.3.0 minor band plus the 6.3.x patch
  line up to 6.3.5. Library source is untouched; the delta is entirely
  codegen + the version-pinned stdlib snapshot
  (`~/.cyrius/versions/6.3.5/lib/`) the build resolves against. Clears
  the toolchain-drift warning (cycc had already advanced to 6.3.5 while
  the manifest still pinned 6.2.11).
- **Binary growth (standalone bsp)**: 97,608 → 98,560 B (**+952 B,
  +0.98 %**) on 6.3.5. Honest growth-tax from the 6.2.11 → 6.3.5
  prelude/codegen widening (unreachable-fn footprint 68,743 → 69,925 B,
  373 → 384 fns). Consistent with the default-expected growth pattern
  documented for the Cyrius roadmap
  ([[feedback_perf_deltas_growth_tax_default]]); `CYRIUS_DCE=1` NOPs the
  unreachable code in place (same file size, inert bytes) per the
  release-build contract. 79/79 tests pass, 13/13 benches sub-μs
  (min 419–838 ns, variance-level noise vs 6.2.11), 25K fuzz iters across
  3 harnesses (10K intersect + 10K aabb + 5K blockmap) clean.
- **`dist/bsp.cyr` header** bumped to `Version: 1.1.5`. Bundle code is
  byte-identical to 1.1.4 — pure pin change, no source drift; downstream
  consumers (cyrius-doom) re-fetch the same 849-line concatenation.
- **CI** unchanged structurally — `cyrius.cyml`'s `cyrius =` line remains
  the single source of truth for the toolchain version that ci.yml /
  release.yml install, so the 6.3.5 bump propagates with no workflow
  edits. Release job's pre-flight HTTP check will gate on the 6.3.5
  tarball being published upstream.

## [1.1.4] - 2026-06-15

### Changed

- **Cyrius 6.2.11** — toolchain pin bump from 6.0.1. Pure pin move, no
  source changes. Crosses two minor bands (6.0.x → 6.1.0 → 6.2.x) plus
  the 6.2.x patch line up to 6.2.11. Library source is untouched; the
  delta is entirely codegen + the version-pinned stdlib snapshot
  (`~/.cyrius/versions/6.2.11/lib/`) the build resolves against.
- **Binary growth (standalone bsp)**: 94,640 → 97,608 B (**+2,968 B,
  +3.14 %**) on 6.2.11. Honest growth-tax from the 6.0.1 → 6.2.11
  prelude/codegen widening (unreachable-fn footprint 65,572 → 68,743 B,
  365 → 373 fns). Consistent with the default-expected growth pattern
  documented for the Cyrius roadmap
  ([[feedback_perf_deltas_growth_tax_default]]); `CYRIUS_DCE=1` NOPs the
  unreachable code in place (same file size, inert bytes) per the
  release-build contract. 79/79 tests pass, 13/13 benches sub-μs
  (min 419–908 ns, variance-level noise vs 6.0.1), 25K fuzz iters across
  3 harnesses (10K intersect + 10K aabb + 5K blockmap) clean.
- **`dist/bsp.cyr` header** bumped to `Version: 1.1.4`. Bundle code is
  byte-identical to 1.1.3 — pure pin change, no source drift; downstream
  consumers (cyrius-doom) re-fetch the same 849-line concatenation. (The
  6.2.x `cyrius distlib` emits a reformatted header/separator style; the
  curated header is preserved here so the bundle stays a one-line diff.)
- **CI** unchanged structurally — `cyrius.cyml`'s `cyrius =` line remains
  the single source of truth for the toolchain version that ci.yml /
  release.yml install, so the 6.2.11 bump propagates with no workflow
  edits. Release job's pre-flight HTTP check will gate on the 6.2.11
  tarball being published upstream.

## [1.1.3] - 2026-05-21

### Changed

- **Cyrius 6.0.1** — toolchain bump from 5.5.2. Pure pin move, no
  source changes. Picks up the v5.7.x → v6.0.1 band: v5.8.x sum-type
  / `Result<T,E>` / `?` / exhaustive-match infrastructure, v5.11.x
  annotation arc (`fn foo(): i64`), v5.11.59 DCE-aware undef-fn
  reachability filter, v5.11.60 `_exec3` argv/envp byte-contract fix,
  v5.11.65 CVE-05 tok_names mangle-path overflow fix, v6.0.0
  `cyrc → cybs` + `cc5 → cycc` binary rename ceremony, v6.0.1
  stdlib-resolution path hotfixes.
- **Binary growth (standalone bsp)**: 76,496 → 94,640 B (**+18,144 B,
  +23.7 %**) on 6.0.1. Honest growth-tax from the v5.8.x sum-type
  emit + v5.11.x annotation-arc rt-table widening. Cyrius roadmap
  documents this as default-expected
  ([[feedback_perf_deltas_growth_tax_default]]); v6.0.x byte-array
  literal peephole + v6.0.x dead-code careful sweep are expected
  to recover a portion. 79/79 tests pass, 13/13 benches sub-μs
  (variance-level noise), 25K fuzz iters across 3 harnesses clean.
- **`cyrius.cyml`** — `version = "${file:VERSION}"` single-source pattern
  (matches patra/vani/sakshi/mihi). Legacy `cyrius.toml` compat shim
  removed — the .cyml manifest is the only manifest now.
- **`dist/bsp.cyr` header** bumped to `Version: 1.1.3`. Bundle content
  otherwise byte-identical to 1.1.2 — pure pin change; downstream
  consumers (cyrius-doom ≥ 0.27.0) re-fetch the same 849-line
  concatenation.
- **CI modernization** (matches patra/cyrius-doom CI shape):
  - `.cyrius-toolchain` retired — was pinned to `5.5.0`, would have
    installed wrong cyrius for the 6.0.1 pin in `cyrius.cyml`. CI now
    reads the toolchain version from `cyrius.cyml` (single source of
    truth, same pattern vani/yukti/patra/cyrius-doom use).
  - Version-pinned install layout: `~/.cyrius/versions/$V/{bin,lib}/`
    (cycc 6.0.1's stdlib resolver prefers this path).
  - Pre-flight HTTP check on the cyrius release tarball — surfaces a
    clear error if `cyrius.cyml`'s `cyrius =` pin was bumped ahead of
    the published release (same pattern patra v1.9.0 CI added).
  - Docs-job required-files list: `cyrius.toml` → `cyrius.cyml`.
  - New `Verify version consistency` step in the docs job —
    `${file:VERSION}` template-aware (resolves to `VERSION` contents),
    asserts VERSION file == cyrius.cyml `version =` line and that
    VERSION appears in CHANGELOG.md.

### Forward (1.2.x outlook)

- Sum-type / `Result<T,E>` adoption survey at the public API boundary
  — `bsp_*_r` Result-returning variants for the small surface that
  can fail (out-of-bounds subsector lookups, blockmap-cell ranges
  outside the grid). Additive only; existing `i64`-return signatures
  preserved per SemVer.
- Type annotations (`: i64`, `: Result`) on the public surface as
  consumer-side documentation. Mechanical sweep; zero-codegen-change.
- O3 real-DCE re-bench once the cyrius pass lands — current 65 KB
  of NOPed unreachable code becomes a real binary shrink.

## [1.1.2] - 2026-04-20

### Changed

- **Cyrius 5.5.2** — toolchain bump from 5.5.0. Pure pin move, no source
  changes. Picks up the 5.5.2 enum-constant sc_num fold: every `enum`
  variant read now emits `mov rax, imm32` (5 bytes) instead of the old
  `mov rcx, gvaddr; mov rax, [rcx]` indirect load (~10 bytes). bsp is
  enum-dense (`BspFixed`, `BspNode`, `BBox`, `BlockmapConst`,
  `FrustumField`) so the win compounds.
- **Binary shrink (standalone bsp)**: 77,944 → 76,496 B (**−1,448 B,
  −1.86 %**) on 5.5.2. All 13 benches still sub-μs (variance-level
  noise), 79/79 tests pass, 25K fuzz iters clean.
- **`dist/bsp.cyr` header** bumped to `Version: 1.1.2`. Bundle content
  otherwise identical — the bump is a pure pin change and consumers
  re-fetch the same 849-line concatenation.

## [1.1.1] - 2026-04-20

### Added

- **`[lib]` section in `cyrius.cyml`** — lists `src/*.cyr` modules in
  include order for the `cyrius distlib` target. Migrated the manifest
  from TOML→CYML (keeping `cyrius.toml` alongside for build-tool
  compatibility during the 5.x transition). Pattern mirrors
  `libro/cyrius.cyml`.
- **`dist/bsp.cyr`** — single-file bundled distribution (849 lines)
  concatenating the 9 `src/*.cyr` modules in dependency order (fixed →
  aabb → intersect → tree → traverse → query → blockmap → frustum).
  Generated to match Cyrius's single-pass forward-reference resolution.
  Consumers vendor this one file into their own `lib/` — identical shape
  to how `libro` consumes `sigil.cyr` and `patra.cyr`.

### Consumers

- **cyrius-doom 0.26.0** — first real consumer of bsp-as-a-library. Prior
  versions had `Composes: bsp` in CLAUDE.md but rolled their own BSP
  traversal in `render.cyr`. 0.26.0 vendors `dist/bsp.cyr` to
  `lib/bsp.cyr` and replaces `map_point_on_side` / `render_bsp_node`'s
  ad-hoc primitives with `bsp_point_on_side` / `bsp_node_child_r/l` /
  `bsp_is_subsector` / `bsp_subsector_idx` (112-byte node layout is
  already compatible — same field offsets).

### Gates

- Tests 79/79, benches 13/13 sub-μs, fuzz 25K iters — unchanged from
  1.1.0 (no source logic changes; this is purely packaging).

## [1.1.0] - 2026-04-20

### Fixed (signed-shift correctness audit)

DOOM escaped these because its coords are integer-fx (`x << 16`), which
makes the low bits align to cancel under Cyrius's logical `>>`. Any
consumer with sub-pixel fx, scene-graph deltas, or AABBs that span the
origin would hit them immediately. Per bsp's own CLAUDE.md rule
("Do not use bare `>>` on signed values — use `asr()`"), all violating
sites fixed. Reproduction probes demonstrated the bugs on real inputs
before fixing.

- **`aabb_center_x` / `aabb_center_y`** — bare `>> 1` on a possibly-negative
  sum. `aabb_center_x(left=-5, right=3)` returned `9223372036854775807`
  (INT64_MAX) instead of `-1`. Now uses `asr(...,1)`.
- **`bsp_point_seg_dist`** — bare `>> 8` on signed `pdx`/`pdy` garbled the
  projected-point dot product for any input quadrant whose `>>8`-scaled
  delta wasn't divisible by 256 (i.e. almost anything outside DOOM's
  integer-fx vertex grid). Now uses `asr(..., 8)`.
- **`bsp_point_seg_dist` degenerate/sub-precision path** — when `>>8`
  scaling collapsed `len_sq` to zero, the function returned
  `|p - sx1|` regardless of which endpoint was actually closer,
  breaking symmetry. Extracted `_approx_dist(dx, dy)` helper and the
  degenerate path now picks the nearer of `sx1`/`sx2`.
- **`frustum_test_point` / `frustum_test_aabb`** — bare `>> 4` on signed
  deltas AND on signed plane normals. Any frustum with a viewer not at
  the origin, or with inward-pointing normals that went negative, could
  reject or accept the wrong side. Refactored to pre-scale signed
  operands once via `asr(...,4)` (also dedups the `(left-vx)>>4`,
  `(top-vy)>>4` etc. repeated sub-expressions across the 8 corner
  tests).

### Changed

- **Cyrius 5.5.0** — toolchain bump from 4.6.2. cc3 → cc5 rename;
  `cyrius.toml` pin raised to 5.5.0; `.cyrius-toolchain` 4.0.0 → 5.5.0.
- **Negative literals modernized** — `0 - 2147483648` → `-2147483648`
  (`aabb_init`), `return 0 - 1` → `return -1` (ray_cast, nearest_seg,
  point_in_subsector initial "no-seg-yet" sentinel).

### Added

- 5 new assertions in `tests/bsp.tcyr` under the "negative-coord /
  signed-shift correctness (1.1.0)" group guarding the three regression
  cases above.
- Internal `_approx_dist(dx, dy)` helper (not public API).

### Quality gates (on 5.5.0)

- **Tests**: 79/79 pass (was 74; +5 regressions).
- **Benches**: 13/13 still sub-microsecond. `point_seg_dist` min 489ns,
  `point_on_side` min 838ns, `fx_mul` min 489ns. No regression attributable
  to the asr() changes; variance within run-to-run noise.
- **Fuzz**: `fuzz_intersect` 10K + `fuzz_aabb` 10K + `fuzz_blockmap` 5K —
  all pass clean with the fixes in.
- **Lint**: 0 warnings across all 9 src modules.

## [1.0.1] - 2026-04-14

### Changed

- **Cyrius 4.6.2** — rebuilt and verified. 74/74 tests pass, 13 benchmarks all sub-microsecond, 3 fuzz harnesses (25K iterations) pass. No code changes. Minimum version bumped.

## [1.0.0] - 2026-04-13

### Stable Release

BSP is production-ready. API stable. Used as primary spatial geometry library for cyrius-doom.

### Changed

- **Cyrius 4.4.3 modernization** — negative literals (`-val` instead of `0 - val`), compound assignment (`+=` for loop counters and accumulators), `cyrius = "4.4.3"` minimum in cyrius.toml. 10+ modernizations across fixed.cyr, frustum.cyr, blockmap.cyr, intersect.cyr, query.cyr.
- Bumped from 0.9.0 → 1.0.0 to signal API stability. No breaking changes.

### Stability Guarantees

- **API**: All public functions documented in README/CLAUDE.md stable across 1.x
- **Zero globals**: library uses no global state (consumer owns all data)
- **Zero dependencies**: no stdlib in library core (only in tests/bench/fuzz harnesses)
- **Zero heap allocation**: callers provide all buffers
- **All math is 16.16 fixed-point**: no FPU required
- **74 tests**, 13 benchmarks, 3 fuzz harnesses passing on cc3 4.4.3
- Byte-identical compilation on both x86_64 and aarch64 backends

## [0.9.0] - 2026-04-13

### Changed

- **cyrius.toml [deps] system** — stdlib dependencies (string, alloc, vec, fmt, args, assert, bench) declared in cyrius.toml and auto-resolved via `cyrius deps`. Removed 24 unused vendored stdlib modules from `lib/` (31 → 7 files).
- **Fuzz harnesses fixed** — replaced `str_to_int(str_from(argv(N)))` with `atoi(argv(N))` (stdlib function). Removed unused `vec.cyr` include from fuzz files. Eliminates `str_to_int`/`str_from` undefined function warnings.
- **Segment intersection refactored** — extracted `_seg_check()` helper from `bsp_seg_intersect()` for cleaner factoring of division-free range checks.
- Removed stale `cyrb.toml` (replaced by `cyrius.toml`)
- Minimum Cyrius version: 3.10.1 (auto-include, undefined function diagnostic)
- 74 tests, 13 benchmarks, 3 fuzz harnesses (25K iterations) all passing

## [0.7.1] - 2026-04-09

### Changed
- Cyrius toolchain pinned to v3.2.5 (cc3 compiler, minimum version)

## [0.7.0] - 2026-04-09

### Fixed

- **Critical: asr() for all signed right shifts** — Cyrius >> is logical (zero-fill), not arithmetic. All `fx_mul`, `fx_to_int`, `fx_div`, and cross-product calculations now use `asr()` for correct negative value handling
- `fx_to_int` rewritten — removed broken OR bitmask approach, uses `asr(f, 16)` 
- `fx_div` overflow guard uses `asr()` instead of bare `>>`
- All `>> 8` shifts in intersect.cyr replaced with `asr(x, 8)`

### Changed

- Requires cyrius 2.4.0+ (expanded globals)
- 74 tests still passing after shift fixes

## [0.6.0] - 2026-04-08

### Added

- Sakshi 0.7.0 slim profile available in `lib/sakshi.cyr` for consumers

## [0.5.2] - 2026-04-08

### Fixed

- `fx_div` overflow on large numerators — guard with scaled-down division
- `bsp_ray_cast` SIGFPE on extreme coordinate values — division-free sign checks before dividing
- `bsp_point_seg_dist` crash on near-zero length segments — guard `len_sq <= 0`
- `aabb_width` / `aabb_height` returning negative for inverted boxes — clamp to 0

### Added

- `fuzz/fuzz_intersect.cyr` — 10,000 random segment pairs, found + fixed SIGFPE
- `fuzz/fuzz_aabb.cyr` — 10,000 random AABB operations, found + fixed negative width
- `fuzz/fuzz_blockmap.cyr` — 5,000 random insert + query cycles

## [0.5.1] - 2026-04-08

### Changed

- CI uses `cyrb test` / `cyrb bench` via install script
- Added `cyrb.toml` with stdlib deps, test and bench entries
- Release workflow bootstraps Cyrius from upstream install script

## [0.5.0] - 2026-04-08

### Added

- **fixed.cyr** — Minimal 16.16 fixed-point math (fx_mul, fx_div, fx_abs, fx_min, fx_max, fx_clamp)
- **aabb.cyr** — Axis-aligned bounding boxes: init, set, add_point, contains_point, intersects, merge, width, height, center
- **intersect.cyr** — Segment-segment intersection (division-free), ray casting, line-of-sight check, point-to-segment distance
- **tree.cyr** — BSP node layout (112 bytes/node), accessors, subsector flag helpers, set/get functions
- **traverse.cyr** — Point-on-side test (cross product), point-to-subsector BSP walk, AABB visibility stub
- **query.cyr** — Point-in-subsector (convex test), nearest segment, count entities in AABB
- **blockmap.cyr** — Grid spatial index: init, insert segment across cells, query by point+radius, cell accessors
- **frustum.cyr** — 2D view frustum: build from viewer angle, test point, test AABB (4-corner rejection)
- **lib.cyr** — Public API: single `include "src/lib.cyr"` pulls all modules

### Design

- Zero globals, zero dependencies, pure functions
- All math is 16.16 fixed-point
- No heap allocation — callers provide all buffers
- No I/O, no rendering — pure geometry

### Tests

- **tests/bsp.tcyr** — 74 assertions across 15 test groups
- Covers: fixed-point arithmetic, AABB ops, segment intersection, ray casting, point-on-side, BSP traversal, blockmap queries, line-of-sight

### Benchmarks

- **benches/bsp.bcyr** — 13 benchmarks, all sub-microsecond
- fx_mul: 449ns, seg_intersect: 465ns, find_subsector(3-node): 484ns, blockmap_query: 838ns, check_sight(8 walls): 502ns

### Build

- Test binary: 30KB (includes stdlib + test harness)
- Library alone: ~2KB compiled (821 lines across 8 modules)
- Compiles clean with cc2 2.2.0+

## [0.1.0] - 2026-04-08

### Added

- Project scaffolded
- Architecture defined: tree, traverse, query, intersect, aabb, blockmap, frustum
