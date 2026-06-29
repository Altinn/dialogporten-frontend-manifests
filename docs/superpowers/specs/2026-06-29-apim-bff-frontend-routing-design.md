# APIM routing for BFF + Frontend — Design

**Date:** 2026-06-29
**Branch:** `arealmaas/bff-frontend-apim-routing`
**Status:** Approved

## Goal

Put the arbeidsflate `bff` and `frontend` apps behind Azure API Management (APIM)
at the public edge, so that:

- all traffic on `/api*` is routed to **bff**
- all other traffic is routed to **frontend**

Model the implementation on the dialogporten manifests repo (kinshasa workspace),
which already exposes its apps through APIM using the
`apim.dis.altinn.cloud/v1alpha1` `Api` + `Backend` custom resources.

## Background / current state

The arbeidsflate cluster routing **already performs this path split** at the
Traefik gateway, on a single hostname per environment
(`manifests/apps/{bff,frontend}/base/resources.yaml`):

- `bff` `HTTPRoute` → `matches: path PathPrefix /api` → Service `bff`
- `frontend` `HTTPRoute` → `matches: path PathPrefix /` → Service `frontend`
- per-env hostnames patched in `manifests/environments/<env>/apps/<app>/ingress-hostname.yaml`:
  - at23 → `arbeidsflate.at22.dis-core.altinn.cloud` (existing host kept as-is)
  - tt02 → `arbeidsflate.tt02.dis-core.altinn.cloud`
  - yt01 → `arbeidsflate.yt01.dis-core.altinn.cloud`
  - prod → `arbeidsflate.prod.dis-core.altinn.cloud`

APIM therefore replicates the same split at the public edge and forwards to the
existing cluster ingress, which re-splits to the right Service. No app,
Deployment, Service, or HTTPRoute changes are required.

## Reference pattern (kinshasa)

- `manifests/apim/base/` holds one `Api` + one `Backend` per exposed app, plus a
  `kustomization.yaml` listing them.
- Each `Api`: `apiType: http`, a single version (`subscriptionRequired: false`,
  `protocols: [https]`), a `/*`-catch-all OpenAPI stub covering all HTTP verbs,
  `serviceUrl: set-by-env`, and a policy that binds the API to its backend via
  `set-backend-service backend-id="{{.backendID}}"` (where `backendID` is resolved
  with `idFromBackend`).
- Each `Backend`: `title` / `description` + `url: set-by-env`.
- Per-env `manifests/environments/<env>/apim/kustomization.yaml` references
  `../../../apim/base` and JSON-patches each `Backend /spec/url` and
  `Api /spec/versions/0/serviceUrl` to that environment's ingress URL.
- The env `manifests/environments/<env>/kustomization.yaml` adds `apim` to its
  `resources` list.

## Decisions

1. **Public path layout: root (Q1 = A).**
   - `frontend` `Api` → `path: ""` (catch-all / default API)
   - `bff` `Api` → `path: api`
   - Public URLs: `https://<apim-host>/*` → frontend, `https://<apim-host>/api/*` → bff.
   - Chosen over a `/arbeidsflate` product prefix because arbeidsflate is the
     end-user web UI (clean root URLs) and this maps 1:1 onto the existing
     `/` vs `/api` gateway split.

2. **Policy: minimal passthrough (Q2 = A).**
   - Each `Api` policy is `set-backend-service` + `<base/>` only.
   - kinshasa's maintenance-mode bypass block is intentionally **not** carried over.

## Design

### New files

```
manifests/apim/base/
  frontend.yaml          # kind: Api,     path: ""
  frontend-backend.yaml  # kind: Backend
  bff.yaml               # kind: Api,     path: api
  bff-backend.yaml       # kind: Backend
  kustomization.yaml     # resources: the 4 files above
manifests/environments/at23/apim/kustomization.yaml
manifests/environments/tt02/apim/kustomization.yaml
manifests/environments/yt01/apim/kustomization.yaml
manifests/environments/prod/apim/kustomization.yaml
```

### Edited files

Add `- apim` to the `resources:` list of each:

```
manifests/environments/at23/kustomization.yaml
manifests/environments/tt02/kustomization.yaml
manifests/environments/yt01/kustomization.yaml
manifests/environments/prod/kustomization.yaml
```

