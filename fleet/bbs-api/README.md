# Fleet bundle — not yet populated

This directory will hold the Kubernetes resources for `bbs-api` once there is an
image to run (Phase 1 of the [roadmap](../../docs/roadmap.md)).

It is empty on purpose. Fleet applies every manifest in a bundle directory, so an
illustrative or placeholder `Deployment` here would be reconciled onto the
cluster — with an image that does not exist, or worse, with one that does. The
sketch of the resources lives in [architecture.md](../../docs/architecture.md#deployment)
as prose and a manifest excerpt, where it cannot be applied by accident.

## What will go here in Phase 1

- `fleet.yaml` — `defaultNamespace: ham-bbs`, kustomize dir `.`, takeOwnership.
- `resources.yaml` — one `Deployment` and one `Service`.

Constraints that are not optional, because they come from the volume rather than
from preference:

- `nodeSelector: kubernetes.io/hostname: spainip-k3s` — the data claim is
  `ReadWriteOnce` on local-path, bound to that node. `bbs` and `bbs-aprs` carry
  the same selector.
- `replicas: 1` — three writers already share one SQLite write lock. A second API
  replica buys latency, not throughput, and the design rejects it until the
  database is not SQLite.
- The image is pinned by digest, never by tag.
- Readiness probes `/readyz`; liveness probes `/healthz`. Readiness must not
  consult APRS-IS.
- No `hostPort`: `bbs` owns port 2222 on the node, and nothing here needs a host
  port.
