# Contributing to BSP

## Development

1. Install Cyrius compiler (cc5, 5.5.0+)
2. `cyrius build src/lib.cyr build/bsp` to compile
3. Test programs in `programs/`

## Constraints

- All math must be 16.16 fixed-point — no floating point
- Zero dependencies — pure geometry
- No rendering, no I/O in the library itself

## License

GPL-3.0-only.
