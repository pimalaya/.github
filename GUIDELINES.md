# Documentation and naming guidelines

How every Pimalaya repository documents itself, sets up its code, and names its public items. Written for both humans and AI agents, and referenced by the org-wide [CONTRIBUTING.md](./CONTRIBUTING.md). Read the [Pimalaya README](https://github.com/pimalaya) for what exists, [ARCHITECTURE.md](./ARCHITECTURE.md) for the shared architecture, this document for conventions, then the repository's own CONTRIBUTING.md and cairn/ folder.

> This is a living draft, iterated with usage. When a repository contradicts it, the repository needs realigning. Flag the discrepancy.

## How to read this document

Each rule is one paragraph with a stable id and a strength. The id is `scope-nnn`, where the scope names the area (for example cargo, readme, naming) and the number is stable: it never changes once assigned, even if rules are added, removed, or reordered around it. The strength is MUST for a hard requirement or SHOULD for a strong preference that a repository may override with a reason. MUST and SHOULD are mixed freely within a scope.

Scopes are organised into three groups. Code covers the source, its packaging, the repository file skeleton, commits, and code documentation. Markdown covers the documentation files, split by file. Audit covers tests, security, and licensing.

Templates are embedded throughout and are load-bearing. When creating or auditing a file, start from the template rather than reinterpreting the prose.

## Requesting a conformance check

Ask to check a repository against specific ids (`readme-003`, `naming-007`), against a whole scope (`cargo`), against a group (`Code`), or against everything. The answer is a table, one row per rule in scope, each row PASS, FAIL, or N/A with the evidence (a file and line, or the command output). Nothing in scope is left out of the table. Resolution is then iterated row by row.

## Table of contents

- Code: [repo](#repo), [commits](#commits), [nix](#nix), [cargo](#cargo), [crate](#crate), [header](#header), [inline](#inline), [logging](#logging), [naming](#naming), [cli](#cli)
- Markdown: [markdown](#markdown), [readme](#readme), [contributing](#contributing), [changelog](#changelog), [cairn](#cairn)
- Audit: [tests](#tests), [security](#security), [license](#license)

# Code

## repo

**repo-001** (MUST): every repository ships the same documentation skeleton: README.md, CHANGELOG.md, LICENSE-MIT and LICENSE-APACHE, deny.toml, a cairn/ folder, CONTRIBUTING.md when the repository deviates from the org-wide guide, SECURITY.md when applicable, and config.sample.toml when the binary reads a config file. Only deviate when the repository genuinely lacks the concept.

## commits

**commits-001** (MUST): commits follow the [conventional commits specification](https://www.conventionalcommits.org/en/v1.0.0/), applied flexibly. Keep the subject imperative and scoped. Describe the why in the body when it is not obvious.

## nix

**nix-001** (SHOULD): flake and packaging conventions live here. No rules are captured yet. Add them as the flake.nix and package.nix conventions settle, so a repository can be checked against them.

## cargo

Applies to Cargo.toml and, by extension, to any other language-specific manifest such as package.json. The Rust manifest rules below are the settled ones. Manifest rules for other ecosystems are added here as they are needed.

**cargo-001** (MUST): the library manifest uses the package field order shown below, description right after name.

```toml
[package]
name = "io-example"
description = "Example client library for Rust"
version = "0.1.0"
authors = ["soywod <pimalaya.org@posteo.net>"]
rust-version = "1.87"
edition = "2024"
license = "MIT OR Apache-2.0"
categories = ["api-bindings", "no-std"]
keywords = ["io-free", "no-std", "coroutine", "socket", "example"]
homepage = "https://pimalaya.org"
documentation = "https://docs.rs/io-example/latest/io_example"
repository = "https://github.com/pimalaya/io-example"

[package.metadata.docs.rs]
all-features = true
rustdoc-args = ["--cfg", "docsrs"]

[features]
default = ["rustls-ring"]
client = []
rustls-ring = ["client", "pimalaya-stream/rustls-ring", "dep:anyhow", "dep:pimalaya-stream"]
rustls-aws = ["client", "pimalaya-stream/rustls-aws", "dep:anyhow", "dep:pimalaya-stream"]
native-tls = ["client", "pimalaya-stream/native-tls", "dep:anyhow", "dep:pimalaya-stream"]
vendored = ["pimalaya-stream?/vendored"]

[[example]]
name = "std_example"
path = "examples/std_example.rs"
required-features = ["rustls-ring"]

[dev-dependencies]
env_logger = "0.11"

[dependencies]
log = { version = "0.4", default-features = false }
pimalaya-stream = { version = "0.0.1", default-features = false, optional = true }
thiserror = { version = "2", default-features = false }
```

**cargo-002** (MUST): the description matches the README one-sentence description, with no trailing dot.

**cargo-003** (MUST): the license is always MIT OR Apache-2.0.

**cargo-004** (MUST): io- libraries carry the api-bindings and no-std categories, and mix the family keywords (io-free, no-std, coroutine) with domain words.

**cargo-005** (MUST): for libraries, documentation points at docs.rs and the docs.rs metadata block enables all features with the docsrs cfg.

**cargo-006** (MUST): features follow the layered shape. default enables the default TLS provider. client = [] gates the std-blocking client, the blessed std-gating. Each TLS feature implies client, selects the pimalaya-stream provider, and pulls the optional client-side deps via dep:. vendored forwards to the weak pimalaya-stream dependency.

**cargo-007** (MUST): every example gets its own example block with explicit name and path, plus required-features when it needs a gated layer.

**cargo-008** (MUST): dependencies are alphabetical, each with default-features = false and only the needed features enabled, so nothing silently drags std or unused code into the no_std core. Layer-specific deps are optional and pulled by the features needing them. Dev-dependencies follow the same discipline.

**cargo-009** (MUST): binaries drop the documentation field, the docs.rs metadata block, and the no-std category. They declare a lib or bin name only when it must differ from the package default, and they add the release profile.

```toml
[profile.release]
lto = "fat"
codegen-units = 1
strip = "symbols"
panic = "abort"
```

## crate

**crate-001** (MUST): #![no_std] is unconditional on libraries, never feature-gated. extern crate alloc; is declared whenever the crate allocates. extern crate std; only when std is genuinely needed, usually behind the client feature.

**crate-002** (MUST): deliberately-std utility crates exposing no I/O-free coroutines (pimalaya-stream wrapping TLS providers and sockets, the pimalaya-* helpers) are exempt. They carry no #![no_std] and no extern crates, and their lib.rs opens directly with the docsrs attribute. Their modules sit flat at the crate root, and a runtime-named module (std, tokio) is only introduced the day two runtimes genuinely coexist.

**crate-003** (MUST): the golden rule for feature-gating is that a cargo feature is justified only when it pulls additional crates into the build, std included. The client feature gating the std-blocking client is the canonical example. When gating some code would not change the crate set at all, do not gate it: remove the feature and ship the code unconditionally.

**crate-004** (MUST): imports take from core and alloc as much as possible, and from std only the strict minimum core and alloc cannot provide. They are organised in blocks separated by one empty line, in this order: core, alloc, std, third-party crates, crate. super is never used, in-crate paths always go through crate. Within a block, imports from the same crate are merged into a single use. The only reason for two use declarations on the same crate is a feature gate on one of them.

```rust
use core::fmt;

use alloc::{string::String, vec::Vec};

#[cfg(feature = "client")]
use std::io::{Read, Write};

use serde::Deserialize;
use url::Url;

use crate::rfc6749::state::Oauth20State;
```

**crate-005** (MUST): compile_error! is banned, and so is any other way of failing the build over a cargo feature combination. A crate never refuses to compile because a feature is missing, redundant, or paired with another. Gate the module or the item on the features it genuinely needs, and let the call site bail! at runtime with a message naming what to enable. A partial build stays usable, the failure reads as a sentence rather than a macro error inside a dependency, and the decision lands where the user can act on it. A crate still carrying a compile_error! feature guard is migrating away from it.

```rust
// Wrong: the build dies for a combination the caller may not control.
#[cfg(not(any(feature = "rustls", feature = "native-tls")))]
compile_error!("Either feature `rustls` or `native-tls` must be enabled");

// Right: the module is gated, and the call site explains itself.
#[cfg(any(feature = "rustls", feature = "native-tls"))]
pub mod tls;

#[cfg(not(any(feature = "rustls", feature = "native-tls")))]
pub fn connect_tls(url: &Url) -> Result<StreamStd> {
    bail!("Cannot open a TLS connection to `{url}`: this build carries no TLS provider, rebuild with the `rustls-ring`, `rustls-aws` or `native-tls` feature")
}
```

**crate-006** (MUST): a feature gate is written once, at the declaration of what it gates. A module declared behind a #[cfg(feature = "x")] never repeats that cfg on the items inside it, nor on that file's own imports, and a #[cfg] never sits on a block inside a function body when it belongs on the item. A second gate inside an already-gated file is noise at best and a bug at worst: it can compile the module to something empty for a caller that enabled exactly the right feature, and nothing reports it.

## header

**header-001** (MUST): the lib.rs header (libraries) or main.rs header (binaries) is the equivalent of the retired per-repo ARCHITECTURE.md. It is a concise document, structured by sections, avoiding dash lists, describing the whole architecture of the crate and linking to inner resources (modules, cairn/ files, examples).

**header-004** (MUST): the architecture header obeys inline-001's four-line paragraphs like every other header, and nothing else about it shrinks. Trimming it means denser paragraphs, never fewer sections: it is the one document carrying what a reader cannot recover from the code, so a section is compressed or it stays as it is, never dropped. The test per paragraph is whether a competent reader who does not know the crate still learns the same architectural fact from the shorter version. When the fact is gone rather than compressed, the cut went too far.

**header-002** (MUST): lib.rs starts with #![no_std], followed by #![cfg_attr(docsrs, feature(doc_cfg))], then a blank line, then the header docs. main.rs starts directly with its header docs, since binaries are std and publish no rustdoc.

**header-003** (MUST): libraries never include the README as their rustdoc (no doc attribute including README.md). The README and the lib.rs header are two different documents by design, the public presentation versus the architecture.

```rust
#![no_std]
#![cfg_attr(docsrs, feature(doc_cfg))]

//! # io-example
//!
//! I/O-free Example coroutines built on io-http: every network
//! exchange is a resumable state machine emitting read and write
//! requests instead of performing I/O itself.
//!
//! ## Layout
//!
//! The source tree is organised by RFC, one folder per RFC, so the
//! RFC number is the version discriminator. [...]
```

## inline

**inline-001** (MUST): each module opens its header docs with a markdown title (`//! # Title`), then one or two sentences saying what the module is. Two is the maximum, not a target. Further paragraphs follow only when genuinely needed, for its place in the codebase and its relations with other components, and each of them is at most four lines. A paragraph reaching for a fifth line is not concise enough: cut it rather than rewrap it.

**inline-002** (MUST): each pub item (type, struct, enum, function, const, field, variant) is documented by one summary line. That is the ceiling, not a floor. When one line genuinely cannot carry it, a blank doc line follows, then at most three further lines. Comments and inline docs wrap at 80 columns. Docs on shared APIs stay protocol-agnostic, and per-protocol nuance goes on the protocol-specific items.

**inline-003** (MUST): no empty lines between enum variants or struct fields. The doc comment of each item is separator enough. Blank lines keep separating methods and other items.

```rust
//! # Access token request
//!
//! Exchanges an authorization code for an access token against the
//! token endpoint (RFC 6749 section 4.1.3).
//!
//! Consumed by the authorization code grant, next to the auth request
//! and auth response modules.

/// The parameters of the access token request.
///
/// The optional client secret rides along for providers issuing one
/// even to public clients.
pub struct ExampleRequestAccessTokenParams { /* ... */ }
```

**inline-004** (MUST): avoid in-code // comments, since code should be clear enough on its own. When a situation is genuinely non-obvious, prefix the comment with one of these five tags, and no other: NOTE (a non-obvious fact the next reader needs: constraint, invariant, spec quirk), TODO (deferred work, the code is correct meanwhile), FIXME (known-wrong or fragile, needs repair), HACK (a deliberate workaround kept on purpose), SAFETY (justification above an unsafe block, the official Rust convention enforced by clippy's undocumented_unsafe_blocks lint). A tag is not a licence to annotate: NOTE earns its place only where a competent reader of the surrounding code would otherwise get it wrong, which is rare, and the first move when one feels necessary is to make the code say it instead. Fixing a bug is not by itself such a case: the reasoning behind a fix belongs in the commit message, the changelog and the Cairn log, not beside the line that changed, where it decays into narration of a defect nobody can see any more.

**inline-005** (MUST): structural section separators (dashed // banners) are banned. When an impl block grows too big to navigate, split it into several impl blocks, each introduced by its own doc comment, or split the module into several files, or, in extreme cases, generate the repetitive parts with a macro.

**inline-006** (MUST): CLI crates document every pub item, because clap renders doc comments as the CLI help. The first paragraph (two lines max) is what -h shows. The following paragraphs complete the --help page. Such docs are the user interface rather than developer documentation, so they are the one exception to inline-002's ceiling: the summary line stays genuinely short and padding is cut as hard as anywhere else, but help a user needs to operate the command is never amputated to fit three lines. An exit-code table, a flag that can discard data, and the contract of a command a configuration names all earn their length.

**inline-007** (MUST): a commented sample configuration (config.sample.toml and friends) documents each key the way inline-002 documents a pub item: one line, then at most three more when one cannot carry it. It is read by someone setting the tool up, so the gotchas stay, a default that is not obvious first among them, while the essay around them goes.

**inline-008** (MUST): the ceilings above cut length, never reasons. Every comment answers one of two questions, and they are not worth the same: what the code does, which the code already says and which therefore goes, and why it is the way it is, which nothing else records. A bug a line prevents, an invariant two layers depend on, an RFC requirement, why the tempting simpler thing is wrong: that is compressed to the ceiling, and where it truly cannot fit, the sharpest sentence survives alone and the rest goes. Deleting a why to meet a line count is a regression even though nothing compiles differently.

## logging

**logging-001** (MUST): libraries only use debug and trace, warn exceptionally, error in really rare cases. debug marks the beginning and the end of a function or coroutine, tracking where the code goes, and is usually followed by a trace carrying the input or output data. trace covers the steps inside the execution.

**logging-002** (MUST): in a coroutine, never log at the beginning of the resume loop, since it only produces noise. Log when the state changes, at the end of match arms for example, carrying the data in a trace when applicable.

**logging-003** (MUST): messages carry no prefix, since the log crate already provides the module path. They start lowercase and take no trailing dot.

```rust
pub fn compose(&self, email: &str) -> Result<Vec<ServiceConfig>> {
    debug!("begin config compose");
    trace!("email: {email}");

    // ...

    debug!("end of config compose");
    trace!("{configs:?}");
    Ok(configs)
}
```

**logging-004** (MUST): applications additionally use info when performing an action. debug and trace remain for app-specific internals (config loading, UI) with the same rules in mind: debug at beginning and end, often followed by a trace with input or output, trace for in-process operations. warn signals something definitely wrong, not crucial, that the user can fix. Anything else stays trace. error is reserved for parallel operations that must not fail fast, parallel discovery being the canonical example: one mechanism erroring is logged while the others continue. error flags something that went wrong, never a mechanism that gracefully discovered nothing, which is trace.

## naming

**naming-001** (MUST): I/O-free libraries carry the io- prefix, and io-pim- when the domain is the PIM lowest common denominator, like io-pim-discovery. Important cross-library CLIs get their own repository. Smaller ones ship as an in-repo cli feature, off by default.

**naming-002** (MUST): files and modules are snake_case everywhere. kebab-case is banned, and #[path] attributes are banned. The mod.rs choice is content-based: a pure aggregator (only mod declarations and re-exports) lives in foo/mod.rs, and a module with code of its own is a sibling foo.rs next to the foo/ folder. Never both foo/mod.rs and foo/foo.rs.

**naming-003** (MUST): the source tree of a library mirrors how the specification itself is organised, as closely as possible. Standardized domains are structured by RFC, one module per RFC (OAuth, IMAP, SMTP). Provider APIs are structured by API version (Gmail, Microsoft Graph), scoped by domain below the version when the API reference does so (Gmail: v1 then users). Everything inside is flattened, and code reused by several modules lives at the crate root. The client module spanning the RFC modules is the canonical example.

**naming-004** (MUST): there are no re-exports at the crate root, and consumers use module-qualified paths. One blessed exception: io-imap re-exports the foreign imap-types and imap-codec crates it is built on, locking their versions and smoothing onboarding.

**naming-005** (MUST): types live next to the code that owns them, never in a types catch-all module and never behind a private mod types plus a doc-inlined pub use re-export (that flatten is retired). A type attached to a single coroutine or function, its Params, Options, Response, Error and other companions, lives in that coroutine's own file. A type used independently by several coroutines or functions gets its own public module file, named after the type or its family: one file per type by default, one file per family when the per-type split would be too granular or the API design groups them. The module name is part of the public path, exactly like io-oauth's rfc6749::state::Oauth20State (the state module holds the shared CSRF value), and no re-export hides it. A module that carries its own shared types is the sibling foo.rs next to its foo/ folder, holding those types plus the pub mod declarations. A folder whose mod.rs only aggregates stays a pure aggregator.

**naming-006** (MUST): public items follow the `<Domain><Target><Verb><Ext>` pattern, reading from the largest scope down to the narrowest (`ImapMailboxCreate`: Imap then Mailbox then Create, its error `ImapMailboxCreateError`). Domain is the library or protocol scope, version-scoped when the protocol is versioned (`Oauth20`, `Http11`), bare otherwise (`Imap`, `Smtp`). Target is what the item is about (`Mailbox`, `Client`, `Message`). Verb is only for coroutines, functions performing an action, and their direct derivates (`Create`, `List`, `Fetch`, `Send`): it comes after the target, and the target is omitted when the action applies to the whole exchange (`ImapSend`, `Http11Send`). Ext is for derivates like `Error`, `Result`, `Params`, `Options`, `Yield`, `State`, `Stream`.

**naming-007** (MUST): the Domain prefix is strict, and every pub item carries it. Two exceptions: types re-exported from a foreign crate keep their upstream names, and the shared std toolkit crates (pimalaya-stream, pimalaya-cli, pimalaya-config) are exempt, since the crate name and module path already namespace them (`pimalaya_cli::printer::StdoutPrinter`, `pimalaya_stream::stream::Stream`).

**naming-008** (MUST): pure data objects have no verb, so it is omitted (`Oauth20ClientSource`, `Oauth20AccessTokenSuccessParams`). This applies only to objects standing free of any single coroutine, typically spec-defined wire shapes shared across exchanges.

**naming-009** (MUST): companions mirror their parent's target and verb. The error of `ImapMailboxCreate` is `ImapMailboxCreateError`, never `ImapCreateMailboxError`. Data companions follow the same rule (`ImapMailboxSelectData`, `GmailMessagesListParams`): data directly related to one coroutine never drops the verb.

**naming-010** (MUST): identifiers shorten authorization to auth, but RFC wire tokens are never renamed. The authorization_pending error code keeps its spelling.

**naming-011** (MUST): std clients spanning several RFC modules live in a crate-root client module, keep the version-scoped type name (`Oauth20ClientStd`) and version-less methods. A future protocol version adds a sibling client, unified behind a version-agnostic wrapper only once one exists.

**naming-012** (MUST): log macro messages start lowercase. User-facing error messages start with a capital. Neither carries a trailing dot.

**naming-013** (MUST): the private State enum of a coroutine names each variant after the action in flight, as a present-tense verb, never after what the driver is waiting for: `Start` for the entry point, then `Send`, `Read`, `Copy`, `Rename`, `Probe`, `Scan`, `CreateTmp`, `FetchBaseline`, as io-imap does. An `Await` or `Pending` prefix is banned, since it names the driver's posture rather than the step the coroutine is at. The Display impl reads the same way, a present verb and its object (`read time`, `copy into tmp`, `rename into place`), so a log line says what the coroutine is doing.

## cli

Rules applying to the command-line interface of a product, meaning the cli module of a repository shipping a binary. They describe how a CLI meets someone who has not configured it yet, which is the same encounter in every Pimalaya product.

**cli-001** (MUST): items under the cli module carry no product prefix, since nothing there is meant to be consumed as a library and the binary already names itself. `Cli`, `Command`, `Config`, `AccountConfig`, `ConfigPathsArg` and `Transport` are the canonical names, not their `Comodoro`-prefixed or `Himalaya`-prefixed variants, and this overrides naming-006 and naming-007 for that subtree. A domain prefix survives only where a CLI spans several domains and the bare name would collide (`MailboxCommand` against `MessageCommand`).

**cli-002** (MUST): a CLI reads its configuration from a TOML document holding named accounts, and resolving one is where a command discovers it has nothing to run against. The three ways that fails each name what is missing and what to do about it: a missing configuration names the path it looked for, which is the one `-c` gave or the default location so a mistyped path shows up as itself, a missing named account lists the accounts the configuration does hold, and a missing default account names both ways of picking one.

**cli-003** (MUST): a wizard generates a configuration, it never edits one. It asks the fewest questions that produce a working account, derives the account name rather than prompting for it, and hands back a ready-to-place `[accounts.<name>]` table. Editing an account, adding a second one by hand and everything the questions do not cover belong to the file and the user's editor, against the documented config.sample.toml. A `configure` command runs the wizard on demand, and is the only entry point that skips the welcome, since it was asked for by name.

**cli-004** (MUST): writing the generated account never rewrites what a human wrote. A configuration file that does not exist yet is written whole. One that exists is appended to as plain text, never parsed and re-serialized, so its comments, its ordering and its formatting survive. Two invariants guard the append, both of them properties of the shared accounts table rather than of any one product: the account name must be free, since a second `[accounts.<name>]` table makes the whole document fail to parse and takes the working accounts down with it, and the generated account claims `default` only when no other account does, since two defaults resolve to whichever one the account map yields first.

**cli-005** (MUST): a bare invocation, with no subcommand, is what a newcomer runs first. It offers the wizard when it finds no configuration, and prints the help otherwise. Any command that needs an account raises the same offer, and that offer is a hook rather than a gate: the command carries on afterwards either way, so accepting gives it a chance to work and declining leaves it to fail on the configuration it still has not got. A bare invocation has nothing to carry on to, so a declined offer falls back to the help.

**cli-006** (MUST): interactivity is decided by the streams, never assumed. Nothing prompts when stdin is not a terminal or when `--json` is set, since a cron job cannot answer and a JSON consumer wants a failure it can read: both get the error that names the way out. A generated document goes to stdout whenever stdout is redirected, so `<binary> configure > config.toml` works, and every prompt, banner and confirmation renders on stderr so it never pollutes that document.

**cli-007** (MUST): the welcome a first run prints frames the product in a sentence, names the configuration file that is missing, says what the wizard covers and what stays hand-written, links config.sample.toml, and mentions that `configure` runs the same wizard later so declining costs nothing. The `--help` footer carries the bug tracker and the sponsoring links, through pimalaya-cli's `footer!` macro.

# Markdown

## markdown

Rules applying to every markdown file (README, CONTRIBUTING, CHANGELOG, cairn/).

**markdown-001** (MUST): never hard-wrap. Each paragraph or bullet stays on one long line, and editors soft-wrap.

**markdown-002** (MUST): no em dashes. Prefer short, separate sentences over both dashes and semicolons. Use a semicolon only when two clauses are too tightly linked to split into sentences. A colon may still introduce.

**markdown-003** (MUST): file and path references are written bare (config.sample.toml) or as a markdown link, never wrapped in backticks. Backticks are for code identifiers a user types or sets, such as flags, feature names and config keys, and those are allowed inline everywhere, including the README (see readme-002).

**markdown-004** (MUST): shell command blocks are fenced with sh, not bash nor shell.

**markdown-005** (SHOULD): stay concise yet precise, with signal-dense sentences carrying the subject, the operation, and the reason. No marketing prose, no motivation paragraphs. Prefer paragraphs over dash lists in prose. Lists are for genuinely enumerable content.

**markdown-006** (MUST): no paragraph runs longer than three lines as rendered. Longer is the signal that it is not concise enough, so it is cut or split at a real seam, never rewrapped, markdown-001 forbidding hard wrapping in the first place. Expect a trimmed file to gain lines rather than lose them: the rule buys scannability, not bytes.

## readme

**readme-001** (MUST): the README is the public documentation. It exists for users to understand what the library or application does and how to get it.

**readme-002** (MUST): the README carries no code. That means no API snippets a reader would paste into a program, no type or function signatures, and no library usage examples: that technical documentation belongs on docs.rs (autogenerated from the inline docs) and behind --help for CLIs. Backticks are not code: naming a user-facing token inline is fine and encouraged, a CLI flag (`--json`), a cargo feature (`rustls-ring`), a config key, a URL scheme, since these are things a user types or sets rather than an API to document. Shell blocks for installation and a few real command lines are expected for CLIs (see readme-012). The one place real configuration code appears is an application's provider recipes (see readme-011).

**readme-003** (MUST): the header has two flavours, picked by whether logo.svg exists at the repo root, and never mixed. The HTML flavour is only for the flagship binaries shipping a logo (himalaya, himalaya-tui, neverest). A screenshot goes right under it, and an optional caution or warning callout flags pre-stable status.

```html
<div align="center">
  <img src="./logo.svg" alt="Logo" width="128" height="128" />
  <h1><Icon> <Name></h1>
  <p><One-line tagline></p>
  <p>
    <a href="https://matrix.to/#/#pimalaya:matrix.org"><img alt="Matrix" src="https://img.shields.io/badge/chat-%23pimalaya-blue?style=flat&logo=matrix&logoColor=white"/></a>
    <a href="https://fosstodon.org/@pimalaya"><img alt="Mastodon" src="https://img.shields.io/badge/news-%40pimalaya-blue?style=flat&logo=mastodon&logoColor=white"/></a>
  </p>
</div>
```

**readme-004** (MUST): the markdown flavour is for every other repo. The docs.rs badge comes first and is skipped when no library is published on crates.io.

```markdown
# <Maybe icon> <Name> [![Documentation](https://img.shields.io/docsrs/<crate>?style=flat&logo=docs.rs&logoColor=white)](https://docs.rs/<crate>/latest/<crate>) [![Matrix](https://img.shields.io/badge/chat-%23pimalaya-blue?style=flat&logo=matrix&logoColor=white)](https://matrix.to/#/#pimalaya:matrix.org) [![Mastodon](https://img.shields.io/badge/news-%40pimalaya-blue?style=flat&logo=mastodon&logoColor=white)](https://fosstodon.org/@pimalaya)

<One-sentence description without trailing dot>
```

**readme-005** (SHOULD): for io- libraries the description may be followed by the layers block, adjusted to the layers the crate actually ships.

```markdown
This library is composed of 3 feature-gated layers:

- Low-level **I/O-free** coroutines: no_std-compatible state machines containing the whole <domain> logic, usable anywhere
- Mid-level **light client**: a standard, blocking client wrapping a stream you opened yourself
- High-level **full client**: the light client plus TCP connections and TLS negotiations handled for you
```

**readme-006** (MUST): binaries with a visible interface show it between the one-line description and the table of contents: the flagship binaries above and every GUI application. A single centered screenshot suits a single screen. An application with several screens uses a horizontally scrolling row instead. Screenshots live in a screenshots/ folder at the repository root. Libraries ship no screenshot. The scrolling row is a plain HTML table, one screenshot per column in a single row, each image given a fixed pixel width so the combined width overflows the README column and GitHub renders a horizontal scrollbar rather than wrapping onto a second line.

```html
<table><tr>
<td><img src="screenshots/first.png" width="200" alt="<what the screen shows>" /></td>
<td><img src="screenshots/second.png" width="200" alt="<what the screen shows>" /></td>
</tr></table>
```

**readme-007** (MUST): sections appear in order. Libraries: Table of contents, Features, RFC coverage, Usage, Examples, License, Social, Sponsoring. CLIs and TUIs: Table of contents, Features, RFC or API coverage (when meaningful), Installation, Configuration, Usage, License, Social, Sponsoring. AI policy and Contributing carry no section of their own: they exist only as table of contents entries linking out (see readme-013), placed where their section used to sit, AI policy before License and Contributing between Social and Sponsoring. When the tool targets named providers, the Configuration section gains one subsection per provider, each nested under Configuration in the table of contents.

**readme-008** (MUST): the Features section has one bullet per feature, two lines max each, worded for users rather than implementers. No RFC references, since they belong to the coverage section.

```markdown
- **Device authorization grant**: sign in by typing a short code on another device, for hosts without a browser.
```

The TLS support block keeps this exact shape, and the section closes with the tip callout when the crate uses cargo features.

```markdown
- Full standard, blocking client with **TLS** support:
  - [Rustls](https://crates.io/crates/rustls) with ring crypto (requires `rustls-ring` feature, enabled by default)
  - [Rustls](https://crates.io/crates/rustls) with aws crypto (requires `rustls-aws` feature)
  - [Native TLS](https://crates.io/crates/native-tls) (requires `native-tls` feature)
```

**readme-009** (MUST): the RFC or API coverage section states what the crate supports in protocol terms, each RFC (or provider API) linked, with no code identifiers, just an explanation of what is covered.

```markdown
| RFC    | What is covered                                                                             |
|--------|---------------------------------------------------------------------------------------------|
| [6749] | The OAuth 2.0 framework: authorization code grant, client credentials grant, token issuance and refresh |
| [7591] | Dynamic client registration: register a public client without any provider console          |

[6749]: https://www.rfc-editor.org/rfc/rfc6749
[7591]: https://www.rfc-editor.org/rfc/rfc7591
```

**readme-010** (MUST): for CLIs and TUIs, Installation subsections appear in order: Pre-built binary, Cargo, Nix, Sources. Configuration describes the wizard behavior when one exists, then the canonical config paths and overrides, linking to config.sample.toml for the full field reference.

```markdown
A configuration is loaded from the first valid path among:

- $XDG_CONFIG_HOME/<name>/config.toml
- $HOME/.config/<name>/config.toml
- $HOME/.<name>rc

Override the path with -c <PATH> or <NAME>_CONFIG=<PATH>. Multiple paths can be passed at once, separated by :. The first one is the base and the rest are deep-merged on top. The full field reference lives in [config.sample.toml](./config.sample.toml).
```

**readme-011** (MUST): when the tool targets named providers whose setup discovery cannot fully automate, the Configuration section gains one subsection per provider, each listed (nested under Configuration) in the table of contents. The subsection carries that provider's ready-made configuration block (endpoints, scopes, and a public client where one exists) as a fenced toml snippet, plus the caveats that setup trips over. This is the one place a README carries configuration code: the recipes render on the repository page and stay in a single source of truth, while config.sample.toml keeps only the annotated field skeleton and points here for the per-provider blocks.

**readme-012** (MUST): Usage and Examples are redirects, not manuals. Usage points to docs.rs for libraries and to --help for CLIs (which may inline a few real-world command lines). Examples points to the examples folder and to tests when they demonstrate usage.

```markdown
## Usage

The whole API is documented on [docs.rs](https://docs.rs/<crate>/latest/<crate>), including runnable snippets for every coroutine and client.

## Examples

Complete runnable programs live in [./examples](./examples); the tests also demonstrate real usage.
```

**readme-013** (MUST): the AI policy and the contributing guide live once at the org level and are never restated in a repository. The README carries them as table of contents entries only, each linking straight to the org file, so the reader sees they exist without the user guide carrying meta content. Contributing points at the repository's own CONTRIBUTING.md when it ships one (see contributing-002), at the org guide otherwise.

```markdown
- [AI policy](https://github.com/pimalaya/.github/blob/master/AI_POLICY.md)
- [License](#license)
- [Social](#social)
- [Contributing](./CONTRIBUTING.md)
- [Sponsoring](#sponsoring)
```

**readme-014** (MUST): License and Social are byte-identical across repos, and License states the dual licensing by linking both files with no further prose.

```markdown
## License

This project is licensed under either of:

- [MIT license](LICENSE-MIT)
- [Apache License, Version 2.0](LICENSE-APACHE)

## Social

- Chat on [Matrix](https://matrix.to/#/#pimalaya:matrix.org)
- News on [Mastodon](https://fosstodon.org/@pimalaya) or [RSS](https://fosstodon.org/@pimalaya.rss)
- Mail at [pimalaya.org@posteo.net](mailto:pimalaya.org@posteo.net)
```

**readme-015** (MUST): Sponsoring closes the README with the NLnet banner, the year-by-year grant list (2022 to 2023 NGI Assure, 2023 to 2024 NGI Zero Entrust, 2024 to 2026 NGI Zero Core, 2026 to 2027 NGI Zero Commons Fund), then the six donation badges (GitHub Sponsors, Ko-fi, Buy Me a Coffee, Liberapay, thanks.dev, PayPal). The block is byte-identical across repos. Copy it from an existing README rather than retyping it.

**readme-016** (MUST): the donation badges link where [.github/FUNDING.yml](./.github/FUNDING.yml) says they link, that file being the single source of truth for where the money goes. It is what GitHub renders in the repository sidebar, so a README pointing elsewhere contradicts the button next to it. An account that moves is changed there first, then propagated to every README in the same pass, since a stale badge is a dead link nobody reports.

## contributing

**contributing-001** (MUST): the standard contributing guide lives once at the org level, in [.github/CONTRIBUTING.md](./CONTRIBUTING.md). GitHub serves it as the default for every repository that does not ship its own. It covers the reading order (Pimalaya README, then the org guides, then the local docs), the Nix development environment, the layered build checks, lint, test, audit, dependency overrides, and the commit style.

**contributing-002** (MUST): a repository adds its own CONTRIBUTING.md only when something differs from the standard, and that file documents only the differences, opening with the same reading order.

```markdown
# Contributing guide

Thank you for investing your time in contributing to <Name>.

Whether you are a human or an AI agent, read these in order before touching the code:

1. the [Pimalaya README](https://github.com/pimalaya) for what the project is and how its repositories stack;
2. the [Pimalaya CONTRIBUTING](https://github.com/pimalaya/.github/blob/master/CONTRIBUTING.md) guide, which chains to the shared architecture and guidelines;
3. the inline header documentation, starting with src/lib.rs (or src/main.rs): it is the architecture document of this crate;
4. the cairn/ folder for the development history and living plans (the Cairn convention: spec/, changes/, log/).

Everything below documents only what differs from the Pimalaya standards.

## <Repo-specific section, e.g. the feature matrix to build against>
```

## changelog

**changelog-001** (MUST): the CHANGELOG uses the Keep a Changelog 1.0.0 format with SemVer, entries grouped under Added, Changed, Fixed, Removed. Each item opens with a one-line (two max) past-tense summary of the change. When more context is needed, one or more indented paragraphs follow after a blank line. Both stay concise and never verbose: the summary line is a summary, not the explanation folded into the first line.

**changelog-002** (MUST): a release section reports the net changes relative to the previous version, not a complete history log. Interior churn is folded into final-state entries, and history belongs to the cairn/ log.

```markdown
## [Unreleased]

### Added

- Added the `grant` account config field.

  Selects the OAuth 2.0 grant flow run by the auth commands; defaults to `authorization-code`, the previous implicit behavior.

### Changed

- Enabled PKCE by default with the S256 method, aligning with OAuth 2.1.
```

## cairn

**cairn-001** (MUST): the cairn/ folder is the development memory of the repository, used by AI agents and humans to track what is done during development. It follows the Cairn convention (github.com/pimalaya/cairn), which supersedes the former docs/ folder. spec/ holds the current design as one file per capability, the living truth. changes/ holds in-flight proposals, each a folder with a proposal, a task list, and a spec delta. log/ holds the dated history, one entry per landed change. A landed change is folded into spec/ and logged, so the spec always reflects current truth and nothing is lost. The activation stanza lives in AGENTS.md at the repository root, and cairn/verify.sh checks conformance.

```text
cairn/
  spec/       current design, one file per capability
  changes/    in-flight proposals, one folder each (proposal, tasks, delta)
  log/        dated history, one entry per landed change
  verify.sh   conformance checker (optional, vendored from pimalaya/cairn)
```

# Audit

## tests

**tests-001** (MUST): tests are never adjusted to fit AI-generated code. The code is adjusted to fit correct behaviour, verified against the relevant RFC or upstream spec.

More test conventions (layout, naming, coverage expectations) are added here as they settle, so a repository can be checked against them.

## security

**security-001** (MUST): when applicable, SECURITY.md carries a Supported Versions table reflecting the current version line, and a Reporting a Vulnerability section pointing at the repository's issue tracker or a private contact.

## license

**license-001** (MUST): every crate, library and application alike, is dual-licensed MIT OR Apache-2.0, with LICENSE-MIT and LICENSE-APACHE at the repository root and no per-file license headers.

**license-002** (MUST): AGPL is retired. A repository still carrying it migrates back to the dual license.
