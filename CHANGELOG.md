# Changelog

All notable changes to BSP will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
