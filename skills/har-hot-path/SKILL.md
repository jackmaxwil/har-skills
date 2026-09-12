---
name: har-hot-path
description: Rust runtime performance — measure before changing, profiler and benchmark commands, black_box, allocation and clone rules, arenas, buffer reuse, bounds-check elision, type sizes, hashing, inline and cold, release-profile and PGO tuning. Load before optimizing Rust or writing a frame, render, or parse loop.
---

# Hot paths

## Measure first

1. Reproduce on a realistic workload. Microbenchmarks supplement a decision; they never make it.
2. Never profile or benchmark a debug build. 10-100x differences are routine and they are not your code.
3. Wall time is what users feel but swings on memory layout alone. For A/B, instruction and cycle counts have far lower variance.
4. A mediocre measurement beats none. Do not build the perfect harness before starting.

A change with no before/after number is not an optimization, it is a rewrite.

## Build for profiling

```toml
[profile.profiling]
inherits = "release"
debug = "line-tables-only"
strip = false
```

Symbols for the profiler, no size explosion, and the shipped `release` profile stays stripped. Add `RUSTFLAGS="-C force-frame-pointers=yes"` when stacks are truncated, and `-C symbol-mangling-version=v0` when names are unreadable. On Linux, `cargo flamegraph` needs `--no-rosegment` when the binary was linked with `lld` (the default since 1.90) or `mold`, or the stacks are wrong.

## Tools

| Tool | Command | Answers | Platform |
| --- | --- | --- | --- |
| samply | `cargo install samply && samply record ./target/profiling/app` | where wall time goes; opens Firefox Profiler | all; on Linux it samples **on-CPU only**, on macOS both |
| perf | `perf record -g ./target/profiling/app && perf report` | sampling plus real PMU counters — the only honest source for cache and branch events | Linux |
| flamegraph | `cargo flamegraph --profile profiling --bin app -- args` | one flame graph | needs `perf` (Linux) or `dtrace` (macOS, sudo) |
| Instruments | `cargo instruments -t 'Time Profiler' --profile profiling` | mac-native CPU, allocations, GPU and frame timing | macOS |
| heaptrack | `heaptrack ./target/profiling/app` | allocation traces over time | Linux |
| DHAT | `valgrind --tool=dhat ./target/profiling/app` | which call sites allocate, and hot `memcpy` | Linux/macOS |
| cachegrind | `valgrind --tool=cachegrind ./app && cg_annotate cachegrind.out.*` | exact, reproducible instruction counts, flat — best for a small regression diff | Linux/macOS |
| callgrind | `valgrind --tool=callgrind ./app && callgrind_annotate callgrind.out.*` | the same counts plus caller/callee attribution | Linux/macOS |
| iai-callgrind | `cargo bench` with the `iai-callgrind` dev-dependency | deterministic counter benchmarks, low enough variance for CI | Linux/macOS |
| hyperfine | `hyperfine --warmup 3 './old' './new'` | end-to-end wall time A/B with statistics | all |
| criterion / divan | `cargo bench` with that dev-dependency | per-function regression tracking; divan is lighter to write | all |
| cargo-bloat | `cargo bloat --release --crates` | what is actually in the binary | all |
| cargo-llvm-lines | `cargo llvm-lines \| head -20` | monomorphization bloat, the icache offender | all |

Cachegrind's and Callgrind's cache and branch simulation is a rough model, not the machine — read it as diagnostic, and use `perf stat` against the real PMU when cache or branch behaviour is the question.

**Nightly compiler diagnostics**, pinned like any other tool (`har-supply`), and not runtime measurements:

```
RUSTFLAGS=-Zprint-type-sizes cargo +nightly build --release   # layout, padding, fat enum variants
```

Profile first, benchmark second: the profiler says *where*, the benchmark says *whether it moved*.

## Benchmarks lie without `black_box`

Criterion, divan, a release build and instruction counts do nothing to stop LLVM from constant-folding a literal input, hoisting the work out of the loop, or deleting a pure result nobody reads. `std::hint::black_box` is the documented best-effort barrier, and it applies to **its own output**, so the input and the result are two separate calls:

```rust
b.iter(|| black_box(parse(black_box(&input))));
```

It is a benchmarking tool only. Never cite it for correctness, constant-time, or any security property.

## Allocation

