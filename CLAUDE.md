# BSP — Claude Code Instructions

## Project Identity

**BSP** — Binary Space Partitioning. Spatial geometry primitives for Cyrius.

- **Type**: Shared library — geometry foundation for games, compositors, spatial systems
- **License**: GPL-3.0-only
- **Language**: Cyrius (native)
- **Version**: SemVer, version file at `VERSION`
- **Status**: Scaffolded, pre-implementation

## Genesis Layer

Part of **AGNOS**. Genesis repo: `/home/macro/Repos/agnosticos`.

- **Standards**: `agnosticos/docs/development/applications/first-party-standards.md`
- **Shared crates**: `agnosticos/docs/development/applications/shared-crates.md`

## Architecture

```
src/
  lib.cyr         — public API
  tree.cyr        — BSP tree construction (partition line selection, recursive split)
  traverse.cyr    — front-to-back / back-to-front traversal from viewpoint
  query.cyr       — point-in-subsector, nearest-node, containment
  intersect.cyr   — line-line, line-plane, ray cast, segment clipping
  aabb.cyr        — axis-aligned bounding boxes (construct, intersect, contains, merge)
  blockmap.cyr    — grid spatial index (cell size, insert, query range)
  frustum.cyr     — view frustum planes, point/AABB rejection
  fixed.cyr       — 16.16 fixed-point math (or include from stdlib)
```

## Key Constraints

- **All math is 16.16 fixed-point** — no FPU, matches DOOM and kernel conventions
- **Zero dependencies** — pure geometry, no I/O, no rendering, no allocation
- **Fixed buffers** — max node count, max seg count defined at compile time
- **Target size** — 1-2KB compiled
- **Pure functions** — no state, no globals (except lookup tables)

## Development Process

1. **P(-1)** — vidya entry for BSP concepts before implementation
2. Implement in Cyrius
3. Test with programs/ (visualize BSP output as text or PPM)
4. Benchmark traversal and query operations
5. Integrate into cyrius-doom first, then kiran

## DO NOT

- **Do not commit or push** — user handles git
- **NEVER use `gh` CLI**
- Do not use floating point — all 16.16 fixed-point
- Do not add rendering code — this is pure geometry
