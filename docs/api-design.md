# API design

Everything here is a decision, not a menu. Where a convention was chosen over a
plausible alternative, the reason is stated so it can be argued with.

## Conventions

### Base path and versioning

```text
https://<host>/v1/...
```

The major version is in the path. Within `/v1`, changes are additive only: a new
field, a new endpoint, a new enum member a client can ignore. Removing a field,
renaming one, tightening validation, or changing a status code requires `/v2`.

Deprecation is announced with headers before it happens:

```http
Deprecation: true
Sunset: Wed, 01 Apr 2027 00:00:00 GMT
Link: <https://.../v2/boards>; rel="successor-version"
```

A `Sunset` date is a promise; it goes in the release notes the day it is set.

### Naming and shape

- `snake_case` in JSON. The database columns are already snake_case and the
  translation keys are lowercase; introducing `camelCase` at the wire would add a
  third convention.
- Collections are `{"items": [...], "next_cursor": "..."}` — an object, never a
  bare array. A bare array cannot grow: adding `total` or `next_cursor` later
  breaks every client.
- IDs are opaque strings in responses even where the column is an integer. A
  client that parses `"42"` as an int is a client that breaks when the key type
  changes.
- `null` and absent mean the same thing: not set. Clients must not distinguish.
- Timestamps are RFC 3339 in UTC with a `Z` suffix. **The database stores
  `"2026-11-13 09:41 UTC"`**, and that conversion happens once, in the
  serialisation layer. Nothing else in the system may parse a timestamp.

### Authentication

Two credential types, both `Authorization: Bearer <token>`.

| Type | Lifetime | Use |
| --- | --- | --- |
| Session token | 12 h idle, 7 d absolute | A human client that logs in |
| API key | Long-lived, revocable | Scripts and integrations |

Session tokens are opaque, 32 random bytes, base64url. Only their SHA-256 is
stored — the same discipline as the BBS's own reset tokens, so a database copy
cannot be replayed. They are **rows, not memory**: the service is stateless, and a
deploy must not log every client out.

API keys carry scopes rather than full authority:

```text
bbs.read      read anything the principal may read
bbs.write     post, edit own content, send APRS
bbs.admin     sysop operations
```

Scopes are checked in middleware, once per request, against the route's declared
requirement. A route that forgets to declare one is a startup error, not a
runtime hole — the router table is validated at boot against the OpenAPI spec's
`security` blocks.

**No cookies.** No ambient authority, no CSRF surface, nothing a browser attaches
on its own.

### Errors

RFC 9457 `application/problem+json`:

```json
{
  "type": "https://bbs.ea7klk.es/problems/reply-too-deep",
  "title": "Thread nesting exceeds 5 levels",
  "status": 422,
  "detail": "Message 812 is already 5 levels below its root.",
  "instance": "/v1/messages/812/replies",
  "code": "reply_too_deep",
  "errors": [{"field": "parent_id", "reason": "depth_limit"}]
}
```

`code` is the stable machine-readable discriminator; `type` is a URL that may
later resolve to documentation. Clients branch on `code`, never on `title`.

Status codes are used for their meaning:

| Code | When |
| --- | --- |
| 400 | Malformed request — not JSON, missing required field |
| 401 | No credential, or an expired/unknown one |
| 403 | Authenticated, not permitted |
| 404 | Absent, or invisible to this principal |
| 409 | Conflict, e.g. callsign already registered |
| 412 | `If-Match` did not match |
| 422 | Well-formed but refused by a rule — depth, length, invalid callsign |
| 429 | Rate limited; carries `Retry-After` |
| 502 | Upstream APRS-IS failed |

The distinction between 403 and 404 is deliberate and consistent: a resource a
principal may not **know exists** returns 404. The BBS's login deliberately does
not reveal whether an account exists, and the API must not leak what the terminal
hides.

### Pagination

Cursor-based, in both directions:

```http
GET /v1/aprs/messages?direction=sent&limit=50&cursor=eyJpZCI6MTIzfQ
```

```json
{"items": [...], "next_cursor": "eyJpZCI6NzR9", "has_more": true}
```

Cursors are opaque, base64 of `{"id": ...}` — the ordering key. Offset pagination
was rejected: a new message arriving between pages shifts every row, so a client
paging through history sees duplicates and misses rows. The APRS page-size
selector in the web UI already shows why this matters once a receiver is writing
continuously.

`limit` defaults to 25, caps at 200. A cap is a promise the service can keep; a
`limit=100000` that silently returns 200 rows is a lie.

### Idempotency

