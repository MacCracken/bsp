# Contributing to BSP

## Development

1. Install the Cyrius toolchain at the version pinned in `cyrius.cyml`
   (currently **6.5.20**). The pin — not whichever `cycc` is on your PATH —
   decides which stdlib snapshot `cyrius build` vendors into `lib/`.
2. `cyrius build src/lib.cyr build/bsp` to compile.
3. `tests/bsp.tcyr` is the test suite. `programs/test_bsp.cyr` is a
   hand-run smoke test that always exits 0 and gates nothing.

## Gates

Every change must pass all of these:

```sh
cyrius build src/lib.cyr build/bsp     # watch the unreachable-fn byte count
cyrius test tests/bsp.tcyr             # must report "0 failed"
cyrius bench benches/bsp.bcyr          # 13 ops
cyrius fuzz                            # 3 harnesses, 25K standard gate
cyrius audit                           # fmt + lint + docs + tests + bench
cyrius distlib                         # regenerate dist/bsp.cyr after ANY src change
```

## Constraints

- All math must be 16.16 fixed-point — no floating point
- Zero dependencies — `src/` includes nothing, not even stdlib
- Zero globals, zero heap allocation — callers own all memory
- No rendering, no I/O in the library itself
- Never use bare `>>` on a signed value: Cyrius `>>` is LOGICAL. Use
  `asr(v, n)`, the `fx_*` helpers, or the native `>>>` — and parenthesise
  `>>>` over a sum, since it binds tighter than `+`

## Testing expectations

- **Property tests beat value tests.** "every point added to a box is contained
  in it" catches what "width >= 0" cannot.
- **Mutation-test every new gate.** A test or fuzz invariant you have not seen
  *fail* proves nothing. Run it against the previous release: it must be RED
  there and GREEN on your branch. Two gates in this repo passed on code they
  were written to catch until this was enforced.
- **Behaviour-preserving changes need a differential**, not an argument: inline
  the old implementation beside the new one, run a few hundred thousand random
  inputs through both, and assert equality. Cover negative, sub-precision and
  ±2e9 coordinates.
- **Check the sampler, not just the invariant.** A power-of-two LCG's low bits
  have short periods, so `state % n` correlates with the call index — mix the
  high bits down before taking a modulus.

## Documentation

`cyrius audit`'s docs stage must report `ok: docs complete`. A doc comment
counts only when it is **directly adjacent** to the `fn` — a single blank line
between a comment and its function makes it invisible to the checker.

## License

GPL-3.0-only.
