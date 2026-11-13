# Decisions

Each entry records what was decided, what it cost, and what was rejected. The
point is that a future reader can tell whether the reasoning still applies, rather
than re-litigating from the conclusion.

---

## ADR-001: The SSH BBS remains canonical

**Context.** There are three ways into one SQLite file, and a rule has to come
from somewhere. Both existing applications have drifted from each other at least
twice in production, each time invisibly.

**Decision.** `ea7klk/simple-ham-bbs` is canonical for data, schema, and rules. In
any disagreement the terminal is right. Schema changes and rule changes land
there first; this repository follows.

**Alternatives rejected.**
- *Peer services negotiating.* Two implementations disagreeing have no
  tiebreaker, and the disagreement is invisible until a user notices.
- *The API becomes canonical* because it is the most modern. Newest is not most
  correct, and the terminal is the surface with the most behaviour in it.

**Consequences.** A rule the API needs but the terminal does not have cannot be
added here alone — it goes upstream first. That is slower and it is the point.

---

## ADR-002: Extract the domain rules rather than reimplement them

**Context.** `ham-bbs-web` holds a second copy of the sanitising rule, the depth
cap, the token formats, the throttles and the APRS section classifier. Both drifts
we found were in that copy: a five-character group-tag bound that dropped
`NEAPRS`, and a join that omitted the required message body.

**Decision.** The rules move into an importable `domain/...` package in
`simple-ham-bbs`. This service imports it. No rule may exist only here.

**Alternatives rejected.**
- *Reimplement, and test against the other two.* Three copies to keep in step is
  worse than two, and the tests that would catch drift between two copies are
  exactly the tests nobody writes.
- *Have the API shell out to the terminal app.* Requires an interactive TTY and
  makes every request a session; unusable.

**Consequences.** Phase 0 is a prerequisite for all implementation work in this
repository. The extraction itself is mechanical but touches the canonical
application, so it needs care and its existing suite as the safety net.

---

## ADR-003: Spec first, not code first

**Context.** The contract is the deliverable. Code-first generation produces a
document accurate about what the code does, which is not the same as accurate
about what the service means.

**Decision.** `openapi/bbs-api.yaml` is authored first, the router is validated
against it at boot, types are generated from it, and CI contract-tests the served
shapes.

**Alternatives rejected.**
- *Code-first with comment annotations.* The annotations drift the moment someone
  is in a hurry, and nothing fails.
- *No formal spec, just a Markdown API guide.* Cannot generate a client, cannot
  validate, and Swagger UI was requested.

**Consequences.** A handler cannot exist without a spec entry — which is the
desired friction.

---

## ADR-004: Bearer tokens, not cookies

**Context.** `ham-bbs-web` uses a session cookie. That is right for a
browser-rendered application and wrong for an API.

**Decision.** `Authorization: Bearer` with two credential types: short-lived
session tokens for human clients, scoped API keys for automation. Tokens are
stored as SHA-256 hashes, like the BBS's own reset tokens.

**Alternatives rejected.**
- *Cookies for API clients.* Pulls in CSRF protection and ambient authority for
  no benefit, and makes scripts awkward.
- *JWT.* Self-contained tokens cannot be revoked, and revocation matters here — a
  sysop disabling an account should end its sessions. A row can be deleted.

**Consequences.** Every request costs one indexed read. Acceptable: the whole
database is in memory page cache.

---

## ADR-005: Cursor pagination, not offset

**Context.** The APRS receiver writes continuously. Offset pagination over a
table being written to shows duplicate and missing rows as a client pages.

**Decision.** Opaque cursors over the ordering key (`id`), `limit` capped at 200.
Collections always return an object with `items` and `next_cursor`, never a bare
array.

**Alternatives rejected.**
- *Offset/limit.* Simple, and demonstrably wrong under concurrent writes.
- *Bare arrays.* Cannot grow a `total` field without breaking every client.

**Consequences.** No random page access. Nothing in this API needs it.

---

## ADR-006: Idempotency keys are mandatory on APRS sends

**Context.** An APRS send is a packet on the air. A client that times out and
retries has transmitted twice, and the recipient sees a duplicate that cannot be
recalled.

**Decision.** Every operation with an effect outside the database — sends, bot
commands, the Thursday actions — requires `Idempotency-Key`. Replays return the
original response, status included, for 24 hours.

**Alternatives rejected.**
- *Advisory keys.* A client that forgets one transmits twice, silently. The cost
  of the accident is borne by a third party with a radio.
- *Server-side deduplication by content.* Two identical messages at different
  times are legitimate; deduplicating them loses real traffic.

**Consequences.** The client must hold a key across retries — which the generated
client wrapper does, once per user action.

---

## ADR-007: One binary serves the API, the docs and the console

**Context.** The deployment is one node with a `ReadWriteOnce` volume. The API
cannot be scaled horizontally anyway.

**Decision.** Static frontend assets and Swagger UI are `go:embed`ed and served by
the same process at `/console` and `/docs`.

**Alternatives rejected.**
- *Separate frontend deployment.* A second image and a second rollout to keep in
  step, buying scaling that cannot be used.
- *Server-rendered Go templates for the console.* Would make it a second copy of
  `ham-bbs-web` and would never exercise the API over HTTP, which is the
  console's reason to exist.

**Consequences.** A frontend change requires an API deploy. Given one deployable
unit, that is a feature.

---

## ADR-008: The API never acknowledges APRS messages

**Context.** The `bbs-aprs` pod receives messages and acknowledges every one
addressed to an APRS-enabled account, whether or not anyone is using a client.
That is deliberate: an acknowledgement is owed to the sender, not to a session.

**Decision.** No acknowledgement endpoint, no ack worker in this service. Neither
does the web app.

**Alternatives rejected.**
- *Ack on read, client-driven.* Messages would go unacknowledged whenever nobody
  was looking, and two clients reading the same message would ack twice.

**Consequences.** A client cannot "mark as received"; there is nothing to mark.
Documented in `ham-bbs-web/docs/shared-database.md` and here so it is not
re-litigated.

---

## ADR-009: Bot commands take structured input, not free text

**Context.** The bot command strings are a wire format: `LAST10 THURS`,
`MINE10 EA7KLK`, `?APRSM`. A client assembling them by hand will eventually send
`MINE10 THURS EA7KLK`, which the bot reads as a callsign of `THURS`. That exact
bug was written and caught during the section work.

**Decision.** `POST /v1/aprs/bots/aprsph/commands` takes `{command, count, target,
phrase, thursday}` and the service builds the string via the shared package. A
`raw` variant exists for escape hatches, validated and never split.

**Alternatives rejected.**
- *Free text only.* Maximum flexibility, and the client is responsible for a
  protocol it does not own.
- *Structured only.* Cannot express a command the bot adds tomorrow.

**Consequences.** Two request shapes on one endpoint, documented with `oneOf`.

---

## ADR-010: No implementation until Phase 0 lands

**Context.** The temptation is to start the API now, against the current
`ham-bbs-web` code, and extract later. That is how a third copy happens: "later"
arrives after the API is in production and has its own tests asserting the wrong
behaviour.

**Decision.** This repository contains design until the domain package exists
upstream.

**Alternatives rejected.**
- *Build against `ham-bbs-web`'s code by copying it.* Every copy is a future
  drift.
- *Vendor the whole application.* Same copy, with a build step.

**Consequences.** Visible, deliberate idleness in this repository until a
prerequisite is met elsewhere. Recorded here so the idleness is not mistaken for
abandonment.