Every `POST` that has an effect outside this database — APRS sends, bot commands,
and the Thursday actions — **requires** `Idempotency-Key`:

```http
POST /v1/aprs/messages
Idempotency-Key: 6f1d0f9a-...
```

This is not politeness. An APRS send is a packet on the air. A client that retries
after a timeout because it did not see the response has transmitted twice, and
the recipient sees a duplicate. The service stores the key with the outcome for
24 hours and replays the original response on a repeat, including its status.

Writes that only touch our own database (posting a message, creating a board)
accept the header but do not require it: a duplicate there is visible and
cheap to delete, whereas a duplicate packet is already on the air.

### Conditional requests

Reads return `ETag`. A client may send `If-None-Match` and get `304` with no
body. Edits accept `If-Match` on the mutable text fields and return `412` if the
row changed underneath — optimistic concurrency without a version column.

### Rate limits

Mirroring the BBS's own, not a softer set:

| Action | Limit |
| --- | --- |
| Login attempts | 5 per 15 min per account+address, 15 min lockout |
| Registration | 5 per hour per address |
| Password reset mail | 3 per hour per account |
| Authenticated requests | 600 per minute per token |
| APRS sends | 20 per hour per account |

Responses carry `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`, and
a 429 carries `Retry-After`. Limits are enforced in the service, not left to the
ingress, because the ingress does not know who the principal is.

## Resources

The full inventory of what the two existing surfaces do, and how each maps.

### Sessions and accounts

| Method | Path | Notes |
| --- | --- | --- |
| `POST` | `/v1/sessions` | callsign + password → token. Throttled per account+address |
| `GET` | `/v1/sessions/current` | who am I, including `is_sysop`, `enable_aprs` |
| `DELETE` | `/v1/sessions/current` | revoke this token |
| `POST` | `/v1/accounts` | register. 409 if the callsign exists, 422 if the password is under 8 characters |
| `POST` | `/v1/accounts/activation-codes` | (re)send the code |
| `POST` | `/v1/accounts/activation` | verify a code, activating the account |
| `POST` | `/v1/password-resets` | request mail. Always 202, whether or not the account exists |
| `PUT` | `/v1/password-resets/{token}` | set the new password, consuming the token |
| `POST` | `/v1/email-changes` | start the double confirmation |
| `POST` | `/v1/email-changes/confirmations` | redeem one of the two links |

`POST /v1/password-resets` returning 202 unconditionally is deliberate: a
different answer for an unknown address is an account enumeration oracle. The
terminal behaves the same way.

### Users and directory

| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/v1/users/me` | own profile, including fields not shown publicly |
| `PATCH` | `/v1/users/me` | profile; `email` change returns 202 and starts confirmation |
| `GET` | `/v1/directory` | paginated station list — the terminal's Station directory |
| `GET` | `/v1/directory/{callsign}` | one station |
| `GET` | `/v1/users` | sysop only; includes `registered`, `disabled` |
| `PATCH` | `/v1/users/{callsign}` | sysop only; `is_sysop`, `disabled` |

The directory never exposes `email`, `password_hash`, `first_seen`, `last_seen`
or `activated`. Those are in one serialiser that only the sysop route may use;
there is no code path where a public route can accidentally select the private
shape.

### Boards and messages

| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/v1/boards` | with message counts |
| `POST` | `/v1/boards` | sysop; id derived from the name as the terminal does |
| `GET` | `/v1/boards/{board}` | one board |
| `PATCH` | `/v1/boards/{board}` | sysop; rename, description |
| `DELETE` | `/v1/boards/{board}` | sysop; cascades as the terminal does |
| `GET` | `/v1/boards/{board}/messages` | `view=tree|flat`, default tree |
| `POST` | `/v1/boards/{board}/messages` | top-level post |
| `GET` | `/v1/messages/{id}` | one message |
| `PATCH` | `/v1/messages/{id}` | sysop; subject and body |
| `DELETE` | `/v1/messages/{id}` | sysop; deletes the subtree |
| `POST` | `/v1/messages/{id}/replies` | 422 `reply_too_deep` past five levels |

`view=tree` returns nested `replies`, mirroring what both existing surfaces show.
`view=flat` returns rows with `parent_id` and `depth`, for clients that paginate
or render their own tree. The web UI's flat rendering with an indent class is the
same data; the API lets the client decide.

The depth cap is enforced by the domain call, not by the handler and certainly
not by the client. A `POST` that would exceed it is refused with 422 and writes
nothing — the same guarantee the terminal gives.

### Bulletins (local)

| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/v1/bulletins` | newest first. Ordering comes from `position`, descending — the newest is the *last* row by position, which is easy to get backwards |
| `POST` | `/v1/bulletins` | sysop |
| `GET` | `/v1/bulletins/{id}` | |
| `PATCH` | `/v1/bulletins/{id}` | sysop |
| `DELETE` | `/v1/bulletins/{id}` | sysop |

### APRS messaging

| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/v1/aprs/settings` | `enable_aprs`, and the SSID-0 fact |
| `PATCH` | `/v1/aprs/settings` | enable/disable. 409 when disabled and a send is attempted |
| `POST` | `/v1/aprs/messages` | send. Splits at 67 characters; **requires `Idempotency-Key`** |
| `GET` | `/v1/aprs/messages` | `direction=sent\|received`, `section=direct\|aprsph\|thursday\|groups` |
| `GET` | `/v1/aprs/messages/{id}` | with parts, per-part status and the raw packet |
| `DELETE` | `/v1/aprs/messages/{id}` | one of the caller's own |
| `DELETE` | `/v1/aprs/messages?ids=a,b,c` | bulk, as the web UI's checkboxes do |

The `section` filter exists because bot traffic is not ordinary traffic; the
classifier is the shared one — tag in the body first, peer second — so the API
groups messages exactly as the terminal and the web UI do.

`POST /v1/aprs/messages` returns `201` with the parts and their per-part status,
not a bare `202`. The BBS actually transmits during the request, so the honest
answer is the result, including which parts failed and why. A caller that gets
`201` knows the packet left; a caller that gets `502` knows it did not.

### The two bots

| Method | Path | Notes |
| --- | --- | --- |
| `POST` | `/v1/aprs/bots/aprsph/commands` | `{"command": "last", "count": 10, "thursday": true}` |
| `POST` | `/v1/aprs/thursday/join` | `CQ HOTG <message>`; **message required**; 409 outside Thursday |
| `POST` | `/v1/aprs/thursday/leave` | `U HOTG`; requires `{"confirm": true}` |
| `POST` | `/v1/aprs/thursday/keepalive` | `K HOTG` |

The bot commands take structured input and build the protocol string in the
shared package, rather than accepting free text. The strings are a wire format —
`LAST10 THURS`, `?APRSM`, `MINE10 EA7KLK` — and a client assembling them from
memory is a client that will get `MINE10 THURS EA7KLK` wrong. A `raw` variant
takes a free-form command for the escape hatch, validated and never split.

`join` requires a message because the group server ignores a bare command; the
terminal has always required one, and a production join failed without it.

### APRS bulletins (received)

| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/v1/aprs/bulletins` | categories with counts; empty ones omitted |
| `GET` | `/v1/aprs/bulletins?category=weather` | live bulletins, ten-day window |
| `GET` | `/v1/aprs/bulletins/{id}` | provenance: group, slot, seen count, raw packet |

Read-only. The receiver owns that table, including retention.

### Content and metadata

| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/v1/about` | BBS name, location, topic — the terminal's About screen |
| `GET` | `/v1/resources` | radio resources |
| `GET` | `/v1/languages` | supported locales |
| `GET` | `/v1/version` | build, commit, schema version |

### Operations

| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/healthz` | process alive; no dependencies |
| `GET` | `/readyz` | database open, schema current |
| `GET` | `/docs` | Swagger UI |
| `GET` | `/openapi.json` | the contract itself |

Readiness deliberately does **not** consult APRS-IS. Reception is the receiver
pod's job; an API restarted because an upstream was down would be restarted for
nothing.

## Localisation

Every resource that returns prose returns the strings the caller asked for, via
`Accept-Language`, falling back to `en`. Error `title` and `detail` are
translated; `code` never is. Content itself — message bodies, bulletins — is
stored once and returned as written, because it is user text, not UI.

Translations come from the same `translations.json` the other two surfaces use.
The API adds no keys that exist only here except for error titles, and those are
added to all four languages in the same commit.

## What the API will not do

- **Acknowledge APRS messages.** The `bbs-aprs` receiver acknowledges every
  inbound message for an APRS-enabled account, whether or not anyone is using any
  client. A second ack path would duplicate packets on the air.
- **Receive APRS.** That is the receiver's job, and it must not stop when the API
  is redeployed.
- **Serve HTML from `/v1`.** The frontend is a separate concern served from the
  same binary at a different path, and it talks to the API over HTTP like any
  other client.
- **Expose raw SQL or a generic query endpoint.** Every read is a designed
  resource with a shaped response.
