# How Pimalaya works

This document explains how the Pimalaya project is structured and the conventions every repository follows. It is written for **both humans and AI agents**: it is the shared context you need before touching any Pimalaya codebase.

It is intentionally **generic**: it describes patterns that apply to every current and future Pimalaya crate, and uses specific crates only as examples. Each repository also ships its own `CONTRIBUTING.md` and `ARCHITECTURE.md` with project-specific details.

**To contribute (human or AI), read in this order:**

1. the [Pimalaya README](https://github.com/pimalaya) (organization profile): what exists and how the pieces fit together;
2. **this document** (Pimalaya `ARCHITECTURE.md`): the shared architecture and conventions;
3. the repository's `CONTRIBUTING.md`: how to build, test and submit changes there;
4. the repository's `ARCHITECTURE.md`: that specific crate's internals.

> This is a living draft. If something here contradicts a repository's own docs or current code, the code wins; please flag the discrepancy.

## Table of contents

- [1. Mission](#1-mission)
- [2. The big picture](#2-the-big-picture)
- [3. The I/O-free approach](#3-the-io-free-approach)
- [4. One crate, up to three layers](#4-one-crate-up-to-three-layers)
- [5. Library conventions](#5-library-conventions)
- [6. How applications share infrastructure](#6-how-applications-share-infrastructure)
- [7. Rust code style](#7-rust-code-style)
- [8. Documentation and changelog](#8-documentation-and-changelog)
- [9. Licensing](#9-licensing)
- [10. Build, test and lint](#10-build-test-and-lint)
- [11. Notes for AI agents](#11-notes-for-ai-agents)

## 1. Mission

Pimalaya improves open-source tooling around **Personal Information Management** (PIM): emails, contacts, calendars, tasks and timers. It pursues two goals:

1. Provide **I/O-free** Rust libraries dedicated to the PIM domain, so application developers never have to reinvent IMAP, JMAP, CalDAV, Maildir, and friends.
2. Provide quality applications (CLIs and TUIs) built on top of those libraries.

## 2. The big picture

Pimalaya is a set of small, **independent repositories**; there is no monorepo, and cross-repo dependencies are ordinary Cargo dependencies. They stack into layers (see the flowchart on the [organization profile](https://github.com/pimalaya)):

- **Applications**: user-facing CLIs and TUIs.
- **Domain libraries**: one per PIM domain, exposing a backend-agnostic API (for example `io-email`, `io-addressbook`, `io-calendar`).
- **Protocol and storage libraries**: one per wire protocol or on-disk format (for example `io-imap`, `io-jmap`, `io-smtp`, `io-maildir`, `io-vdir`, `io-webdav`, `io-http`, `io-oauth`).
- **Foundation**: shared transport helpers (stream, TLS, SASL) live in `pimalaya-stream`.
- **Application framework**: argument parsing, configuration, terminal UI and service discovery, shared by every app (see section 6).

Naming follows the layer: library crates are `io-<thing>` (a domain, protocol or storage concern) or `pimalaya-<thing>` (framework); applications take a product name. Each layer depends only on layers below it, and **no library ever reaches below its layer to perform I/O itself**, which is the whole point of the next section.

## 3. The I/O-free approach

This is the single most important idea in Pimalaya. Read it carefully.

**A library computes _what_ I/O to perform; it never performs the I/O itself.** All protocol and storage logic lives in `no_std` state machines called *coroutines*. The actual side effects (sockets, TLS, filesystem, clock, randomness) are performed by the caller. This is the "sans-I/O" pattern, and it buys:

- **Portability**: the core is `no_std`, so it runs on servers, in the browser (WASM), on embedded targets, or inside someone else's runtime.
- **Runtime independence**: the same coroutine drives a blocking client, an async client, or an in-memory test or fuzz harness. The library does not choose blocking vs async for you.
- **Testability**: logic is exercised by feeding scripted replies, with no network and no disk.

### The coroutine contract

Every coroutine implements a per-crate `*Coroutine` trait whose single method is:

```rust
fn resume(&mut self, arg: Option<Reply>) -> State<Yield, Return>
```

`State` has two variants:

- `Yielded(Y)`: an intermediate step. `Y` is a `Wants*` request describing the I/O the caller must perform next (for example `WantsRead`, `WantsWrite`, `WantsFileExists`, `WantsRandom { len }`, `WantsTime`). The caller performs it and feeds the result back as the matching `Reply` on the next `resume`.
- `Complete(R)`: terminal. By convention `R = Result<Output, Error>` carrying the final value.

A caller's driver loop (blocking flavor) looks like this:

```rust
let mut arg = None;
loop {
    match coroutine.resume(arg.take()) {
        State::Yielded(Wants::Read) => {
            let n = stream.read(&mut buf)?;
            arg = Some(Reply::Read(buf[..n].to_vec()));
        }
        State::Yielded(Wants::Write(bytes)) => {
            stream.write_all(&bytes)?;
            arg = Some(Reply::Write);
        }
        State::Complete(result) => break result,
    }
}
```

**Do not put I/O in the coroutine core. Do not put domain logic in the driver.** That separation is what every layer below depends on.

## 4. One crate, up to three layers

A Pimalaya crate is organized as up to three layers, each behind a Cargo feature, so a consumer pulls in only what it needs. This is the **dual library/CLI** pattern: a single repository can be a pure library, or the same library plus a ready-to-run binary.

1. **I/O-free coroutines** (`no_std` core, always present). The entire protocol, format or domain logic. No I/O, no async runtime, no `std`.
2. **Std client** (`client` feature, optional). A thin, blocking driver that runs the coroutines against `std` I/O (`std::net`, `std::fs`, `std::time`, ...), so consumers who just want a working client do not have to write the driver loop themselves.
3. **CLI or TUI** (binary feature, optional). A user-facing front-end that consumes the std client, reads TOML configuration, and renders output.

A repository stops at whichever layer makes sense:

- **Pure library**: layers 1 and 2 (for example `io-imap`, `io-vdir`). Published to crates.io as a library.
- **Library plus CLI**: layers 1 to 3 in one repo, the CLI gated behind a feature so library consumers never pull in the argument parser or terminal dependencies (for example `ortie` and `pimconf`, which both ship coroutines, a blocking client, and a CLI).
- **Standalone application**: a repo whose identity is the product (for example `himalaya`, `neverest`, `cardamum`). It wires several libraries together and owns the I/O loop.

The rule of thumb: a layer is always feature-gated, never assumed. Gate features on **items** (`mod`, `fn`, variant), not around blocks inside a function body. Never use `compile_error!` to forbid a feature combination; gate the module and `bail!` at the call site instead.

## 5. Library conventions

Conventions shared by every `io-*` crate. When in doubt, copy an existing library such as `io-imap` (networked) or `io-vdir` (filesystem).

### Crate setup

- `#![no_std]` is declared **unconditionally**.
- `extern crate std;` is gated on `#[cfg(feature = "client")]`; there is no separate `std` feature (`client` implies `std`).
- `alloc` is available. Core collections are `BTreeMap` / `BTreeSet`, not `HashMap` (which needs a random hasher and `std`).

### Coroutine template

One crate's coroutines set the template for all others; for the networked libraries that template is `io-imap`'s `auth_*` and `login` coroutines. A new coroutine mirrors them:

- A single `new` (plus an `opts`-style constructor only when needed), not a pile of overlapping constructors.
- A `try!`-style macro to short-circuit errors inside `resume`.
- Dedicated `State` variants rather than a generic catch-all.
- A `fmt::Display` impl describing the coroutine.
- Normalized errors of the shape `"<PROTOCOL> <operation> failed: <cause>"`.
- A consistent unit-test layout per coroutine.

### Modules and visibility

- `mod.rs` files contain **only** module declarations (`pub mod ...`); no code, no `pub use`, no items.
- Do **not** re-export types at the crate root. Callers use module-qualified paths, for example `io_email::mailbox::Mailbox`.
- The `types`/`utils` exception: those submodules are private `mod` plus `#[doc(inline)] pub use module::*;` in the parent `mod.rs`, so callers reach them through a flat path.
- A feature-gated module file carries no internal `#[cfg]`; gate it once at the parent `mod` declaration.

### Errors and logging

- `Result` / `Error` / state enums use tuple shape for one payload field, struct shape for two or more, unit for none.
- Low-level `io-*` libraries log only through `trace!`, never `info!` / `debug!` / `warn!` / `error!`. Log messages start lowercase; user-facing error messages start capitalized. Neither carries a trailing period.

## 6. How applications share infrastructure

Applications do not each reinvent argument parsing, configuration or discovery. They share a small framework, and a set of command conventions.

### Framework crates

- **`pimalaya-cli`**: argument parsing, interactive prompts, spinner, terminal helpers, build scripts.
- **`pimalaya-config`**: configuration loading (TOML) and secrets handling.
- **`pimalaya-tui`**: terminal UI building blocks for the TUIs.
- **`pimconf`**: discovery of PIM services (Thunderbird autoconfig, DNS SRV per RFC 6186, well-known URLs per RFC 6764).

### Command conventions

- Subcommands carry their data **inside their own variant** and chain via `execute(self, prior_spec)`; there is no shared options struct flattened across every variant.
- Per-protocol behavior lives on `pub` fields of the backend client structs, never as new methods on the shared domain API.
- A domain library's shared API is a strict **least common denominator**: it exposes only what every targeted backend supports. Backend-specific capabilities move to protocol-specific commands.
- Shared commands address messages by their stable identifier (for email, the IMAP UID); protocol-only addressing (such as `--seq`) belongs only in protocol-specific commands.
- User-facing output is consistent: spinner messages carry no trailing ellipsis (the spinner already animates), and error messages are capitalized and dotless.

### Output streams and exit codes

Applications treat the standard streams uniformly, which is what makes them scriptable:

- Almost every application supports **structured JSON output** alongside the default human-readable output (typically via a global `--output json` flag).
- **All program output goes to `stdout`**, both successful data and error messages. Errors are not written to `stderr`.
- The **exit code is the only signal** that distinguishes data from an error: a zero exit code means the `stdout` payload is the result, a non-zero exit code means it is an error.
- **`stderr` carries logs only** (the `trace`/log stream), never the primary output.

This keeps `stdout` a single clean channel a caller can parse (especially as JSON) regardless of success or failure, and lets it switch on the exit code to interpret what it received.

## 7. Rust code style

- Always use `use` imports; never write fully-qualified paths inline (no `std::str::from_utf8(...)` at the call site).
- Import order: `core`, blank line, `alloc` + `std`, blank line, third-party crates, blank line, `crate::`. Never `super::`.
- Prefer **methods on structs/enums** over free functions; a module full of `fn`s that all take the same first argument is a struct waiting to happen.
- Prefer `pub` fields and `Struct { ... }` literals over trivial `::new(a, b, c)` constructors; reserve constructors for non-trivial work.
- Do not extract a helper (function or macro) until it has at least two call sites; inline single-use code.
- Order items in a file top-down by abstraction level: high-level types and public API at the top, private helpers at the bottom, tests last.
- Keep code clean and concise but not cramped: use blank lines and named locals for readability.
- Wrap comments and inline docs at 80 columns. Avoid bare `//` comments; when a comment is needed, prefix it with a greppable tag (`NOTE` / `TODO` / `HACK` / `SAFETY` / `FIXME`).
- Every public type, struct and enum gets a one-line `///` description so the generated docs stay clean.
- Never use em dashes in code, comments or docs; use `:` or `;` instead.

## 8. Documentation and changelog

- READMEs and doc-comments are concise yet precise: one short sentence (protocol + operation + wrapped type), plus at most one more for non-obvious behavior.
- Do not hard-wrap prose in markdown files; each paragraph or bullet stays on a single long line.
- README layout and section order are standardized across repositories; follow an existing application README for apps and an existing library README for libraries.
- `CHANGELOG.md` follows [Keep a Changelog 1.0.0](https://keepachangelog.com/en/1.0.0/): bullets start with `- ` and a past-tense verb, with an optional indented paragraph after a blank line.

## 9. Licensing

Every Pimalaya crate, library or application, is **dual-licensed under MIT OR Apache-2.0**, with no per-file license header.

A few repositories are currently still licensed under AGPL-3.0; these are being migrated back to the dual MIT OR Apache-2.0 license. New work should assume the dual license.

## 10. Build, test and lint

- Repositories pin their toolchain with a `flake.nix`. Run cargo through the dev shell: `nix develop --command cargo <...>` (not `nix-shell`).
- After any code change, run `cargo fmt` on the affected crate(s).
- Before opening a PR, make sure `cargo test`, `cargo clippy` and `cargo deny check` pass.
- Validate libraries against **both** their `no_std` core and their `client` feature; never let `std`-only code leak into the core.

## 11. Notes for AI agents

If you are an automated agent working on a Pimalaya repository:

- **Read before you write.** Read the current state of a file immediately before editing it; the working tree may have changed.
- **Respect the layering.** Never add I/O to a coroutine core, never add domain logic to a client, and never add a method to a shared domain API to express backend-specific behavior.
- **Match the surrounding code.** Mirror the existing module's naming, comment density and idioms. When adding a coroutine, copy the crate's reference coroutine.
- **Do not invent.** Verify a file, function, feature or flag still exists before relying on it. If a convention here conflicts with the code, follow the code and report the conflict.
- **Secrets.** Never print or echo tokens, passwords or other secrets in output, logs or commit contents.
- **Disclose AI involvement** as required by the repository (most repos have an "AI disclosure" section in their README).
- **Scope your commits** to the task; do not opportunistically reformat unrelated code.

---

Questions, corrections and proposals are welcome: open an issue on the relevant repository or reach the project via [Matrix](https://matrix.to/#/#pimalaya:matrix.org).
