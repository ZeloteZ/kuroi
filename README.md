# kuroi

`kuroi` is a Steam account management tool with OIDC authentication, API-key based automation, and visibility-aware account sharing.

## Production-ready highlights

- Secrets are required via environment variables (no insecure runtime fallbacks).
- Steam account passwords are encrypted at rest in the database.
- Default compose works without reverse proxy (localhost ports).
- Traefik override compose supports existing external Traefik instances.
- GitHub Actions workflow builds and pushes multi-arch Docker images to GHCR.

## 1) Configuration

Copy the template and set all required values:

```bash
cp .env.example .env
```

Minimum required:

```env
POSTGRES_PASSWORD=CHANGE_ME_DB_PASSWORD
APP_SECRET=CHANGE_ME_LONG_RANDOM_SECRET
FRONTEND_URL=http://localhost:3000
VITE_API_BASE_URL=http://localhost:8000
```

For OIDC (optional):

```env
OIDC_ENABLED=true
OIDC_ISSUER_URL=https://auth.example.com
OIDC_CLIENT_ID=your_client_id
OIDC_CLIENT_SECRET=your_client_secret_or_empty
OIDC_REDIRECT_URI=https://api.example.com/auth/oidc/callback
OIDC_TOKEN_AUTH_METHOD=auto
OIDC_USE_PKCE=true
VITE_OIDC_ENABLED=true
```

## 2) Deploy without reverse proxy

```bash
docker compose up -d --build
```

Access:

- Frontend: `http://localhost:3000`
- API: `http://localhost:8000`

## 3) Deploy behind existing Traefik

Set Traefik-specific variables in `.env`:

```env
TRAEFIK_NETWORK=traefik_proxy
TRAEFIK_ENTRYPOINT=websecure
TRAEFIK_DOMAIN=kuroi.example.com
TRAEFIK_API_DOMAIN=api.kuroi.example.com
TRAEFIK_TLS=true
```

Then deploy with the override:

```bash
docker compose -f docker-compose.yml -f docker-compose.traefik.yml up -d --build
```

Notes:

- This assumes Traefik is already running.
- `TRAEFIK_NETWORK` must already exist as an external Docker network.
- `docker-compose.traefik.yml` removes exposed host ports and routes traffic through Traefik labels.

## 4) Deploy from GHCR images (no local build)

Set these values in `.env`:

```env
GHCR_OWNER=your-github-user-or-org
IMAGE_TAG=latest
POSTGRES_PASSWORD=CHANGE_ME_DB_PASSWORD
APP_SECRET=CHANGE_ME_LONG_RANDOM_SECRET
```

If packages are private, log in first:

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u "$GHCR_USER" --password-stdin
```

Start using prebuilt images:

```bash
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
```

Behind existing Traefik:

```bash
docker compose -f docker-compose.prod.yml -f docker-compose.traefik.yml up -d
```

## 5) GHCR image publishing

Workflow: `.github/workflows/docker-publish.yml`

What it does:

- Triggers on pushes to `main`, version tags (`v*`), and manual runs.
- Builds backend + frontend images for `linux/amd64` and `linux/arm64`.
- Pushes images to `ghcr.io/<owner>/kuroi-backend` and `ghcr.io/<owner>/kuroi-frontend`.
- Uses GitHub Actions cache for faster rebuilds.

Repository requirements:

- Packages permission enabled (workflow uses `GITHUB_TOKEN`).
- Default branch should be `main` for `latest` tagging behavior.

## 6) Security checklist

Before production rollout:

- Set strong unique `APP_SECRET`.
- Set strong unique `POSTGRES_PASSWORD`.
- Use HTTPS domains for OIDC redirect and frontend URL.
- Keep `.env` out of Git (already ignored).
- Rotate API keys regularly.
