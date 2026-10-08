# Coolify Docker Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy OpenKOS to a VPS through Coolify as a production Docker Compose application with FrankenPHP web, queue, scheduler, external PostgreSQL, persistent private uploads, and a repeatable rollback path.

**Architecture:** Use the existing multi-stage image from `.docker/php/Dockerfile.prod`, publish it to `ghcr.io/rayhanvito/openkos`, and run `compose.production.yaml` as a normal Git-based Compose application in Coolify. The `web` service receives the public domain on internal port `8080`; `queue` and `scheduler` stay internal. Coolify supplies runtime variables, PostgreSQL is external, and the named upload volume is mounted at `/app/storage/app/private`.

**Tech Stack:** Docker Buildx, FrankenPHP/PHP 8.4, Laravel 13, React/Inertia/Vite, PostgreSQL, GitHub Actions, GHCR, Coolify, Traefik-managed HTTPS.

## Global Constraints

- Keep the production Dockerfile at `.docker/php/Dockerfile.prod` and the build context at the repository root.
- Keep the runtime image HTTP-only on internal port `8080`; terminate TLS in Coolify's proxy.
- Use normal Git-based Docker Compose mode in Coolify, not Raw Compose mode.
- Never commit `.env.production`, `APP_KEY`, database passwords, SMTP passwords, registry tokens, or other secrets.
- Pin production to a version tag or digest; do not use `latest` for the stable deployment.
- Keep PostgreSQL external to `compose.production.yaml`.
- Persist and back up `/app/storage/app/private`, including runtime plugins.
- Preserve the existing web, queue, scheduler, and health-check roles.
- Do not modify third-party GitHub URLs or dependency metadata.

## Audit Baseline

- `docker buildx build --check --file .docker/php/Dockerfile.prod .` passed with no warnings.
- Plain `docker compose -f compose.production.yaml config` fails when `.env.production` is absent because all three services use `env_file`.
- Compose parses successfully when `OPENKOS_ENV_FILE=.env.example` is supplied.
- Production Compose has no PostgreSQL service and expects an external database.
- Both image-publishing workflows already target `ghcr.io/rayhanvito/openkos`.

## Files to Change

- Modify `compose.production.yaml`: remove the required repository-relative env file, pass Coolify-managed variables to all services, and expose only internal web port `8080`.
- Modify `.env.example`: remove obsolete host-port variables and document that production values are entered in Coolify.
- Modify `docs/deployment.md`: add the Coolify setup, variables, first install, smoke checks, backup, and rollback runbook.
- Leave `.docker/php/Dockerfile.prod` and the two publish workflows unchanged unless a concrete build or registry test fails.

---

### Task 1: Make production Compose compatible with Coolify

**Files:**
- Modify: `compose.production.yaml:1-109`
- Modify: `.env.example:1-20`
- Test: Docker Compose rendered configuration

**Interfaces:**
- Consumes: Coolify environment variables and `OPENKOS_IMAGE`.
- Produces: A Compose file that parses without `.env.production`, gives identical Laravel runtime configuration to `web`, `queue`, and `scheduler`, and exposes only internal port `8080`.

- [x] **Step 1: Capture the current failure.**

Run:

```powershell
docker compose -f compose.production.yaml config
```

Expected: the command reports that `.env.production` is missing.

- [x] **Step 2: Replace `env_file` with an explicit shared environment mapping.**

Add a top-level YAML anchor and merge it into each service:

