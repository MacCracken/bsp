# BSP — Claude Code Instructions

## Project Identity

**BSP** — Binary Space Partitioning. Spatial geometry primitives for Cyrius.

- **Type**: Shared library — geometry foundation for games, compositors, spatial systems
- **License**: GPL-3.0-only
- **Language**: Cyrius (native, compiled via cycc 6.5.20)
- **Version**: SemVer, single source of truth at `VERSION` (referenced
  via `version = "${file:VERSION}"` in `cyrius.cyml`)
- **Binary contribution**: ~2KB compiled (1154 lines across 8 modules
  in `dist/bsp.cyr`)
- **Status**: v1.2.4 — STABLE on Cyrius 6.5.20. **168 tests passing**,
  13 benchmarks, 3 fuzz harnesses (25K-iter standard gate),
  **0 undocumented public functions**.

  **1.2.4 rewrote `fx_div`**, which was wrong by >1 % on 95 % of divisions
  with a dividend past 32767 (worst case 241x off): the scale-down fired
  at |a| > 32767 when `a << 16` only overflows at 2^47, and because `asr`
  is FLOOR it sent positive divisors 1..255 to 0 (returning the
  divide-by-zero sentinel for representable quotients) and negative
  -1..-255 all to -1. The sentinel was also always +FX_MAX, inverting the
  sign. **This changed numeric output of `bsp_ray_cast` and
  `bsp_point_seg_dist`** — deliberately; the old values were wrong. The
  `|a| <= 32767` path is bit-identical.

  1.2.3 carried the **full audit** that found five correctness defects and
  two defects in the fuzz harnesses themselves. Highest-value lessons:
  - **`aabb_init`'s sentinels were ±2^31 in an i64 library** — any world
    coordinate past 32767.99 exceeded the seed, so a box grown from one
    point at (40000,40000) measured 7232 units wide. Now ±2^50
    (`AABB_EMPTY`), bounded ABOVE by `frustum_test_aabb`, which
    multiplies `asr(edge,4)` by up to 2^12 and needs the seed under 2^55.
  - **`bsp_find_subsector` never validated child indices** — a cyclic
    tree hung forever, and a NEGATIVE child was read as a leaf (two's
    complement sets bit 15) returning subsector 32763.
  - **The fuzz harnesses returned the raw LCG state**, so `% n`
    correlated with the call index — every `% 10` came out EVEN, making
    a single-point box unreachable. `_fz_rand` now mixes high bits down.
  - **A gate you have not mutation-tested is not a gate.** Widening the
    fuzz coordinates alone did NOT catch the aabb bug; the harness also
    lacked the property ("every point added must be contained"). Both
    were needed, and only running it against 1.2.2 proved it.

  1.2.3 is also a **free pin move plus one language upgrade**: the
  6.5.19 → 6.5.20 builds are **byte-identical** (same SHA256, 118,176 B),
  and `asr()` became a one-line alias for the native `>>>` operator
  (arithmetic right shift, added in Cyrius **v6.4.46** — unavailable
  while BSP was pinned to 6.3.5). That collapses a branch + negate +
  add + shift + negate into a single `sar`, **−128 B of emitted code**
  for a function called from 43 sites. No API or behaviour change:
  proven by a 600,576-case `asr` ≡ `>>>` differential and a 600,000-
  iteration whole-API differential (6 seeds, all 32 public functions,
  zero divergence), plus 1.8M fuzz iterations.

  6.5.20 fixes a silent `switch`/`match` miscompile (a case body
  leaving by anything but `return` hit a mis-patched jump table).
  **BSP is not exposed** — it uses no `switch`, `match`, or `#derive`.

- **Previously**: v1.2.2 — STABLE on Cyrius 6.5.19. 103 tests passing,
  13 benchmarks (10–311 ns/op), 3 fuzz harnesses (25K-iter standard
  gate). 1.2.2 is a **toolchain catch-up** release: the pin moves
  6.3.5 → 6.5.19, crossing the 6.4.x (86 patches) and 6.5.x (19
  patches) bands. No geometry source changed; behaviour proven
  identical to 6.3.5 over a 3M-iteration cross-toolchain differential
  probe (6 seeds, every public function, zero divergence). Standalone
  binary 98,240 → **118,176 B (+19,936 B, +20.3 %)** — the largest
  growth-tax this project has taken, entirely prelude/stdlib widening
  (dead-code accounting 384 fns/69,598 B → 433 fns/81,549 B) plus
  +896 B for the 6.3.12 W^X split into two `PT_LOAD` segments.
  Also: fuzz harnesses renamed `*.cyr` → `*.fcyr` so
  `cyrius fuzz` discovers them (it never had — CI now runs them);
  `atomic` + `result` declared in `[deps] stdlib` (they were
  transitive-only and so never re-vendored by `cyrius lib sync`);
  `cyrius vet` phantom-dep and stale-doc fixes.

  **Release lineage** (detail in CHANGELOG.md): 1.2.1 fixed `asr()` to
  FLOOR rather than round-toward-zero (cyrius-doom RC-F2). 1.2.0 was
  the deep audit / optimization release — `bsp_nearest_seg` far-query
  sentinel fix plus instruction-count hoists, 79 → 94 assertions.
  1.1.1–1.1.5 were packaging and pin moves; 1.1.1 added the `[lib]`
  section and the `dist/bsp.cyr` bundle. 1.1.0 landed the signed-shift
  correctness audit.
