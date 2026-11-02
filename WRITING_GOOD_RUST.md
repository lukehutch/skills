# Prompting LLMs to Write Correct, Idiomatic Rust

Merged and filtered field guide, revision 2 (2 November 2026). Combines the earlier version of this guide with three further research reports, filtered for current relevance and verified where possible. Numbered superscripts link to the references.

[1. How this guide was filtered](#method) [2. Harness rules (highest leverage)](#harness) [3. Ownership and borrowing](#own) [4. Errors, panics, destructors](#err) [5. Types and API boundaries](#types) [6. Async and concurrency](#async) [7. Unsafe and edition 2024](#unsafe) [8. Performance and structure](#perf) [9. Drop-in configuration](#config) [References](#refs)

## 1. How this guide was filtered

Inclusion criteria: a rule is kept only if (a) it is timeless and sound Rust engineering practice, or it addresses a structural property of LLM agents that does not disappear as models improve; (b) it would actually change the output of a current frontier model, so rules that a capable model already follows reliably (for example `&str` instead of `&String`, iterator idioms, basic trait derivation) are omitted; and (c) it is either verified against primary documentation, verified by compiling code, or explicitly marked as a practitioner heuristic.

Age policy: measured findings from models more than six months old (GPT-4, GPT-4o, o3, Claude 3.5 through Claude Opus 4, Gemini 2.5) are no longer used as evidence of current behavior. They are retained only where the mechanism they demonstrate is model-independent (for example, that a deterministic compiler loop converts unverifiable output into verifiable output), and then only as supporting context.

Grades: \[V\] verified here by compiling or running the toolchain (rustc 1.97.0, edition 2024); \[O\] official documentation or project policy; \[S\] structural argument independent of model generation; \[P\] practitioner heuristic.

On crate-name hallucination: per your instruction this finding (based on Claude 4 / Gemini 2.5 era models) is removed. One related failure is not a hallucination problem and does not go away with model quality: **training-cutoff drift**. The previous revision of this guide recommended the rand 0.9 API, but rand 0.10.0 was released on 8 February 2026 and renamed `Rng` to `RngExt`, `OsRng` to `SysRng` and `choose_multiple` to `sample`[\[5\]](#r5), which I did not know. Code written from memory against 0.9 does not compile against 0.10. Rule H3 below exists for this reason.

## 2. Harness rules (highest leverage)

H1. Define "done" as a set of commands that pass, and require their output as evidence. \[O\]\[S\]

`cargo fmt --all`; `cargo check --all-targets`; `cargo clippy --all-targets --all-features -- -D warnings`; `cargo test --all-targets` plus `cargo test --doc`; for libraries, `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps`. Anthropic identifies giving the model a way to verify its work as the single highest-leverage practice[\[1\]](#r1). Re-run the full set after every repair: in CRUST-Bench, repairing toward passing tests lowered build success by 5% to 20%[\[6\]](#r6) (older models, but the mechanism, that one fix can break another check, is model-independent).

Correction to input report C: `cargo doc --no-deps -- -D warnings` is rejected by cargo ("unexpected argument '-D'"). Use the `RUSTDOCFLAGS` form. \[V\]

H2. Put zero-exception rules in lints and hooks; put judgment in prose. \[O\]\[V\]

Anthropic's guidance: if a CLAUDE.md rule is still violated despite emphasis, it belongs in a hook[\[1\]](#r1). Two mechanisms block the commonest "make it green" shortcut, which is suppressing the lint rather than fixing the code:

- `clippy::allow_attributes = "deny"` rejects any `#[allow(...)]`; `clippy::allow_attributes_without_reason = "deny"` requires a `reason`. Legitimate exceptions then take the form `#[expect(clippy::lint, reason = "...")]`, which also warns when the exception becomes unnecessary (stable since Rust 1.81). Both lints were confirmed to fire on rustc 1.97 \[V\].
- Prevent the agent from weakening the configuration itself: deny agent edits to `clippy.toml`, `rustfmt.toml` and `rust-toolchain.toml` via Claude Code permission rules, and fail CI if the `[lints]` tables change without review.

H3. For any dependency, read the locked version's real API before writing calls against it. \[V\]\[S\]

Training data is a snapshot; crates.io is not. Instruct the agent to look up the version in `Cargo.lock` and read the source under `~/.cargo/registry/src/` or `docs.rs/<crate>/<version>` whenever it uses an API it has not already seen compile in this repository. The rustc-dev-guide gives the same instruction: tell the model to read the relevant material rather than treating it as a source of truth[\[9\]](#r9). The rand 0.10 case above demonstrates the failure on a current model. Verified rand 0.10 form:

```
use rand::RngExt; // rand 0.10: was `use rand::Rng;` in 0.9

fn roll() -> u8 {
    rand::rng().random_range(1..=6)
}
```

H4. Write instructions as rationale plus preference order, not as absolute prohibitions, and omit code examples from CLAUDE.md. \[O\]

The Claude Code team reported in July 2026 that examples and "do not do X" instructions are no longer best practice for current frontier models (Fable 5, Opus 4.8), and that they cut the Claude Code system prompt by 80% accordingly[\[2\]](#r2). An absolute ban on `.clone()` is wrong in the cases where cloning an `Arc` into a spawned task is correct; a stated rationale and a resolution order (rule O1) generalize correctly. Keep CLAUDE.md short and limited to facts the agent cannot infer from the repository[\[1\]](#r1); enforcement belongs in H2.

H5. Treat edits to tests, assertions and lint configuration as out of scope during a fix. \[S\]\[O\]

Under a pass/fail objective an agent can satisfy the check instead of the requirement: weakening an assertion, special-casing a test input, adding `#[ignore]`. Anthropic's pattern is to write a failing test first and then fix the code without modifying the test[\[1\]](#r1); a reviewer in a fresh context, which has not seen the reasoning that produced the change, is a second check[\[1\]](#r1).

H6. When editing existing code, upgrade the touched functions to the repository standard rather than imitating the surrounding code. \[S\]\[P\]

Code in the immediate context conditions generation more strongly than general instructions; a function surrounded by `unwrap()` and `Box<dyn Error>` will tend to acquire more of both. State the rule explicitly and scope it to the functions changed, so refactors do not sprawl.

H7. For non-trivial work, fix types and signatures before implementation. \[S\]

Have the public types, error enums and function signatures written with `todo!()` bodies and reviewed before bodies are written. Signatures constrain the space of valid implementations and expose ownership problems before effort is spent on them. CRUST-Bench supplies hand-written Rust interfaces for exactly this reason[\[6\]](#r6). For large ports, a written mapping document (Bun's 600-line PORTING.md for its Zig-to-Rust rewrite[\[8\]](#r8)) serves the same role at repository scale.

## 3. Ownership and borrowing

O1. Resolve borrow conflicts structurally, in this order, before cloning. \[P\]\[V\]

1. Narrow the borrow: end it earlier with an inner block, or reorder statements so the `&mut self` call completes first.
2. Split the borrow: destructure `let Self { a, b } = self;` or pass disjoint fields (`&mut self.state, &self.config`) to a free function or helper. Field-level disjointness is visible to the borrow checker inside one function body but not across a `&mut self` method call.
3. Move out without allocating: `std::mem::take`, `std::mem::replace`, `Option::take`.
4. Change the representation: indices into an owning `Vec` (arena) for graphs and parent links, instead of `Rc<RefCell<_>>` or self-referential lifetimes.
5. Clone deliberately when ownership transfer is the actual semantics (an `Arc` into a task, a small config into a thread), not to end a borrow.

```
struct Ledger { items: Vec<u32>, total: u64 }

impl Ledger {
    fn add_all(&mut self) {
        let Self { items, total } = self; // two disjoint borrows, no clone
        for &x in items.iter() {
            *total += u64::from(x);
        }
    }
}
```

O2. A clone introduced to end a borrow can silently discard writes. \[V\]

This is a correctness defect, not only a performance one: subsequent mutation lands on the copy. rustc catches the trivial case (an unused assignment), but not the case where the copy is used, so the output looks right while the canonical state is unchanged. Verified to compile without any warning under `-D warnings` and Clippy:

```
// Wrong: `x` reaches the returned string but never reaches `self.items`.
fn add(&mut self, x: u32) -> String {
    let mut items = self.items.clone(); // clone added to dodge a borrow conflict
    self.note("add");                   // &mut self call
    items.push(x);
    format!("{items:?}")
}

// Right: sequence the &mut self call first; no clone is needed.
fn add(&mut self, x: u32) -> String {
    self.note("add");
    self.items.push(x);
    format!("{:?}", self.items)
}
```

O3. Treat lifetime parameters spreading across many structs as a design signal. \[P\] If borrowed views are being threaded through long-lived types to satisfy the compiler, the type should usually own its data, or the borrowed view should be confined to a short-lived struct used within one call.

## 4. Errors, panics, destructors

E1. Use the lazy form of fallback combinators whenever the fallback computes, allocates, has side effects or can panic. \[V\]

Arguments are evaluated before the call, so `opt.unwrap_or(m["default"])` indexes (and can panic) even when `opt` is `Some`. Use `unwrap_or_else`, `ok_or_else`, `map_or_else`. The detecting lint, `clippy::or_fun_call`, is in the nursery group and therefore off by default; it was confirmed to fire when enabled explicitly \[V\]. Enable it individually.

E2. Libraries return typed errors; applications may aggregate. \[O\]\[P\]

Library crates: one `thiserror` enum per crate or subsystem, with a variant per failure a caller may need to distinguish, causes preserved via `#[source]`/`#[from]`. No `anyhow::Error`, `Box<dyn Error>` or `String` errors in public library signatures, since callers cannot match on them. Binaries: `anyhow::Result` with `.with_context(|| ...)` at I/O and configuration boundaries.

E3. Panic only on violated invariants, and say which invariant. \[O\]

`unwrap()` is denied outside tests (lint, H2). `expect("...")` is permitted where prior logic guarantees success and the type system cannot express it; the message states the invariant. Decide lock poisoning explicitly (`unwrap_or_else(PoisonError::into_inner)` recovers the data) rather than reflexively unwrapping `lock()`.

E4. `Drop` must not panic, block or report failure; fallible teardown gets an explicit consuming method. \[O\]\[V\]

A panic in `drop` during unwinding aborts the process, and `drop` has no way to return an error. Expose `fn close(self) -> Result<(), E>` that performs the fallible work, and keep `Drop` as best-effort cleanup that ignores errors.

```
impl Sink {
    pub fn close(mut self) -> std::io::Result<()> {
        if let Some(mut f) = self.file.take() {
            f.flush()?;
            f.sync_all()?;
        }
        Ok(())
    }
}
impl Drop for Sink {
    fn drop(&mut self) {
        if let Some(mut f) = self.file.take() {
            let _ = f.flush(); // best effort only
        }
    }
}
```

## 5. Types and API boundaries

T1. Encode mutually exclusive states as enum variants carrying their data. \[O\] Replace sets of booleans and co-dependent `Option` fields with one enum, so impossible combinations cannot be constructed. Replace `bool` parameters in public functions with two-variant enums. Microsoft's guidelines for AI-assisted Rust list strong types as a design rule for this reason[\[10\]](#r10).

T2. Parse at the boundary into newtypes with fallible constructors. \[O\]\[V\] Validate once in `TryFrom`/`FromStr`; the existence of a value of the type then certifies validity, and inner code needs no re-checks. Keep the field private so the constructor cannot be bypassed.

T3. Match your own enums exhaustively, without a `_ =>` arm. \[S\] A catch-all arm converts the compile error you want when a variant is added into silent misbehavior. For public enums that downstream crates match on and that you expect to extend, mark them `#[non_exhaustive]` instead.

T4. Choose typestate versus a runtime state enum by where the state lives. \[P\] Typestate (`Builder<Ready>` via `PhantomData`) when transitions are decided in code and misuse should be a compile error. A runtime enum when the state is loaded from storage, serialized, or held in a heterogeneous collection.

T5. Keep `Rc`, `RefCell` and lock types out of public signatures. \[P\] Wrap shared state in a newtype (for example a `#[derive(Clone)]` struct around `Arc<Inner>`) so the synchronization strategy can change without an API break, and so `!Send` types do not leak into users' async code.

## 6. Async and concurrency

A1. Never hold a `std::sync` (or `parking_lot`) guard across `.await`; scope it in a block. \[O\]\[V\]

Tokio's documentation states the compiler will not prevent this when the future is not required to be `Send`, and that it almost never yields correct concurrent code because it readily deadlocks[\[3\]](#r3). With `tokio::spawn` the held guard makes the future `!Send` and compilation fails, which is the case the agent then "fixes" badly. Enforce with `clippy::await_holding_lock`.

```
async fn bump(state: Arc<Mutex<u64>>) {
    let n = {
        let mut g = state.lock().unwrap_or_else(PoisonError::into_inner);
        *g += 1;
        *g
    }; // guard dropped before the await
    persist(n).await;
}
```

A2. Use `std::sync::Mutex` for plain data; `tokio::sync::Mutex` only when the guard must span an `.await`. \[O\] Tokio's docs say the blocking mutex should be preferred where it can be used, and that a dedicated task owning the resource, reached by messages, is often better than an async mutex[\[3\]](#r3). Several published CLAUDE.md templates state the opposite; follow the docs.

A3. Own every spawned task's lifetime. \[O\]\[P\] Keep `JoinHandle`s (or use `JoinSet`) so tasks are awaited or aborted at shutdown and their panics are observed; signal shutdown with a cancellation token. Every branch of a `tokio::select!` in a loop must be cancellation-safe, since the losing branch's future is dropped mid-operation; consult each method's documented cancel-safety.

A4. In public traits, write async methods as RPITIT with an explicit `Send` bound. \[V\]\[O\]

A bare `async fn` in a public trait cannot have `Send` required by callers; rustc warns ("use of `async fn` in public traits is discouraged as auto trait bounds cannot be specified") \[V\]. Declare `fn get(&self, k: u64) -> impl Future<Output = Option<Vec<u8>>> + Send;` and implement it with `async fn`. Neither form is dyn-compatible; use `#[async_trait]` or manual boxing only where `dyn Trait` is required.

A5. Keep blocking work off runtime worker threads; bound every queue. \[O\]\[P\] Blocking file I/O, synchronous drivers and CPU-heavy work go to `spawn_blocking`, a dedicated thread, or rayon. Use bounded channels so backpressure exists; an unbounded channel needs a documented reason why its size is bounded in practice.

## 7. Unsafe and edition 2024

U1. `unsafe_code = "deny"` by default; exceptions are narrow, documented and Miri-tested. \[O\] Allow it only in the smallest module that needs it, with a `// SAFETY:` comment per block naming the invariant (`clippy::undocumented_unsafe_blocks`), a `# Safety` doc section per `unsafe fn`, a safe wrapper API, and `cargo +nightly miri test` on its tests. `unsafe` introduced to get past a borrow error is a defect[\[8\]](#r8).

U2. Edition 2024 soundness changes. \[O\] `std::env::set_var`/`remove_var` are `unsafe` because they are unsound once other threads may read the environment[\[4\]](#r4); call them only before spawning threads, or better, pass configuration explicitly. `extern` blocks must be `unsafe extern`, `no_mangle` becomes `#[unsafe(no_mangle)]`, and `unsafe_op_in_unsafe_fn` warns by default.

## 8. Performance and structure

P1. State performance acceptance criteria and benchmark hot paths; passing tests says nothing about complexity. \[S\] The documented case is an LLM-generated SQLite reimplementation (Claude Opus 4 era) whose primary-key lookup was 20,171× slower than SQLite because the planner did not recognize `INTEGER PRIMARY KEY` aliases, compounded by per-statement AST clones and page copies on cache hits[\[7\]](#r7). The model is old; the lesson, that compiling and passing tests does not establish algorithmic adequacy, is not. Ask for the complexity of each hot operation and add a `criterion` benchmark.

P2. Audit per-iteration allocation. \[P\] `.clone()`, `.to_vec()`, `.to_string()`, `format!` and `collect()` inside loops or per-request paths, and log messages formatted before the level check.

P3. Library hygiene. \[P\] Business logic in `lib.rs` with a thin `main.rs`, so integration tests and benchmarks can reach it. Library crates depend on heavy crates with `default-features = false` and opt into what they use, so they do not force features on downstream users. Workspaces centralize versions and lints in `[workspace.dependencies]` and `[workspace.lints]`.

P4. Target-specific traps worth stating in context when relevant. \[O\]\[P\] On `wasm32-unknown-unknown`, `std::time::Instant::now()` and `std::thread` panic at runtime although they compile. In stdio-based protocol servers (for example MCP), anything written to stdout corrupts the protocol stream; route logs to stderr.

## 9. Drop-in configuration

### 9.1 Lints (root Cargo.toml)

Self-correction: a gate run with `-D warnings` promotes every warning to an error, so `pedantic = "warn"` (recommended in the previous revision and in report C) and `nursery = "warn"` (report C) are effectively deny-level in CI. Blanket nursery denial in particular invites churn and suppression attempts. The set below is curated so that every enabled lint is one you are willing to treat as an error; run `pedantic` separately as a non-blocking advisory pass if wanted.

```
[workspace.lints.rust]
unsafe_code = "deny"
unsafe_op_in_unsafe_fn = "deny"

[workspace.lints.clippy]
unwrap_used = "deny"
expect_used = "warn"
panic = "warn"
todo = "deny"
unimplemented = "deny"
dbg_macro = "deny"
allow_attributes = "deny"
allow_attributes_without_reason = "deny"
await_holding_lock = "deny"
let_underscore_future = "deny"
or_fun_call = "warn"
undocumented_unsafe_blocks = "deny"
large_futures = "warn"

# member crates:  [lints] workspace = true
```

```
# clippy.toml
allow-unwrap-in-tests = true
allow-expect-in-tests = true
allow-dbg-in-tests = true
```

### 9.2 Stop hook (`.claude/hooks/rust-gate.sh`)

Exit code 2 returns stderr to the agent and prevents it from ending the turn[\[1\]](#r1).

```
#!/usr/bin/env bash
set -u
run() { if ! out=$("$@" 2>&1); then printf '%s\n' "$out" >&2; exit 2; fi; }
run cargo fmt --all -- --check
run cargo clippy --all-targets --all-features --quiet -- -D warnings
run cargo test --all-targets --quiet
run cargo test --doc --quiet
exit 0
```

### 9.3 CLAUDE.md block

Rationale-first, no code examples, per H4.

```
# Rust conventions

## Verification
Done means: cargo fmt --all; cargo clippy --all-targets --all-features -- -D warnings;
cargo test --all-targets; cargo test --doc. Report the commands and their output.
Lint configuration, rust-toolchain.toml and existing test assertions are not yours to change;
if one of them blocks correct work, stop and explain why.

## Dependencies
Your knowledge of crate APIs may predate the locked versions. Before calling an API you have not
seen compile in this repo, read the locked version (Cargo.lock, ~/.cargo/registry/src, or
docs.rs/<crate>/<version>). Prefer std where it suffices.

## Design order
For non-trivial changes, propose types, error enums and pub signatures (todo!() bodies) first.
Model exclusive states as enums with data; parse inputs into newtypes at boundaries.
Match this crate's own enums exhaustively, without a wildcard arm.

## Borrowing
Resolve borrow errors by narrowing or splitting borrows, mem::take/replace, or an index-based
representation. Clone only when transferring ownership is the intended semantics; a clone made to
end a borrow can silently drop later writes.

## Errors and panics
Library code returns thiserror enums; binaries use anyhow with context. unwrap is for tests;
expect states the invariant that guarantees success. Use the _else form of fallback combinators
when the fallback computes or can panic. Drop never panics; fallible teardown is close(self).

## Async
No std/parking_lot guard across .await: scope it. std Mutex for data; tokio Mutex only when a
guard must span an await. Keep JoinHandles; select! branches must be cancel-safe. Public async
trait methods: -> impl Future<Output = T> + Send. Blocking work goes to spawn_blocking.

## Editing existing code
Bring the functions you touch up to these conventions instead of matching nearby legacy code.
```

## References

1. Anthropic. "Best practices for Claude Code." Claude Code documentation. [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)
2. S. Willison. "A Fireside Chat with Cat and Thariq from the Claude Code team." 21 July 2026. [simonwillison.net/2026/Jul/21/cat-and-thariq/](https://simonwillison.net/2026/Jul/21/cat-and-thariq/)
3. Tokio documentation. `tokio::sync::Mutex`. [docs.rs/tokio/latest/tokio/sync/struct.Mutex.html](https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html)
4. The Rust Edition Guide. "Unsafe functions" (Rust 2024). [doc.rust-lang.org/edition-guide/rust-2024/newly-unsafe-functions.html](https://doc.rust-lang.org/edition-guide/rust-2024/newly-unsafe-functions.html)
5. rust-random/rand. CHANGELOG.md, entries 0.10.0 (2026-02-08) to 0.10.2 (2026-07-02). [github.com/rust-random/rand/blob/master/CHANGELOG.md](https://github.com/rust-random/rand/blob/master/CHANGELOG.md)
6. "CRUST-Bench: A Comprehensive Benchmark for C-to-safe-Rust Transpilation." arXiv:2504.15254 (older models; cited for mechanism). [arxiv.org/html/2504.15254v3](https://arxiv.org/html/2504.15254v3)
7. KatanaQuant. "Your LLM Doesn't Write Correct Code. It Writes Plausible Code." March 2026; companion repository KatanaQuant/kqsqlite. [blog.katanaquant.com](https://blog.katanaquant.com/p/your-llm-doesnt-write-correct-code); [github.com/KatanaQuant/kqsqlite](https://github.com/KatanaQuant/kqsqlite)
8. The Pragmatic Engineer. "The Pulse: What can we learn from Bun's rapid Rust rewrite with AI?" 2026. [blog.pragmaticengineer.com](https://blog.pragmaticengineer.com/the-pulse-what-can-we-learn-from-buns-rapid-rust-rewrite-with-ai/)
9. Rust Compiler Development Guide. "LLM guidance: writing." [rustc-dev-guide.rust-lang.org/llm-guidance/writing.html](https://rustc-dev-guide.rust-lang.org/llm-guidance/writing.html)
10. Microsoft. "Pragmatic Rust Guidelines: AI." [microsoft.github.io/rust-guidelines/guidelines/ai](https://microsoft.github.io/rust-guidelines/guidelines/ai)
11. Input reports: (A) "Executive Summary / Quick Prompting Checklist" deep-research report; (B) "Prompting LLMs (Claude) for Rust Code: Best Practices and Pitfalls"; (C) "Engineering First-Time-Right Rust with Large Language Models" (.docx). Supplied by the user; their claims were verified or filtered before inclusion.

Verification environment: rustc and Clippy 1.97.0, edition 2024, rand 0.10.3, thiserror 2, tokio 1. All Rust snippets above (in context of minimal enclosing types) compiled and passed `cargo clippy --all-targets -- -D warnings`; lint and cargo-command claims marked \[V\] were tested directly.