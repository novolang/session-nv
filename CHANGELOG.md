# Changelog

All notable changes to session-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `sessstore` — `SessStore[e]`, the store trait with an effect
  parameter, and the reason the whole package holds together: a memory
  store costs `[mutate]`, a file store costs `[fs]`, and the middleware
  costs whichever one the caller mounted.  `SessMemory` with a capacity
  and an eviction count; `SessFiles` whose constructor touches no
  filesystem; and `SessRemote`, the shape a network store takes as a
  value, declared without depending on a client.
- `sess` — `Sess`, `SessRecord`, `SessPolicy` and `SessChange`, all
  `[]`.  Change tracking so a read does not become a write; two
  timeouts rather than one, with `expiry_reason` naming which ended a
  session; `rotate` carrying the old identifier so the old record can
  be deleted; and flash messages where `take_flash` empties and
  `peek_flash` does not.
- `sesscookie` — the cookie half over cookie-nv: `__Host-` attributes
  by default, the identifier and client-side kinds, key rotation as a
  list with `verified_by` naming which key matched, and
  `write_client`'s refusal of a session past the browser's 4096-byte
  ceiling with both numbers in the fault.
- `sessid` — `SessId`, minted from the operating system's generator at
  `[fs, rand]` and compared in constant time.  `parse` refuses
  everything outside the alphabet before a store or a file name is
  built from it; `entropy_bits` says 122 for a UUID and 256 for the
  wide form.
- `sessmw` — `load` and `save`, both effect-polymorphic over the store;
  `sign_in`, which rotates and saves in one call; `sign_out`;
  `sweep` for a caller's own scheduler; and `load_faults`, so an
  unsealed cookie is counted rather than hidden.
- `sesserr` — `SessFault`, with `means_no_session` as the predicate
  that turns a bad cookie into a sign-in page rather than a 500.
- `tests/` — 66 API tests against the signatures, red until the bodies
  land.  Two of them declare different effect rows over the same trait
  calls, which is the compiler asserting the store design.

### Known

- Every body is `todo()`.  `novo test` is red, `novo pkg build` is
  green, and the shard rows that measure the design —
  `effect-budget`, `dep-layer`, `no-discharge-in-core` — pass.
- No `tests/embedded_probe.nv`: this is a `host` package, so it makes
  no device claim to check.
- No Redis or database store ships.  `SessRemote` is the shape one
  takes; writing one is about twenty lines against a client the
  deployment already has.
