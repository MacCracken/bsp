# Changelog

All notable changes to BSP will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
