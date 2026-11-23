# Backend <-> frontend contract (v1)

The single source of truth for the data between the `backend` (Laravel + Filament, template `laravel-cms`) and the `frontend` (Astro,
template `astro-cms-frontend`), the two submodules of this repository. A change is ONE pull request in EACH repository, linked to
each other: the backend (exporter + golden test), the frontend (Zod schema, golden copy, fixture) and this file.

Conventions: JSON keys are **camelCase**, dates are ISO 8601, texts are plain text (never HTML). The version is the
handshake: the frontend accepts only `schemaVersion: 1` and fails the build otherwise. A field added later is **optional** in the
frontend schema (an older backend does not send it); a field that changes meaning or is removed is `schemaVersion: 2`.

## 1. Content export: `GET /cms-api/v1/export`

- Authorization: `Authorization: Bearer <CMS_EXPORT_TOKEN>`. Read only by the frontend build (Jenkins, local development).
  An unset token disables the endpoint (404), a wrong token answers 401. With `CMS_API_ALLOWED_IPS` set, other
  addresses get 403.
- Contains only published pages, ordered by `sort`, then `id`.
- When the content breaks the contract the endpoint answers `422 {"message", "problems": [...]}` instead of a payload the
  build would reject. The same `problems` stop the Publish button.

```jsonc
{
  "schemaVersion": 1,
  "generatedAt": "2026-11-18T08:00:00+00:00",
  "contentHash": "sha256:…",              // hash of the payload without generatedAt and contentHash
  "settings": {
    "siteName": "Example Site",            // string, required
    "description": "…",                    // string | null, default meta description
    "contactEmail": "hello@example.com",   // string | null, a valid e-mail
    "footerText": "…",                     // string | null
    "ogImage": Image | null                // default Open Graph image
  },
  "pages": [
    {
      "id": 1,
      "slug": "home",                      // ^[a-z0-9]+(-[a-z0-9]+)*$, unique; a path segment of the URL
      "isHome": true,                      // exactly one page; rendered at `/`, the others at `/<slug>/`
      "sort": 10,
      "title": "Welcome",                  // string, required
      "description": "…",                  // string | null, meta description
      "blocks": [ Block ],
      "updatedAt": "2026-11-10T09:00:00+00:00",  // optional: when the page itself was last saved (ISO 8601 with offset)
      "noindex": false,                    // optional: true = noindex, left out of the sitemap, llms.txt and Markdown versions
      "ogImage": Image | null              // optional: Open Graph image of this page (the site image is used when null or absent)
    }
  ]
}
```

`updatedAt`, `noindex` and `ogImage` of a page are the **optional fields of v1**: the backend sends all three. The frontend uses `updatedAt`
as the `lastmod` of the sitemap (never the build time, so a rebuild does not touch the dates of unchanged pages), `noindex` for the robots
meta tag, the sitemap, `llms.txt` and the Markdown versions, and `ogImage ?? settings.ogImage` as the Open Graph image.

### Blocks

```jsonc
{ "type": "heading", "level": 2 | 3, "text": "…" }
{ "type": "text", "body": "…" }            // plain text, a blank line separates paragraphs
{ "type": "image", "image": Image, "alt": "…", "caption": "…" | null }   // alt is required
```

New block types are added to the exporter, to `blockSchema` in `frontend/src/lib/cms/schema.ts`, to
`frontend/src/components/Blocks.astro`, to the Markdown renderer `frontend/src/lib/cms/markdown.ts` and here, together.

### Files and media

```jsonc
// File
{ "url": "/_cms/<sha256[0:20]>.png", "mime": "image/png", "bytes": 12345, "sha256": "<64 hex>" }
// Image = File +
{ "width": 1200 | null, "height": 630 | null }
```

- The URL of a file is always `/_cms/<first 20 characters of sha256>.<ext>`; the extension follows `mime`
  (`image/webp` webp, `image/png` png, `image/jpeg` jpg, `image/avif` avif). No other type is served (no SVG).
- The backend writes the file to the `cms-public` disk before it answers, and the web server serves that disk at
  `/_cms/` (`docker/nginx.example`). Names are content addressed, so a file never changes: an edited image gets a
  new name and reaches the site only after a publish.
- The frontend does not download images. It checks that the URL matches the sha256 and sends `HEAD` to every file
  (200 and the size from `bytes`), otherwise the build fails.
- The fixture (`CMS_SOURCE=fixture`) carries relative paths into `src/lib/cms/fixture-media/` instead of URLs; the build copies
  them to `dist/_cms`.