- Pre-size anything whose final length you can estimate. `Vec::with_capacity(n)` turns four allocations into one for a twenty-element push loop.
- Declare the buffer outside the loop and `clear()` it per iteration. `clear` keeps capacity; a fresh `Vec::new()` does not.
- `read_line(&mut line)` plus `line.clear()` instead of `.lines()`, which allocates a `String` per line.
- `vec![0; n]` beats `resize`/`extend` — the OS hands back zero pages.
- `dst.extend_from_slice(src)` to append a slice, not a hand-written `push` loop: it states the bulk-copy intent and `std` specializes it to `copy_from_slice` for `Copy` elements. Its bound is `T: Clone`, so `dst.extend(src.iter().copied())` where that suits better. `reserve` first when a larger composition makes the final length known.
- `swap_remove` is O(1) where `remove` is O(n). `retain` for bulk removal.
- `ok_or_else` / `unwrap_or_else` / `map_or_else` whenever the default costs anything; `ok_or(build())` always evaluates.

**Arenas**, when a profile shows many small heterogeneous allocations that all die together — a parser, an AST, a query plan, a per-request graph. A bump allocator (`bumpalo`) turns each allocation into a pointer bump and the whole phase into one reset. Two conditions: no reference may escape the phase, and nothing in the arena may need a prompt `Drop`, because a bump reset does not run destructors. Use a drop-aware wrapper or explicit cleanup where it does.

## Clones

Cloning to silence the borrow checker is the defect. A deliberate clone in a cold path is fine; the pattern to kill is one the compiler talked you into.

| Situation | Instead of | Use |
| --- | --- | --- |
| Overwrite a vec from another | `dst = src.clone()` | `dst.clone_from(&src)` — reuses `dst`'s allocation |
| Immutable payload, **uniquely owned**, fixed size | `String` / `Vec<T>` | `Box<str>` / `Box<[T]>` — 2 words, no refcount |
| Immutable payload **genuinely shared** across threads | `String` / `Vec<T>` clone | `Arc<str>` / `Arc<[T]>` — clone is a refcount bump |
| Immutable payload shared on one thread | `Arc` | `Rc` — no atomics |
| Mutate a shared value if unshared | clone-then-mutate | `Rc::make_mut` / `Arc::make_mut` — copies only when refcount > 1 |
| Move a value out of a field | clone the field | `mem::take` / `mem::replace` |
| Two owners of mutable state | clone both sides | restructure, or `Rc`/`Arc` once |

`Arc` is not the default shared-payload type — it costs an allocation and atomic refcount traffic on every clone and drop, and the Performance Book calls `Rc`/`Arc` expensive when sharing is rare. Reach for `Box<str>`/`Box<[T]>` when ownership is unique, borrow `&Arc<T>` rather than cloning it, and measure the contention when a hot `Arc` is cloned across threads. `Vec::into_boxed_slice` is a finalization step at the ownership boundary; it reallocates when `len < capacity`, so it is not free.

## Buffers

| Situation | Do |
| --- | --- |
| Length known or estimable | `Vec::with_capacity(n)` |
| Rebuilt every iteration or frame | hoist out, `clear()` per pass |
| Allocation profiling shows a small bounded common length | `SmallVec<[T; N]>` — **measured**, see below |
| Hard maximum known, no heap wanted | `ArrayVec<[T; N]>` |
| Finished growing, stored many times | `into_boxed_slice()` — 2 words instead of 3, and it states the invariant |
| Result only ever iterated | return `impl Iterator<Item = T>`, skip the `Vec` |

`SmallVec` and `ArrayVec` reduce allocation rate and are not free defaults: inline storage enlarges every enclosing object and adds a branch on each access, so inside a struct that appears a million times they can cost more cache than they save allocator. Adopt them when a profile shows the bounded common length **and** you have measured the enclosing object's size.

## Strings and slices

`as_` is free, `to_` allocates, `into_` consumes. The prefix is a cost contract; `har-api` owns the naming rule, this is what it buys you at runtime.

| Situation | Take or return | Why |
| --- | --- | --- |
| Parameter, read-only | `&str` | zero cost, no monomorphized copies |
| Parameter, callee stores it | `String` | one clone, at the caller's choice, visible |
| Return, borrowed from an input | `&str` | free |
| Return, transform usually a no-op | `Cow<'a, str>` | zero allocations in the common case |
| Shared, immutable, crosses threads | `Arc<str>` | refcount clone, smaller than `String` |
| Encoding irrelevant | `&[u8]` | every `&str` construction pays UTF-8 validation |

Reach for `&[u8]` rather than an unchecked conversion; `har-unsafe` owns any unsafe fast path. (`har-io` owns correctness when the bytes are user-visible text.)

## Iterators

