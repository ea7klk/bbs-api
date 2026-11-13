# Project Agent Instructions

Guidance for AI agents (Codewhale, Claude Code, etc.) working in this repository.

## Read this first

**This repository is a design, not an implementation.** There is no Go code and
no frontend yet, on purpose. See
[ADR-010](docs/decisions.md#adr-010-no-implementation-until-phase-0-lands): the
rules this service needs are being extracted into an importable package in
`ea7klk/simple-ham-bbs`, and writing the service before that exists would create a
third copy of them — the specific failure this design is built to avoid.

If you are asked to implement, the first question is whether
`simple-ham-bbs/domain` exists. If it does not, say so and point here. Do not
start a partial copy.

## What this service is

A REST API over the HAMNET Radio BBS, plus a reference frontend. It is an HTTP
adapter: it owns routing, auth, serialisation, pagination, rate limiting and the
OpenAPI contract, and delegates every domain rule to the shared package.

- **Canonical application:** `ea7klk/simple-ham-bbs`. Owns the schema, the rules,
  the APRS receiver and the acknowledgement worker.
- **Sibling surface:** `ea7klk/ham-bbs-web`. Server-rendered HTML, same database.
- **This repository:** third surface, same database, same rules, HTTP instead of
  a form.

## The invariants

1. **The SSH BBS is canonical.** Where this service and the terminal disagree, the
   terminal is right. Schema and rule changes land there first.
   ([ADR-001](docs/decisions.md#adr-001-the-ssh-bbs-remains-canonical))
2. **No rule exists only here.** If a handler needs "and also cap the depth at
   five", the cap belongs in the shared domain call. A rule written in this
   repository is a rule that will drift.
   ([ADR-002](docs/decisions.md#adr-002-extract-the-domain-rules-rather-than-reimplement-them))
3. **The spec is authoritative.** `openapi/bbs-api.yaml` is authored before the
   handler. The router is validated against it at boot; types are generated from
   it; CI contract-tests the served shapes.
   ([ADR-003](docs/decisions.md#adr-003-spec-first-not-code-first))
4. **The API never acknowledges APRS messages.** The `bbs-aprs` receiver does,
   for every account, whether or not anyone is using a client. A second ack path
   means duplicate packets on the air.
   ([ADR-008](docs/decisions.md#adr-008-the-api-never-acknowledges-aprs-messages))
5. **Sends require `Idempotency-Key`.** A retried send is a second packet the
   recipient can see, and it cannot be recalled.
   ([ADR-006](docs/decisions.md#adr-006-idempotency-keys-are-mandatory-on-aprs-sends))
6. **Bot commands are built from structured input** in the shared package, never
   assembled by a client. The strings are a wire format.
   ([ADR-009](docs/decisions.md#adr-009-bot-commands-take-structured-input-not-free-text))
7. **The database is opened by the canonical owner's rules** —
   `?_busy_timeout=5000&_journal_mode=WAL&_txlock=immediate` with
   `MaxOpenConns(1)` — and the stale-file guards are inherited, not reinvented.
8. **Nothing is fetched at runtime.** Swagger UI and the frontend are embedded in
   the binary. The deployment serves a network where a CDN dependency means a
   blank page.

## Numbers that must not be re-invented

If one of these appears in this repository, it belongs in the shared package
instead.

| Rule | Value |
| --- | --- |
| APRS message field | 67 characters |
| Reply nesting cap | 5 levels below a root |
| Activation code | 4 characters, `A-Z` and `2-9`, 24 h |
| Password reset link | 2 h, 3 requests per hour per account |
| Email change link | 24 h, two links, both required |
| Login throttle | 5 failures per 15 min per account+address, 15 min lockout |
| Registration throttle | 5 per hour per address |
| APRS bulletin retention | 10 days from last heard |
| Password minimum | 8 characters |

The authoritative values live in `simple-ham-bbs`. These are recorded here only so
a reader can recognise drift.

## Working in this repository

```sh
# Validate the contract (once implementation work begins)
npx @redocly/cli lint openapi/bbs-api.yaml

# YAML sanity, which is all that can be checked today
python3 -c "import yaml; yaml.safe_load(open('openapi/bbs-api.yaml'))"
```

The contract must pass lint with no errors. Every operation needs an
`operationId` and a `summary`; every tag must be declared; every error response
references `Problem`; every `$ref` resolves.

`.redocly.lint-ignore.yaml` records four deliberate exceptions with their
reasons. Adding an entry is a last resort — fix the contract instead.

## Documentation conventions

- Decisions go in `docs/decisions.md` as ADRs, with the alternatives that were
  rejected and why. A decision without its rejected alternatives is a preference.
- When a design choice rests on something that actually happened — a production
  incident, a bug that shipped — say so and name it. That is why several
  documents here cite specific failures: they are the evidence, and they stop the
  same mistake being made a third time.
- Do not describe a capability as present when it is designed. Mark status
  explicitly, as the README does.

## GitHub operations

Use the `gh` CLI for **all** GitHub operations — issues, pull requests, reviews,
releases, workflow runs, repository settings, and API queries. It is already
installed and authenticated in this environment, so do not ask for a token or
re-run `gh auth login`.

- Prefer the specific subcommand (`gh release`, `gh run`, `gh pr`, `gh issue`,
  `gh api` only for what has no subcommand).
- Confirm the repository with `gh repo view ea7klk/bbs-api` before mutating
  anything, and never push, publish, or change settings unless the user asked.
- Treat `gh` output as ground truth; read it rather than assuming success.

## Commit messages

Conventional commits: `docs:`, `feat:`, `fix:`, `refactor:`, `test:`, `chore:`,
`ci:`. One logical change per commit. Say why in the body when the diff does not
make it obvious.

## Deployment (when there is something to deploy)

Follow the pattern the sibling repositories use: test, build, publish to GHCR on
a release, pin the digest in the Fleet bundle in `fleet/bbs-api/`. Never deploy
with `docker compose` or `kubectl set image` on the cluster host — there is no
Docker daemon there and Fleet reverts drift.

The pod must carry the same `nodeSelector` as `bbs` and `bbs-aprs`, because the
volume is `ReadWriteOnce` on local-path and is pinned to one node.