- Unused files are removed by `cms:prune-public-media` (daily): files that no build of the last 7 days referenced.

## 2. Publish callback: `POST /cms-api/v1/publishes/{id}/status`

Jenkins reports the result of a build started by the Publish button.

- Authorization: `Authorization: Bearer <CMS_JENKINS_CALLBACK_TOKEN>` (404 when unset, 401 when wrong).
- Body: `{ "result": "SUCCESS" | "UNSTABLE" | "FAILURE" | "ABORTED" | …, "buildUrl": "https://…", "contentHash": "sha256:…", "message": "…" }`.
  Only `result` is required. `SUCCESS` and `UNSTABLE` count as deployed (`UNSTABLE` adds a note), everything else as failed.
- Answer: `200 {"status": "success" | "failed"}`.
- A result that arrives after the publish was ended by the timeout or by an admin is recorded with a note; it does not reopen the publish.

## 3. Trigger: backend -> Jenkins

`POST {CMS_JENKINS_JOB_URL}/buildWithParameters` with basic auth (`CMS_JENKINS_USER` + API token) and the form
parameter `CMS_PUBLISH_ID=<id>`. Jenkins answers `201` with the queue item in `Location`. As a fallback for a missing
callback the backend polls `<queue item>/api/json`, then `<build>/api/json`.

## 4. Golden export, golden copy and fixture

`backend/tests/Fixtures/content-export.golden.json` is the output of the exporter over the sample data of
`backend/tests/Support/SampleContent.php` (guarded by `GoldenExportTest`; `pixel.png` next to it is the one media file). The frontend keeps a
**copy** of both in `frontend/contract/` so that it builds and checks on its own, and its fixture `frontend/src/lib/cms/fixture.json` is generated from
the copy (`pnpm fixture:build`).

`pnpm fixture:check` in the frontend compares:

1. the schema version of the copy with the version the frontend accepts (always);
2. the copy with the backend originals, byte for byte, **when `../backend` exists** (this monorepo with both submodules checked out);
3. the fixture with the copy.

Without `../backend` (the frontend repository on its own, for example in Jenkins) only 1 and 3 run, so a backend change that
was not copied is caught the next time someone works in the monorepo. After changing the exporter or the sample data:

```sh
cd backend && GOLDEN_UPDATE=1 php artisan test --filter=GoldenExportTest
cd ../frontend && pnpm contract:sync        # copies the golden file and pixel.png into contract/, rebuilds the fixture
```

## 5. Routes

Pages: `/` for the home page, `/<slug>/` for the others (`build.format: 'directory'`); Markdown versions at `/index.md` and `/<slug>.md`
(`noindex` pages have none). The backend is on a **subdomain** (default, `admin.<domain>`) or under `/admin` of the site's domain
(exception, README "Where the backend lives"). The static site and `/_cms/` media come from the web server; `/admin`, `/cms-api`,
`/forms`, `/livewire-<hash>` and `/up` from the backend.

## 6. Forms: `POST /forms/contact`

The static site has no server, so the forms of the site post to the backend (described in full in `backend/docs/forms.md`).

- A plain `<form method="post" action="{backend origin}/forms/contact">` from the site (cross-origin is fine: a form post is a navigation, no CORS
  needed; the backend lists the site in `CMS_FORMS_ALLOWED_ORIGINS` and checks `Origin`). No CSRF token, session or cookie.
- Fields: `name`, `email`, `message`, the hidden honeypot field `website` (must stay empty), `recaptcha_token` (reCAPTCHA v3, action `contact`) and
  optionally `return_to` (absolute URL on an allowed origin).
- A plain post is answered with `303` to `return_to` (default `/` of the posting site) with `?form=<code>#form-result`; with `Accept: application/json` the answer is
  `{"status": "sent"|"error", "code": "…", "message": "…"}`. Codes: `sent`, `invalid`, `origin_forbidden`, `rate_limited`, `captcha_missing`,
  `captcha_failed`, `captcha_unavailable`, `unavailable`, `failed`.
- A post without a reCAPTCHA token is rejected (`captcha_missing`, message "Enable JavaScript or contact us at …"): the page and the contact
  alternative next to the form render without JavaScript, the form itself needs it.
- The form's URL is a build argument of the frontend (`PUBLIC_FORM_ENDPOINT`, never read at runtime); the template ships no form component:
  the first project that needs one adds it.
