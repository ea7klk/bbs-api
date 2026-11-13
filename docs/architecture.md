# Architecture

## What the service is responsible for

bbs-api is an HTTP adapter. It owns:

- routing, request decoding, response encoding
- authentication and authorisation
- pagination, filtering, sorting
- rate limiting and idempotency
- the OpenAPI contract and its documentation UI
- operational concerns: health, readiness, structured logs, request IDs

It does **not** own: the schema, the business rules, the APRS receiver, the
acknowledgement worker, or the notion of what a valid message is. Those belong to
the canonical BBS.

## One implementation of each rule

This is the load-bearing decision, and it exists because of a real failure.

`ham-bbs-web` duplicates the SSH BBS's rules in Go. Both copies have drifted at
least twice in production:

- A five-character limit on APRS group tags silently sent `NEAPRS` traffic into
  the wrong section; the other copy had the right bound.
- The web form sent a bare `CQ HOTG`, which the terminal client had always
  refused to send without a message body, so the join went out and the group
  server ignored it.

Each drift was invisible until someone noticed behaviour differing between two
surfaces reading the same rows. A third copy would make that class of bug
three-way: which of the three is right when they disagree, and how would anyone
notice?

So bbs-api imports the rules rather than restating them:

```
github.com/ea7klk/simple-ham-bbs/domain   <- canonical, extracted there
        |
        +-- imported by  simple-ham-bbs/bbs/cmd/bbs   (SSH app)
        +-- imported by  ham-bbs-web                  (web app, eventually)
        +-- imported by  bbs-api                      (this service)
```

### What "extracted" means concretely

Today both applications are `package main` with the rules inline. The extraction
is a mechanical move, not a rewrite, and it happens **in `simple-ham-bbs` first**:

| Move | From | To |
| --- | --- | --- |
| Sanitising rule | `bbs/cmd/bbs/sanitize.go` | `domain/text` |
| Message write path, depth cap | `mutations.go`, `boards.go` | `domain/forum` |
| Token formats, TTLs, activation codes | `tokens.go`, `account_mail.go` | `domain/account` |
| Registration and login throttles | `ratelimit.go` | `domain/throttle` |
| APRS split, command, ack construction | `aprs.go`, `aprs_ack.go` | `domain/aprs` |
| Message section classifier | `aprs_scope.go` | `domain/aprs` |
| Models and migrations | `database.go` | `domain/store` |

The SSH application then becomes a terminal UI over the same package. Its
behaviour must not change; that is the merge criterion for the extraction, and
its existing test suite is the evidence.

**Until the extraction lands, this repository has no implementation.** Writing it
first would mean writing a third copy, which is the thing this design exists to
prevent. Phase 0 in the [roadmap](roadmap.md) is therefore a prerequisite, not a
convenience.

## Layering

```
cmd/bbs-api/            main: flags, config, server construction
internal/httpapi/       routing, middleware, handlers, request/response types
internal/auth/          session tokens, API keys, scopes, rate limits
internal/openapi/       embedded spec + Swagger UI assets
domain/...              imported from simple-ham-bbs; no HTTP knowledge
```

Dependencies point one way: `httpapi` may import `auth` and `domain`; `domain`
imports neither. A domain function must be callable from a terminal session, an
HTTP handler, and a test with equal ease — that constraint is what keeps it
honest.

### Handlers are thin

A handler may: decode, validate shape, authorise, call one domain function,
serialise, map errors to status codes. It may not contain a business rule. If a
handler needs "and also cap the depth at five", the cap belongs in the domain
call, where the terminal client already gets it.

## Database access

### The same file, by the canonical owner's rules

bbs-api reads and writes `/var/lib/bbs/bbs.sqlite`, the file `simple-ham-bbs`
owns. It must be opened exactly as the other two do:

```text
?_busy_timeout=5000&_journal_mode=WAL&_txlock=immediate
```

plus `SetMaxOpenConns(1)`. `_txlock=immediate` is what serialises read-then-write
transactions across processes; without it a web write and a terminal write
interleave into a lost update. This is not a tuning choice, it is the contract.

### Three writers is a real change

Until now there were two. A third raises the contention on a single SQLite write
lock, and the honest position is that it may not be sustainable:

- SQLite allows **one writer at a time** across all processes. Three idle
  applications cost nothing; three busy ones queue.
- The volume is `ReadWriteOnce` on local-path, so every pod mounting it must be
  scheduled to the same node. That gives no horizontal scale and no node
  redundancy.

The design therefore ships with a **write budget** and a measurement obligation:

| Concern | Position at v1 |
| --- | --- |
| Reads | Unbounded within rate limits; reads dominate and cost little |
| Writes | Rate limited per token, and measured |
| Scale-out | Not possible on one node — the API is a single replica by design |
| Trigger to revisit | Sustained write queueing, or any need for two replicas |

`docs/roadmap.md` records the ceiling and what would have to change (Postgres)
rather than pretending SQLite scales indefinitely.

### Stale-file detection, inherited

`ham-bbs-web` learned this the expensive way on 2026-11-12: the volume was
re-provisioned and a restore landed, and the web app kept serving a deleted inode
for fifteen hours while the canonical BBS moved on. Logins failed for accounts
that existed on disk the whole time.

bbs-api inherits both guards, because it holds the same handle for the same
reason:

- **Refuse to create the database.** A missing file means the canonical side has
  not created it; seeding an empty one produces a plausible-looking BBS with none
  of the real data.
- **Watch the inode.** Compare `os.SameFile` on a timer; on replacement or
  removal, exit and let the supervisor restart onto the current file.

Both live in the domain package so all three surfaces share one implementation.

## Process model

- Single Go binary, static, no cgo beyond the SQLite driver.
- **No in-memory state.** Sessions are rows, not a map. This is a deliberate
  departure from `ham-bbs-web`, whose sessions live in process memory and are
  lost on restart; an API that logged every client out on deploy would be worse
  than useless for automation.
- Graceful shutdown: stop accepting, drain in-flight requests with a deadline,
  close the database.
- One structured log line per request: method, path template, status, duration,
  request id, principal, bytes.

## Deployment

Same shape as its siblings, because it must share the volume:

- Fleet bundle in this repository (`fleet/bbs-api/`), namespace `ham-bbs`.
- Digest-pinned image, `correctDrift` on the GitOps side.
- `nodeSelector` to the node holding the claim, as `bbs` and `bbs-aprs` already do.
- Its own `Service`, no `IngressRoute` until the front door is decided (see
  [roadmap](roadmap.md)).
- Readiness gates on a successful database open plus a schema-version check, so a
  pod that cannot see the canonical file never receives traffic.

## Failure modes, named

| Failure | Consequence | Mitigation |
| --- | --- | --- |
| Database file replaced under the process | Serves stale data silently | Inode watch; exit and restart |
| Database file missing | Body would be seeded empty | Refuse to create; wait, then fail loudly |
| Schema older than expected | Queries fail in confusing ways | Readiness check on `schema.version` |
| Two API replicas | Both write; SQLite serialises but latency spikes | One replica by design; revisit with Postgres |
| APRS send retried by a client | Duplicate packet on the air | `Idempotency-Key`, mandatory on sends |
| Upstream APRS-IS down | Sends fail; reads unaffected | Explicit 502 with the APRS-IS reason |
| Rule drift with the terminal | Different behaviour per surface | Shared package; no rule exists only here |

## Observability

- Request logs as above.
- Counters worth having from the start: requests by route and status, send
  attempts by outcome, idempotency replays, rate-limit rejections, database busy
  retries.
- `/healthz` (process alive) and `/readyz` (database open, schema current,
  receiver-independent). Never let readiness depend on APRS-IS: reception is the
  receiver pod's job, and an API that reported unready because an upstream was
  down would be restarted for no reason.

## Security posture

- TLS terminated by the ingress; the service itself listens plain HTTP on the
  cluster network, as its siblings do.
- Bearer tokens only. No cookies, so no CSRF surface and no ambient authority.
- Passwords are verified through the domain package, which implements the same
  PBKDF2 parameters as the BBS. This service never sees a plaintext password
  beyond the length of one request, and never logs it.
- Every string a client supplies goes through the sanitising rule on write, and
  content is cleaned again on read — the same rule, from the same package.
- Rate limits are per principal and per source address, mirroring the BBS's own
  login and registration limits rather than inventing softer ones.