- One terminal `collect()`. Intermediate ones allocate for nothing.
- `try_fold` / `try_for_each` when items are `Result`/`Option` or a condition stops the pipeline and the output is an accumulator — it short-circuits without materializing a temporary. Keep `collect` when you genuinely need the collection.
- `filter_map` over `filter().map()`. `extend` into an existing collection over collect-then-append.
- `iter().copied()` for small `Copy` types, `chunks_exact` over `chunks` — both codegen better.
- `for x in &xs`, never `for i in 0..xs.len()`. Indexed access keeps the bounds check the iterator elides.
- Implement `size_hint` (or `ExactSizeIterator`) on custom iterators so `collect` and `extend` pre-allocate.

**When a hot loop must index** — zipped arrays, sliding windows — the safe ways to get the bounds check elided, in order: iterate or `zip` if the shape allows; otherwise narrow the slice **once** before the loop (`let xs = &xs[..n];`) or assert the range up front, so LLVM proves the rest. Read the generated code before believing it worked. `get_unchecked` is the last resort, and it converts a compiler-checked fact into an unsafe invariant you now own (`har-unsafe`).

## Type sizes

`-Zprint-type-sizes` is the only honest source. The compiler already reorders fields, so hand-ordering is noise unless `#[repr(C)]`.

Box the rare fat variant — it shrinks every value of the enum, common variants included:

```rust
enum Node {
    Leaf(u32),
    Branch(Box<(Node, Node, Metadata)>),
}
```

Freeze a hot type so a later field addition fails the build:

```rust
static_assertions::assert_eq_size!(FrameSlot, [u8; 64]);
```

## Hashing

`std::collections::HashMap` uses a randomly seeded SipHash 1-3 — strong against HashDoS, slow on integer and short keys — and std explicitly reserves the right to change that. The alternatives are not interchangeable, and none of them is HashDoS-resistant:

| Map | Hasher | Use |
| --- | --- | --- |
| `std::collections::HashMap` | SipHash 1-3, randomly seeded | anything keyed by attacker-supplied data |
| `hashbrown::HashMap` | **foldhash** by default | its own docs say the default is not HashDoS-resistant |
| `rustc_hash::FxHashMap` | FxHasher — rustc-hash 2.x kept the name and replaced the algorithm | internal maps keyed by ids |
| `ahash::HashMap` | AHash, opt-in keyed | when its keying is actually configured |

Benchmark against your real key distribution. Published rankings — including rustc's own historical numbers for Fx versus AHash — are measurements of one workload on one version, and do not transfer. Keep the default hasher on any map keyed by data a peer controls; `har-threat` owns that judgement.

## IO

`har-layout` owns stdout and stderr discipline, buffering, and explicit `flush`. The runtime reason is the same rule from the other side: `println!` takes the stdout lock on **every** call, and an unbuffered `File` is a syscall per read or write.

## Codegen cost

A generic function is compiled once per instantiation: build time, binary size, and instruction cache all pay. `cargo llvm-lines | head -20` names the offenders, `cargo bloat --release --crates` says what shipped. The fix — a thin generic shell delegating to one non-generic `inner` — is `har-layout`'s, under compile-time budget; here it matters because the duplicated bodies are what evict the icache.

## `#[inline]` and `#[cold]`

Default: no attribute. The compiler sees the body within a crate, and `lto` extends that across crates.

- Small non-generic public functions callers should inline across a crate boundary — `Deref`, `AsRef`, accessors: yes.
- Sprinkling it across a module costs build time and buys nothing.
- There is **no language rule against it on private or generic functions**, and the Performance Book explicitly recommends splitting a mixed hot/cold function into an `#[inline(always)]` hot half and an `#[inline(never)]` wrapper. So after a profile points at a call, `#[inline]` or `#[inline(always)]` anywhere is permitted — then re-measure, and check code size, because inlining loses to icache pressure as often as it wins.

The inverse is underused. When a profile shows a hot success path polluted by infrequent work — error formatting, a diagnostic, a recovery routine — extract it and mark the outlined function `#[cold]`, optionally with `#[inline(never)]`:

```rust
#[cold]
#[inline(never)]
fn report(span: Span, found: u8) -> Error { Error::Unexpected { span, found } }
```

Not a blanket annotation for every `Err` — a simple check is cheaper left inline.

## Release-profile tuning

`har-supply` owns the profile as policy; these are the runtime trade-offs. **None of them has a promised number** — try it, measure it on the deployment target, and record the artifact and CPU compatibility you just committed to.

