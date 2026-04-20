# BSP — Claude Code Instructions

## Project Identity

**BSP** — Binary Space Partitioning. Spatial geometry primitives for Cyrius.

- **Type**: Shared library — geometry foundation for games, compositors, spatial systems
- **License**: GPL-3.0-only
- **Language**: Cyrius (native, compiled via cc5 5.5.0)
- **Version**: SemVer, version file at `VERSION`
- **Binary contribution**: ~2KB compiled (837 lines across 9 modules)
- **Status**: v1.1.1 — STABLE on Cyrius 5.5.0. 79 tests passing, 13 benchmarks sub-microsecond, 3 fuzz harnesses (25K iterations). 1.1.1 adds `[lib]` manifest section and `dist/bsp.cyr` single-file bundle — first downstream consumer is cyrius-doom 0.26.0 (swaps its ad-hoc BSP traversal for bsp library primitives). 1.1.0 landed the signed-shift correctness audit (`asr()` for all signed shifts; fixed `aabb_center_*` INT64_MAX wrap and `bsp_point_seg_dist` symmetry on sub-precision segments).
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
- **vidya** — `content/cyrius/field_notes.toml` for Cyrius-specific gotchas (logical shift, fixed-point overflow)

## Development Process

### P(-1): Research (before implementing)

1. Check vidya for BSP/geometry patterns
2. Read relevant DOOM Black Book chapter for real-world usage
3. Document algorithm source in function header comment

### Work Loop (continuous)

1. Work phase — implement, fix, optimize
2. Build check: `cyrius build` or `cat tests/bsp.tcyr | cc5 > build/test && ./build/test`
3. Run tests: `cyrius test tests/bsp.tcyr` — assert "0 failed"
4. Run benchmarks: `cyrius bench benches/bsp.bcyr` — all sub-microsecond
5. Run fuzz: `cyrius fuzz` — fuzz/fuzz_intersect.cyr, fuzz/fuzz_aabb.cyr, fuzz/fuzz_blockmap.cyr
6. Review — correctness (algebraic properties), performance (ns/op), robustness (no crashes on random input)
7. Documentation — CHANGELOG, roadmap
8. Version check — VERSION, cyrius.toml all in sync

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
- **asr() for signed shifts.** Cyrius >> is logical. Use `asr(val, bits)` or the `fx_*` functions.
- **Guard all divisions.** Check denominator AND denominator-after-scaling for zero.
- **Property tests > value tests.** "distance is always >= 0" catches more bugs than "distance(5,5) == 7".
- **No globals.** Every function takes its data as arguments. The consumer owns the memory.
- **Sakshi available.** `lib/sakshi.cyr` included for consumer-side tracing, not used by library itself.

## Build & Test Commands

```sh
# Build
cyrius build src/lib.cyr build/bsp

# Test (74 assertions)
cyrius test

# Benchmark (13 ops, all sub-microsecond)
cyrius build benches/bsp.bcyr build/bench && ./build/bench

# Fuzz (3 harnesses: intersect, aabb, blockmap)
cyrius build fuzz/fuzz_intersect.cyr build/fuzz_i && ./build/fuzz_i

# Consumer usage via cyrius.toml git dep:
# [deps.bsp]
# git = "https://github.com/MacCracken/bsp"
# tag = "0.9.0"
# modules = ["src/lib.cyr"]
```

## DO NOT

- **Do not commit or push** — the user handles all git operations
- **NEVER use `gh` CLI** — use `curl` to GitHub API only
- Do not add globals — library must have zero global state
- Do not add I/O or syscalls — pure geometry only
- Do not use floating point — all 16.16 fixed-point
- Do not use bare `>>` on signed values — use `asr()` or `fx_*` functions
- Do not skip fuzz testing — it finds bugs that unit tests miss
- Do not add rendering code — this is pure geometry

## Documentation Structure

```
Root files (required):
  README.md, CHANGELOG.md, CLAUDE.md, CONTRIBUTING.md, SECURITY.md,
  CODE_OF_CONDUCT.md, LICENSE, VERSION, cyrius.toml

tests/:
  bsp.tcyr — 74 assertions across 15 test groups

benches/:
  bsp.bcyr — 13 benchmarks (fx_mul through check_sight)

fuzz/:
  fuzz_intersect.cyr — random segments, rays, distances (10K iters)
  fuzz_aabb.cyr — random AABB operations (10K iters)
  fuzz_blockmap.cyr — random insert + query (5K iters)
```

## CHANGELOG Format

Follow [Keep a Changelog](https://keepachangelog.com/). Performance claims MUST include benchmark numbers. Fuzz results (bugs found, iterations survived) should be noted.