- **Genesis repo**: [agnosticos](https://github.com/MacCracken/agnosticos)
- **Standards**: [First-Party Standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-standards.md)
- **Shared crates**: [shared-crates.md](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/shared-crates.md)

## Consumers

| Project | Usage |
|---------|-------|
| cyrius-doom | BSP rendering, collision detection, map geometry |
| kiran | Spatial queries, scene graph partitioning, culling |
| aethersafha | Window occlusion, compositor region queries |
| phylax | Zone-based threat mapping, spatial anomaly detection |

## Architecture

```
src/
  lib.cyr         — public API (single include pulls all modules)
  fixed.cyr       — minimal 16.16 fixed-point math (fx_mul, fx_div, fx_abs, etc.)
  aabb.cyr        — axis-aligned bounding boxes (init, contains, intersects, merge)
  intersect.cyr   — segment intersection (division-free), ray cast, line-of-sight, point-seg distance
  tree.cyr        — BSP node layout (112 bytes/node), accessors, subsector helpers
  traverse.cyr    — point-on-side test, point-to-subsector BSP walk
  query.cyr       — point-in-subsector, nearest segment, count in AABB
  blockmap.cyr    — grid spatial index (insert, query by point+radius)
  frustum.cyr     — 2D view frustum construction and AABB culling
```

## Key Design Constraints

- **Zero globals** — consumers are already at their global limit. Pure functions only.
- **Zero dependencies** — no stdlib, no I/O, no rendering, no allocation
- **Zero heap allocation** — callers provide all buffers
- **All math is 16.16 fixed-point** — no FPU, matches DOOM and kernel conventions
- **Target size**: 1-2KB compiled contribution to consumer binary
- **Pure functions** — no state, no side effects. Data in, result out.

## References

- **[BSP FAQ](http://www.faqs.org/faqs/graphics/bsptree-faq/)** — BSP tree theory
- **[Game Engine Black Book: DOOM](https://fabiensanglard.net/gebbdoom/)** — BSP in practice (chapters 7-9)
- **vidya** — `content/cyrius/language/core.cyml` for the operator/precedence
  tables and the numbered gotcha list (logical `>>` vs arithmetic `>>>`,
  non-C precedence, term-operator intrinsic gap); `content/cyrius/field_notes/`
  for per-project notes (`doom.cyml` is the closest to BSP's domain)

## Development Process

### P(-1): Research (before implementing)

1. Check vidya for BSP/geometry patterns
2. Read relevant DOOM Black Book chapter for real-world usage
3. Document algorithm source in function header comment

### Work Loop (continuous)

1. Work phase — implement, fix, optimize
2. Build check: `cyrius build src/lib.cyr build/bsp`
3. Run tests: `cyrius test tests/bsp.tcyr` — assert "0 failed"
4. Run benchmarks: `cyrius bench benches/bsp.bcyr` — see the ns/op note below
5. Run fuzz: `cyrius fuzz` — auto-discovers fuzz/*.fcyr (intersect, aabb, blockmap)
6. Review — correctness (algebraic properties), performance (ns/op), robustness (no crashes on random input)
7. Documentation — CHANGELOG, roadmap
8. Version check — VERSION, cyrius.cyml, dist/bsp.cyr header all in sync

### Task Sizing

- **Low/Medium**: Batch freely
- **Large**: Small bites, verify each
- **If unsure**: Treat as large. Research via vidya first, then externally for information

### Refactoring

- Refactor when the code tells you to
- Never refactor speculatively. Wait for the third instance.
- Every refactor must pass tests + benchmarks + fuzz

### Key Principles

- **Fuzz first.** The fuzz harnesses found 3 real bugs that 74 unit tests missed (SIGFPE, negative width, division by zero).
- **Division-free when possible.** Segment intersection uses sign checks instead of division — avoids SIGFPE on degenerate inputs.
- **asr() for signed shifts.** Cyrius `>>` is LOGICAL; `>>>` is the
  ARITHMETIC (sign-preserving) one — the REVERSE of JS/Java. Use
  `asr(val, bits)` or the `fx_*` functions. `asr()` is now just
  `val >>> bits`, but keep calling it: `>>>` is on the **TERM** tier
  with `* / %`, binding TIGHTER than `+ - & | ^`, so `a + b >>> 16`
  parses as `a + (b >>> 16)`. Direct `>>>` over a sum MUST be
  parenthesised — `(a + b) >>> 16`. `asr()` has no such trap.
- **Guard all divisions.** Check denominator AND denominator-after-scaling for zero.
- **Property tests > value tests.** "distance is always >= 0" catches more bugs than "distance(5,5) == 7".
- **Mutation-test every new gate.** A test or fuzz invariant you have not
  seen FAIL is not evidence of anything. Run it against the previous
  release: it must be RED there and GREEN here. In 1.2.3 this caught two
  self-deceptions — widening the fuzz coordinates alone did *not* find the
  `aabb_init` bug (the harness's only real invariant, `width >= 0`, is
  satisfied by a corrupted box), and `np == 1` was unreachable because the
  PRNG's low bit alternated. Both looked like working gates.
- **Check the sampler, not just the invariant.** A power-of-two LCG's low
  bits have short periods, so `state % n` correlates with the call index.
  Mix high bits down before taking a modulus.
- **No globals.** Every function takes its data as arguments. The consumer owns the memory.
- **Benchmarks mean something again (6.5.19).** The bench harness now measures
  the clock-read floor (~1.3 µs) and subtracts it from every sample, so per-op
  averages are real: 10–311 ns/op instead of a uniform ~1.3 µs of timer
  overhead. Pre-1.2.2 guidance said perf wins could only be read as
  instruction-count/binary-size deltas — that is obsolete. Compare ns/op
  directly, but still confirm a win with a differential harness.
- **Measure binary size in a FRESH tree, one build per tree.** `cyrius build`
  auto-vendors the stdlib into `lib/` (gitignored), and the snapshot it picks
  comes from the **`cyrius =` pin in `cyrius.cyml`** — *not* from which `cycc`
  you run. A second build in a tree that already has `lib/` silently reuses the
  first vintage. Swapping compilers inside one working tree therefore measures a
  mixed configuration that never ships: doing that here produced a −5,664 B
  "shrink" when the honest number was +19,936 B. Use
  `git archive <tag> | tar -x -C <clean-dir>` and build once.

## Build & Test Commands

```sh
# Build
cyrius build src/lib.cyr build/bsp

# Release build: NOPs dead functions in-place — same file size, but
# unreachable code becomes inert. Used by release.yml.
CYRIUS_DCE=1 cyrius build src/lib.cyr build/bsp

# Test (168 assertions across 20 groups)
cyrius test tests/bsp.tcyr

# Benchmark (13 ops, 10–252 ns/op; harness subtracts the ~1.3us timer floor)
cyrius build benches/bsp.bcyr build/bench && ./build/bench

# Fuzz — all 3 harnesses, auto-discovered from fuzz/*.fcyr
cyrius fuzz
# Or one harness directly, with explicit iters + seed:
cyrius build fuzz/fuzz_intersect.fcyr build/fuzz_i && ./build/fuzz_i 200000 12345

# Refresh the vendored stdlib in lib/ after a pin bump (lib/ is gitignored)
cyrius lib sync

# Regenerate the consumer bundle after ANY src change or version bump.
# Writes dist/bsp.cyr (stamped from VERSION) + dist/bsp.deps (stdlib sidecar).
cyrius distlib

# Whole-project gate: fmt + lint + docs + tests + bench
cyrius audit

# Consumer usage: vendor dist/bsp.cyr, or declare a cyrius.cyml git dep:
# [deps.bsp]
# git = "https://github.com/MacCracken/bsp"
# tag = "1.2.4"
# modules = ["dist/bsp.cyr"]
```

## DO NOT

- **Do not commit or push** — the user handles all git operations
- **NEVER use `gh` CLI** — use `curl` to GitHub API only
- Do not add globals — library must have zero global state
- Do not add I/O or syscalls — pure geometry only
- Do not use floating point — all 16.16 fixed-point
- Do not use bare `>>` on signed values — it is LOGICAL. Use `asr()`,
  `fx_*`, or the native `>>>` (parenthesised over any sum)
- Do not skip fuzz testing — it finds bugs that unit tests miss
- Do not add rendering code — this is pure geometry

## Documentation Structure

```
Root files (required):
  README.md, CHANGELOG.md, CLAUDE.md, CONTRIBUTING.md, SECURITY.md,
  CODE_OF_CONDUCT.md, LICENSE, VERSION, cyrius.cyml

tests/:
  bsp.tcyr — 168 assertions across 20 test groups

benches/:
  bsp.bcyr — 13 benchmarks (fx_mul through check_sight)

fuzz/:
  fuzz_intersect.fcyr — random segments, rays, distances (10K iters)
  fuzz_aabb.fcyr — random AABB operations (10K iters)
  fuzz_blockmap.fcyr — random insert + query (5K iters)
  # .fcyr, not .cyr — that extension is what `cyrius fuzz` discovers.
  # Each takes optional [iters] [seed] argv for stress runs.

dist/ (generated by `cyrius distlib`):
  bsp.cyr — single-file bundle consumers vendor (committed)
  bsp.deps — sidecar 6.5.19+ emits from `[deps] stdlib`; gitignored, NOT
             shipped. Those are the harness deps, not the bundle's — the
             bundle imports no stdlib, so shipping it would make a
             consumer's `cyrius deps` demand leaves it doesn't need.
```

## CHANGELOG Format

Follow [Keep a Changelog](https://keepachangelog.com/). Performance claims MUST include benchmark numbers. Fuzz results (bugs found, iterations survived) should be noted.
