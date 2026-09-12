# session-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

Sessions for a web application.  A signed cookie carrying an
identifier with the data in a store, **or** an encrypted cookie
carrying the whole session with no store at all — both behind one
interface, so swapping between them is a line in a constructor rather
than a rewrite.

It is what you reach for when a person signs in on one request and has
to still be signed in on the next.

It is a port of [tower-sessions](https://github.com/maxcountryman/tower-sessions)
and Flask's `flask.session`.  Six modules, and a reader should know
which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **middleware** | `sessmw` | you are mounting the session layer |
| the **session** | `sess` | you are in a handler, reading or writing one |
| the **store** | `sessstore` | you are choosing where sessions live, or writing a store |
| the **cookie** | `sesscookie` | you are deciding attributes, or rotating a key |
| the **identifier** | `sessid` | you are minting one, or checking one that arrived |
| the **faults** | `sesserr` | you are deciding what to answer |

## Adding it, and checking it

```bash
novo pkg add session-nv          # into your novo.toml
novo pkg build                   # type- and effect-check the package
novo test --isolate tests/sess_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: session-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

The four lines a route wrapper is:

```novo
use civil
use sess
use sessid
use sessmw
use sessstore

fn handle(req: HttpRequest, s: Sess) -> (HttpResponse, Sess)
    let seen = sess.get_or(s, "visits", "0")
    (http.server.text(200, "seen ${seen} times\n"),
     sess.set(s, "visits", "${str.to_int(seen) ?? 0 + 1}"))

fn serve_one(sessions: SessFiles, o: SessOptions,
             req: HttpRequest) -> HttpResponse [io, fs, rand]
    let id = sessid.new() !
    let now = a_clock_reading()
    let loaded = sessmw.load(sessions, o, cookie_header_of(req), id, now) !
    let (resp, after) = handle(req, sessmw.session_of(loaded))
    let out = sessmw.save(sessions, o, after, now) !
    if sessmw.has_cookie(out)
        http.server.with_header(resp, sessmw.set_cookie_header(), out.set_cookie)
    else
        resp
```

Load, call, save, attach.  The handler's signature says it takes a
session, which is the whole reason this is two functions rather than a
wrapper — see below.

And at the sign-in:

```novo
let out = sessmw.sign_in(sessions, o, signed_in_session, sessid.new() !, now) !
```

## The load-bearing interface

`sessstore.SessStore[e]` — the store trait, with an **effect
parameter**.

```novo norun:pseudo
pub trait SessStore[e]
    fn sess_get(self, id: SessId) -> Result<?SessRecord, SessFault> [e]
    fn sess_put(self, r: SessRecord) -> ?SessFault [e]
    fn sess_drop(self, id: SessId) -> ?SessFault [e]
    fn sess_touch(self, id: SessId, expires_at: CivilDateTime) -> ?SessFault [e]
    fn sess_sweep(self, now: CivilDateTime) -> Result<Int, SessFault> [e]
```

The `[e]` is the argument.

A session store is the one part of a web application whose cost
genuinely differs by deployment.  An in-memory store mutates a cell:
`[mutate]`.  A file store reads and writes a directory: `[fs]`.  A
store over a database opens a socket: `[net]`.  All three are
legitimate, and a single program uses more than one — memory in its
tests, files on a developer's machine, a database in production.

Without the effect parameter this package would have to pick.  It could
declare the union, `[fs, net, mutate]`, and then every application that
mounts the memory store in a test declares `[fs]` and `[net]` for a
store that touches neither — which turns the effect row from a
description into a ceiling, and a ceiling tells a reviewer nothing.  Or
it could ship three unrelated interfaces and make swapping a rewrite.

With it, the middleware binds the parameter:

```novo norun:pseudo
pub fn load<S: SessStore[e]>(sessions: S, o: SessOptions, cookie_header: Str,
                             new_id: SessId, now: CivilDateTime)
    -> Result<SessLoaded, SessFault> [e]
```

and is charged what the caller's store supplies — `[mutate]` against
`SessMemory`, `[fs]` against `SessFiles`, `[net]` against a store
somebody wrote over their own client.  The application's declared row
then says what its session backend *actually does*.

The tests hold both halves of that claim: `sessstore_tests.nv` has one
test declaring `[io, mutate]` and another declaring `[io, fs]`, calling
the same five trait methods.  The compiler is the thing asserting it.

### Why the middleware is two functions and not a wrapper

The obvious shape is `fn(handler) -> handler`, which is what the
standard library's `http.server.wrap` does and what `mw_access_log`
is.  It does not work here: a session middleware has to **give the
handler the session** and take it back afterwards, and a
`fn(HttpRequest) -> HttpResponse` has nowhere to put one.

Three shapes were weighed.  Threading it through a request field means
editing the standard library's `HttpRequest`.  Holding it in a cell
keyed by a request identifier means the session layer owns mutable
global state with a lifetime nobody can see.  Two functions and an
explicit value means the handler's signature says it takes a session —
which is also the only one of the three a reader can follow.

## The layer, and why

`host`.  A store touches the machine, so the package is not `core`.

Three of the six modules are `[]` throughout — `sesserr`, `sess` and
`sesscookie` — and one, `sessid`, performs only when it **mints**.
That split is why "does rotation carry the data?", "does a read dirty
the session?", "is this cookie past the browser's ceiling?" and "which
timeout ended it?" are all tests that run in a millisecond with nothing
installed.

The one row that surprises a reader is `[fs, rand]` on `sessid.new`.
The `[fs]` is the operating system's entropy device: the standard
library's `rand` module is xoshiro256\*\*, which is a fine shuffle and
a terrible session identifier, so minting goes through rand-nv's
`rng.os_bytes` and reading a device is `[fs]`.  `sessid.of_bytes` is
the `[]` twin for a caller who brought their own bits — and it is what
every test here uses.

## Five things this package refuses to let happen quietly

**Session fixation.**  `sessmw.sign_in` rotates the identifier and
deletes the old record, in one call, because the two halves separated
are two halves somebody forgets one of.  Without rotation, an attacker
who plants a known identifier in somebody's browser *before* they sign
in holds an authenticated session *after* they do.  It is thirty years
old and it survives because the fix is a step every framework makes
optional.

**A store write on every page view.**  `Sess` carries a `SessChange`,
every mutator sets it, and `sessmw.save` writes only when
`sess.needs_save` says to — which is false for the overwhelming
majority of requests.  A middleware that saved unconditionally would
put a store write behind every page.

**A cookie the browser silently discards.**  RFC 6265 § 6.1 asks for at
least 4096 bytes per cookie and every browser treats it as a maximum,
dropping a larger one with **no error anywhere**.  The next request
arrives with no session, which from the application's side is
indistinguishable from the person logging out — and it happens on the
request that added the item that tipped it over.
`sesscookie.write_client` answers a `Result`, and
`sesscookie.client_bytes` is the measurement to take *before* writing.

**A store that answers questions an attacker is asking.**  `sess_get`
answers `?SessRecord`, and `None` covers three cases: never issued,
expired, deleted.  Telling them apart would make the store an oracle —
one bit per guess against a space this package exists to keep
unguessable.  `sess_drop` is silent about whether there was anything to
drop, for the same reason.

**A session kept alive forever by a stolen cookie.**  `SessPolicy`
carries **two** timeouts and `sess.is_expired` checks both.  A
deployment with only an idle timeout has sessions that an automated
loop refreshes indefinitely — which is exactly the session an attacker
has.  `sess.expiry_reason` says which one ended it, because the two
have different fixes and one message for both tells a person nothing.

## The store you probably want is not in here

`SessMemory` and `SessFiles` ship.  A Redis- or database-backed store
does not, and that is deliberate: depending on a Redis client here
would make every application with a file store compile one it never
calls.

What ships instead is `sessstore.SessRemote`, the **shape** such a
store takes — the key prefix, whether the server expires rows itself,
whether `touch` is one command — plus `remote_key` and `remote_ttl`.
A deployment writes about twenty lines implementing `SessStore[net]`
over its own client, the `[net]` lands on the `impl`, and the
middleware is charged it through `[e]` with nothing in this package
changing.

## Its relation to the registry's own sign-in

novo-lang.org's registry authenticates publishes with a **bearer
token**, hashed and compared against a table — not a session.  That is
the right shape for the publishing path: a token is presented by a
command-line tool that has no browser, no cookie jar, and no redirect
to come back from, and a session cookie would be all three of those
problems for nothing.

Where this package would fit is the half that has a browser: a person
signing in on the registry's web pages to manage their own packages.
That flow has a redirect, a cookie jar, and a privilege change at the
moment of sign-in — which is `sessmw.sign_in` and the reason it
rotates.

## What it depends on

| | |
| --- | --- |
| cookie-nv | the whole cookie half — including the five attribute combinations a browser silently ignores, which a session cookie is exactly the wrong place to get wrong |
| calendar-nv | what an expiry is; both timeouts are arithmetic on times the caller passes in |
| crypto-nv | one function, `digest.ct_eq`: the identifier comparison, which is against a value an attacker chose |
| uuid-nv | the default identifier — a version 4, 122 unpredictable bits |
| rand-nv | 256 bits, for the deployment that wants more than a UUID has |

## Related

- [cookie-nv](https://github.com/novolang/cookie-nv) — the layer under
  `sesscookie`
- [oauth2-nv](https://github.com/novolang/oauth2-nv) — what puts a
  person in a session in the first place; its `OauthPending` is
  something to keep in one
- [router-nv](https://github.com/novolang/router-nv) — the other half
  of a request's front door
