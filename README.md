# Fish-Identifier

Identify fish flawlessly.

[Identify Today](https://trey16885.github.io/Fish-Identifier/)

Powered by Forgegrit

A single-page web app: take a photo of a fish, and it comes back with an
identification from `claude-fable-5` via the Forgegrit API.

## Hosting

The app is a static site, so GitHub Pages serves it as-is. Pages gives it
HTTPS, which browsers require before granting camera access — this is why the
link above is the way to use it, rather than opening the file locally.

Under **Settings → Pages**, either source works:

- **Deploy from a branch** — choose the branch and `/ (root)`. Nothing else is
  needed; `.github/workflows/pages.yml` is simply unused.
- **GitHub Actions** — `.github/workflows/pages.yml` publishes the repository
  root on every push to `main`, and can also be run by hand from the Actions
  tab.

All asset paths are relative, so the site works from a project URL
(`/Fish-Identifier/`), from a user site at the domain root, and from a custom
domain without changes. `.nojekyll` keeps Pages from running the files through
Jekyll.

## Using it

1. **Register or log in** with a Forgegrit email and password. Registering
   creates the account and then signs in immediately.
2. **Take a picture** with the live camera, or choose an existing photo.
3. The photo is sent on its own and the identification comes back — there is no
   message box; the prompt is always `What fish is this`.

The API key returned at login is kept in `localStorage`, so the session
survives a reload until you log out. Each visitor signs in with their own
account and their key stays in their own browser — nothing is committed to the
repository or shared between visitors.

On a phone, "Add to Home Screen" installs it as a standalone app
(`manifest.webmanifest`).

### Running it locally

Serve it over `http://localhost`, which counts as a secure origin for camera
access; `file://` does not:

```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

## How it talks to the API

Base URL: `https://api-forgegrit.serveousercontent.com`

| Step | Request |
| --- | --- |
| Register | `POST /register` with `{email, password}` |
| Log in | `POST /authenticate/email` with `{email, password}` → `{api_key}` |
| Identify | `POST /v1/chat/completions` with `Authorization: Bearer <api_key>` |

The completions endpoint expects the message `content` to be a **plain string**,
with the image passed alongside it as raw base64 in an `images` array — no
`data:` prefix, and not as OpenAI-style content blocks (those are rejected by
the inference backend):

```json
{
  "model": "claude-fable-5",
  "messages": [
    { "role": "user", "content": "What fish is this", "images": ["<base64>"] }
  ],
  "stream": false
}
```

Photos are drawn to a canvas and scaled down to 1024px on the long edge, then
encoded as JPEG at quality 0.85 to keep the upload small.

### CORS

`OPTIONS /v1/chat/completions` currently answers `401 Missing API key`, because
the key check also runs on the preflight — and browsers never send an
`Authorization` header on a preflight. A preflight must answer with a 2xx or
the browser refuses to send the request at all, reporting only `Failed to
fetch`. `/register` and `/authenticate/email` answer `204`, so logging in works
and identifying does not.

Until the API lets `OPTIONS` through without a key, identification cannot work
from a browser; there is no client-side way around it, as the `Authorization`
header is what forces the preflight in the first place. The app tells the two
apart by sending a preflight-free probe: if that reaches the API, the network
is fine and the preflight is what was refused.

### Retries

The tunnel in front of the API drops a noticeable share of requests — measured
at roughly one in ten — as an empty `502` (no body, `Connection: close`, no
`server` header, so it is the tunnel rather than the app) or as a dead
connection. Both are transient and succeed on a retry, so every request makes
up to three attempts with a short backoff. Over 15 measured runs that left no
user-visible failures.

## Layout

| File | |
| --- | --- |
| `index.html` | The whole app — markup, styles, and logic, no build step and no dependencies |
| `manifest.webmanifest`, `icon.svg`, `icon-512.png`, `apple-touch-icon.png` | Icons and home-screen install metadata |
| `.nojekyll` | Tells Pages to serve the files untouched |
| `.github/workflows/pages.yml` | Pages deployment, for the "GitHub Actions" source |
