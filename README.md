# session-nv

A **session** is the state a web application keeps about one visitor
across requests. The browser carries an HTTP cookie and the server
matches it to that state. This package provides both of the ways that
is done: a signed cookie carrying an identifier with the data in a
**store**, and an encrypted cookie carrying the whole session with no
store at all. Both are behind one interface. It is a port of
[tower-sessions](https://github.com/maxcountryman/tower-sessions) and
of Flask's `flask.session`, and it is built on
[cookie-nv](https://novo-lang.org/packages/cookie-nv).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What a session is

HTTP is stateless: each request stands alone. A **session identifier**
is an unpredictable value the server mints once and the browser returns
on every later request, in a cookie. The server looks the identifier up
and finds what it knows about that visitor.

A **store** is where that state lives: a table in memory, a directory
of files, a row in a database. This package ships the first two and
describes the shape of the third.

A **client-side session** skips the store. The whole session is
encrypted into the cookie itself, so the server keeps nothing. Two
consequences follow: the session cannot be revoked short of rotating
the key, and it must fit in a cookie.

A **flash message** is a value written on one request and read on the
next, once. It is how a redirect carries "your changes were saved".

Two timeouts end a session. The **idle timeout** counts from the last
request. The **absolute timeout** counts from when the session was
created, however active it has been.

**Session fixation** is the attack in which somebody plants a known
identifier in a victim's browser before they sign in, so that the
attacker holds the identifier of an authenticated session afterwards.
The defence is to mint a new identifier at the moment of sign-in and
delete the old record.

The current time arrives as an argument. Nothing in this package reads
a clock.

## Install

```
novo pkg add session-nv
```

## Example

```novo
use civil
use sess
use sessid
use uuid

fn main() [io]
    // The current time, read by the caller. Nothing here reads a clock.
    let now = civil.datetime(civil.date_from_epoch_day(20708), civil.midnight())

    // A session that has just been created, under a new identifier.
    let s = sess.fresh(sessid.of_uuid(uuid.nil()), now)

    // Reading does not mark the session as needing a write.
    println("visits so far: ${sess.get_or(s, "visits", "0")}")

    // Writing does. `needs_save` is what a middleware asks before it
    // touches the store.
    let after = sess.set(s, "visits", "1")
    println("write it back: ${sess.needs_save(after, sess.defaults())}")

    // Idle and absolute timeouts, both checked, and the one that ended
    // the session named.
    println("expired: ${sess.is_expired(after, sess.defaults(), now)}")
    println("reason: ${sess.expiry_reason(after, sess.defaults(), now)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: session-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `sesserr` | Every fault, the status it should produce, and whether it is safe to report. |
| `sessid` | The identifier: minting one, reading one that arrived, and comparing two. |
| `sess` | The session as a value: the data, the flashes, the change tracking and the two timeouts. |
| `sessstore` | The store trait, a memory store, a file store, and the shape of a remote one. |
| `sesscookie` | The cookie: its attributes, reading an identifier or a whole session out of one, and writing one back. |
| `sessmw` | The four calls a request makes: load, save, sign in and sign out. |

## How to choose an entry point

**A request calls `sessmw.load`, then the handler, then
`sessmw.save`.** `load` reads the cookie, fetches the record and
answers the session. `save` writes it back if it changed and answers
the `Set-Cookie` header to attach, if any.

**`sessmw.sign_in` is called at the moment a person's privileges
change.** It rotates the identifier and deletes the old record in one
call. **`sessmw.sign_out` destroys the session** and writes the cookie
that removes it.

**`sessmw.sweep` removes expired records.** A store whose backend
expires rows itself does not need it.

**`sesscookie.defaults` is the identifier cookie** and
`sesscookie.client_side` is the encrypted whole-session cookie.
Changing between them is one call.

**`sessstore.memory` is for tests, `sessstore.files` for a single
machine.** For anything else, implement `SessStore[e]` over your own
client. See "Writing your own store".

## The rules a user needs

1. **The middleware is two calls, not a wrapper.** A session
   middleware has to give the handler the session and take it back, and
   a function from a request to a response has nowhere to put one. The
   handler's signature therefore says it takes a session.
2. **Rotate the identifier when privileges change.**
   `sessmw.sign_in` does it, together with deleting the old record,
   because the two halves separated are two halves somebody forgets one
   of. Without rotation a planted identifier survives the sign-in.
3. **`sesscookie.defaults()` names the cookie `__Host-sid`.** The
   `__Host-` prefix is the only host-locking cookies have: a browser
   refuses such a cookie if it carries a `Domain` or a `Path` other
   than `/`. Without it, anything that can run script on one subdomain
   can set a session cookie for another.
4. **A read does not mark a session dirty and a write does.**
   `sess.needs_save` is false for the great majority of requests, so a
   store write does not sit behind every page view. Set
   `SessPolicy.save_unchanged` when an idle timeout must be refreshed
   on every request.
5. **A client-side session must fit in a cookie.** RFC 6265 section
   6.1 asks for at least 4096 bytes per cookie and browsers treat that
   as a maximum, dropping a larger one with no error anywhere. The next
   request then arrives with no session, which from the server's side
   looks exactly like a sign-out. `sesscookie.write_client` answers a
   `Result` and `sesscookie.client_bytes` is the measurement to take
   first.
6. **Set both timeouts.** `sess.is_expired` checks both, and
   `sess.expiry_reason` says which one ended the session, because the
   two have different fixes. A deployment with only an idle timeout has
   sessions an automated loop refreshes forever, which is exactly the
   session a thief has.
7. **A store's `sess_get` answers nothing for three different
   reasons**: never issued, expired, deleted. It does not distinguish
   them, because doing so would make the store an oracle worth one bit
   per guess. `sess_drop` is silent about whether there was anything to
   drop, for the same reason.
8. **Compare identifiers with `sessid.id_eq`.** It is a constant-time
   comparison, because the value being compared is one an attacker
   chose.
9. **`sessid.new` costs `[fs, rand]`.** The operating system's
   generator is a device, and reading a device is `[fs]`. The standard
   library's `rand` module is xoshiro256\*\*, which is a fine shuffle
   and a poor session identifier. `sessid.of_bytes` is the effect-free
   twin for a caller bringing its own bits.
10. **Session values are strings.** A session is serialised into a
    store and back, and a typed bag would make this package a
    serialisation framework. A caller with structure puts JSON in a
    value.
11. **A flash message is consumed by reading.**
    `sess.take_flash` answers the messages and the session without
    them. `sess.peek_flash` reads without consuming.
12. **The cookie's keys are a list, newest first.** The first signs or
    encrypts and every key still verifies, so a key can be rotated
    without signing everybody out. `sesscookie.rotate_key` pushes a new
    one on the front.
13. **Check the cookie's attributes at start-up.**
    `sesscookie.problems` answers the combinations a browser silently
    drops, from cookie-nv's own rules.
14. **A session identifier belongs in a log only in its short form.**
    `sessid.short_of` is that form. The full value is a credential.

## Timeouts and sizes

| `SessPolicy` field | `defaults()` |
| --- | --- |
| `idle_secs` | 1800, a thirty-minute idle timeout |
| `absolute_secs` | 43200, a twelve-hour absolute timeout |
| `cookie_ceiling` | 4096 bytes |
| `save_unchanged` | false |

Zero means no timeout in each of the first two, which is a choice
rather than a default. `sesscookie.cookie_ceiling()` is the same 4096.

## Writing your own store

`SessStore[e]` is a trait with an effect parameter, so the store says
what it costs and the middleware is charged the same.

| Store | Effects |
| --- | --- |
| `SessMemory` | `[mutate]` |
| `SessFiles` | `[fs]` |
| A store over a database client | `[net]`, or whatever that client costs |

`sessmw.load`, `save`, `sign_in`, `sign_out` and `sweep` each bind that
parameter, so an application mounting the memory store in its tests
declares `[mutate]` and not `[fs]` and `[net]` as well.

A store implements five methods: `sess_get`, `sess_put`, `sess_drop`,
`sess_touch` and `sess_sweep`. `sessstore.SessRemote` describes the
shape a remote one takes, with the key prefix, whether the server
expires rows itself and whether touching is one command, and
`remote_key` and `remote_ttl` supply the two values such a store needs.
The implementation is about twenty lines over the caller's own client,
and nothing in this package changes.

## What is not included

- **A store for Redis or a database.** Depending on such a client here
  would make every application with a file store compile one it never
  calls. See above.
- **A clock.** Every time is an argument.
- **Cross-site request forgery tokens.** A session is where one would
  be kept, and the token itself belongs to whatever renders the form.
- **Authentication.** What puts a person in a session is
  [oauth2-nv](https://novo-lang.org/packages/oauth2-nv) or a password
  check the application makes.
- **A build for a microcontroller.** A store touches the machine.

## Related packages

- [cookie-nv](https://novo-lang.org/packages/cookie-nv) is the layer
  under `sesscookie`: the attributes, the prefix rules, and the signing
  and encryption of a cookie value. This package depends on it.
- [oauth2-nv](https://novo-lang.org/packages/oauth2-nv) is what signs a
  person in. Its `OauthPending` is a value to keep in a session between
  the redirect out and the callback back.
- [router-nv](https://novo-lang.org/packages/router-nv) is the other
  half of a request's front door.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is the
  civil date and time both timeouts are arithmetic over. This package
  depends on it.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) supplies
  `digest.ct_eq`, the constant-time comparison of rule 8. This package
  depends on it.
- [uuid-nv](https://novo-lang.org/packages/uuid-nv) supplies the
  default identifier, a version 4 UUID with 122 unpredictable bits.
  This package depends on it.
- [rand-nv](https://novo-lang.org/packages/rand-nv) supplies the
  operating system's generator, and the 256-bit identifier
  `sessid.new_wide` mints. This package depends on it.

## Tests

```bash
novo test tests/sesserr_tests.nv     # the faults and their statuses
novo test tests/sessid_tests.nv      # minting, parsing, comparing
novo test tests/sess_tests.nv        # change tracking, the timeouts, flashes
novo test tests/sesscookie_tests.nv  # the attributes, the ceiling, key rotation
novo test tests/sessstore_tests.nv   # the trait, at two effect rows
novo test tests/sessmw_tests.nv      # load, save, sign in, sign out
```

The reference implementations are tower-sessions and Flask's session
support. RFC 6265 section 6.1 is where the cookie size floor comes
from, and cookie-nv's own rules decide the attributes.

No test reads a clock or installs a store. Every time is a literal and
every session is built from public constructors, so "does rotation
carry the data across", "does a read mark the session dirty", "is this
cookie past the browser's ceiling" and "which timeout ended it" are
each one comparison.

`tests/sessstore_tests.nv` holds the claim about the effect parameter:
one test declares `[io, mutate]` and another declares `[io, fs]`,
calling the same five trait methods against the memory store and the
file store. The compiler decides that before the run starts.

The tests compile today and fail at run, each on the
`not implemented: session-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one
at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| Every `pub struct`, `pub enum` and `pub trait` in the six modules | the types are declared |
| `sesserr.is_retryable`, `.is_configuration_fault`, `.means_no_session`, `.status_for`, `.is_client_safe` | no |
| `sesserr.SessFault.message` | no |
| `sessid.new`, `.new_wide`, `.of_bytes`, `.parse`, `.of_uuid` | no |
| `sessid.text_of`, `.short_of`, `.id_eq`, `.is_uuid`, `.as_uuid`, `.entropy_bits` | no |
| `sess.defaults`, `.policy`, `.check_policy` | no |
| `sess.fresh`, `.of_record`, `.to_record` | no |
| `sess.get`, `.get_or`, `.has`, `.set`, `.remove`, `.keys`, `.len`, `.clear` | no |
| `sess.flash`, `.take_flash`, `.take_all_flashes`, `.peek_flash`, `.flash_count` | no |
| `sess.rotate`, `.destroy`, `.touch`, `.superseded` | no |
| `sess.needs_save`, `.needs_cookie`, `.is_fresh`, `.is_dirty`, `.was_rotated`, `.was_destroyed` | no |
| `sess.is_expired`, `.expiry_reason`, `.expires_at` | no |
| `sessstore.memory`, `.memory_len`, `.memory_evicted`, and `SessStore[mutate] for SessMemory` | no |
| `sessstore.files`, `.file_path`, and `SessStore[fs] for SessFiles` | no |
| `sessstore.remote_defaults`, `.remote_key`, `.remote_ttl` | no |
| `sesscookie.defaults`, `.client_side`, `.with_name`, `.with_attrs`, `.is_client_side`, `.problems` | no |
| `sesscookie.read_id`, `.read_client`, `.verified_by` | no |
| `sesscookie.write_id`, `.write_client`, `.write_deletion` | no |
| `sesscookie.client_bytes`, `.cookie_ceiling`, `.rotate_key`, `.key_count` | no |
| `sessmw.options`, `.with_policy`, `.eager`, `.set_cookie_header`, `.has_cookie` | no |
| `sessmw.load`, `.session_of`, `.expiry_reason_of`, `.load_faults` | no |
| `sessmw.save`, `.sign_in`, `.sign_out`, `.sweep` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