### Resource shape

Both `Api` resources share the kinshasa shape, differing only in
`metadata.name`, `spec.path`, `displayName`/`description`, the OpenAPI stub
`info.title`, and the `backendID` `idFromBackend.name`:

- `arbeidsflate-frontend` — `path: ""`, backendID → `arbeidsflate-frontend`
- `arbeidsflate-bff` — `path: api`, backendID → `arbeidsflate-bff`

Policy (identical for both):

```xml
<policies>
  <inbound>
    <set-backend-service backend-id="{{.backendID}}" />
    <base />
  </inbound>
  <backend><base /></backend>
  <outbound><base /></outbound>
  <on-error><base /></on-error>
</policies>
```

Labels follow the existing convention: `app.kubernetes.io/name: <name>`,
`app.kubernetes.io/part-of: arbeidsflate`.

### Per-env URL patches

Each env's `apim/kustomization.yaml` patches four targets (2 `Backend.url`,
2 `Api …/serviceUrl`):

| env | frontend url (`path: ""`) | bff url (`path: api`) |
|------|---------------------------|-----------------------|
| at23 | `https://arbeidsflate.at22.dis-core.altinn.cloud` | `https://arbeidsflate.at22.dis-core.altinn.cloud/api` |
| tt02 | `https://arbeidsflate.tt02.dis-core.altinn.cloud` | `https://arbeidsflate.tt02.dis-core.altinn.cloud/api` |
| yt01 | `https://arbeidsflate.yt01.dis-core.altinn.cloud` | `https://arbeidsflate.yt01.dis-core.altinn.cloud/api` |
| prod | `https://arbeidsflate.prod.dis-core.altinn.cloud` | `https://arbeidsflate.prod.dis-core.altinn.cloud/api` |

### Routing semantics (why the bff backend url keeps `/api`)

APIM strips the matched `Api.path` prefix from the request and appends the
remainder to the `Backend.url`. APIM resolves the **most specific** API path
first, so `/api/*` binds to `arbeidsflate-bff` even though the `""` frontend API
would also prefix-match.

- `GET /api/v1/dialogs` → matches bff (`path: api`) → remainder `/v1/dialogs`
  → `https://arbeidsflate.<env>…/api` + `/v1/dialogs` = `…/api/v1/dialogs`.
  The cluster `bff` HTTPRoute (`PathPrefix /api`) then routes it to Service `bff`. ✓
- `GET /some/spa/route` → matches frontend (`path: ""`) → remainder
  `/some/spa/route` → `https://arbeidsflate.<env>…` + `/some/spa/route`.
  The cluster `frontend` HTTPRoute (`PathPrefix /`) routes it to Service `frontend`. ✓

The bff `Backend.url` **must** include `/api`: APIM strips the `api` path
segment, so the prefix has to be re-supplied by the backend URL, otherwise the
cluster gateway would not match the bff route.

## Known risk

kinshasa's `Api` resources all use **non-empty** `path` values. Option A relies
on the `apim.dis.altinn.cloud/v1alpha1` operator accepting `path: ""` for the
catch-all/default API. The CRD schema is owned by an external platform operator
(not present in either repo) and has not been verified here.

**Fallback if the operator rejects an empty `path`:** mount the product under a
prefix instead — `frontend` at `path: arbeidsflate`, `bff` at
`path: arbeidsflate/api` (the Q1-B layout) — with the same backend URLs. This is
a localized change to the two `Api` `path` values; nothing else in the design
moves.

## Validation

Per the manifests validation baseline, the following must succeed after the change:

```
kustomize build manifests/environments/at23
kustomize build manifests/environments/tt02
kustomize build manifests/environments/yt01
kustomize build manifests/environments/prod
```

Each build's output must include the `arbeidsflate-frontend` and
`arbeidsflate-bff` `Api` + `Backend` resources with the correct per-env URLs.

## Out of scope

- No changes to Deployments, Services, ConfigMaps, ExternalSecrets, HPAs,
  ApplicationIdentities, or the existing HTTPRoutes.
- No maintenance-mode policy (explicitly deferred — Q2 = A).
- No new environments; the existing four (at23, tt02, yt01, prod) only.
