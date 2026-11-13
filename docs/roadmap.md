# Roadmap

Sequenced by dependency, not by appeal. Phase 0 blocks everything: it is where
the rules move, and without it there is nothing to import.

## Phase 0 — extract the domain (in `simple-ham-bbs`, not here)

**Blocks all of this repository.** The rules currently live in `package main` in
two places. Until they are importable, any implementation here is a third copy —
the exact outcome this design exists to avoid.

- Mechanical move of the rule modules into `domain/...` in `simple-ham-bbs`.
- The SSH application becomes the first consumer. Its behaviour must not change;
  its existing suite is the evidence, and the merge criterion is that suite
  passing untouched.
- The stale-file guards move too, so all surfaces share one implementation.

Done when: `simple-ham-bbs` builds against the extracted package with no
behavioural change, and the package has no HTTP knowledge.

## Phase 1 — read-only API

The smallest thing that is useful and proves the contract.

- `cmd/bbs-api`, layered as in [architecture.md](architecture.md).
- Auth: `POST /v1/sessions`, bearer tokens as rows.
- Reads: boards and messages, bulletins, directory, APRS history with section
  filtering, APRS bulletins, about/resources.
- Problem+json errors, cursor pagination, `ETag`.
- Spec served at `/openapi.json`, Swagger UI at `/docs`, both embedded.
- Contract tests in CI.
- Fleet bundle, digest-pinned, one replica.

Done when: a script can read every published thing in the BBS without a browser,
and CI proves the spec matches what is served.

## Phase 2 — writes

- Registration, activation, password reset, email change.
- Posts and replies (depth cap from the domain), sysop edits and deletes.
- Bulletin create/edit/delete.
- APRS send with mandatory `Idempotency-Key`; the APRS bots.
- Rate limits mirroring the BBS's own.
- Scopes enforced, validated against the spec at boot.

Done when: the API can do everything the two existing surfaces can, and a
retried send does not transmit twice.

## Phase 3 — reference frontend

- Vite + TypeScript console at `/console`, types generated from the spec.
- The nine screens in [frontend.md](frontend.md).
- Embedded in the same image, no Node in production.

Done when: every API operation is exercised by a real screen, and the console was
uses only the public API.

## Phase 4 — retire the duplicated rules

The point of the whole exercise.

- Move `ham-bbs-web` onto either the API or the shared package, so the rules
  operate whether or not a person is using a client.
- Delete its copies of the sanitising, depth, throttle, token and section logic.
- At that point three surfaces share one implementation, and a rule change is
  made once.

Not scheduled until phases 1–3 are in production; it is a migration with its own
risk, and the web UI working correctly today is worth more than its internal
tidiness.

## What this design deliberately does not include

| Deferred | Why |
| --- | --- |
| Postgres migration | The real ceiling, but it changes the schema contract with two running applications. Trigger: sustained write queueing, or a need for more than one API replica. |
| APRS receive in the API | The `bbs-aprs` pod owns it, and it must not stop when the API redeploys. Same reasoning that separated it from the SSH container. |
| APRS acknowledgements | The receiver does them for every account. A second ack path means duplicate packets on the air. |
| WebSocket / SSE live feed | Useful once reception volume justifies it; pagination covers the need today. |
| Public unauthenticated read access | The BBS requires an account. Whether the API should be open is a policy question, not a technical one, and it is not answered here. |
| OAuth2 / OIDC | Bearer tokens are sufficient for one BBS and its operators. Adding a provider is a dependency and an availability risk for no benefit yet. |
| Multi-tenancy | There is one BBS. |
| Generated SDK publication | Consumers can generate from the spec. |

## The ceiling, stated plainly

SQLite, one node, one writer, `ReadWriteOnce` volume. This API adds a third
writer to that arrangement. It will be fine for the traffic this BBS sees — three
applications, tens of requests a second at most — and it will not scale
horizontally, ever, without leaving SQLite.

That is an accepted limit, not an oversight. The design keeps the boundary in one
place (`domain/store`) so the eventual move to a networked database changes one
package rather than three applications.
