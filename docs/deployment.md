# Production Deployment

OpenKOS production builds the multi-stage FrankenPHP image directly on the
Coolify server from `.docker/php/Dockerfile.prod`. GHCR publishing remains an
optional release path; a registry credential is not required for the default
Coolify deployment.

The production Compose reference contains only the application roles:

- `web`: FrankenPHP HTTP server on container port `8080`
- `queue`: database-backed Laravel queue worker
- `scheduler`: Laravel scheduler

PostgreSQL is external. Database-backed cache and sessions remain the default,
and production does not require Redis.

## Coolify setup

Create a normal Git-based Docker Compose application in Coolify. Use these
settings:

- Repository: `https://github.com/rayhanvito/OpenKos`
- Build strategy: Docker Compose
- Base directory: `/`
- Compose file: `compose.production.yaml`
- Mode: normal Git-based Compose, not Raw Compose

Set the HTTPS domain on the `web` service and route it to internal port `8080`.
Do not publish or attach domains to `queue` or `scheduler`. Coolify's proxy
terminates TLS; the containers remain HTTP-only inside the Compose network.

The Compose file builds all three services from the same Dockerfile and tags the
local image as `openkos:coolify` by default. Keep `OPENKOS_IMAGE` unset for this
mode. If you later switch to a published registry image, set it to a version
tag or digest, never `latest` for a stable release. For example:

```text
OPENKOS_IMAGE=ghcr.io/rayhanvito/openkos:1.0.0
```

If the GHCR package is private, configure a read-only package credential in
Coolify before deploying.

## Configuration

Enter production variables and secrets in Coolify's Environment Variables
panel. Do not commit `.env.production` or secret values to the repository. At
minimum configure:

```text
APP_NAME=OpenKOS
APP_ENV=production
APP_DEBUG=false
APP_URL=https://<deployment-domain>
APP_KEY=<generated Laravel key>
TRUSTED_PROXIES=<trusted Coolify proxy address or network>
DB_CONNECTION=pgsql
DB_HOST=<PostgreSQL host>
DB_PORT=5432
DB_DATABASE=<PostgreSQL database>
DB_USERNAME=<PostgreSQL username>
DB_PASSWORD=<PostgreSQL password>
SESSION_DRIVER=database
CACHE_STORE=database
QUEUE_CONNECTION=database
FILESYSTEM_DISK=local
MAIL_MAILER=smtp
MAIL_HOST=<SMTP host>
MAIL_PORT=587
MAIL_USERNAME=<SMTP username>
MAIL_PASSWORD=<SMTP password>
MAIL_FROM_ADDRESS=<verified sender>
MAIL_FROM_NAME=OpenKOS
OPENKOS_UPLOAD_VOLUME=openkos_uploads
LARAVEL_OPTIMIZE=true
```

Generate `APP_KEY` with `php artisan key:generate --show` before entering it.
Use `TRUSTED_PROXIES=*` only when the application is unreachable except through
the trusted private proxy network.

The Compose-managed `openkos_uploads` volume is mounted at
`/app/storage/app/private` for all three services. This covers tenant
documents, payment proofs, branding assets, and runtime plugins.

## Branding storage verification

Logo and favicon files use `FILESYSTEM_DISK` and are streamed through the
application's branding routes, including when the disk is private. Automated
coverage uses Laravel's local fake disk; there is no remote-storage test
harness. Before enabling a remote or private production disk, manually verify
upload, replacement, removal, and the `Content-Type` returned by both branding
routes. Also verify that an Inertia navigation after a favicon change updates
the browser tab; this remains an integration check rather than a browser test.

The application is HTTP-only inside the Compose network. Terminate TLS in
Traefik, Caddy, Cloudflare Tunnel, or another external load balancer and
forward `X-Forwarded-For`, `X-Forwarded-Host`, `X-Forwarded-Port`, and
`X-Forwarded-Proto`. Configure `TRUSTED_PROXIES` only for proxies that are
actually trusted. `TRUSTED_PROXIES=*` is appropriate only when the application
is unreachable except through a trusted private proxy network.

## First installation and release sequence

Deploy the Coolify resource once, then open a terminal for the `web` service and
run the interactive installer:

```bash
php artisan app:install
```

Choose `ID`, `Asia/Jakarta`, the real HTTPS application URL, and the owner
credentials. The installer runs the initial migrations and creates the owner
account. Do not use `--fresh` after real data exists.

For later releases, publish a new immutable image tag, update `OPENKOS_IMAGE`,
and deploy from Coolify. Run pending migrations once from the `web` service:

```bash
php artisan migrate --force
```

Migrations are never run by web, queue, or scheduler startup. Each long-lived
container initializes Laravel's runtime caches using the configuration injected
into that container before starting its process. This keeps configuration-
specific cache files in the actual runtime container.

After every release, verify the health endpoint and application flow:

```bash
curl --fail --silent --show-error https://<deployment-domain>/up
```

Then verify the landing page, owner login, queue, scheduler, and one persisted
upload. The current route set supports route caching; if future routes use
closures, remove route caching from the runtime optimization step before
deployment.

The `openkos_uploads` volume is durable across container replacement on the same
Docker host, but it is not a cross-host or multi-replica filesystem. Back it up
before replacing the host. If the application is later scaled across hosts,
move uploads to shared object storage or shared storage first.

Runtime plugin packages use the same private persistent storage under
`storage/app/private/plugins`. Keep that path available to the web, queue, and scheduler
containers. After installing, enabling, disabling, or updating a runtime plugin, restart
FrankenPHP workers, queue workers, and the scheduler so each process boots the new plugin
set. Runtime plugin installation never changes the root Composer files.

## Backup and rollback

Back up PostgreSQL daily with retention outside the application container. Back
up the `openkos_uploads` volume, including `storage/app/private/plugins`, tenant
documents, payment proofs, and branding files. Rehearse a restore into a
non-production Coolify resource before relying on the backup.

Keep the previous image tag or digest in the release notes. To roll back, set
`OPENKOS_IMAGE` back to that value and redeploy. Restore PostgreSQL only when an
incompatible migration requires it; never use `migrate:fresh` for rollback.

## Optional GHCR image publishing

The stable workflow publishes version tags `1.2.3`, `1.2`, `1`, and `latest`
when GitHub Actions is available. Nightly builds publish `nightly` and an
immutable tag containing the UTC build date and commit SHA. Nightly builds never
move `latest`.

Coolify does not need these registry images in the default configuration because
`compose.production.yaml` builds the image locally. Use a version tag or digest
only when intentionally switching to registry-based deployments.
