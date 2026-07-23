# Contributing guide

Thank you for investing your time in contributing to Pimalaya.

Whether you are a human or an AI agent, read these in order before touching any code:

1. the [Pimalaya README](https://github.com/pimalaya) for what the project is and how its repositories stack;
2. this guide, together with [ARCHITECTURE.md](./ARCHITECTURE.md) (the shared architecture and conventions), [GUIDELINES.md](./GUIDELINES.md) (how everything is documented and named) and [AI_POLICY.md](./AI_POLICY.md) (how AI is used here, and what an AI-assisted contribution must satisfy);
3. the repository's inline header documentation, starting with src/lib.rs (or src/main.rs for binaries): it is the architecture document of that crate;
4. the repository's own CONTRIBUTING.md when it exists (it documents only what differs from this guide) and its cairn/ folder (development history and living plans, following the Cairn convention).

## Development environment

The environment is managed by [Nix](https://nixos.org/download.html): nix develop spawns a shell with the right toolchain, and every cargo command below assumes it (or prefix them with nix develop --command).

Without Nix, install a recent stable toolchain via [rustup](https://rust-lang.github.io/rustup/); each crate pins its minimum in the rust-version field of Cargo.toml.

## Build

Libraries expose up to three feature-gated layers: the I/O-free coroutines (no feature required, no_std), the light client (client feature, wrapping a stream you opened yourself) and the full client (one TLS feature among rustls-ring, enabled by default, rustls-aws and native-tls). Check every layer, since gated code must never leak into the always-on coroutine core:

```sh
cargo build --no-default-features                    # coroutines only, no std leak
cargo build --no-default-features --features client  # light client, no TLS deps
cargo build --release                                # full client (default TLS)
```

When touching feature gates or imports, build with and without each feature. Binaries build with a plain cargo build; their features are listed in Cargo.toml.

## Lint, test, audit

```sh
cargo test --all-features                  # unit + doc tests
cargo clippy --all-targets --all-features
cargo fmt                                  # CI checks cargo fmt --check
cargo deny check                           # advisories, licenses, sources
```

Run clippy and fmt at the end of every piece of work: a change is not done until both pass clean.

Every public item carries an inline doc and the coroutine module examples are real doctests; keep both complete across the feature matrix:

```sh
RUSTFLAGS="-D missing_docs" cargo check --all-features
RUSTFLAGS="-D missing_docs" cargo check                       # feature-gated modules too
RUSTDOCFLAGS="-D rustdoc::broken_intra_doc_links" cargo doc --all-features
```

Runnable examples live in the examples folder when the crate ships some; run one with cargo run --example followed by its name.

## Override dependencies

All Pimalaya crates publish on crates.io and patch siblings locally when needed. To build against a locally modified dependency, add to Cargo.toml:

```toml
[patch.crates-io]
io-http.path = "/path/to/io-http"
```

## Commit style

Commits follow the [conventional commits specification](https://www.conventionalcommits.org/en/v1.0.0/), applied flexibly: keep the subject imperative and scoped, and describe the why in the body when it is not obvious.
