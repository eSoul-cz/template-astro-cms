# Project

Static website (Astro) with a headless CMS (Laravel + Filament). Editors work in the admin of the backend; the public site is a static
build that is regenerated and deployed by the **Publish** button.

This repository is the thin **monorepo** of the project: shared documentation and tools, and two git submodules, each its own repository
with its own pipeline. Nothing is built or deployed from here.

| Path | What |
| --- | --- |
| `frontend/` | Submodule: the Astro 7 static site (created from the `astro-cms-frontend` template). Content comes from the CMS export at build time. |
| `backend/` | Submodule: Laravel 13 + Filament 5, pages and site settings, the export API, the Publish button, the form endpoint (created from the `laravel-cms` template). |
| `docs/contract.md` | The contract between the halves (versioned, guarded by the golden export). |
| `docker/nginx.example` | One NGINX site definition for the site and the backend (backend on a subdomain; the path variant is commented). |
| `bin/add-submodules` | Adds the two repositories as submodules (`composer submodules -- <frontend-url> <backend-url>` does the same). |

## Getting started

1. Clone with the submodules: `git clone --recurse-submodules git@github.com:eSoul-cz/<repository>.git` (an existing clone: `git submodule update --init --recursive`), then `git lfs pull` where a repository keeps `assets/` or `.pen` files in LFS.
2. Agent skills: each repository commits its `skills-lock.json` and `.agents/skills/`, so they come with the clone; `npx skills experimental_install` restores or refreshes them. Optional personal layers (installed once per user, not part of the project): `npx @esoul-cz/skills install project helpers`; `npx @esoul-cz/skills layers` lists them.
3. Local environment: [Local development](#local-development), and the README of the backend and the frontend.
4. Access, asked from the project lead or devops: the GitHub repositories, the Freelo project (`CONTEXT.md` › Freelo), the Jenkins jobs of the backend and the frontend (the `Jenkins` check runs on every pull request), Komodo (read-only, for the stacks and their logs) and the project's vault in 1Password (secrets, the export token and staging logins live there, never in Git). Then read `AGENTS.md` and `CONTEXT.md`.

## Setup

1. Create the two repositories from the templates (`composer create-project esoul-cz/template-astro-cms-frontend:dev-main` and
   `composer create-project esoul-cz/template-laravel-cms:dev-main`; the variants are published as `dev-main` only, so Composer needs the version, or the Packagist-less equivalent), push each to its own GitHub repository.
2. In this repository (`git init` first): `sh bin/add-submodules <frontend-repository-url> <backend-repository-url>`, then commit
   `.gitmodules` and the two pointers. Every other checkout runs `git submodule update --init --recursive`.
3. Fill `CONTEXT.md` (archetype `C-headless`, where the backend lives, business time zone, tasking) and set up the two repositories
   as their own READMEs say.

The submodule pointers move when a half is released (`git add frontend backend`). A change to the contract is a pull request in each repository.

## Where the backend lives

NGINX serves the frontend and the backend from **one site definition** (`docker/nginx.example`). The default, and the recommendation, is a
**subdomain**; a path of the site's domain is a documented exception for a project that is forced onto one hostname. Record the choice in
`CONTEXT.md`.

| | Subdomain (default) | Path `/admin` (exception) |
| --- | --- | --- |
| Admin URL | `https://admin.example.com/admin` | `https://example.com/admin` |
| nginx | a site block (static + `/_cms/`) and an `admin.` block that routes `/admin`, `/cms-api`, `/forms`, `/livewire-*`, `/up` to php-fpm and serves `/cms-media/`, `/_cms/` and the panel assets | one block: the static site, the backend's own paths routed to php-fpm |
| `APP_URL` (backend) | `https://admin.example.com` | `https://example.com` |
| `SESSION_DOMAIN` / `SESSION_PATH` | `null` / `/` (host-only cookie; never `.example.com`) | `null` / `/admin` |
| `CMS_ADMIN_PATH_MODE` (backend) | `false` | `true`: the Livewire update and upload routes move under `/admin/livewire/` |
| `CMS_API_URL` (frontend build) | `https://admin.example.com/cms-api/v1` | `https://example.com/cms-api/v1` |
| `CMS_FORMS_ALLOWED_ORIGINS` | `https://example.com` | `https://example.com` |

- **Why a subdomain:** the admin session cookie is host-only and never reaches the public site, the admin has its own CSP and access rules (VPN or IP
  allowlist on `admin.` without touching the site), and the public site exposes no PHP routes.
- **Why the path variant needs more:** a host-only cookie is sent with every request to the host, so on a path the cookie has to be scoped to the backend
  (`SESSION_PATH=/admin`). Livewire posts its updates and uploads outside `/admin` by default, and those would no longer carry the cookie;
  `CMS_ADMIN_PATH_MODE=true` moves them under `/admin`. Test the admin login, a page save and an image upload after switching.
- CORS is never needed: the static build fetches the export server-side and visitors' browsers only navigate (the form post is a navigation).
- `CMS_API_ALLOWED_IPS` has to contain the build agent in both modes (see the setup checklist).
- Switching later means changing nginx, the backend `.env`, `CMS_API_URL` in the frontend `Jenkinsfile` and the DNS record; the exported media URLs are relative (`/_cms/<hash>`).

## Local development

```sh
# backend (see backend/README.md for the full setup)
cd backend && composer install && cp .env.example .env && php artisan key:generate && php artisan migrate
php artisan make:filament-user                      # admin at http://localhost:8080/admin
# set CMS_EXPORT_TOKEN=<random string> in backend/.env

# frontend
cd frontend && pnpm install
CMS_SOURCE=fixture pnpm dev                         # no backend needed
# or against the running backend: CMS_API_URL=http://localhost:8080/cms-api/v1, CMS_API_TOKEN=<CMS_EXPORT_TOKEN>
```

After a backend change to the exporter: `cd backend && GOLDEN_UPDATE=1 php artisan test --filter=GoldenExportTest`, then `cd ../frontend && pnpm contract:sync`
(see `docs/contract.md`, "Golden export, golden copy and fixture").

## Publish flow

```text
editor ── Publish ─▶ StartPublishCommand ─▶ Jenkins job of the frontend (CMS_PUBLISH_ID=<id>) ─▶ Checks ─▶ image (FROM scratch, dist/) ─▶ Komodo stack
   ▲  (Filament)      │ checks the export,    │                                  │ GET /cms-api/v1/export                 │ deploy hook: releases/<id>, `current` switch
   │                  │ writes a `publishes`  │                                  │ (bearer token, Zod-validated,          ▼
   │                  │ row (queued)          │                                  │  /_cms media HEAD-checked)       web/current -> releases/<id>
   └──── status ◀──── POST /cms-api/v1/publishes/<id>/status (callback, bearer token) ◀── `cms` callback of the frontend Jenkinsfile
```

1. An editor saves pages and settings in Filament. Nothing is public yet: the **Publish** page shows "unpublished changes".
2. **Publish** (`StartPublishCommand`): refuses when a publish is already queued or running, when Jenkins is not
   configured, or when the export has problems (the same rules as the Zod schema). Otherwise it stores a `publishes` row
   (`queued`) and calls `{CMS_JENKINS_JOB_URL}/buildWithParameters` with `CMS_PUBLISH_ID`.
3. The frontend job fetches `GET /cms-api/v1/export` (bearer `CMS_EXPORT_TOKEN`). While serving it, the backend copies
   every referenced image to `storage/app/cms-public/<sha256[0:20]>.<ext>`; nginx serves it at `/_cms/`. The image build validates the export with Zod,
   HEAD-checks the media, builds, runs `pnpm verify` and the Komodo hook deploys `dist/` as a new release.
   **In CI the build fails when the export is unreachable or invalid**: it never falls back to the fixture or to old data.
4. The `finally` block reports `{result, buildUrl}` to `POST /cms-api/v1/publishes/<id>/status`
   (bearer `CMS_JENKINS_CALLBACK_TOKEN`). `SUCCESS` and `UNSTABLE` mean deployed.
5. The callback is best effort. **Refresh status** polls Jenkins (queue item, then build), and a publish without a result
   after `CMS_PUBLISH_TIMEOUT` minutes (default 30) is marked failed so the button cannot stay blocked. A late callback
   is still recorded, with a note.

Media: names are content addressed, so files are immutable and cacheable for a year. `cms:prune-public-media` (daily 03:30)
deletes files that no build of the last 7 days referenced. Rollback is redeploying the previous image of the frontend.

### Setup checklist

- Backend `.env`: `CMS_EXPORT_TOKEN`, `CMS_JENKINS_JOB_URL` (`https://<jenkins>/job/<job>/job/main`, the job of the frontend), `CMS_JENKINS_USER`,
  `CMS_JENKINS_API_TOKEN`, `CMS_JENKINS_CALLBACK_TOKEN`, `CMS_API_ALLOWED_IPS`, and for the forms `CMS_FORMS_ALLOWED_ORIGINS`, `CMS_FORMS_RECIPIENTS`,
  `CMS_FORMS_RECAPTCHA_SECRET`, `TRUSTED_PROXIES`.
- Jenkins credentials (Secret Text) of the frontend job: the export token (= `CMS_EXPORT_TOKEN`) and the callback token
  (= `CMS_JENKINS_CALLBACK_TOKEN`), named in its `Jenkinsfile`.
- **`CMS_API_ALLOWED_IPS` has to contain the address of the build agent.** The build fetches `/cms-api` over the public
  URL, so a changed agent address makes the export (and so every publish) answer 403 until the list is updated. Record the address in the frontend `README.md`.
- nginx: `docker/nginx.example`. The static release root (`web/current`) and the backend directories (`storage/app/cms-public`, `storage/app/cms-media`, `public/`)
  must be readable by nginx.
- Staging: build the frontend with `PUBLIC_NOINDEX=true` (noindex meta, `Disallow: /`, no sitemap, no `llms.txt` entries) behind HTTP basic auth. Production leaves it unset.

## Notes

- The content model is deliberately minimal (pages with heading / text / image blocks, one settings record). Extend the
  exporter, the Zod schema, `Blocks.astro`, the Markdown renderer and `docs/contract.md` together, in a pull request in each repository.
- A site without a CMS needs only the `astro-static` template; if it needs a form it starts its small backend from `laravel-cms`.
- `esoul-guidelines.json` of this repository holds the layers for the shared documentation; each submodule has its own.
- This repository has a `composer.json` with `esoul-cz/guidelines` as a dev dependency: `composer install`, then `vendor/bin/esoul-guidelines sync`
  (`composer update` runs it) and `vendor/bin/esoul-guidelines check`. Nothing is built from here and it has no pipeline, so nothing checks it in CI; if you
  add one, run `vendor/bin/esoul-guidelines check` after `composer install` (the version of its `composer.lock`).
  The submodules have no `vendor/`: their READMEs ("Guidelines") show how to run the command without PHP in the project (Composer global, the PHAR of the
  recorded version, Docker), and the `astroPipeline` of their `Jenkinsfile` runs `check` with the version recorded in their `esoul-guidelines.lock`.

## Addons

The addons this project took are listed in `.esoul/template.json`; their documentation is linked below.

<!-- @esoul-slot addons -->
