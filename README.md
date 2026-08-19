# har-skills

Nine skills that give AI coding agents high-assurance Rust guidance: what to write, what to refuse to write, and how to prove it.

Each skill is a single dense `SKILL.md` — decision tables, non-panicking alternatives, exact tool invocations. No topic is owned by two skills; they cross-reference instead of repeating each other.

## Install

```bash
npx skills add jackmaxwil/har-skills
```

Install one skill, or a few:

```bash
npx skills add jackmaxwil/har-skills --skill har-unsafe
```

Or use one without installing it:

```bash
npx skills use jackmaxwil/har-skills@har-verify
```

Installs into Claude Code, Cursor, Codex, and every other agent the [skills CLI](https://github.com/vercel-labs/skills) supports.

## The skills

| Skill | Load when |
| --- | --- |
| `har` | Writing or reviewing Rust that must not crash — panic budget, `Result` discipline, newtypes, typestate, integer and overflow rules. |
| `har-api` | Designing a type, trait, or public interface — generics vs `dyn Trait`, standard traits, `From`/`Into`, RAII, the borrow-checker ladder, iterator idioms, `thiserror` vs `anyhow`, semver. |
| `har-concurrent` | Threads share data, or anything takes an `Ordering` argument — `Send`/`Sync`, mutex and condvar protocols, memory ordering, false sharing, async cancellation. |
| `har-unsafe` | Auditing `unsafe` you did not write, or debugging a lifetime, variance, or `Send` error — the UB list, validity vs safety invariants, drop check, panic and leak safety, FFI. |
| `har-verify` | Deciding how to test or prove a change — the ladder from `clippy` through property tests, fuzzing, differential harnesses, `miri`, `loom`, and `kani` proofs. |
| `har-supply` | Before adding a dependency or auditing the graph — `cargo-deny`, `cargo-audit`, transitive `unsafe`, reproducible builds. |
| `har-threat` | Designing or reviewing a boundary that takes input you do not control — trust boundaries, STRIDE, resource bounds, secrets, AEAD and nonce discipline. |
| `har-hot-path` | Before optimizing, or when writing a frame, render, or parse loop — measure first, profiler commands, allocation and clone rules, type sizes, release-profile tuning. |
| `har-layout` | Adding a module or crate, wiring observability, or when builds got slow — workspace shape, naming, `tracing`, CLI output, serde and config, compile-time budget. |

`har`, `har-verify`, and `har-supply` are the base three. The rest load on demand.

## Sources

These skills are original prose distilled from published work by others. The thinking is theirs; any errors in compression are mine. Read the sources — they are all free.

- **[High Assurance Rust](https://highassurance.rs)** — Tiemoko Ballo. The book this family is named for, and the backbone of `har`, `har-supply`, `har-verify`, and `har-threat`: the verification ladder, dependency trust, threat modeling, and the discipline of deriving assurance rather than claiming it.
- **[The Rustonomicon](https://doc.rust-lang.org/nomicon/)** and the [Rust Reference](https://doc.rust-lang.org/reference/behavior-considered-undefined.html) — the UB list, variance, drop check, and aliasing rules behind `har-unsafe`.
- **[Effective Rust](https://effective-rust.com)** — David Drysdale, with the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — trait design, dispatch, ownership ergonomics, and semver behind `har-api`.
- **[Rust Atomics and Locks](https://marabos.nl/atomics/)** — Mara Bos. Memory ordering, `Send`/`Sync`, the cache-coherence cost model, and the primitive-building material behind `har-concurrent`.
- **[The Rust Performance Book](https://nnethercote.github.io/perf-book/)** — Nicholas Nethercote — plus [mre/idiomatic-rust](https://github.com/mre/idiomatic-rust), [corrode.dev](https://corrode.dev), [matklad's writing](https://matklad.github.io), and the `tokio` and `tracing` docs, behind `har-hot-path` and `har-layout`.
- **[Ferrocene](https://ferrocene.dev)** — the qualified-toolchain and build-discipline rules that `har-supply` borrows in their lightweight form: pin the toolchain, clean before shipping, never lower a warning to get a build green.

## Contributing

A change should make an agent write different code. Rules that only describe Rust do not earn their lines. Keep the ownership boundaries: if a topic belongs to another skill, reference it in a clause rather than restating it.
