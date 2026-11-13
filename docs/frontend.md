# The frontend

## Why build one at all

A REST API with a Swagger page is a contract, not a product. The frontend exists
to answer a question the spec cannot: **is this API actually usable by a client
that did not write it?** A reference client that consumes only the public API,
generates its types from the contract, and performs the real workflows — read the
boards, post, send APRS — is the cheapest way to find out.

It also gives the deployment something a human can use at the front door, which is
otherwise a JSON endpoint nobody visits.

## What it is not

- **Not a replacement for `ham-bbs-web`.** That application keeps working,
  server-rendered, as it is. Rewriting it onto this API is a separate decision
  with its own migration; the [roadmap](roadmap.md) records it as deferred.
- **Not another copy of the UI.** It is a small console, not a redesign. The
  design language follows Packet Manual × Spectrum from `ham-bbs-web` so the two
  do not look like different products, but no effort goes into matching its
  screen-for-screen behaviour.

## Shape

A static single-page application, TypeScript, built with Vite, served from the
same binary as the API at `/console`.

```
GET /            -> redirect to /console
GET /console/*   -> embedded static assets
GET /docs        -> Swagger UI
GET /v1/*        -> the API
```

One deployable unit, one image, one digest to pin. The alternative — a separate
frontend container — buys independent scaling that this deployment cannot use
(there is one node and one SQLite writer) at the cost of a second image and a
second rollout to keep in step.

### Why not server-rendered Go templates

`ham-bbs-web` is already server-rendered, and that works well for it. But its
pages are built by the same process that owns the data, so it never exercises the
API. A client that is *forced* through HTTP — auth headers, pagination cursors,
problem+json errors, idempotency keys — is the client that proves the contract.

Server-rendering this console would make it a second copy of `ham-bbs-web` inside
a service that is supposed to be an adapter.

### Why not a framework

No React, no Svelte. The console has perhaps eight screens and no shared state
worth managing; a framework would be more code than the application, and its
build would dominate the pipeline. Vite + TypeScript + a small render helper,
rebuilt only when the frontend sources change.

If this turns out to be wrong — if the client grows interactions that need real
state management — that is a decision to revisit with evidence, and
[decisions.md](decisions.md) is where it goes.

## Generated client

Types and the request functions come from `openapi/bbs-api.yaml` at build time:

```sh
npx openapi-typescript ../openapi/bbs-api.yaml -o src/api/schema.d.ts
```

No hand-written response types. If the contract changes and the frontend does not
compile, that is the intended alarm.

A thin wrapper around `fetch` adds what the API requires and TypeScript cannot
enforce:

- attaching the bearer token
- generating an `Idempotency-Key` for every dangerous `POST` **once per user
  action**, held across retries — the whole point of the key is that a retry
  reuses it
- parsing `problem+json` into a typed error, so a screen can show `detail` and
  branch on `code`
- following `next_cursor` for "load more"

## Shape of a screen

Reusing the Packet Manual × Spectrum language from `ham-bbs-web` — monospace,
hard rules, a title block, the section bar one level down — so the two do not
look like unrelated products:

```
+---------------------------------------------------------------+
| HAMNET RADIO BBS      EA7KLK · SYSOP        [EN]   [Sign out] |  title block
+---------------------------------------------------------------+
| HOME  BULLETINS  BOARDS  DIRECTORY  APRS  EA7KLK  SYSOP       |  section tabs
+---------------------------------------------------------------+
| APRS                                                          |
|                                                               |
| +-- NOTE -------------------------------------------------+   |
| | SENDING AS  Every message you send leaves from EA7KLK-0.|   |
| +---------------------------------------------------------+   |
|                                                               |
|  DIRECT (12)   APRSPH (8)   THURSDAY (479)   GROUPS (14)      |  section bar
|  ---------------------------------------------------------    |
|  4h   EA7KLK-0   EA7KLK-15   BBS loop test, please ignore     |
|  6h   KB3KBR-2   EA1ABC      N:HOTG happy aprsthursday!!!     |
|  ...                                                          |
|                                      [Load more]              |
+---------------------------------------------------------------+
```

The section bar is the same control the web UI uses, for the same reason: bot
traffic is not ordinary traffic, and the counts are what tell you that before you
open anything.

## Screens

The minimum that exercises the whole contract:

| Screen | Exercises |
| --- | --- |
| Login | `POST /v1/sessions`, throttling, 401 shape |
| Boards | list, counts, create/rename/delete as sysop |
| Board view | `view=tree`, reply, depth refusal (`422 reply_too_deep`) |
| Bulletins | ordering, sysop edit |
| APRS console | send with idempotency, split parts, per-part status |
| APRS history | `direction` + `section` filtering, pagination, bulk delete |
| Bots | structured command forms, the Thursday actions and their refusals |
| Directory | pagination, the public/private field split |
| Profile | `merge-patch`, email change starting a confirmation |

Every screen has an explicit empty state and an explicit error state. The API
returns structured errors precisely so the client can do something with them; a
screen that renders "Something went wrong" wastes that.

## Things the console must get right

- **The SSID-0 fact.** Sending shows that traffic leaves from `<callsign>-0`
  whatever is addressed — the same warning the other two surfaces carry, in the
  same words from the same translations.
- **The Thursday gate.** Joining outside Thursday is refused with `409`; the
  button is disabled with the reason shown *before* the request, rather than after
  it fails.
- **A required message on join.** The message field is required in the form, not
  only on the server, because the server's refusal is correct but late.
- **APRS disabled.** A 409 from a send turns into a prompt to enable APRS with a
  link to the setting, not a dead end.
- **No acknowledgements.** The console never offers to ack anything, because the
  receiver does that for every account whether or not anyone is looking.

## Accessibility and deployment constraints

The audience is radio operators, often on a phone in a car or a laptop tethered to
a handheld. So: responsive from 360px, no hover-only affordances, focus visible,
contrast checked against the same palette rules the web UI follows, and no
external assets — the whole console is embedded in the binary and works on a
network with no internet.

The build is one `dist/` directory copied into the image; there is no Node
runtime in production.
