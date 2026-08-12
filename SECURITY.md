# Security Policy

## Scope

BSP is a pure geometry library: no I/O, no syscalls, no allocation, no globals.
The attack surface is entirely **malformed input data** — node arrays, segment
arrays and blockmaps that a consumer built from a map file it did not author.

Cyrius performs **no runtime bounds checking**, and BSP does not allocate, so
every buffer belongs to the caller and its size is the caller's contract. What
BSP guarantees is that it will not turn well-formed *memory* into a hang, an
out-of-bounds read, or a silently wrong index because of hostile *values*:

- `bsp_find_subsector` bounds its walk by `node_count` and rejects cyclic,
  negative, out-of-range and above-16-bit child references. Before 1.2.3 a
  cyclic tree hung the process indefinitely.
- `blockmap_get_cell` returns 0 for out-of-range coordinates, and
  `blockmap_cell_count` / `blockmap_cell_seg` both tolerate that 0.
  `blockmap_cell_seg` also range-checks against the cell's live count, so it
  cannot hand back an uninitialised slot.
- `blockmap_init` rejects unusable grid dimensions rather than returning a
  buffer it never zeroed.
- Divisions are guarded, and segment intersection is division-free by design
  so degenerate input cannot raise SIGFPE.

**Still the caller's responsibility:** buffer sizes. `nodes` must hold
`node_count` records, a blockmap buffer must hold
`BM_DATA + cols * rows * BM_CELL_STRIDE` bytes, and the documented coordinate
and count limits in the README are contracts, not assertions.

## Reporting

Report vulnerabilities to robert.maccracken@gmail.com. Please do not open a
public issue for anything exploitable in a consumer.
