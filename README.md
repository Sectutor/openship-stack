# OpenShip — AppVault curated stack

Open-source, self-hostable deployment platform with built-in CI/CD (Apache-2.0,
10k+ stars, https://github.com/oblien/openship). Point it at a repo — it builds,
ships, routes, and TLS-terminates your app. Drive it from the web dashboard.

This is the **AppVault-curated** compose for one-click installs:

- **Env baked in** — no `.env` file, no `${VAR:?}` traps (the upstream
  `docker/docker-compose.yml` requires a repo-root `.env` and refuses to boot
  without `POSTGRES_PASSWORD`, which breaks engine installs).
- **`OPENSHIP_PUBLIC_URL` is injected by the AppVault agent** at install time
  (same mechanism as `SERVER_URL` for Twenty) so the browser origin is trusted
  and login works from the store launch URL. If you run this compose by hand,
  set it to your public origin (e.g. `https://openship.example.com`) or login
  will be rejected with `403 ORIGIN_REJECTED`.
- **First admin is bootstrapped automatically** by a one-shot `setup` sidecar
  (calls the internal-token-gated `POST /api/system/bootstrap-admin`), so a
  fresh install is immediately login-ready with the documented default admin.
- **Edge (OpenResty :80/:443) is excluded** — AppVault clients already run
  Caddy on 80/443. The control plane (projects, deploys to remote servers via
  SSH, dashboard) works without it; auto-domain routing + Let's Encrypt on the
  SAME box needs an edge container. Escape hatch: run any
  `ghcr.io/oblien/openship-edge` container named `openship_edge` (host
  network) and the API will use it (`OPENSHIP_EDGE_MODE=docker`).

## Default admin (first boot)

| Field | Value |
|---|---|
| Email | `admin@openship.local` |
| Password | `Openship2026!` |

Change it in-app (profile settings) or recover via
`POST /api/system/reset-admin-password` with `X-Internal-Token` from the host.

## Services

| Service | Image | Port | Notes |
|---|---|---|---|
| postgres | postgres:16-alpine | private :5432 | named volume `postgres_data` |
| redis | redis:7-alpine | private :6379 | appendonly, named volume `redis_data` |
| api | ghcr.io/oblien/openship-api:latest | 4000:4000 | mounts host docker.sock (Docker-out-of-Docker), auto-migrates on boot |
| dashboard | ghcr.io/oblien/openship-dashboard:latest | 3001:3001 | the UI (AppVault launch URL) |
| setup | curlimages/curl | — | one-shot admin bootstrap, exits 0 |

## Security notes

- The `api` mounts `/var/run/docker.sock` by design (it builds + runs your
  deployed apps on the host daemon). Run only on a trusted host.
- `BETTER_AUTH_SECRET` / `INTERNAL_TOKEN` are fixed per-stack values shared by
  every AppVault client install. Change them before exposing the instance to
  untrusted networks.
