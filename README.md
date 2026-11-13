# bbs-api — design

A REST API over the HAMNET Radio BBS, with an OpenAPI contract and a reference
frontend.

**Status: design only.** There is no implementation in this repository yet. This
document set is the specification for one: what the service does, how it is
built, and what it deliberately does not do.

## Why

The BBS is reachable two ways today, and both are tightly coupled to their
delivery mechanism:

- **The classic SSH BBS** (`ea7klk/simple-ham-bbs`) — the canonical application.
  It owns the schema, the APRS receiver and acknowledgement worker, and every
  business rule. Its only interface is a 132x24 terminal session.
- **The web front end** (`ea7klk/ham-bbs-web`) — server-rendered HTML, one page
  per screen, forms and cookies. Every rule it applies is a second copy of the
  SSH BBS's.

Neither can serve a script, a mobile client, a map overlay, or anything that is
not a browser form. A third surface is wanted, and it should be the last one that
needs its own copy of the rules.

## What this is

- A REST API covering the whole BBS domain: accounts, sessions, boards and
  threaded messages, bulletins, the station directory, APRS messaging and
  history, APRS bulletins, and the two APRS bots.
- An OpenAPI 3.1 contract as the source of truth, with Swagger UI served at
  `/docs`.
- A reference frontend that consumes that API and nothing else, so the contract
  is exercised by a real client rather than asserted.

## What this is not

- **Not a third implementation of the BBS rules.** The service owns HTTP:
  routing, authentication, serialisation, pagination, rate limiting, the
  contract. Every domain rule — reply depth, text sanitising, sysop bootstrap,
  token formats, APRS splitting, the message-section classifier — is delegated to
  code extracted from the canonical repository. See
  [the architecture](docs/architecture.md#one-implementation-of-each-rule).
- **Not a replacement for either existing surface.** The SSH BBS stays canonical.
  The web front end keeps working.
- **Not a database owner.** The schema belongs to `simple-ham-bbs`.

## Documents

| Document | What it settles |
| --- | --- |
| [docs/architecture.md](docs/architecture.md) | Process model, layering, database access, deployment, failure modes |
| [docs/api-design.md](docs/api-design.md) | Resources, authentication, errors, pagination, idempotency, versioning |
| [docs/openapi-strategy.md](docs/openapi-strategy.md) | How the contract is written, generated from, and kept true |
| [docs/frontend.md](docs/frontend.md) | The reference client |
| [docs/roadmap.md](docs/roadmap.md) | Phases, and what is deliberately deferred |
| [docs/decisions.md](docs/decisions.md) | The decisions worth remembering, with the alternative each rejected |
| [openapi/bbs-api.yaml](openapi/bbs-api.yaml) | The contract, as a working skeleton |

## The three decisions that shape everything

1. **The SSH BBS is canonical.** Where the API and the terminal disagree, the
   terminal wins; schema and rules are changed there first.
   ([ADR-001](docs/decisions.md#adr-001-the-ssh-bbs-remains-canonical))
2. **One implementation of each rule.** A shared Go package extracted from
   `simple-ham-bbs`, imported here — not a third copy.
   ([ADR-002](docs/decisions.md#adr-002-extract-the-domain-rules-rather-than-reimplement-them))
3. **The contract comes first.** `openapi/bbs-api.yaml` is authored before the
   handlers and checked in CI.
   ([ADR-003](docs/decisions.md#adr-003-spec-first-not-code-first))

## Quick shape

```
        client (frontend, scripts, mobile)
                    |
                    | HTTPS, /v1/..., Bearer token
                    v
        +-------------------------------+
        |           bbs-api             |
        |  HTTP  auth  serialise  OAS   |
        +-------------------------------+
                    |
                    | Go call, not HTTP
                    v
        +-------------------------------+
        |  ham-bbs-domain (shared pkg)  |   <- extracted from simple-ham-bbs
        |  rules, storage, APRS, tokens |
        +-------------------------------+
                    |
                    v
        /var/lib/bbs/bbs.sqlite  (owned by the canonical BBS)
```

## Adding to this repository

See [AGENTS.md](AGENTS.md). The short version: a change to a rule belongs in
`simple-ham-bbs` first, a change to the contract belongs in
`openapi/bbs-api.yaml` first, and an implementation must not introduce a rule
that exists only here.
