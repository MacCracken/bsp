# BSP

> Binary Space Partitioning — spatial geometry primitives for the Cyrius ecosystem.

## What It Does

- **BSP tree construction** — build from line segments or polygons
- **BSP traversal** — front-to-back and back-to-front from any viewpoint
- **Point queries** — which subsector/leaf contains a given point
- **Line intersection** — segment-segment, segment-plane, ray casting
- **Bounding boxes** — AABB construction, intersection, containment
- **Blockmap** — grid-based spatial lookup for collision detection
- **Frustum culling** — view cone rejection for rendering

## Design

- **All math is 16.16 fixed-point** — no floating point required, matches Cyrius kernel and DOOM
- **Zero dependencies** — pure geometry, no rendering, no I/O
- **Fixed buffers** — no heap allocation, bounded node/seg counts
- **Cyrius-native** — not ported from Rust

## Consumers

| Project | Usage |
|---------|-------|
| **cyrius-doom** | BSP rendering, collision detection, map geometry |
| **kiran** | Spatial queries, scene graph partitioning, culling |
| **aethersafha** | Window occlusion, compositor region queries |
| **phylax** | Zone-based threat mapping, spatial anomaly detection |

## Size Target

~1-2KB compiled. Geometry is just math.

## Build

Requires the Cyrius toolchain pinned in `cyrius.cyml` (currently 6.5.20).

```sh
cyrius build src/lib.cyr build/bsp
cyrius test tests/bsp.tcyr
cyrius fuzz
```

## Use It

Either vendor the single-file bundle `dist/bsp.cyr`, or declare a git
dependency in your own `cyrius.cyml`:

```toml
[deps.bsp]
git = "https://github.com/MacCracken/bsp"
tag = "1.2.4"
modules = ["dist/bsp.cyr"]
```

## License

GPL-3.0-only

## Project

Part of [AGNOS](https://agnosticos.org) — the AI-native operating system.
