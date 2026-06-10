# How to get a working pCloud API token (when the account has 2FA enabled)

## Goal

Get a usable bearer token for a 2FA-enabled pCloud account so NanoClaw (via the OneCLI proxy) can call `eapi.pcloud.com` endpoints — for search, read, list, etc.

The OneCLI proxy injects `access_token=<TOKEN>` automatically when the credential is stored, so the agent never sees the token directly.

---

## Account facts to establish first

Two things you must know before debugging anything:

1. **Which pCloud datacenter is the account on?**
   - EU accounts → `eapi.pcloud.com`
   - US accounts → `api.pcloud.com`
   - Wrong host returns `2000 Log in failed` — even with correct credentials. That error is misleading; it does not mean "wrong password."
   - Test by hitting both with credentials. The host that gets to a 2FA error instead of `2000` is the right one.

2. **Is 2FA enabled on the account?**
   - If yes, the password-based API login flow (`getauth`) **cannot** be used. This is not a parameter problem — see "Dead ends" below.

---

## The working method: rclone's bundled OAuth app

pCloud OAuth2 normally requires registering your own app to get a `client_id` / `client_secret`. We tried — pCloud's developer console kept returning *"Temporarily unavailable, please contact support team"* (a known flaky page on their side).

**rclone sidesteps this entirely**: it ships with its own pre-registered pCloud OAuth app, so no manual app registration is needed.

### Step 1 — Trigger the OAuth flow

On any host with a browser available (your laptop, the NanoClaw host with port-forwarding, whatever):

```bash
rclone config create pcloud pcloud
```

This opens the standard pCloud OAuth flow in a browser: **login → 2FA → Allow**. Because login happens in the browser, pCloud handles 2FA the way it normally does — the API never has to process a 2FA code at all.

### Step 2 — Extract the token

After rclone has stored its config:

```bash
rclone config dump
```

The bearer token is in the JSON output under the `pcloud` remote's `token` field. It's a **non-expiring** bearer token (until revoked in pCloud → Settings → Security).

### Step 3 — Store it in OneCLI

Save the token as a generic credential in OneCLI's dashboard, targeting `eapi.pcloud.com`. OneCLI then appends `access_token=<TOKEN>` to outgoing requests to that host. The agent never sees the token.

### Step 4 — Verify

From inside the agent:

```bash
curl -s "https://eapi.pcloud.com/userinfo"
```

A working response looks like:

```json
{ "result": 0, "userid": <your-numeric-id>, ... }
```

`result: 0` means success. Any other result code means the token wasn't injected or is invalid.

### Rotation

- Re-run `rclone config create pcloud pcloud` to get a fresh token.
- Revoke old tokens in pCloud → Settings → Security.

---

## Dead ends (and why each fails)

Documented here so future-you doesn't waste an hour rediscovering them.

### 1. `getauth` with username + password

```
GET https://eapi.pcloud.com/userinfo?getauth=1&username=…&password=…
```

→ `1022 Please provide 'code'`. Server wants a 2FA code, but…

### 2. Supplying the 2FA code on the same request

Tried every plausible parameter name (`code`, `tfacode`, `tfa_code`, `otp`, …) as both GET and POST.

→ Always `1022 Please provide 'code'`. The code is never registered server-side.

It's not a timing issue either — an expired code gives a different error. `1022` means "this parameter does not exist in this flow."

### 3. Digest authentication (SHA-1 challenge)

→ `2297 Two factor authentication required`. A clean "need 2FA" — but the response carries no token or session to attach a 2FA code to.

### 4. Looking for a pending-2FA session/token

Checked the JSON body, HTTP response headers, and cookies on every variant.

→ No token field, no `Set-Cookie`, nothing. A cookie-jar follow-up returned `1000 Log in required`. There is **no server-side session handle** that a 2FA code could ever bind to.

### 5. Hitting the assigned backend node directly

For an EU account, the proxy routes you to a backend like `apicph1.pcloud.com` (Copenhagen).

```
GET https://apicph1.pcloud.com/...
```

→ `2000 Log in failed`. Digests are bound to the host that issued them. Never use the backend node directly — always go through `eapi.pcloud.com` (or `api.pcloud.com` for US).

---

## The actual conclusion

> pCloud's password-based API login (`getauth`) is fundamentally incompatible with 2FA accounts — it never hands out the token/session that completing 2FA would require. This isn't a parameter we got wrong; the flow doesn't exist.

This is confirmed by every real-world pCloud client (rclone, keepass2android, the official SDKs) having abandoned password login in favor of OAuth2 for exactly this reason.

OAuth2 works because login + 2FA happens in the browser, where pCloud handles it normally. The API then only ever sees the resulting bearer token.

---

## Quick reference

| What | Value |
|------|-------|
| EU API host | `eapi.pcloud.com` |
| US API host | `api.pcloud.com` |
| Wrong-host symptom | `2000 Log in failed` |
| 2FA-blocked symptoms | `1022 Please provide 'code'`, `2297 Two factor authentication required` |
| Working auth method | OAuth2 via `rclone config create pcloud pcloud` |
| Token type | Bearer, non-expiring (until revoked) |
| How to use | Append `access_token=<TOKEN>` to any method (OneCLI does this automatically once stored) |
| Verify call | `GET /userinfo` → `result: 0` |
