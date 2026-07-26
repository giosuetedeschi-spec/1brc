# 1BRC: build fix, run results, and design notes

This documents the work done to get this repo's solution running on Linux, the
measurements taken, and why the code is built the way it is.

## 1. The bug that blocked the build

`src/main.rs` imported `std::os::windows::fs::{FileExt, MetadataExt}` — a
Windows-only module. On Linux this fails to compile:

```
error[E0433]: failed to resolve: could not find `windows` in `os`
error[E0599]: no method named `read_exact_at` found for `&File`
error[E0599]: no method named `size` found for struct `Metadata`
```

`FileExt::read_exact_at` (positional pread) and `MetadataExt::size` both exist
on Unix too, just under `std::os::unix::fs` instead. One-line fix:

```diff
- os::windows::fs::{FileExt, MetadataExt},
+ os::unix::fs::{FileExt, MetadataExt},
```

No other changes were needed — the rest of the code is platform-agnostic.

## 2. Running it end to end

Environment: 4 vCPUs, 30GB free disk.

```
$ cargo build --release
    Finished `release` profile [optimized] target(s) in 2.49s

$ time cargo run --release --features generator --bin generate 1000000000
real  5m39.309s   (data generation — not part of the challenge timing)

$ ls -lh measurements.txt
13G measurements.txt   (1,000,000,000 lines)

$ time target/release/1brc > out.txt
real   0m27.776s
user   1m41.742s
sys    0m5.561s
```

413 unique city names, output sorted alphabetically as `City: min/mean/max`,
e.g. `Abha: -46.6/18/80.4`.

With the `fxhash` feature enabled:

```
$ cargo build --release --features fxhash
$ time target/release/1brc > out_fx.txt
real   0m24.069s
user   1m29.445s
sys    0m5.139s
```

`diff out.txt out_fx.txt` — **identical**, confirming the faster hasher
doesn't change results. fxhash was ~13% faster wall-clock on this run.

Both are well below the README's reference `9.7s`, which is expected — that
number was presumably taken on a machine with more than 4 cores; this
implementation is CPU-bound and scales with `available_parallelism()`.

## 3. Why it's built this way

The git history (`naive` → `distribute work` → `bigger/reused buffers` →
`better types` → `fxhash`) is the standard 1BRC optimization arc. Each step
targets a specific bottleneck:

- **Chunked positional reads, not a line reader.** `get_aligned_buffer` reads
  fixed 16MB (`CHUNK_SIZE`) slices via `read_exact_at` — no shared file
  cursor, so threads can read disjoint regions concurrently without
  contending on file position. A `CHUNK_EXCESS` (64-byte) backward-lookback
  guarantees the buffer starts on a record boundary and ends on one, so no
  line is split or double-counted across chunk edges.

- **Atomic work-stealing queue instead of a fixed split.** `distribute_work`
  spawns one thread per core; each pulls the next chunk offset from a shared
  `AtomicU64::fetch_add`. This avoids the straggler problem you'd get from
  pre-dividing the file into N equal ranges (later chunks tend to have
  different cache/IO behavior than early ones).

- **Zero-copy parsing.** Inside a chunk, city names are kept as `&[u8]`
  slices into the read buffer (`BorrowedMap<'a>`), never allocated as
  `String` until absolutely necessary. Since there are ~1B rows but only
  ~400 unique cities, avoiding a per-row allocation matters far more than
  the eventual one-time `String` conversion cost.

- **Two-level aggregation to minimize lock contention.** Each thread
  accumulates a full chunk into its own local `HashMap`, then takes the
  shared `Mutex<Map>` lock **once per 16MB chunk**, not once per row. Lock
  acquisition is amortized over ~100K+ rows per lock, so contention is
  negligible even with 4 threads racing on the same mutex.

- **Running aggregates instead of stored samples.** `Records` keeps only
  `count/min/max/sum` as `f32` and derives `mean` on output — O(1) memory
  per city regardless of row count.

- **Swappable hasher (`fxhash` feature).** Rust's default `HashMap` hasher
  (`RandomState`/SipHash) is DoS-resistant but slower than necessary here,
  since city names aren't attacker-controlled input. `FxHash` trades away
  that resistance for raw speed on a hot path called a billion times.

## 4. Known quirks (not fixed, just noting)

- **Mean formatting drops trailing `.0`.** `Records::mean` rounds to one
  decimal, but `f32`'s `Display` impl prints `18` instead of `18.0` for
  whole numbers (e.g. `Abidjan: 2.2/26/47.2` — note `26` not `26.0`). The
  official 1BRC spec expects a consistent one-decimal format. Not touched
  since it wasn't the ask, but a one-line fix would be formatting with
  `{:.1}` instead of relying on `Display`.
- The generator/challenge both assume single-decimal temperatures in
  `-99.9..=99.9`; nothing enforces that beyond the generator's own output.

## 5. Where the remaining time likely goes (not implemented)

For further speedup, the usual next moves on the 1BRC leaderboard are:
`mmap` instead of `read_exact_at` (skip the buffer copy entirely), a custom
open-addressing hash table over raw byte hashes (skip `std::collections::HashMap`
node overhead), and branchless fixed-point parsing of the temperature (since
the format is always one digit before/after an optional `-` and a fixed
one-decimal suffix, it can be parsed without calling into `f32::parse`).
