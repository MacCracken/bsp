# Changelog

All notable changes to BSP will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
