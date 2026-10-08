# url-shortener chart

FastAPI backend + Next.js frontend behind the k3s-bundled Traefik Ingress. The
backend owns every path except the frontend's page (`/`) and assets (`/_next`,
`/favicon.ico`), so short links (`/<code>`) and `/api/*` both reach it.

## Deploy

`git push`. ArgoCD renders this chart with `values.yaml` + `values-<env>.yaml`
for each Application in `apps/` and syncs it (see the repo README). Never
`helm upgrade` it by hand: self-heal would revert it.

| env  | values           | namespace            | URL                      |
|------|------------------|----------------------|--------------------------|
| dev  | `values-dev.yaml`  | `url-shortener-dev`  | https://dev.supabase.win |
| prod | `values-prod.yaml` | `url-shortener-prod` | https://supabase.win     |

## What lives outside the chart

The namespaces and the `backend-secret` (`DATABASE_URL`) and `origin-tls`
Secrets are created by `scripts/bootstrap-cluster.sh` in url-shortener-infra,
not the chart, so pruning or deleting the
Application never deletes credentials and the password never lands in git.

## Behaviour worth knowing

- **Migrations** run as a `pre-install`/`pre-upgrade` hook Job
  (`alembic upgrade head`) before new backend pods start. Disable with
  `backend.migrations.enabled=false`.
- Changing `backend.config` or `publicUrl` rolls the backend pods automatically
  (`checksum/config` annotation).
- `publicUrl` is required; the backend builds short links from it.
- With `ingress.tlsSecret` set, Traefik serves the routes on 443 only; plain
  HTTP to the node returns 404. That's fine behind Cloudflare, which always
  connects over HTTPS ("Full (strict)") and redirects visitors to HTTPS.
- Images are still `:latest`, so a new image is picked up only when pods
  restart. Next step: pin `backend.image.tag`/`frontend.image.tag` to a git
  SHA (from the app repo's CI, or ArgoCD Image Updater).
