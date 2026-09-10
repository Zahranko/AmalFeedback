# Al Amal — Patient Feedback (public form)

The anonymous patient feedback form (`رأيك يهمّنا`), hosted on its own subdomain
and served as a single self-contained `index.html` — CSS, JavaScript and all.
No build step, no dependencies, no framework.

The design is a byte-for-byte copy of the original in-app form
(`D:\repos\CRMS\Feedback\wwwroot\index.html` + `css/site.css`); only the API
wiring differs, because that page was same-origin with its backend and this one
is not.

## The two things that must be set

**1. `API_BASE` — the AlAmalBusiness API origin.** Near the top of the `<script>`
block in `index.html`:

```js
const API_BASE = 'https://api.alamalhospitaljo.com';
```

> ⚠️ That value is still a **placeholder**. Set it to the real API origin
> (the same host `alamal-console` points `API_BASE_URL` at) before the form can
> load departments or accept a submission.

**2. CORS — this subdomain must be allow-listed on the API.** Every call is
cross-origin now, so the browser blocks it before the controller ever sees it.
In `AlAmalBusiness.Api/appsettings.json`:

```json
"Cors": {
  "AllowedOrigins": [
    "https://business.alamalhospitaljo.com",
    "https://feedback.alamalhospitaljo.com"
  ]
}
```

The API never uses `AllowAnyOrigin()`, so an unlisted origin fails closed.

## Endpoints it calls

Both are on `PublicFeedbackController` — the only anonymous surface on the API
besides login, rate limited to 10 requests/minute per IP.

| | |
|---|---|
| `GET /api/public/feedback/departments` | fills the department dropdown; returns a bare array of `{ id, name, isActive }` |
| `POST /api/public/feedback` | submits the form; returns `{ success, error, referenceNumber, createdDate }` |

The staff routes under `/api/Feedback` require a JWT and must never be called
from this page.

## Deploying

Pushing to `main` runs `.github/workflows/deploy.yml`, which FTPs the root of
the repo to Hostinger. It needs three **secrets** and one **variable** under
*Settings → Secrets and variables → Actions*:

| Name | Kind | Value |
|---|---|---|
| `FTP_SERVER` | secret | Hostinger FTP hostname or IP |
| `FTP_USERNAME` | secret | FTP account user |
| `FTP_PASSWORD` | secret | FTP account password |
| `FTP_SERVER_DIR` | variable | the subdomain's document root, trailing slash — e.g. `/public_html/feedback/` |

If you connect Hostinger's own Git deployment instead, no secrets are needed —
it clones this repo into the document root directly. `index.html` is kept at the
repo root so that route works too.

## Local preview

Opening the file directly works for checking layout, but `fetch` needs an
origin, so serve it over HTTP to exercise the form:

```
npx serve .
```

Without a reachable API the page shows
`تعذّر تحميل قائمة الأقسام` in the error banner — that is the expected offline
behaviour, not a bug in the page.
