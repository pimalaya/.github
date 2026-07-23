# AI policy

How AI is used across the Pimalaya organization, and what is expected from contributions that use it. This document is org-wide: every repository links here rather than restating it, so users, downstream packagers and contributors have a single place to check.

## Disclosure

Pimalaya projects are developed with AI assistance. This section documents how.

- **Tools**: Claude Code (Anthropic), invoked locally with a persistent project-scoped memory and a small set of repo-specific rules.
- **Used for**: Refactors, mechanical multi-file edits, boilerplate (feature gates, error enums, derive macros, trait impls), test scaffolding, doc polish, exploratory design conversations.
- **Not used for**: Engineering, critical code, git manipulation (commit, merge, rebase…), real-world tests.
- **Verification**: Every AI-assisted change is read, compiled, tested, and formatted before commit. Behavioural correctness is verified against the relevant RFC or upstream spec, not assumed from the model output. Tests are never adjusted to fit AI-generated code; the code is adjusted to fit correct behaviour.
- **Limitations**: AI models occasionally produce code that compiles and passes tests but is subtly wrong. The verification workflow catches most of this; it does not catch all of it. Bug reports are welcome and taken seriously.
- **Last reviewed**: 12/08/2026

## Contributions

AI-assisted contributions are welcome, held to the same bar as any other. Read the [contributing guide](./CONTRIBUTING.md) and the [guidelines](./GUIDELINES.md) first, then:

- Disclose the assistance in the pull request description, naming the tool and what it produced.
- Own what you submit. You are the author, and "the model wrote it" is not an answer to a review comment.
- Verify before opening: the change builds across the feature matrix, the tests pass, clippy and fmt are clean, and the behaviour is checked against the relevant RFC or upstream spec.
- Never adjust a test to fit generated code, adjust the code instead (rule tests-001 of the guidelines).
- Keep the diff scoped to one intent. Unreviewed bulk output is closed without review.

## Agents

Every repository activates the [Cairn](https://github.com/pimalaya/cairn) convention through an AGENTS.md at its root, with CLAUDE.md chaining to it. An agent working in a Pimalaya repository reads the org [contributing guide](./CONTRIBUTING.md) first, then the repository's own files, and follows Cairn: propose before non-trivial work, then fold the spec and write the log entry once the work lands.
