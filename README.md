# Al Amal — Patient Feedback (public form)

The anonymous patient feedback form (`رأيك يهمّنا`), hosted on its own subdomain
and served as a single self-contained `index.html` — CSS, JavaScript and all.
No build step, no dependencies, no framework.

It started as a byte-for-byte copy of the original in-app form
(`D:\repos\CRMS\Feedback\wwwroot\index.html` + `css/site.css`). It has since
diverged deliberately:

- **Cairo throughout.** The original set IBM Plex Sans Arabic as the body face
  and used Cairo only for headings and buttons; Cairo is now the whole page and
  IBM Plex is no longer loaded.
- **It is an app shell, not a scrolling document.** `<body>` is
  `position:fixed` and never scrolls; the header is pinned at the top, the
  submit button sits in a bar pinned at the bottom, and only the form between
  them scrolls. The shell's height is `--app-h`, which JavaScript keeps equal
  to `visualViewport.height` — so when the keyboard opens the form shrinks
  instead of the focused field disappearing behind it, and the iOS address bar
  can never collapse mid-scroll and shift the layout.
- **Zoom is off.** `user-scalable=no` covers Android; Safari has ignored it
  since iOS 10, so the script also swallows `gesturestart`/`gesturechange`,
  two-finger `touchmove`, double-tap, `Ctrl`+wheel and `Ctrl`+`+`/`-`/`0`.
  Every text control is at least 16px for the same reason — Safari
  auto-zooms on focusing anything smaller, and closing the keyboard does not
  undo it. This is a deliberate accessibility trade-off: patients who rely on
  browser zoom cannot use it here.
- **Every native picker is now a bottom sheet** (a centred dialog above 640px),
  all three sharing one `.sh` component with a drag-to-close handle, a scrim,
  focus trapping and `Escape`:
  - *Country code* — ~198 countries with Arabic names, the twelve most likely
    repeated in a group at the top. Search matches either an Arabic name —
    normalising أ/إ/آ, ة/ه and ى/ي so spelling variants still hit — or the dial
    code's digits.
  - *Department* — was a `<select>`. Search appears once there are more than
    eight. If the fetch fails the button becomes the retry control.
  - *Visit date* — was `<input type="date">`, whose look, format and RTL
    behaviour differ on every device. It is now an Arabic calendar grid with
    اليوم/أمس/قبل يومين shortcuts and month/year selects, and days outside the
    two-year window are simply not selectable.

  None of these is a `<select>` on purpose: a long option list is a blind
  spinning wheel on iOS and an unsearchable wall on Android. Each keeps its
  submitted value in a hidden input, which is why `form.reset()` cannot restore
  them and the reset handler sets all three by hand.
- **No message type is preselected**, and one must be chosen. The original
  defaulted to *شكر وتقدير*, which meant a patient who never touched that row
  silently filed a thank-you.
- **Dates are formatted locally, not through `toISOString()`.** UTC formatting
  returned *yesterday* for any visit logged before 03:00 Amman time.
- The API wiring differs throughout, because the original was same-origin with
  its backend and this page is not.

## The two things that must be set

**1. `API_BASE` — the AlAmalBusiness API origin.** Near the top of the `<script>`
block in `index.html`:

```js
const API_BASE = 'https://alamalhosp-001-site6.itempurl.com';
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

Without a reachable API the page shows `تعذّر تحميل قائمة الأقسام` in the error
banner and the department field reads `تعذّر التحميل — اضغط لإعادة المحاولة` —
that is the expected offline behaviour, not a bug in the page.

Chrome cannot be resized narrower than the display, so to check the phone
layout properly, point an iframe at it rather than trusting the desktop view:

```html
<iframe src="/index.html" width="390" height="780"></iframe>
```