```yaml
x-openkos-environment: &openkos-environment
    APP_NAME: '${APP_NAME:-OpenKOS}'
    APP_ENV: '${APP_ENV:-production}'
    APP_KEY: '${APP_KEY}'
    APP_DEBUG: '${APP_DEBUG:-false}'
    APP_URL: '${APP_URL}'
    TRUSTED_PROXIES: '${TRUSTED_PROXIES:-}'
    DB_CONNECTION: '${DB_CONNECTION:-pgsql}'
    DB_HOST: '${DB_HOST}'
    DB_PORT: '${DB_PORT:-5432}'
    DB_DATABASE: '${DB_DATABASE}'
    DB_USERNAME: '${DB_USERNAME}'
    DB_PASSWORD: '${DB_PASSWORD}'
    SESSION_DRIVER: '${SESSION_DRIVER:-database}'
    CACHE_STORE: '${CACHE_STORE:-database}'
    QUEUE_CONNECTION: '${QUEUE_CONNECTION:-database}'
    FILESYSTEM_DISK: '${FILESYSTEM_DISK:-local}'
    MAIL_MAILER: '${MAIL_MAILER:-smtp}'
    MAIL_HOST: '${MAIL_HOST}'
    MAIL_PORT: '${MAIL_PORT:-587}'
    MAIL_USERNAME: '${MAIL_USERNAME:-null}'
    MAIL_PASSWORD: '${MAIL_PASSWORD:-null}'
    MAIL_FROM_ADDRESS: '${MAIL_FROM_ADDRESS}'
    MAIL_FROM_NAME: '${MAIL_FROM_NAME:-OpenKOS}'
    OPENKOS_MARKETPLACE_URL: '${OPENKOS_MARKETPLACE_URL:-https://marketplace.openkos.id}'

services:
    web:
        environment:
            <<: *openkos-environment
            LARAVEL_OPTIMIZE: '${LARAVEL_OPTIMIZE:-true}'
```

Include the remaining non-secret variables already documented in `.env.example`—locale, logging, session, Redis, AWS, and marketplace limits—in the same anchor. Remove all three `env_file` blocks. Do not place secret values directly in the file.

- [x] **Step 3: Remove host binding from the public service.**

Replace `web.ports` with:

```yaml
expose:
    - '8080'
```

Keep the existing health check at `http://127.0.0.1:8080/up`. Do not publish ports for `queue` or `scheduler`; Coolify will route the domain to the web service's internal port.

- [x] **Step 4: Align the example environment file.**

Remove `OPENKOS_HTTP_BIND` and `OPENKOS_HTTP_PORT` from `.env.example`. Add a comment stating that production variables and secrets are entered in Coolify's Environment Variables panel, not committed to Git.

- [x] **Step 5: Validate the rendered Compose configuration.**

Run:

```powershell
docker compose --env-file .env.example -f compose.production.yaml config --quiet
```

Expected: exit code `0`.

Run:

```powershell
$rendered = docker compose --env-file .env.example -f compose.production.yaml config
if ($rendered -match 'env_file:') { throw 'env_file must not remain' }
if ($rendered -notmatch 'expose:.*8080') { throw 'web must expose 8080' }
```

Expected: no exception.

- [ ] **Step 6: Commit the Compose change.**

```bash
git add compose.production.yaml .env.example
git commit -m "fix: prepare production compose for Coolify"
```

---

### Task 2: Build and publish an immutable production image

**Files:**
- Inspect: `.docker/php/Dockerfile.prod:1-124`
- Inspect: `.github/workflows/publish-stable-image.yml:1-57`
- Inspect: `.github/workflows/publish-nightly-image.yml:1-61`
- Test: Dockerfile syntax, local image build, and GHCR manifest

**Interfaces:**
- Consumes: the repository source and GitHub Actions package-write permission.
- Produces: `ghcr.io/rayhanvito/openkos:<version>` for Coolify.

- [x] **Step 1: Validate Dockerfile syntax.**

```powershell
docker buildx build --check --file .docker/php/Dockerfile.prod .
```

Expected: exit code `0` and no warnings.

- [x] **Step 2: Build a local image.**

```powershell
docker buildx build --file .docker/php/Dockerfile.prod --tag openkos:coolify-check --load .
```

Expected: the build completes and loads an image exposing port `8080` with the existing entrypoint and FrankenPHP command.

- [ ] **Step 3: Publish a stable semver tag.**

Create a release tag such as `v1.0.0` after Task 1 is committed. The existing stable workflow publishes `1.0.0`, `1.0`, `1`, and `latest`.

- [ ] **Step 4: Verify the registry manifest.**

```powershell
docker buildx imagetools inspect ghcr.io/rayhanvito/openkos:1.0.0
```

