## Eko Redirect Hub — Cross-Origin Callback Relay

**A shared, cross-origin redirect landing page for Eko’s web apps.**
It **captures all query/hash parameters**, **posts them to the opener** via `postMessage`, and **auto-closes**. When no opener is available, it **redirects back** to a provided URL with all parameters appended.

---

## Why this exists

Eko's webapps open 3rd-party pages in new tabs, instead of redirecting to them by replacing the current page. But, many 3rd-party tools require a redirection-URL to redirect back to the calling page. This generic solution addresses these needs by providing:

* **One redirection-target page for many apps** (different origins/subdomains).
* **Works with OAuth/OIDC and generic tool callbacks.**
* **No tight coupling** to any provider; forwards *everything* it receives.

---

## Features

* **Generic param pass-through**: forwards *all* query + hash params as-is.
* **Cross-origin safe**: uses `window.opener.postMessage` to pass the payload.
* **Auto-close**: closes the callback tab if script-opened; shows a manual close hint otherwise.
* **Graceful fallback**: if `opener` is missing, navigates to your app’s `return_to` with all params.

---

## Quick Start

### 1) Deploy the redirector

Host the static `index.html` at a neutral domain, e.g. **`https://redirect.eko.in/`**.

### 2) Open the external tool / auth URL from your app

* Add `return_to`, and `state` (with PKCE if OAuth), if required.
* Open in a **script-initiated** popup/tab so the redirector can close itself.

```js
// In your app (e.g., https://app.eko.in)
const state = crypto.randomUUID(); // or a JWS from your backend
sessionStorage.setItem('oauth_state', state);

const auth = new URL('https://tool.example.com/oauth/authorize');
auth.searchParams.set('client_id', '...');
auth.searchParams.set('redirect_uri', 'https://redirect.eko.in/');
auth.searchParams.set('response_type', 'code');
auth.searchParams.set('state', state);

// Prefer embedding these inside `state` if the provider strips them
auth.searchParams.set('return_to', `${window.location.origin}/auth/resume`);

window.open(auth.toString(), 'ekoAuth', 'popup,width=480,height=720');
```

### 3) Listen for the message in your app

* Verify **`state`** matches, if required.
* Resume your flow (e.g., code exchange with PKCE).

```js
// Boot-time listener
window.addEventListener('message', (event) => {
  if (event.origin !== 'https://redirect.eko.in') return; // strict check

  const msg = event.data;
  if (msg?.type !== 'EXTERNAL_TOOL_REDIRECT') return;

  const params = msg.params || {};
  const expected = sessionStorage.getItem('oauth_state');
  if (!expected || params.state !== expected) {
    console.warn('State mismatch; ignoring payload');
    return;
  }
  sessionStorage.removeItem('oauth_state');

  // Continue your flow (exchange code, handle tokens/errors, etc.)
  // exchangeAuthorizationCode(params.code, codeVerifier) ...
});
```

## Repository Structure

* **`/redirect.html`** — the shared callback page (static).
* **`/examples/`** — minimal demo app showing opener → redirector → message flow.
* **`/docs/`** — integration notes, provider quirks, security checklists.

---

## Integration Patterns

### Embedding `target_origin` & `return_to` in `state`

* Encode `{ target_origin, return_to, nonce, exp }` into a **signed** state (JWS/JWT).
* On receipt, your app decodes and verifies it; the redirector remains generic.

### Multiple values per key

* The redirector preserves duplicates (arrays). Your app should accept both **string** or **string\[]** for each key.

---

## Browser Support

* `window.opener.postMessage` — broadly supported.
* Auto-close requires the popup/tab to be **opened by script** (user gesture).
* Fallback navigation covers cases where `opener` is missing or blocked.

---

## Troubleshooting

* **Tab didn’t close** → Ensure popup was opened from a **user gesture** and not blocked by the browser.
* **No message received** → Check `event.origin`, allowlist, and confirm provider returned `target_origin` (or that your app recovered it from `state`).
* **State mismatch** → Make sure you store the request `state` (e.g., `sessionStorage`) *before* opening the popup, and clear it after success.

---

## Roadmap

* **Telemetry hooks** (success/error counters; no secrets).
* **Provider adapters** (docs for common OAuth quirks).
* Serve it via a tiny server/edge function (`/server`) that injects an **allowlist** of permitted `target_origin`s.
