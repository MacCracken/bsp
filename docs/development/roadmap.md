# BSP Development Roadmap

> **v1.2.4** — 122,280 B standalone (cycc 6.5.20). 168/168 assertions across
> 20 groups, 13/13 benches, 3 fuzz harnesses (25K standard gate, clean at 3M
> stress), 0 undocumented public functions. 1,182-line `dist/bsp.cyr` bundle.
>
> 1.2.3 and 1.2.4 together were the **audit arc**: seven real defects, five of
> them silent-wrong-answer rather than crash, plus two defects in the fuzz
> harnesses that explain why the others survived so long.

## Completed

| Version | Milestone |
|---------|-----------|
| v0.5–v1.0 | Pre-stable arc (fixed-point, AABB, intersect, tree, traverse, query, blockmap, frustum modules) |
| v1.1.0 | Signed-shift correctness audit (`asr()` for all signed shifts; `aabb_center_*` INT64_MAX wrap + `bsp_point_seg_dist` symmetry fixes) |
| v1.1.1 | `[lib]` manifest section + `dist/bsp.cyr` single-file bundle (849 lines); cyrius-doom 0.26.0 is first downstream consumer |
| v1.1.2 | Cyrius 5.5.0 → 5.5.2 (enum-constant fold; −1,448 B standalone) |
| v1.1.3 | Cyrius 5.5.2 → 6.0.1; manifest modernized (single `cyrius.cyml`, `${file:VERSION}`); +18,144 B growth-tax; bundle byte-identical to 1.1.2 |
| v1.1.4 / v1.1.5 | Cyrius pin lifts 6.0.1 → 6.2.11 → 6.3.5 (pure pin moves; growth-tax 94,640 → 98,560 B) |
| v1.2.0 | Deep audit / optimization release (pin held at 6.3.5): `bsp_nearest_seg` sentinel fix; instruction-count hoists; first `query.cyr`/`frustum.cyr` tests (79→94); 500K-iter differential proof; −352 B |
| v1.2.1 | `asr()` corrected to FLOOR rather than round-toward-zero (cyrius-doom RC-F2) — negative world coordinates mis-wrapped flats by one texel |
| v1.2.2 | Cyrius 6.3.5 → 6.5.19 (~130 patches). Fuzz harnesses renamed `*.cyr` → `*.fcyr` so `cyrius fuzz` could discover them at all; `atomic`/`result` declared in `[deps] stdlib`. +19,936 B growth-tax, the project's largest |
| v1.2.3 | Cyrius 6.5.20 (byte-identical build); `asr()` became a one-line alias for native `>>>`; **full audit** — `aabb_init` ±2^31 sentinels, `bsp_find_subsector` hang/OOB/negative-child, `blockmap_cell_seg` SIGSEGV, blockmap silent drops, negative-count queries; fuzz PRNG correlation + `fuzz_aabb` blind spot; docs 33 undocumented → 0; 103 → 136 assertions |
| v1.2.4 | **`fx_div` rewritten** — was wrong by >1 % on 95 % of divisions with a dividend past 32767 (worst case 241×), with a sign-inverting sentinel; plus blockmap live-count/dimension guards and the above-16-bit child hole. 136 → 168 assertions |

## Notes on the abandoned 1.2.x language-adoption plan

An earlier version of this roadmap slotted v1.2.1–v1.2.3 as a language-adoption
sweep: `: i64` return annotations, `Result<T, E>` variants for fallible queries,
and a `test_each` table-driven test refactor. **None of that shipped.** Those
patch levels went to correctness instead — the RC-F2 `asr` fix, a two-band
toolchain catch-up, and the audit. The ideas are still open; they are re-slotted
below rather than left describing releases that contain something else.

## v1.3.x — Contract clarity (unscheduled)

The audit surfaced a pile of sentinel-encoded failure modes that a real result
type would make unmissable. This is now motivated by evidence rather than by
mirroring another project's sweep.

| # | Item | Detail |
|---|------|--------|
| 1 | `Result`-shaped variants for the fallible queries | `bsp_find_subsector`, `bsp_nearest_seg`, `blockmap_query_point` all encode failure as a sentinel (`0`, `-1`) that is indistinguishable from a valid answer. Additive `_r` variants only — the `i64` forms stay, per SemVer |
| 2 | `fx_div` saturation is unsignalled | `±FX_MAX` means both "divide by zero" and "quotient out of range" and "the quotient really is FX_MAX". A caller cannot tell them apart |
| 3 | `: i64` return annotations on the public surface | Parse-only, ABI-identical; verify byte-identical codegen |
| 4 | Prefix consistency | `aabb_*`, `frustum_*`, `blockmap_*`, `fx_*` and `asr` lack the `bsp_` prefix. Renaming is a **major** bump — not worth it alone, but worth bundling if a 2.0 ever happens |

## v1.4.x — Performance recovery (gated on Cyrius O3)

`CYRIUS_DCE=1` currently NOPs ~82 KB of unreachable code in place, so the file
size is unchanged. Cyrius O3 (IR-driven real DCE + const-prop + dead-store
elimination) would genuinely shrink it.

| # | Item | Status | Detail |
|---|------|--------|--------|
| 1 | Wait for **Cyrius O3** real DCE | Upstream | ~82 KB of the 122,280 B standalone is unreachable prelude |
| 2 | Re-bench under O2/O3/O4 phases | Pending | bench-history per upstream phase landing |
| 3 | `bsp_count_in_aabb` reloads all four box fields per entity | Open | Found in the 1.2.3 audit, not yet actioned — hoist the bounds out of the loop |

## Future

| Item | Detail |
|------|--------|
| `bsp_bbox_visible` | Currently a placeholder that always returns 1. Either implement it against the frustum or remove it in a major bump — a permanently-true culling test is a trap |
| Blockmap cell capacity | `BM_CELL_MAX_SEGS` = 16 is a hard cap; dense geometry hits it with valid input. 1.2.4 made the drops *reportable*, not absent. Chained or variable-width cells would remove the ceiling |
| 3D BSP support | Currently 2D only (DOOM-style). 3D adds Z-plane partition for true 3D queries |
| Quake-style BSP loader | Read .bsp format directly (BSP1/BSP2/BSP3) — would let bsp consume id-format static maps |
| Dynamic BSP rebuild | Currently static. Incremental rebuild for destructible-geometry consumers (kiran, phylax) |
