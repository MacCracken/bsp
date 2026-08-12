# BSP

> Binary Space Partitioning — spatial geometry primitives for the Cyrius ecosystem.

54 pure functions across 8 modules. No globals, no allocation, no I/O, no
floating point. You bring the buffers; BSP does the arithmetic.

## What It Does

- **Fixed-point math** — 16.16 multiply, divide, clamp, and a sign-preserving
  arithmetic shift (`asr`)
- **Bounding boxes** — AABB construction, growth, containment, intersection,
  merge, centre/extent
- **Segment intersection** — segment-segment and ray-segment, **division-free**
  (sign checks, never a divide), plus point-to-segment distance and
  line-of-sight through a wall list
- **BSP node layout** — the 112-byte node record, its accessors, and the
  subsector-reference encoding
- **Point-to-subsector** — the point-on-side test and the tree walk that
  resolves a point to its leaf
- **Blockmap** — a fixed grid spatial index: insert by bounding box, query by
  point + radius
- **Frustum culling** — build a 2D view wedge from two edge normals and reject
  points or AABBs against it

### What it does *not* do

The README claimed these for several releases; they were never implemented.

- **No tree construction.** `tree.cyr` gives you the node record and its
  accessors — `bsp_node_set`, `bsp_node_set_children` — but nothing builds a
  tree from segments or polygons. The caller (or its map compiler) does that.
- **No ordered traversal.** There is no front-to-back or back-to-front walk.
  BSP provides the decision primitives (`bsp_point_on_side`,
  `bsp_find_subsector`) and you write the traversal loop you need.
- **No segment-plane intersection.** This is a 2D library; the only planes are
  the frustum's two half-planes.
- `bsp_bbox_visible` is a **placeholder that always returns 1**. Real culling
  is `frustum_test_aabb`.

## Design

- **All math is 16.16 fixed-point** — no floating point, matches Cyrius kernel
  and DOOM conventions
- **Zero dependencies** — `src/` includes nothing, not even stdlib
- **Zero globals, zero heap** — every function takes its data as arguments and
  the caller owns all memory
- **No runtime bounds checking** — the limits below are contracts, not asserts,
  except where a guard is explicitly documented
- **Cyrius-native** — not ported from Rust

## Limits

These are real boundaries, not style guidance.

| Limit | Value | What happens past it |
|---|---|---|
| AABB coordinate | `\|coord\| < 2^50` fixed | `aabb_add_point` stops updating that edge; the box is silently wrong |
| Useful 16.16 precision | ~32768 world units | `fx_from_int` overflows the representable range |
| BSP node count | ≤ 32768 | rejected — bit 15 is the subsector flag, so a larger index is unaddressable |
| Child reference | must fit 16 bits | rejected, rather than masked to a wrong subsector |
| Segments per blockmap cell | `BM_CELL_MAX_SEGS` = 16 | rejected; `blockmap_insert_seg` returns the number of cells that dropped it |
| `fx_div` result | saturates to ±`FX_MAX` | saturation carries the sign of the true quotient |

`bsp_find_subsector` is hardened against malformed trees: a cycle, an
out-of-range child, or a negative child returns 0 rather than hanging, reading
out of bounds, or resolving to a garbage subsector.

**Cyrius `>>` is a LOGICAL shift.** Use `asr(v, n)` or the native `>>>` for any
signed value. `>>>` binds tighter than `+`, so parenthesise over a sum.

## Consumers

| Project | Usage |
|---------|-------|
| **cyrius-doom** | BSP rendering, collision detection, map geometry |
| **kiran** | Spatial queries, scene graph partitioning, culling |
| **aethersafha** | Window occlusion, compositor region queries |
| **phylax** | Zone-based threat mapping, spatial anomaly detection |

## Size Target

~1–2 KB of compiled contribution to a consumer binary. Geometry is just math.
(The standalone `build/bsp` is ~122 KB, almost entirely Cyrius prelude that a
real consumer already links.)

## Build & Test

Requires the Cyrius toolchain pinned in `cyrius.cyml` (currently 6.5.20).

```sh
cyrius build src/lib.cyr build/bsp
cyrius test tests/bsp.tcyr
cyrius fuzz
cyrius audit
```

168 assertions across 20 groups, 13 benchmarks, and 3 fuzz harnesses
auto-discovered from `fuzz/*.fcyr` (25,000 iterations as the standard gate;
each takes optional `[iters] [seed]` for stress runs).

## Use It

Either vendor the single-file bundle `dist/bsp.cyr` (1,182 lines, regenerate
with `cyrius distlib`), or declare a git dependency in your own `cyrius.cyml`:

```toml
[deps.bsp]
git = "https://github.com/MacCracken/bsp"
tag = "1.2.4"
modules = ["dist/bsp.cyr"]
```

The bundle imports no stdlib — it is pure geometry, and needs nothing in scope.

## License

GPL-3.0-only

## Project

Part of [AGNOS](https://agnosticos.org) — the AI-native operating system.
