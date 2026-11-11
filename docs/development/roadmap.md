# BSP Development Roadmap

> **v1.3.2** — 183,088 B standalone, 19,248 B release (`CYRIUS_DCE=1`), on
> cycc 6.7.18. 200/200 assertions across 21 groups, 14/14 benches, 3 fuzz
> harnesses (25K standard gate, clean at 3M stress), `cyrius audit` green,
> 0 undocumented public functions. 1,397-line `dist/bsp.cyr` bundle.
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
| v1.2.5 | Cyrius 6.5.20 → 6.6.2 (pure pin move). Shipped with `cyrius audit` red: the 6.6.x audit lints and doc-checks the harnesses too, and flagged 19 undocumented harness functions and one over-length test line |
| v1.2.6 | Cyrius 6.6.2 → 6.6.20; stdlib re-synced; `cyrius.lock` added; `dist/bsp.deps` retired by 6.6.18 distlib; audit green again and now gated in CI. 3M-iteration cross-toolchain differential, zero divergence. +57,184 B standalone growth-tax (+3,936 B release), now the project's largest |
| v1.3.0 | Cyrius 6.6.20 → 6.7.9, the move into the 6.7.x line. No API or behaviour change (3M-iteration differential, zero divergence); sizes unchanged to the byte. CI's missing-fuzz-harness hint repaired for 6.7.9's new exit/message |
| v1.3.1 | Cyrius 6.7.9 → 6.7.18; no `src/` change, zero-divergence differential, −496 B. 6.7.11's write-length lint caught a dropped newline in the bench header — the 1.2.6 CI audit gate's first real catch |
| v1.3.2 | Review / hardening / optimization sweep: 13 defects (fx_div SIGFPE + overflow past 2^55, null check_sight flags, point_seg_dist precision, unit-vector directions in ray_cast and the side test, corner seam in line of sight, blockmap >32768 cells, cell size <= 0, corrupted cell counts, a 1-level "3-node" bench). Side test inlined (find_subsector −55 %, point_on_side −32 %); count_in_aabb −53 %. Bit-identical to 1.3.1 for integer-unit geometry. 168 → 200 assertions |

## Notes on the abandoned 1.2.x language-adoption plan

An earlier version of this roadmap slotted v1.2.1–v1.2.3 as a language-adoption
sweep: `: i64` return annotations, `Result<T, E>` variants for fallible queries,
and a `test_each` table-driven test refactor. **None of that shipped.** Those
patch levels went to correctness instead — the RC-F2 `asr` fix, a two-band
toolchain catch-up, and the audit. The ideas are still open; they are re-slotted
below rather than left describing releases that contain something else.

## Contract clarity (unscheduled; was slotted v1.3.x)

1.3.0 shipped as the Cyrius 6.7.9 move instead. These additive variants are
still minor-bump material whenever they land.

The audit surfaced a pile of sentinel-encoded failure modes that a real result
type would make unmissable. This is now motivated by evidence rather than by
mirroring another project's sweep.

| # | Item | Detail |
|---|------|--------|
| 0 | A validating child-reference helper | `bsp_is_subsector` / `bsp_subsector_idx` decode without validating (0x18005 → subsector 5). Consumer-written walks — cyrius-doom's included — get none of `bsp_find_subsector`'s checks. An additive `bsp_child_ok(ref, node_count)` would give them one call. Documented in `tree.cyr` (1.3.2) |
| 1 | `Result`-shaped variants for the fallible queries | `bsp_find_subsector`, `bsp_nearest_seg`, `blockmap_query_point` all encode failure as a sentinel (`0`, `-1`) that is indistinguishable from a valid answer. Additive `_r` variants only — the `i64` forms stay, per SemVer |
| 2 | `fx_div` saturation is unsignalled | `±FX_MAX` means both "divide by zero" and "quotient out of range" and "the quotient really is FX_MAX". A caller cannot tell them apart |
| 3 | `: i64` return annotations on the public surface | Parse-only, ABI-identical; verify byte-identical codegen |
| 4 | Prefix consistency | `aabb_*`, `frustum_*`, `blockmap_*`, `fx_*` and `asr` lack the `bsp_` prefix. Renaming is a **major** bump — not worth it alone, but worth bundling if a 2.0 ever happens |

## v1.4.x — Performance recovery (gated on Cyrius O3)

`CYRIUS_DCE=1` now *compacts* rather than NOPing in place (true by 6.6.2, and
still at 6.7.18): the release build drops 159,800 B of unreachable code and
ships 19,248 B. The rest of Cyrius O3 (const-prop, dead-store elimination) is
still upstream.

| # | Item | Status | Detail |
|---|------|--------|--------|
| 1 | Wait for **Cyrius O3** real DCE | Partly landed | DCE compaction works (178,992 → 19,248 B at 6.7.18); const-prop / dead-store elimination still upstream |
| 2 | Re-bench under O2/O3/O4 phases | Pending | bench-history per upstream phase landing |
| 3 | `bsp_count_in_aabb` reloads all four box fields per entity | **Done (1.3.2)** | Bounds hoisted, test inlined: 64 entities 373 → 174 ns |

## Future

| Item | Detail |
|------|--------|
| DOOM-exact side test | `bsp_point_side` breaks ties to the FRONT; vanilla `R_PointOnSide` to the BACK, and pre-shifts differently. Fine for rendering; matters only if cyrius-doom ever needs demo-sync fidelity. A separate `bsp_point_side_doom` would be additive (noted in the 1.3.2 review) |
| `bsp_bbox_visible` | Currently a placeholder that always returns 1. Either implement it against the frustum or remove it in a major bump — a permanently-true culling test is a trap |
| Blockmap cell capacity | `BM_CELL_MAX_SEGS` = 16 is a hard cap; dense geometry hits it with valid input. 1.2.4 made the drops *reportable*, not absent. Chained or variable-width cells would remove the ceiling |
| 3D BSP support | Currently 2D only (DOOM-style). 3D adds Z-plane partition for true 3D queries |
| Quake-style BSP loader | Read .bsp format directly (BSP1/BSP2/BSP3) — would let bsp consume id-format static maps |
| Dynamic BSP rebuild | Currently static. Incremental rebuild for destructible-geometry consumers (kiran, phylax) |
