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
block in `index.html`, currently set and correct:

```js
const API_BASE = 'https://api.alamalhospitaljo.com';
```

It is hardcoded rather than configured, and that is deliberate: this is a static
page with no server process, so there is no runtime to read an environment
variable. Nor is one wanted — the value is used by JavaScript in the patient's
browser, so it is public no matter where it is kept. Changing it means editing
this line and pushing. Secrets (the DB connection string, the JWT signing key)
live on the API and never appear here.

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

Hostinger's built-in Git deployment is wired directly to this repository — there
is no build pipeline and no CI, because there is nothing to build.

*hPanel → Websites → the feedback subdomain → Advanced → GIT*, pointed at
`https://github.com/Zahranko/AmalFeedback` on branch `main`, with the
install path left as the subdomain's own document root.

Hostinger does **not** pull on its own. Either press *Deploy* in hPanel after a
push, or make it automatic: copy the webhook URL hPanel shows under the
repository, then add it in GitHub under *Settings → Webhooks* (content type
`application/json`, the `push` event alone).

Because the clone lands in the web root, everything committed here is publicly
fetchable. `.htaccess` blocks the files meant for developers — `README.md` and
anything under `.git` — so keep new repo furniture covered by it, or out of the
repo.

If the repository is private, hPanel shows an SSH public key when you connect
it; add that under *Settings → Deploy keys* (read-only is enough).

## Local preview

Opening the file directly works for checking layout, but `fetch` needs an
origin, so serve it over HTTP to exercise the form:

```
npx serve .
```

Without a reachable API the page shows
`تعذّر تحميل قائمة الأقسام` in the error banner — that is the expected offline
behaviour, not a bug in the page.