| Knob | Buys | Costs, and what it commits you to |
| --- | --- | --- |
| `lto = "thin"` | cross-crate inlining | modest link time |
| `lto = "fat"` | more of the same | much slower builds |
| `codegen-units = 1` | more inlining within a crate | loses build parallelism |
| `panic = "abort"` | smaller binary, no unwind tables | no `catch_unwind`, no unwind-based cleanup, no FFI unwind boundary |
| `-C target-cpu=native` | AVX and friends | binary runs only on that microarchitecture — controlled fleets only; otherwise a conservative baseline plus checked runtime dispatch |
| `mimalloc` / `jemalloc` | often large on allocation-heavy programs | one dependency, one line |
| PGO | real, when the profile is representative | an instrument → run → merge → rebuild cycle, and a profile that must match production input |
| BOLT, after PGO | code layout and icache, on a final native binary | external LLVM BOLT tooling, Linux and deploy-controlled binaries only; `cargo-pgo` automates it and calls BOLT support experimental. Keep a non-BOLT fallback |

`overflow-checks = true` is real time in hot arithmetic, and `har-supply` sets it deliberately. Do not turn it off globally to win a benchmark: pick the explicit operation for the hot expression instead — `har` covers `checked_*` / `wrapping_*` / `saturating_*` — and keep the check everywhere else.

## SIMD

Autovectorization first: shape the loop so LLVM can do it — `chunks_exact`, no early exits, no aliasing doubt — and then **read the assembly** or a counter to confirm it happened. That is most of the available win, with no unsafe and no dispatch.

Explicit SIMD on stable Rust means `std::arch` intrinsics with `#[target_feature]`, behind runtime detection, with a scalar fallback:

```rust
#[cfg(target_arch = "x86_64")]
fn sum(xs: &[f32]) -> f32 {
    if is_x86_feature_detected!("avx2") {
        // SAFETY: avx2 confirmed by the check above.
        unsafe { sum_avx2(xs) }
    } else { sum_scalar(xs) }
}
```

Calling a `#[target_feature]` function on a CPU without the feature is UB (`har-unsafe`, UB #7), so the detection is the whole safety argument. **`std::simd` / `core::simd` is still nightly**, behind `feature(portable_simd)` — allowed only under a pinned, reviewed nightly policy, never as a default for a stable build.

## The 120fps checklist

8.3 ms per frame. Steady-state paint:

1. Zero allocations. Every per-frame buffer is a field on the renderer, `clear()`ed at frame start, never recreated. Growth amortizes to zero within a few frames.
2. No `format!`. Cache formatted text in the model and invalidate on change; if text must be built per frame, `write!` into a reused `String`.
3. No `collect()`. Iterate. If a collection is unavoidable, it is a hoisted one.
4. Every clone is a refcount bump. `Arc<str>` / `Arc<[T]>` for shared frame data. A `String` or `Vec` clone per element per frame is the classic killer.
5. `FxHashMap` for id-keyed lookups, kept allocated across frames.
6. Hot structs stay small, and `assert_eq_size!` says so — layout regressions are otherwise invisible.
7. Anything unbounded — parse, IO, database — runs on a worker. The paint path reads a prepared snapshot.
8. No per-frame log line or span. Sample, or instrument the interaction instead: 120 formatted lines a second is cost and noise.

## Anti-patterns

| Anti-pattern | Fix |
| --- | --- |
| A benchmark with no `black_box` | `black_box(f(black_box(input)))`, and consume the result |
| `String::new()` / `Vec::new()` / `format!` inside a loop | hoist and `clear()`, or `write!` into a reused buffer |
| `.lines()` over a large file | `read_line` into one reused `String` |
| Hand-written `push` loop to append a slice | `extend_from_slice` |
| Chained `collect()`s | one terminal `collect`, or `extend`, or return `impl Iterator` |
| `collect::<Result<Vec<_>,_>>()` only to fold it | `try_fold` / `try_for_each` |
| `for i in 0..v.len() { v[i] }` | `for x in &v`; pre-slice or assert once when indexing is unavoidable |
| `ok_or(expensive())`, `unwrap_or(build())` | the `_else` variants |
| `v = other.clone()` | `v.clone_from(&other)` |
| `Arc<[T]>` for a payload that is never actually shared | `Box<[T]>` |
| `.clone()` added because the borrow checker complained | find the real conflict; `Arc`, `mem::take`, or restructure |
| Large generic body instantiated everywhere | thin generic wrapper delegating to a non-generic `inner` |
| `#[inline]` sprinkled across a module | delete; keep it where a profile put it, or turn on `lto` |
| Hot success path carrying rare error formatting inline | outline it, `#[cold]` |
| Default hasher on an internal id-keyed map | `FxHashMap`; never the reverse on a peer-keyed map |
| `target-cpu=native` in a distributed binary | baseline plus runtime feature detection |
| Optimizing against a debug build or a microbenchmark alone | realistic workload, release build, a profiler |