Expected: the tag resolves and includes `linux/amd64` and `linux/arm64`. Configure a read-only GHCR credential in Coolify if the package is private.

- [ ] **Step 5: Pin Coolify to the immutable release.**

Set `OPENKOS_IMAGE=ghcr.io/rayhanvito/openkos:1.0.0` for the first deployment. After inspection, record the digest and prefer `ghcr.io/rayhanvito/openkos@sha256:<digest>` for repeatable redeployments.

---

### Task 3: Provision VPS and Coolify resources

**Files:**
- Modify: Coolify project/environment settings outside the repository
- Reference: `compose.production.yaml`
- Reference: `docs/deployment.md`

**Interfaces:**
- Consumes: the GitHub repository `rayhanvito/OpenKos`, published GHCR image, deployment domain, and PostgreSQL connection.
- Produces: one Coolify Compose resource with database connectivity, domain, secrets, and persistent storage.

- [ ] **Step 1: Prepare the VPS.**

Confirm Coolify is installed, Docker is available, TCP ports `80` and `443` are open, DNS points to the VPS, and at least 20 GB disk is free.

- [ ] **Step 2: Provision PostgreSQL.**

Create a PostgreSQL resource in the same Coolify project/environment or use a trusted managed PostgreSQL instance. Keep the database private and record its host, port, database, username, and password.

- [ ] **Step 3: Create the Git-based Compose application.**

Use these Coolify settings:

```text
Repository: https://github.com/rayhanvito/OpenKos
Build strategy: Docker Compose
Base directory: /
Compose location: compose.production.yaml
Mode: normal Git-based application, not Raw
```

Coolify's normal Compose mode generates management and proxy configuration; Raw mode requires those labels and networking details to be maintained manually ([Coolify Docker Compose docs](https://coolify.io/docs/applications/builds/docker-compose)).

- [ ] **Step 4: Enter the runtime variables.**

Set at minimum:

```text
OPENKOS_IMAGE=ghcr.io/rayhanvito/openkos:1.0.0
OPENKOS_UPLOAD_VOLUME=openkos_uploads
LARAVEL_OPTIMIZE=true
APP_NAME=OpenKOS
APP_ENV=production
APP_DEBUG=false
APP_URL=https://<deployment-domain>
APP_KEY=<generated Laravel key>
TRUSTED_PROXIES=<trusted Coolify proxy address or network>
DB_CONNECTION=pgsql
DB_HOST=<PostgreSQL host>
DB_PORT=<PostgreSQL port>
DB_DATABASE=<PostgreSQL database>
DB_USERNAME=<PostgreSQL username>
DB_PASSWORD=<PostgreSQL password>
SESSION_DRIVER=database
CACHE_STORE=database
QUEUE_CONNECTION=database
FILESYSTEM_DISK=local
MAIL_MAILER=smtp
MAIL_HOST=<SMTP host>
MAIL_PORT=<SMTP port>
MAIL_USERNAME=<SMTP username>
MAIL_PASSWORD=<SMTP password>
MAIL_FROM_ADDRESS=<verified sender>
MAIL_FROM_NAME=OpenKOS
```

Generate `APP_KEY` with:

```powershell
php artisan key:generate --show
```

Use `TRUSTED_PROXIES=*` only when the application is unreachable except through the trusted Coolify proxy; otherwise enter the actual trusted proxy address/network.

- [ ] **Step 5: Configure the domain.**

Attach the HTTPS domain to `web` and set the receiving internal port to `8080`. Do not attach domains to `queue` or `scheduler`.

- [ ] **Step 6: Verify persistent storage.**

Confirm `openkos_uploads` is mounted at `/app/storage/app/private` for all three services. This covers tenant documents, payment proofs, branding assets, and runtime plugins.

---

### Task 4: First installation and deployment verification

**Files:**
- Reference: `app/Console/Commands/InstallCommand.php:14-73`
- Reference: `compose.production.yaml`
- Test: Coolify deployment logs, health checks, and application smoke tests

