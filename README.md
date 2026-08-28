# Fish-Identifier

Identify fish flawlessly.

A single-page web app: take a photo of a fish, and it comes back with an
identification from `claude-fable-5` via the Forgegrit API.

## Using it

Open `index.html` — over `http://` or `https://`, not `file://`, since browsers
only grant camera access to a secure origin (`localhost` counts):

```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

1. **Register or log in** with a Forgegrit email and password. Registering
   creates the account and then signs in immediately.
2. **Take a picture** with the live camera, or choose an existing photo.
3. The photo is sent on its own and the identification comes back — there is no
   message box; the prompt is always `What fish is this`.

The API key returned at login is kept in `localStorage`, so the session
survives a reload until you log out.

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
encoded as JPEG at quality 0.85 to keep the upload small. A transient `5xx`
from the inference backend is retried once.

## Layout

Everything lives in `index.html` — markup, styles, and logic, no build step and
no dependencies.
