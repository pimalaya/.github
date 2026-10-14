# Maintaining Pimalaya

How issues and pull requests are triaged, and the direction decisions they are measured against. Written for maintainers, contributors and the Pimalaya assistant alike. [ARCHITECTURE.md](./ARCHITECTURE.md) explains how the code is built and [GUIDELINES.md](./GUIDELINES.md) how it is written. This document says what Pimalaya accepts, and why.

> This is a living draft. When a decision is taken on an issue that this document does not cover yet, it is added here.

## How to read this document

Each rule has a stable id (`scope-nnn`) so a reply can cite it. A rule states the decision, then the reason. The reason matters more than the rule: a case the rule does not foresee is decided by its reason.

## Table of contents

- [triage](#triage): how an issue becomes a change
- [scope](#scope): what belongs where
- [retired](#retired): what is no longer extended
- [compat](#compat): what is a contract
- [assistant](#assistant): what the Pimalaya assistant may do

## triage

**triage-001**: a non-trivial change starts as an issue, not a pull request. The maintainer answers with a decision: accepted with a scope, or closed as not planned with the reasons. A pull request opened without that agreement may be closed without review. Reason: a design argued in code is expensive for both sides, and the scarce resource is the maintainer's review time.

**triage-002**: an accepted issue states its scope, and the pull request implements that scope, nothing more. Going further (a broader design, an extra flag, a compatibility change) goes back to the issue first. Reason: #788 rebuilt the whole MIME tree where #780 agreed to re-attach the attachments, and had to be redone.

**triage-003**: a reporter offering to write the patch has priority on it. Nobody else, the assistant included, opens a pull request for that issue unless the reporter steps back or stays silent for a month.

**triage-004**: trivial changes need no issue: typos, documentation fixes, an obvious bug with a test showing it, a new entry in a README list of interfaces.

**triage-005**: every non-trivial change ships its Cairn artifacts (proposal, tasks, delta, spec fold, log entry) and a CHANGELOG entry, as described in each repository's AGENTS.md. AI-assisted contributions follow [AI_POLICY.md](./AI_POLICY.md), disclosure included.

**triage-006**: an issue is closed with a reason the reporter can act on: the rule it conflicts with (cited by id), the issue it duplicates, or the commit fixing it. "Not planned" never comes alone.

**triage-007**: bug reports need the version, the backend, the relevant configuration (secrets removed) and the log at `--log-level trace` (or `RUST_LOG=trace`) of the failing command. Asking for them is the first reply when they are missing.

## scope

**scope-001**: a shared API (himalaya's `mailbox`, `envelope`, `flag`, `message`, `attachment`; cardamum's and calendula's equivalents) exposes only what every backend it targets supports natively. A request to widen it for some backends is redirected to the protocol-specific commands. Reason: a shared command behaving differently per backend is a trap for every frontend built on it (#749 shared drafts, #790 custom keywords).

**scope-002**: protocol-specific commands (`imap`, `jmap`, `gmail`, `msgraph`, `maildir`, `pimdir`, …) aim at the full surface of their backend. Exposing a documented operation is welcome, all the more when the io-* library already implements it (#787). Commands mirroring a vendor REST API keep the vendor's resource names.

**scope-003**: protocol logic lives in the protocol library (io-imap, io-jmap, io-msgraph, …), not in an application. A fix belonging to a protocol is made in its library, then picked up by the application. Shared code never moves sideways from one application into another: it goes down into a library, once a second consumer exists.

**scope-004**: a library stays I/O-free and `no_std` at its core ([ARCHITECTURE.md](./ARCHITECTURE.md) section 3). A request adding I/O, a runtime or a std-only dependency to a coroutine core is declined.

**scope-005**: himalaya composes simple messages (`message compose`, `reply`, `forward`). Rich composition (templates, MML, editors) belongs to dedicated tools such as [mml](https://github.com/pimalaya/mml), piped into `message send` or `message add`.

**scope-006**: account-level behaviour belongs to the application's account configuration, not to its frontends (himalaya-vim, himalaya-emacs, himalaya-tui). Example: saving a copy of a sent message is `message.send.save-copy`, because only the CLI knows whether the backend files sent mail itself.

**scope-007**: secrets are resolved by command (`pass`, `secret-tool`, `security`, `cmdkey`, `keyctl`, …). Pimalaya ships no keyring of its own. Requests for one are pointed at the command that fits the platform.

## retired

**retired-001**: the aggregator crates io-email, io-addressbook and io-calendar are frozen: no new backend, no new feature. Applications own their dispatcher over the protocol crates ([INTEGRATING.md](./INTEGRATING.md) section 6).

**retired-002**: the following crates are retired and never extended or added as a dependency: io-process, io-stream (use pimalaya-stream), io-keyring and mimosa (see scope-007), io-socket, io-hook, io-notify, io-time, io-fs, pimconf (use io-pim-discovery).

**retired-003**: an issue or pull request against a frozen or retired crate is answered with the replacement, then closed.

## compat

**compat-001**: the contract of a CLI is its official command and option names and its `--json` output. Aliases are a convenience and may be removed or repurposed without a major version (a CHANGELOG "Removed" entry is enough). Scripts use official names.

**compat-002**: `--json` output uses camelCase keys, the same on every backend for shared commands.

**compat-003**: shared commands address messages by their stable id (the IMAP UID on IMAP). Protocol-only addressing, such as IMAP sequence numbers, stays in protocol-specific commands.

**compat-004**: versions, releases and publishing are done by the maintainer. A pull request never bumps a version in Cargo.toml.

**compat-005**: every crate is dual-licensed MIT OR Apache-2.0. A contribution under any other license is not accepted.

## assistant

The Pimalaya assistant is an automated account that offers help on issues and pull requests. It speaks as itself and never on behalf of the maintainers.

**assistant-001**: the assistant only comments when it has something concrete: a likely fix to a configuration, a duplicate or an earlier answer, missing information (triage-007), a pointer to documentation, or a rule of this document. Otherwise it stays silent. One comment per issue, unless someone replies to it or mentions it.

**assistant-002**: the assistant suggests, it never decides. It may cite a rule of this document by id, but it never accepts or declines a feature, promises a release, or speaks for the maintainers' preferences. A question of direction not settled here is left to the maintainers.

**assistant-003**: the assistant may open a draft pull request only for a bug with a clear reproduction and a small fix, when triage-003 does not apply, or for an issue a maintainer accepted with a scope. The pull request carries a test failing without the fix, follows triage-005, and is announced by one comment on the issue.

**assistant-004**: the assistant never merges, closes, labels, approves, or pushes to a default branch. Its reviews on pull requests are comments, never approvals or change requests.

**assistant-005**: every message of the assistant reads as automated and ends with a short note saying so, and that a maintainer will follow up.

**assistant-006**: the assistant never demands information, it suggests sharing it and says why: "sharing the output of `himalaya --version` would help narrow it down", not "please post the output of `himalaya --version`". Reason: the person owes the project nothing, and an automated request reads as a gate when it should read as help.
