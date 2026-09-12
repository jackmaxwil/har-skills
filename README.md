# har-skills

Twelve skills that give AI coding agents high-assurance Rust guidance: what to write, what to refuse to write, and how to prove it.

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
| `har-api` | Designing a type, trait, or public interface — generics vs `dyn Trait`, dyn compatibility, sealed and extension traits, RPITIT and edition-2024 capture, standard traits, `must_use`, `From`/`Into`, RAII, the borrow-checker ladder, iterator idioms, error source chains, MSRV, semver. |
| `har-concurrent` | Threads share data, or anything takes an `Ordering` argument — `Send`/`Sync`, mutex and condvar protocols, memory ordering, atomic pointers and reclamation, false sharing, building primitives. |
| `har-async` | Spawning a task or a child process, wiring a stdio or socket protocol, or a future can be dropped mid-flight — executor shape, task lifecycle and shutdown, cancel safety, pipe deadlocks, framing codecs, timeouts, async tests. |
| `har-unsafe` | Auditing `unsafe` you did not write, or debugging a lifetime, variance, or `Send` error — the UB list, validity vs safety invariants, provenance, aliasing models, drop check, `Pin`, FFI, C++ and Python boundaries. |
| `har-verify` | Deciding how to test or prove a change — the ladder from `clippy` through property tests, mutation adequacy, sanitizers, fuzzing, `miri`, `shuttle`/`loom`, `kani`, and deductive proof, plus the CI that turns them into evidence. |
| `har-supply` | Before adding a dependency or auditing the graph — `cargo-vet`, `cargo-deny`, `cargo-audit`, build-script sandboxing, artifact verification, toolchain and edition policy, SBOM, reproducible builds. |
| `har-threat` | Designing or reviewing a boundary that takes input you do not control — trust boundaries, STRIDE, agent and tool-output boundaries, resource bounds, subprocess containment, secrets, AEAD and key schedules. |
| `har-io` | A write must survive a crash, or code reads a clock, an RNG, or user-visible text — durability and `fsync`, transactions and retry classes, monotonic vs wall clock, entropy failure, graphemes and normalization. |
| `har-hot-path` | Before optimizing, or when writing a frame, render, or parse loop — measure first, profiler commands, `black_box`, allocation and clone rules, arenas, bounds-check elision, type sizes, release-profile tuning. |
| `har-target` | A crate must build for a target that is not the machine you are on — `no_std` and embedded, panic handlers and allocators, interrupts, target atomic capability, cross-compilation, WebAssembly, const generics. |
| `har-layout` | Adding a module or crate, wiring observability, or when builds got slow — workspace shape, `ARCHITECTURE.md`, naming, `tracing` and metrics, panic policy, CLI output, serde and config, compile-time budget. |

`har`, `har-verify`, and `har-supply` are the base three. The rest load on demand.

## Sources

These skills are original prose distilled from published work by others. The thinking is theirs; any errors in compression are mine. Read the sources — they are all free.

- **[High Assurance Rust](https://highassurance.rs)** — Tiemoko Ballo. The book this family is named for, and the backbone of `har`, `har-supply`, `har-verify`, and `har-threat`: the verification ladder, dependency trust, threat modeling, and the discipline of deriving assurance rather than claiming it.
- **[The Rustonomicon](https://doc.rust-lang.org/nomicon/)**, the [Rust Reference](https://doc.rust-lang.org/reference/behavior-considered-undefined.html), and the [Unsafe Code Guidelines](https://rust-lang.github.io/unsafe-code-guidelines/) — the UB list, validity, provenance, variance, drop check, and aliasing rules behind `har-unsafe`.
- **[Effective Rust](https://effective-rust.com)** — David Drysdale, with the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) and the [Cargo SemVer reference](https://doc.rust-lang.org/cargo/reference/semver.html) — trait design, dispatch, ownership ergonomics, and compatibility behind `har-api`.
- **[Rust Atomics and Locks](https://marabos.nl/atomics/)** — Mara Bos. Memory ordering, `Send`/`Sync`, the cache-coherence cost model, and the primitive-building material behind `har-concurrent`.
- **[The Rust Performance Book](https://nnethercote.github.io/perf-book/)** — Nicholas Nethercote — plus [mre/idiomatic-rust](https://github.com/mre/idiomatic-rust), [corrode.dev](https://corrode.dev), [matklad's writing](https://matklad.github.io), and the `tracing` docs, behind `har-hot-path` and `har-layout`.
- **The [tokio](https://tokio.rs/tokio/tutorial) tutorial and docs**, [Alice Ryhl](https://ryhl.io/blog/async-what-is-blocking/) on blocking and actors, [sunshowers](https://sunshowers.io/posts/cancelling-async-rust/) on cancellation, and the `tokio-util` codec docs, behind `har-async`.
- **[The Embedded Rust Book](https://docs.rust-embedded.org/book/)** and the [Embedonomicon](https://docs.rust-embedded.org/embedonomicon/), with the [Rust platform support](https://doc.rust-lang.org/rustc/platform-support.html) tiers, behind `har-target`.
- **[NIST SP 800-38D](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-38d.pdf)** on GCM invocation limits, [RFC 5869](https://www.rfc-editor.org/rfc/rfc5869) on HKDF, the RustCrypto AEAD docs, and [OWASP's agentic-security guidance](https://owasp.org/www-project-top-10-for-large-language-model-applications/), behind `har-threat`.
- **[Ferrocene](https://ferrocene.dev)** and the [Safety-Critical Rust Consortium](https://github.com/rustfoundation/safety-critical-rust-consortium) — the qualified-toolchain and build-discipline rules that `har-supply` and `har-verify` borrow in their lightweight form: pin the toolchain, clean before shipping, never lower a warning to get a build green.

## Contributing

A change should make an agent write different code. Rules that only describe Rust do not earn their lines. Keep the ownership boundaries: if a topic belongs to another skill, reference it in a clause rather than restating it.
