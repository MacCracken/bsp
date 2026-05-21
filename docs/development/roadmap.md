# BSP Development Roadmap

> **v1.1.3** — 94,640 B standalone (cycc 6.0.1; +18,144 B
> growth-tax from the v5.11.x annotation rt-table + v5.8.x sum-type
> emit). 79/79 tests, 13/13 benches sub-μs, 25K fuzz iters across
> 3 harnesses clean. Zero deps (stdlib-only). 849-line `dist/bsp.cyr`
> bundle consumed by **cyrius-doom 0.27.x**. Manifest hygiene:
> `cyrius.toml` retired, single `cyrius.cyml` with
> `version = "${file:VERSION}"` (matches patra/vani/sakshi/mihi
> convention). Source content byte-identical to 1.1.2 — pure
> Cyrius-pin lift.

## Completed

| Version | Milestone |
|---------|-----------|
| v0.5–v1.0 | Pre-stable arc (fixed-point, AABB, intersect, tree, traverse, query, blockmap, frustum modules) |
| v1.1.0 | Signed-shift correctness audit (`asr()` for all signed shifts; `aabb_center_*` INT64_MAX wrap + `bsp_point_seg_dist` symmetry fixes) |
| v1.1.1 | `[lib]` manifest section + `dist/bsp.cyr` single-file bundle (849 lines); cyrius-doom 0.26.0 is first downstream consumer |
| v1.1.2 | Cyrius 5.5.0 → 5.5.2 (enum-constant fold; −1,448 B standalone) |
| v1.1.3 | Cyrius 5.5.2 → 6.0.1 (covers v5.8.x language arc, v5.11.x annotation arc, v6.0.0 rename ceremony, v6.0.1 path hotfixes); manifest modernized (single `cyrius.cyml`, `${file:VERSION}`); +18,144 B growth-tax; bundle content byte-identical to 1.1.2 |

## v1.2.x — Language-adoption arc

**Theme** — mirror cyrius-doom's 0.27.x adoption sweep at the
bsp public-surface boundary. The 5.8.x → 6.0.1 language gains
(sum types, `Result<T, E>`, `?`, exhaustive match, parse-only
return-type annotations) are now mature in stdlib. bsp's
all-pure-functions / no-globals / no-IO design makes it a
particularly clean canvas for the adoption sweep — there's no
state to thread, so `Result` adoption is purely about contract
clarity at the API boundary.

### v1.2.0 — `: i64` return annotations on public surface

Mechanical sweep. Same shape as vani 0.9.3's annotation pass.
Parse-only, zero-codegen-change, ABI-identical. Documents the
return contract inline; sets up for v1.2.1's `Result` adoption.

| # | Item | Surface | Detail |
|---|------|---------|--------|
| 1 | `: i64` annotation on every `bsp_*` public fn | all 9 modules | Mechanical sweep; verify byte-identical codegen |
| 2 | Re-bench standalone bsp under the annotation pass | benches/ | Confirm variance-level deltas only |
| 3 | Regen `dist/bsp.cyr` with annotated signatures | dist/ | Bundle line count rises ~5 % (annotation suffixes) |

### v1.2.1 — `Result<T, E>` for fallible queries

Adopt the v5.8.28 `lib/result.cyr` carve-out at the small
fallible-query surface. bsp's pure-geometry primitives mostly
can't fail (`aabb_contains` is total; `fx_mul` saturates;
`bsp_point_on_side` is total), but a handful of queries DO have
a "couldn't find it" / "out of range" outcome currently encoded
as sentinel returns:

| # | Item | Module | Detail |
|---|------|--------|--------|
| 1 | `bsp_find_subsector_r` returns `Result<u32, BspError>` | traverse.cyr | `Err(BspOutOfBounds)` for point outside any subsector |
| 2 | `bsp_blockmap_query_r` returns `Result<count, BspError>` | blockmap.cyr | `Err(BspBlockmapCellOutOfRange)` for queries outside grid |
| 3 | `bsp_nearest_seg_r` returns `Result<SegRef, BspError>` | query.cyr | `Err(BspNoSegs)` when subsector is empty |
| 4 | Existing `i64`-return variants preserved per SemVer | all | Additive only — `_r` suffix on Result variants |

### v1.2.2 — Test surface refactor onto `lib/test.cyr`

Adopt the v5.7.43 `test_each(cases, fn)` helper. Current
`tests/bsp.tcyr` is ~79 hand-rolled asserts grouped by 14
sections. Table-driven cuts the boilerplate and makes property
extension trivial.

| # | Item | Detail |
|---|------|--------|
| 1 | Add `"test"` to bsp's stdlib pin | One-line manifest |
| 2 | Convert fixed-point / AABB / intersect asserts → `test_each` | ~40 asserts collapsed |
| 3 | Extend corpus once boilerplate drops | Add edge cases per group (overflow, sub-precision, zero-extent) |

## v1.3.x — Performance recovery (gated on Cyrius O3)

Same gate as cyrius-doom's 0.29.x. Currently bsp's release binary
flags ~65 KB of unreachable code as NOPed (via CYRIUS_DCE=1)
— the file size is unchanged because DCE NOPs in-place. Cyrius
O3 (IR-driven real DCE + const-prop + dead-store-elim) will
genuinely shrink the binary.

| # | Item | Status | Detail |
|---|------|--------|--------|
| 1 | Wait for **Cyrius O3** real DCE | Upstream | Recovers ~65 KB out of current 94,640 B standalone |
| 2 | Re-bench under O2/O3/O4 phases | Pending | bench-history per upstream phase landing |

## Future

| Item | Detail |
|------|--------|
| 3D BSP support | Currently 2D only (DOOM-style). 3D adds Z-plane partition for true 3D queries. |
| Quake-style BSP loader | Read .bsp file format directly (BSP1/BSP2/BSP3) — would let bsp consume id-Software-format static maps directly |
| Dynamic BSP rebuild | Currently static. Incremental rebuild for destructible-geometry consumers (kiran, phylax) |
