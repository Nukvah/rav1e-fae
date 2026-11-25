# rav1e for fae

rav1e 0.8.1 exactly as published on crates.io (upstream commit 38806721f718e7ae6919e72385bc305bf824ef81,
BSD-2-Clause, see LICENSE and PATENTS), plus one patch for fae, a photo editor that encodes AVIF with it.
The first commit is the unchanged crate, so `git diff` against it shows the whole patch.

## The patch

After the tiles (which run in parallel), rav1e ran its frame-wide loop filter steps on one thread. For a
big still image on many cores that was most of the encode: a 50 MP photo on 24 threads took ≈ 7.5 s at
speeds 3 to 8 alike. These steps now run in parallel; the output is byte-identical (search for "fae patch"):

- `src/cdef.rs` `cdef_filter_tile`: the rows of 64x64 filter blocks (upstream: "should parallelize this").
  Each filter block writes only its own area of the output and reads the deblocked copy, so rows don't
  touch each other's data.
- `src/lrf.rs` `lrf_filter_frame`: the restoration stripes, each with its own stripe-filter buffer.
  A stripe writes only its own rows and reads the CDEF'd / deblocked copies.
- `src/deblock.rs` `sse_optimize`: the planes' tallies for the deblocking level (integer sums).

These are per-frame steps, so video frames get the same speed-up. `Cargo.toml` also quiets two lints
the newer compiler raises on upstream code.

## Moving to a newer rav1e

Commit the new crate unchanged, redo the three changes, and check that encodes give the same bytes as the
unpatched crate (fae's `tests/backend/encoders_check.py` checks the rav1e files).
