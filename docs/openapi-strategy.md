# The OpenAPI contract

## Spec first

`openapi/bbs-api.yaml` is written before the handlers and is the source of truth.
Not "documentation generated from the code" — the other way round.

The reason is that the contract is the product. A client author reads the spec;
if the spec and the running service disagree, the client breaks and the spec was
a lie. Code-first generation gets you a document that is always accurate about
what the code *does*, which is not the same as accurate about what the service
*means*, and it cannot express what is not yet built.

Spec-first also forces the resource design to be settled before handlers exist.
Every awkward decision in this document — where bot commands get validated, what
`201` means for a send, how sections are filtered — was made while writing the
spec, which is cheaper than making it while debugging.

## How drift is prevented

Three mechanisms, in increasing strength.

**1. Boot-time route validation.** The router table is built from the spec at
startup. Every route the service serves must appear in the spec with the same
method and path template; every route in the spec must have a handler. A mismatch
is a startup error, so the service cannot run in a state where it claims more or
less than it does.

```go
// internal/httpapi/routes.go
func Routes(spec *openapi3.T) (http.Handler, error) {
    // fails if a handler has no spec entry, or a spec entry has no handler
}
```

**2. Generated types.** Request and response structs are generated from the spec
(`oapi-codegen`), not hand-written. A handler that returns a field the spec does
not declare does not compile. This is what makes the spec authoritative rather
than aspirational.

**3. Contract tests in CI.** A test walks the spec and, for each operation,
asserts the served response *shape*: required fields present, types correct, no
undeclared fields. It runs against a seeded in-memory database with no network.
A spec change that the handlers cannot satisfy fails the build; a handler change
the spec does not describe fails too.

The third is the expensive one and the one that pays. Without it, mechanisms 1
and 2 only check structure — a field could still be `string` in the spec and
always be empty in practice.

## Validation

The spec is linted in CI with `redocly lint`, on these rules:

| Rule | Why |
| --- | --- |
| Every operation has `operationId`, `summary`, `tags` | A docs UI without them is a list of paths |
| Every response has a schema or is `204` | "unknown" responses defeat the purpose |
| Every error response references `Problem` | One error shape, not one per handler |
| No `$ref` to a missing component | A dangling ref breaks Swagger UI silently |
| `additionalProperties: false` on request bodies | Unknown fields are rejected, not ignored — silently dropping a field is how a client ships a bug |

Breaking a rule fails the build. The rules are in `.redocly.yaml`.

Four problems are explicitly ignored in `.redocly.lint-ignore.yaml`, each with a
reason. They are not suppressions: three endpoints (`/healthz`, `/readyz`,
`/version`) are unauthenticated by design and cannot answer 4XX, and the second
server entry is a workstation host. The file is reviewed, and adding to it should
be rarer than fixing what it would hide.

## Documentation UI

Swagger UI is served at `/docs`, rendered from the same file, **embedded in the
binary** with `go:embed`. Nothing is fetched at runtime: the deployment is a
radio-adjacent environment where a page that depends on a CDN is a page that
sometimes does not load, and the rule the other BBS surfaces follow — no external
assets — applies here too.

The UI is read-only: "Try it out" is enabled against the same origin, which is
useful, but `Try it out` sends real packets for the APRS endpoints. Those
operations carry `x-dangerous: true` in the spec, and the UI's request
interceptor warns before sending. A documentation page that can silently transmit
under the operator's callsign would be a poor joke.

## What the spec does not attempt

- **Exhaustive examples.** Examples are on the operations where the shape is
  surprising (errors, APRS sends, bot commands). A spec with an example for every
  field is a spec nobody keeps current.
- **Generated SDKs as a deliverable.** The TypeScript client for the frontend is
  generated in the frontend's build, not published. A second consumer can
  generate its own.
- **Non-HTTP guarantees.** Ordering, retry semantics and idempotency are described
  in prose in [api-design.md](api-design.md), because OpenAPI cannot express them
  and a `description` that tries is worse than a document that says it properly.
