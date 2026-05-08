# lazy-multica — project context for Claude Code

懒猫微服 lpk wrapper for [multica-ai/multica](https://github.com/multica-ai/multica).

## Architecture

**Retag-only main image + 2 pre-mirrored dependencies** (multi-service stack).

- `main` = upstream `ghcr.io/multica-ai/multica-web` (Next.js 16 standalone, port 3000).
  `docker/Dockerfile` is a single-line `FROM ghcr.io/multica-ai/multica-web@sha256:...`
  so the lazycat-ci `lpk-build.yml` reusable mirrors it into the
  lazycat registry as `${LAZYCAT_IMAGE}`.
- `backend` = upstream `ghcr.io/multica-ai/multica-backend` (Go server +
  REST + WebSocket, port 8080), pre-mirrored to `registry.lazycat.cloud`.
- `postgres` = `pgvector/pgvector:pg17`, pre-mirrored to `registry.lazycat.cloud`.

No upstream source vendoring, no `patches/`. Multica's modified Apache
2.0 license restricts SaaS hosting + commercial embedding + frontend
logo modification (see `LICENSE`); we redistribute only the prebuilt
images upstream publishes themselves.

## Routing

Lazycat reverse-proxies a single subdomain (`multica.<box-domain>`)
to `main:3000`. The Next.js web image already proxies internally
via `next.config.ts` rewrites:

| Public path  | Next.js rewrites to (REMOTE_API_URL=http://backend:8080) |
|--------------|----------------------------------------------------------|
| `/api/*`     | `http://backend:8080/api/*`                              |
| `/ws`        | `http://backend:8080/ws`                                 |
| `/auth/*`    | `http://backend:8080/auth/*`                             |
| everything else | served by Next.js                                     |

WebSocket upgrade flows through the Next.js standalone server's
rewrite handler — confirmed in upstream `apps/web/next.config.ts`.

## Service wiring

| Env var (consumer)      | Source                                         |
|-------------------------|------------------------------------------------|
| `DATABASE_URL` (backend)| Constructed inline; password = `stable_secret` |
| `JWT_SECRET` (backend)  | `stable_secret "JWT_SECRET"`                   |
| `MULTICA_APP_URL`       | `https://{{.S.AppDomain}}` (lazycat-injected)  |
| `FRONTEND_ORIGIN`       | same as `MULTICA_APP_URL`                      |
| `CORS_ALLOWED_ORIGINS`  | same as `MULTICA_APP_URL`                      |
| `GOOGLE_REDIRECT_URI`   | `https://{{.S.AppDomain}}/auth/callback`       |
| `RESEND_*` / `GOOGLE_*` | deploy_params (optional)                       |
| `ALLOW_SIGNUP` / `ALLOWED_*` | deploy_params (optional)                  |
| `MULTICA_DEV_VERIFICATION_CODE` | deploy_param (optional). When set, sprig `if` flips `APP_ENV` from `production` → `development` so upstream honors the fixed code. |
| `APP_ENV`               | `production` by default; flipped to `development` only if `MULTICA_DEV_VERIFICATION_CODE` is set |
| `POSTGRES_PASSWORD` (postgres + DATABASE_URL) | `stable_secret "POSTGRES_PASSWORD"` — same key on both ends |

`.S.AppDomain` is a built-in lazycat template var that resolves to
`<subdomain>.<box-domain>` (e.g. `multica.maolv.heiyu.space`) at render
time, so the wrapper needs no required deploy_param. Other built-ins
under `.S`: `BoxName`, `BoxDomain`, `AppDomain`, `OSVersion`,
`IsMultiInstance`, `DeployUID`, `DeployID`. Reference: lazycat
[advanced-manifest-render](https://developer.lazycat.cloud/advanced-manifest-render.html).

`stable_secret` produces 128 hex chars — fits Postgres passwords and JWT
HMAC secrets without trimming.

## depends_on / startup ordering

Compose `depends_on:` is start-order-only, NOT health-gated. So:

- `backend` may start before postgres accepts connections → `migrate up`
  fails → container exits → lazycat restarts → succeeds on retry.
- `main` may start before backend's `/api/health` is up → Next.js still
  serves static pages, and once backend recovers requests start working.

We rely on lazycat's default restart-on-failure to converge. No custom
wait-for script — multica's image is alpine-minimal and we don't vendor
to add one.

## Bumping the upstream multica images

Re-resolve the three digests:

```sh
# multica-web
TOKEN=$(curl -fsSL 'https://ghcr.io/token?service=ghcr.io&scope=repository:multica-ai/multica-web:pull' | jq -r .token)
curl -sSI "https://ghcr.io/v2/multica-ai/multica-web/manifests/v0.2.27" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.oci.image.index.v1+json" \
  -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  | grep -i digest

# multica-backend (same shape)
# pgvector
TOKEN=$(curl -fsSL 'https://auth.docker.io/token?service=registry.docker.io&scope=repository:pgvector/pgvector:pull' | jq -r .token)
curl -sSI "https://registry-1.docker.io/v2/pgvector/pgvector/manifests/pg17" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.docker.distribution.manifest.list.v2+json" \
  | grep -i digest
```

1. Update `docker/Dockerfile` (the `FROM ... @sha256:...` line) and the
   matching version tag.
2. Re-run `lzc-cli appstore copy-image` for backend + postgres if their
   digests change:

   ```sh
   lzc-cli appstore copy-image ghcr.io/multica-ai/multica-backend:vX.Y.Z
   lzc-cli appstore copy-image docker.io/pgvector/pgvector:pg17
   # update the registry.lazycat.cloud/lee/... URLs in lzc-manifest.template.yml
   ```

3. Bump `version` and `git tag vX.Y.Z`.

## Release flow (per skill rules — read before tagging)

1. `git tag v0.0.1 && git push origin v0.0.1` — `release.yml` runs.
   `publish-appstore` step **fails** on first release (app not yet
   registered) — that is expected. The `.lpk` is still attached to the
   GitHub Release.
2. `gh release download v0.0.1 -p '*.lpk'` →
   `scp` to your test box → `lpk-manager install …` → smoke-test in a
   real browser. Capture real screenshots, source the real icon from
   the upstream marketing site or first-page UI.
3. Bump to `v0.0.2`, replace placeholder icon + screenshots, run
   `bootstrap-app.yml` (no `app_id`) — that's the version that gets
   locked into the appstore review record.

## Known limitations

- **AI execution is local, not on the box.** Multica's design separates
  the server (this lpk) from the daemon (which runs on each user's own
  machine and shells out to `claude` / `codex` / etc.). The lpk by
  itself doesn't run any AI model.
- **OAuth callback URL is auto-derived.** It tracks the lazycat
  `AppDomain`, so it changes only if the box domain itself changes.
  When configuring Google OAuth credentials, set the redirect URI to
  `https://multica.<your-box-domain>/auth/callback` to match.
- **License is modified Apache 2.0.** Internal use is fine; SaaS
  hosting / commercial embedding / frontend logo removal are not.

## Source-of-truth files

- `docker/Dockerfile` — pins the **web** (main) image
- `lazycat/lzc-manifest.template.yml` — pins **backend** + **postgres**
  via `registry.lazycat.cloud/lee/...` URLs
- `lazycat/lzc-deploy-params.yml` — install-time inputs
- `lazycat/package.template.yml` — appstore metadata; `version` is
  substituted from CI
- `lazycat/appstore.yml` — extra metadata used by the appstore submit
  step (description, screenshots, keywords, source)