**Interfaces:**
- Consumes: the running PostgreSQL resource, Coolify variables, image, DNS, and TLS.
- Produces: migrated schema, initial owner account, healthy services, and a reachable HTTPS landing page.

- [ ] **Step 1: Deploy once and inspect logs.**

Deploy the Coolify resource and confirm image pull, environment interpolation, container creation, and health-check status for `web`, `queue`, and `scheduler`.

- [ ] **Step 2: Run the first-install command in the web container.**

```bash
php artisan app:install
```

Use `ID`, `Asia/Jakarta`, the real HTTPS application URL, and the owner credentials. Do not use `--fresh` after real data exists.

- [ ] **Step 3: Verify the health endpoint.**

```bash
curl --fail --silent --show-error https://<deployment-domain>/up
```

Expected: HTTP success.

- [ ] **Step 4: Verify the user-facing flow.**

Open `https://<deployment-domain>/`, confirm the public listings page, log in as owner, open the dashboard, and confirm the About repository link points to `https://github.com/rayhanvito/OpenKos`.

- [ ] **Step 5: Verify queue and scheduler.**

Confirm runtime logs show `php artisan queue:work` and `php artisan schedule:work` running continuously. Trigger one queued application action and verify completion.

- [ ] **Step 6: Verify volume persistence.**

Upload one branding asset or tenant document, redeploy the same image, and confirm the file remains available.

---

### Task 5: Document releases, backups, and rollback

**Files:**
- Modify: `docs/deployment.md:1-97`
- Reference: `compose.production.yaml`
- Test: backup/restore rehearsal and rollback deployment

**Interfaces:**
- Consumes: the deployed Coolify resource, PostgreSQL backup tooling, and Docker volume backup tooling.
- Produces: an operator runbook for releases and recovery.

- [x] **Step 1: Add the Coolify runbook.**

Document repository/Compose settings, required variables, web port `8080`, external PostgreSQL, GHCR authentication, volume path, and the first-install command. Link to the official Coolify Compose and health-check documentation ([health checks](https://coolify.io/docs/applications/configuration/health-checks)).

- [x] **Step 2: Document the release sequence.**

Publish the new image tag, update `OPENKOS_IMAGE`, deploy, run pending migrations once, and verify `/up`, landing page, login, queue, scheduler, and persistent files. Keep migrations backward-compatible with the previous image.

- [x] **Step 3: Document backups.**

Back up PostgreSQL daily with retention outside the application container. Back up the `openkos_uploads` volume, including `storage/app/private/plugins`, documents, payment proofs, and branding files. Rehearse restore into a non-production resource.

- [x] **Step 4: Document rollback.**

Keep the previous image tag or digest in release notes. Roll back by restoring `OPENKOS_IMAGE` to that version and redeploying. Restore the database only when an incompatible migration requires it. Never use `migrate:fresh` in rollback.

- [ ] **Step 5: Commit the runbook.**

```bash
git add docs/deployment.md
git commit -m "docs: add Coolify deployment and rollback runbook"
```

## Final Verification Checklist

- [x] Dockerfile syntax check exits `0`.
- [x] Full single-platform image build exits `0`.
- [x] Compose renders with `.env.example` and no `.env.production` dependency.
- [ ] GHCR contains the selected immutable image tag or digest.
- [ ] Coolify targets `web:8080` and has no public queue/scheduler domain.
- [ ] PostgreSQL connection succeeds and `app:install` completes.
- [ ] `/up`, `/`, login, queue, scheduler, and private uploads work after redeploy.
- [ ] PostgreSQL and `openkos_uploads` restore procedures have been rehearsed.
- [ ] Previous image tag/digest is recorded and rollback has been tested.

## Self-Review

- The plan covers one deployable subsystem: the existing Docker image, its Coolify Compose runtime, and its operational runbook.
- It addresses the verified `.env.production` parse failure, internal port routing, external PostgreSQL, persistent storage, image immutability, first install, health checks, and rollback.
- It introduces no new application dependency, database container, or runtime abstraction.
- Deployment-specific values are entered through Coolify rather than committed to the repository.
