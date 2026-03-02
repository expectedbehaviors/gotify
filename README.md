# Gotify Helm Chart

Baseline for [Gotify](https://gotify.net) (self-hosted push notifications). Uses [bjw-s app-template](https://github.com/bjw-s/app-template). Override TZ, ingress host, and TLS in your values. Use OIDC at ingress (e.g. nginx + oauth2-proxy) to protect the app.

## Subcharts

| Subchart | Source | Values prefix | Description |
|----------|--------|---------------|-------------|
| **gotify** (app-template) | [bjw-s helm-charts](https://github.com/bjw-s/helm-charts) | `gotify.*` | App template: controllers, persistence, ingress, env. |
| **onepassworditem** | [expectedbehaviors/OnePasswordItem-helm](https://github.com/expectedbehaviors/OnePasswordItem-helm) | `onepassworditem.*` | Optional secrets sync into the release namespace. |

All inputs: **`gotify.*`** (controllers, persistence, ingress, env), **`onepassworditem.enabled`**, **`onepassworditem.items`**. Defaults: see `values.yaml`.

## Configuration reference (all inputs)

Every value is documented below. Source: [bjw-s app-template](https://github.com/bjw-s/helm-charts) (as `gotify.*`) and [expectedbehaviors/OnePasswordItem-helm](https://github.com/expectedbehaviors/OnePasswordItem-helm) (as `onepassworditem.*`).

### Subchart: gotify (bjw-s app-template)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `gotify.global.fullnameOverride` | string | — | Override full name (e.g. `gotify`). |
| `gotify.defaultPodOptions.securityContext` | object | — | Default pod security. |
| `gotify.controllers.main.enabled` | bool | `true` | Enable main controller. |
| `gotify.controllers.main.type` | string | `"deployment"` | Controller type. |
| `gotify.controllers.main.replicas` | int | `1` | Replicas. |
| `gotify.controllers.main.containers.main.image.repository` | string | — | Image (e.g. `gotify/server`). |
| `gotify.controllers.main.containers.main.image.tag` | string | — | Image tag. |
| `gotify.controllers.main.containers.main.env` | object | — | Env vars (TZ, GOTIFY_DEFAULTUSER_PASS via secretRef, etc.). |
| `gotify.controllers.main.containers.main.probes` | object | — | liveness, readiness, startup. |
| `gotify.controllers.main.containers.main.resources` | object | — | requests/limits. |
| `gotify.service.main.enabled` | bool | `true` | Enable main service. |
| `gotify.service.main.ports` | object | — | Ports (e.g. http: 80). |
| `gotify.ingress.main.enabled` | bool | `true` | Enable ingress. |
| `gotify.ingress.main.hosts` | list | — | Hosts (host, paths, pathType). |
| `gotify.ingress.main.tls` | list | `[]` | TLS (secretName, hosts). |
| `gotify.ingress.main.annotations` | object | `{}` | Ingress annotations. |
| `gotify.persistence.data.enabled` | bool | `true` | Enable data PVC. |
| `gotify.persistence.data.size` | string | — | PVC size (e.g. 1Gi). |
| `gotify.persistence.data.existingClaim` | string | — | Use existing PVC. |

### Subchart: onepassworditem

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `onepassworditem.enabled` | bool | `true` | Create OnePasswordItem resources; set `false` if supplying secrets another way. |
| `onepassworditem.defaultVault` | string | `""` | Default vault for items. |
| `onepassworditem.items` | list | `[]` | List of `{ item, name, type }`; `name` must match Secret refs in container env. |

## Chart contents

- **App:** Gotify (gotify/server) via bjw-s app-template; port 80.
- **Secrets:** Optional — use onepassworditem or a Secret for `GOTIFY_DEFAULTUSER_PASS` (default admin password) via `envFrom` or `valueFrom.secretKeyRef`.
- **Ingress:** Host/path and TLS; enable OIDC at ingress so the app is not publicly open.
- **Persistence:** Single PVC for `/app/data` (SQLite DB, app config); chart creates **data** PVC or use `existingClaim`.

## Prerequisites

- **1Password Connect** (optional): if you use `onepassworditem.enabled: true` to sync credentials. The chart depends on [expectedbehaviors/OnePasswordItem-helm](https://github.com/expectedbehaviors/OnePasswordItem-helm); secrets are created in the release namespace (`.Release.Namespace`). Set `onepassworditem.items[]` with `{ item, name, type }` and reference the Secret name in your app-template env.

## Requirements

| Dependency | Notes |
|------------|--------|
| **PVC** | Chart creates **data** PVC (default 1Gi); or set `existingClaim`. |
| **Namespace** | e.g. `gotify`; create or use Argo CD project. |
| **OIDC** | Configure nginx ingress auth (oauth2-proxy) for this host so the app is not public. |

## Usage

Override in your values (or Argo CD):

- **Ingress:** `gotify.ingress.main.hosts[0].host`, `gotify.ingress.main.tls` (e.g. `gotify.example.com`, TLS secret name).
- **TZ:** `gotify.controllers.main.containers.main.env.TZ`.
- **1Password:** Set `onepassworditem.enabled: true` and `onepassworditem.items[]`; reference Secret names in container env (e.g. `valueFrom.secretKeyRef`).
- **First run:** Default admin user is `admin`; set password on first login or provide `GOTIFY_DEFAULTUSER_PASS` via a Secret. Create applications in the Gotify UI for API tokens (scripts, CI/CD, notifications).

## Key values

| Area | Where | What to set |
|------|--------|-------------|
| Persistence | `gotify.persistence.data` | size, storageClass, or existingClaim. |
| Ingress | `gotify.ingress.main.hosts` | Host and TLS for your domain. |
| Default password | Optional Secret with key `GOTIFY_DEFAULTUSER_PASS` | Set via envFrom or valueFrom if desired. |

## Values (onepassworditem)

| Key | Description |
|-----|-------------|
| `onepassworditem.enabled` | If `true` (default), the subchart creates OnePasswordItem resources. Set `false` if you supply secrets another way. |
| `onepassworditem.items` | List of `{ item, name, type }`. `name` must match the Secret name used in your container env. |

## Install

**From this repo:**

```bash
helm dependency update
helm install gotify . -f my-values.yaml -n gotify --create-namespace
```

**From Helm repo (expectedbehaviors):**

```bash
helm repo add expectedbehaviors https://expectedbehaviors.github.io/gotify
helm install gotify expectedbehaviors/gotify -f my-values.yaml -n gotify --create-namespace
```

## Render & validation

Run `helm dependency update` first, then:

```bash
helm template gotify . -f my-values.yaml -n gotify
```

## Argo CD

Point your Application at this repo (path: `.`) and pass your values (e.g. from a private values repo or inline). Namespace typically `gotify`.

## Next steps

1. Enable OIDC at ingress for this host.
2. Deploy; log in and create applications/tokens for notifications.
3. Integrate with scripts or CI/CD (e.g. Gotify notifier).
