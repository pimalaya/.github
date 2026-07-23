# Integrating a Pimalaya library

This document is for developers building on the Pimalaya protocol libraries, whether inside the organization or outside it. It answers one question: given your runtime and given who owns your transport, which layer of a protocol crate do you consume, and what is left for you to write.

Read the [Pimalaya README](https://github.com/pimalaya) for what exists, then [ARCHITECTURE.md](./ARCHITECTURE.md) for the sans-I/O approach and the three-layer crate shape, then this document. [GUIDELINES.md](./GUIDELINES.md) covers naming and documentation and matters once you contribute back rather than consume.

The running example is io-imap, the crate where the client layer is most complete. io-smtp and the io-http family share the shape; where they differ, the difference is called out.

> This is a living draft. If it contradicts the code, the code wins; please flag the discrepancy.

## Table of contents

- [1. Which client do you want](#1-which-client-do-you-want)
- [2. Invariant versus opinionated coroutines](#2-invariant-versus-opinionated-coroutines)
- [3. The session coroutine](#3-the-session-coroutine)
- [4. The client traits](#4-the-client-traits)
- [5. What is not a public contract](#5-what-is-not-a-public-contract)
- [6. Aggregation belongs to you](#6-aggregation-belongs-to-you)

## 1. Which client do you want

The decision is three-way, and the axes are your runtime and the ownership of your transport. Pick the case you are in before reading anything else.

### 1.1 std and blocking, opening your own sockets

This is the common case: an ordinary blocking program that wants an IMAP connection and does not care how the socket is made. Use the full client and reuse its connect, which is one call covering transport selection, the optional STARTTLS upgrade, the greeting, PREAUTH detection and the SASL exchange.

```rust
use io_imap::{client::ImapClientStd, session::ImapSessionOpenOptions};
use pimalaya_stream::{sasl::SaslPlain, tls::Tls};

let sasl = SaslPlain { authzid: None, authcid, passwd: passwd.into() };
let opts = ImapSessionOpenOptions::default();
let (mut client, capabilities) = ImapClientStd::connect(&url, &Tls::default(), Some(sasl), opts)?;
```

`ImapClientStd::connect` lives behind the TLS cargo features: `rustls-ring` (enabled by default), `rustls-aws` or `native-tls`. Each of them implies `client` and pulls pimalaya-stream with the matching provider, so `cargo add io-imap` is already this case. io-smtp is the same shape through `SmtpClientStd::connect`, which additionally takes the EHLO domain and a `starttls` flag, since SMTP carries its identity in every handshake.

Nothing in this case is worth reimplementing: the URL scheme table, the port defaults, the proxy resolution and the TLS upgrade are all behind that one call.

### 1.2 std and blocking, but the transport is yours

This is the case people miss, and it is the one that matters most for anything unusual. You are blocking and on std, but the bytes do not come from a socket you are allowed to open: a JNI upcall bridge handing you a stream owned by the host application, a pre-authenticated socket proxy that already performed the handshake, a TLS stack your product mandates, an in-memory double in a test. You do not need your own client, and you do not need the coroutines directly. You need the light client.

The light client is the `client` cargo feature on its own, with no TLS backend:

```sh
cargo add io-imap --no-default-features --features client
```

That feature gates std and nothing else: it pulls in no TLS provider, no URL parser and no sockets. You then wrap whatever you have in the `ImapStream` trait and hand it over:

```rust
use io_imap::client::{ImapClient, ImapClientStd, ImapStream};

impl ImapStream for MyTransport {
    fn as_any_mut(&mut self) -> &mut dyn Any {
        self
    }

    fn set_read_timeout(&self, _timeout: Option<Duration>) -> io::Result<()> {
        Ok(())
    }
}

let mut client = ImapClientStd::new(MyTransport::open()?);
let capabilities = client.capability()?;
```

`ImapStream` is `Read + Write + Send + Any`, plus the two methods above. `as_any_mut` exists so a caller can downcast back to its concrete handle when it needs something type-specific. `set_read_timeout` bounds a blocking read, which is what lets the mailbox watch worker wake up periodically to check its shutdown flag during a silent IDLE; a transport that cannot honor a timeout returns `Ok(())` and manages its own read semantics. The command methods come from the `ImapClient` trait, so it has to be in scope. `ImapClientStd::set_stream` swaps the stream in place, which is what a reconnection or a hand-driven STARTTLS upgrade needs.

sirup and limier live in this case: sirup hands the client a pre-authenticated unix socket whose greeting is PREAUTH, limier hands it a bridge that crosses JNI on every read. Neither ships a client of its own, and neither should.

The same applies to io-smtp through `SmtpClientStd::new` and its `SmtpStream` trait, which carries a blanket implementation for any `Read + Write + Send + Any`, so wrapping is usually nothing more than passing the stream.

### 1.3 Any other runtime

Async, an embedded executor, a fiber runtime, anything that is not blocking std: implement the client trait over your own pump, and answer the session coroutine's transport requests with your own sockets. There is no tokio client to depend on, and after the session coroutine there is nothing left for one to contain: your runtime supplies the socket, `ImapSessionOpen` supplies the handshake, the trait supplies the commands.

```rust
use io_imap::{
    client::{ImapClient, ImapClientError},
    coroutine::{ImapCoroutine, ImapCoroutineState, ImapYield},
};

impl ImapClient for MyClient {
    fn run<C, T, E>(&mut self, mut coroutine: C) -> Result<T, ImapClientError>
    where
        C: ImapCoroutine<Yield = ImapYield, Return = Result<T, E>>,
        ImapClientError: From<E>,
    {
        // resume, answer WantsRead and WantsWrite against your transport,
        // return on Complete
    }
}
```

Implement that one method and the whole command surface follows. `ImapClientAsync` is the same deal for a transport that returns futures. Failures your transport raises that are not `std::io::Error` ride in `ImapClientError::Transport`, which boxes any `Error + Send + Sync`.

Both traits live in the client module, so this case still enables the `client` feature. That feature buys std, not I/O: you are not pulling a socket implementation, a TLS provider or a URL parser you are not going to use.

The examples folder of io-imap holds the runnable versions: examples/std_coroutine.rs and examples/tokio_coroutine.rs pump a coroutine by hand on each runtime, examples/std_client_light.rs implements `ImapStream` over a caller-owned rustls stream, examples/std_client_full.rs is the one-call connect. For the coroutines the traits deliberately do not cover (see the next section), the wirings inside `ImapClientStd` are the reference to copy: `watch_mailbox`, `fetch_body_stream`, `fetch_bodies_stream` and `append_stream`.

## 2. Invariant versus opinionated coroutines

The line that makes the client surface predictable runs through the coroutines themselves, and you can read it off a type.

A coroutine yielding the standard `ImapYield` is invariant. Its only requests are read these bytes and write those bytes, every client answers them identically, and there is exactly one sensible wrapper. Such a coroutine is a defaulted method on the client trait, and you inherit it.

A coroutine declaring its own yield enum is opinionated. It asks for something a runtime answers in its own way, so implementations are expected to diverge, and a consumer's version being different from Pimalaya's is not a defect. Such a coroutine is not on the trait, and wiring it by hand is the intended usage rather than a workaround.

io-imap has exactly five opinionated coroutines: `ImapMailboxWatch` and `ImapIdle`, which add an `Event` variant, and the streamed APPEND and the two streamed FETCHes (`ImapMessageAppendStream`, `ImapMessageFetchStream`, `ImapMessageFetchStreamBatch`), which add `WantsStream` and `BodyChunk` variants so a message body moves straight between the socket and your storage without landing in memory whole. They map one to one onto the five places `ImapClientStd` makes a runtime-flavoured choice: a thread plus a bounded channel plus a read-timeout shutdown poll for the watch, `impl Read` as the source for the streamed APPEND, `impl Write` sinks for the streamed FETCHes. On tokio, a cancellation token and a `select!` are the better arrangement, and the standard one would be in your way.

The rule is compiler-enforced rather than policed in review. The trait's `run` is bounded on `Yield = ImapYield`, so an opinionated coroutine cannot be defaulted even by accident, and a new coroutine's yield type already answers whether it belongs in the client.

## 3. The session coroutine

The handshake is where protocol knowledge accumulates, so it does not live in a client. `ImapSessionOpen`, in the session module of the no_std core, covers everything between a bare address and an authenticated session, and it yields transport requests alongside the usual reads and writes:

```rust
ImapSessionOpenYield::WantsTcpConnect { host, port }
ImapSessionOpenYield::WantsTlsConnect { host, port }
ImapSessionOpenYield::WantsUnixConnect(path)
ImapSessionOpenYield::WantsTlsUpgrade
ImapSessionOpenYield::WantsRead
ImapSessionOpenYield::WantsWrite(bytes)
```

The consequence is worth stating plainly: you receive a checklist, not an instruction to bring a ready socket. The coroutine tells you which socket to open and when to upgrade it, and you answer with whatever your runtime has. Ordering is enforced by the state machine rather than by you remembering that the greeting precedes STARTTLS and that CAPABILITY must be re-issued afterwards, because a caller who skips a step is never asked for the next one.

You build it from an `ImapSessionTransport` (`Tcp`, `Tls` or `Unix`, with `ImapSessionTransport::from_url` behind the `url` feature reading the imap, imaps and unix schemes), an optional SASL mechanism, and `ImapSessionOpenOptions`. It completes with `ImapSessionOpenData`, carrying the capabilities observed in the session's final state and whether the greeting was PREAUTH.

Everything it knows, you inherit for free by pumping it:

- PREAUTH detection, so a pre-authenticated proxy socket skips the SASL step instead of authenticating twice against a server that has no credentials to check.
- The RFC 4959 SASL-IR policy, which follows the advertised `SASL-IR` capability by default, with `ImapSessionOpenOptions::sasl_ir` overriding it in both directions for a server that advertises it falsely (Coremail, on 126.com and 163.com, which no capability inspection can predict).
- `auto_id`, chaining an RFC 2971 ID round-trip right after authentication, which mail.qq.com and Fastmail require.
- The STARTTLS trailing-byte refusal. RFC 3501 section 6.2.1 forbids bytes past the tagged STARTTLS response, so their presence means someone injected plaintext commands the server would replay inside the TLS session. The coroutine fails with `StartTlsInjection` rather than performing the upgrade, and it re-reads the capability list over TLS instead of carrying the pre-upgrade one across.

`ImapClientStd::connect` is a pump over this coroutine answering the transport requests with pimalaya-stream, and it is around thirty lines. Yours will be the same length. io-smtp exposes the twin `SmtpSessionOpen`: similar silhouette, no shared code, because SMTP is greeting, EHLO, STARTTLS, upgrade, EHLO again with the domain riding along, while IMAP is STARTTLS inline, upgrade, then CAPABILITY, with PREAUTH able to skip authentication entirely.

## 4. The client traits

io-imap ships two traits, `ImapClient` for blocking transports and `ImapClientAsync` for transports that return futures. Each has exactly one required method, `run`, and close to forty defaulted commands generated from a single list, so the two surfaces cannot drift apart.

```rust
fn run<C, T, E>(&mut self, coroutine: C) -> Result<T, ImapClientError>
where
    C: ImapCoroutine<Yield = ImapYield, Return = Result<T, E>>,
    ImapClientError: From<E>;
```

The error type is fixed to the crate's own `ImapClientError` rather than left associated. An associated error would need `From` bounds for all forty coroutine error types at every call site, which is unusable; wrap `ImapClientError` in your own error instead.

The `Send` treatment differs between the two traits, and the asymmetry is deliberate. The async trait declares its return type explicitly as `impl Future<Output = ...> + Send`, with `Send` as a supertrait so `&mut Self` carries through. A plain `async fn` in a trait cannot express that the future it returns is `Send`, so anything built from the default bodies would fail to compile under `tokio::spawn`, which is the first thing a worker-spawning consumer reaches for. The blocking trait carries no such bound: a blocking call hands back a value rather than a future, there are no auto-traits to pin down, and requiring `Send` would exclude a perfectly good thread-affine client such as a JNI bridge.

Neither trait is dyn-compatible, because `run` is generic, and that is a decision rather than an oversight. The dynamism the crate actually needs sits one layer down, at `Box<dyn ImapStream>`, which already spans TCP, TLS, unix sockets, a proxy socket and a foreign bridge behind a single concrete client type. A multi-backend application gets no help from a boxed client either, since IMAP and JMAP are different traits and the dispatch has to happen in the product's own backend layer. For the same reason the defaults carry no `where Self: Sized`: that bound exists only to keep a trait dyn-compatible despite a non-dyn-compatible method, and it was never what allowed custom commands anyway, since `run` already accepts any coroutine you write on the standard yield and `ImapRaw` covers arbitrary command bytes.

## 5. What is not a public contract

Two crates on crates.io are shared internals of Pimalaya's own products, not libraries to build on: pimalaya-cli (argument parsing, printers, prompts, spinner, tables, wizards) and pimalaya-config (TOML loading and secret resolution). Their APIs follow the needs of himalaya, neverest, cardamum, calendula and friends, they change without notice, and nothing about them is designed for a foreign consumer. Treat them as private. This is repeated here because it lives in their own documentation, and nobody reads a lib.rs header before running cargo add.

pimalaya-stream is the opposite. It is public and supported, and reusing it is the intended path for anyone wiring a client against io-imap, io-smtp, io-http or a sibling. It opens and upgrades blocking streams (TCP, TLS, unix sockets, SOCKS5 and HTTP CONNECT proxy resolution, the plain-to-TLS upgrade that STARTTLS needs) and carries the `Tls` and `Sasl` configuration vocabulary the protocol crates share. It stays on 0.x, which means breaking changes land in minor bumps and never in patch bumps, so pin a minor and read its changelog before upgrading. Reimplementing TCP, TLS, proxy resolution and the STARTTLS upgrade because a crate looked internal is the wrong move, and the result is worse than what is already there.

## 6. Aggregation belongs to you

The domain aggregator crates, io-email, io-addressbook and io-calendar, are frozen. They take no new backends and no new features. Import the protocol crates you need and own the layer above them.

The reason is structural rather than circumstantial. An aggregator is a matrix of backends against domains, so each new provider fills an entire column even where the cells are convert-only wrappers, and the cost grows multiplicatively. Worse, its shared API is a least common denominator, which is a ratchet: every backend added shrinks the intersection and ejects a partial-coverage feature that used to work. Value decreases monotonically while maintenance increases. Watch and search were simply the first domains where the intersection hit zero, since there is no IDLE for Graph or Gmail, and IMAP SEARCH, JMAP Email/query and Graph $search have incompatible semantics.

Underneath that, the layer had no owner. Interfaces want protocol-specific richness and each one wants a slightly different slice of it; sync engines want a very small verb interface they define themselves. An ownerless layer between the two serves the empty intersection of their needs.

So the abstraction goes to the consumer that defines its requirements. himalaya, cardamum and calendula each own their dispatcher over the protocol crates they chose, and that arrangement works: aggregation in the product, protocols in the libraries. A product-owned core is still a least common denominator across its backends, but it escapes the ratchet through two valves worth keeping: the trade-offs are product decisions with a single owner, and the split between shared commands and protocol-specific commands gives partial-coverage features somewhere to live instead of being ejected. Where two consumers end up with byte-identical conversion code, extract a small focused crate for that one conversion, once the second call site exists.

---

Questions, corrections and proposals are welcome: open an issue on the relevant repository or reach the project via [Matrix](https://matrix.to/#/#pimalaya:matrix.org).
