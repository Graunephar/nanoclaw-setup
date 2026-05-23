# How to set up Signal for NanoClaw via signal-cli-rest-api

## Goal

Make the agent send via a different Signal identity so its messages appear in a **different color** from the user's own messages in Signal.

The existing Signal adapter (`signal.ts`) sends using the user's own Signal account — Signal shows those as "sent by you" on the right side, same color as your phone's messages.

The fix: route the agent's outbound messages through a **separate signal-cli-rest-api Docker container**, also linked as a secondary device on the same account. When that container sends to the user's own number (Note to Self), Signal renders those messages on the **left side** (received), giving them a different color. This works without needing a second phone number.

---

## Step 1 — Write a new channel adapter (`signal-rest.ts`)

NanoClaw's existing Signal adapter talks to signal-cli's native TCP JSON-RPC daemon. signal-cli-rest-api exposes an HTTP REST API instead:

- **Inbound**: poll `GET /v1/receive/{number}` every 3 seconds. Each call consumes pending messages from the Signal server (destructive read), so only one consumer should poll.
- **Outbound**: `POST /v2/send` with `{"message": "...", "number": "+from", "recipients": ["+to"]}`. Supports `text_mode: "styled"` with `text_styles` array for bold/italic/monospace formatting.

Export the shared utility functions from `signal.ts` so `signal-rest.ts` can reuse them without duplication:
- `chunkText` — splits long messages at newlines respecting a 4000-char limit
- `resolveMentions` — replaces Signal's placeholder characters with `@name`
- `parseSignalStyles` — converts Markdown-like syntax to Signal's offset-based style ranges
- The `SignalEnvelope`, `SignalDataMessage`, `SignalQuote`, `SignalMention` interfaces

The new adapter (`src/channels/signal-rest.ts`) registers itself as channel type `signal-rest`. It includes an `EchoCache` to filter out sync messages (echoes of messages it just sent) to avoid loops. It handles Note to Self messages by checking for `syncMessage.sentMessage` and only routing those where the destination is the account's own number.

Wire it into `src/channels/index.ts`:

```typescript
import './signal-rest.js';
```

Build the host TypeScript: `pnpm run build`.

---

## Step 2 — Start the Docker container

Look at how the existing Kuma signal-api is configured to replicate the pattern:

```bash
docker inspect signal-api
```

The new container should be **localhost-only** (not reachable from the internet). Use `127.0.0.1:9285` as the host binding (contrast with `0.0.0.0` which binds all interfaces). Port 9285 was chosen as one above the existing Kuma instance on 9284.

```bash
mkdir -p "$HOME/.local/share/signal-agent"
docker run -d \
  --name signal-agent \
  --restart always \
  -p 127.0.0.1:9285:8080 \
  -v "$HOME/.local/share/signal-agent:/home/.local/share/signal-cli" \
  -e MODE=native \
  bbernhard/signal-cli-rest-api:latest
```

Verify it is up:

```bash
curl -s http://127.0.0.1:9285/v1/about
```

Should return version info JSON.

---

## Step 3 — Link a Signal account to the container

### The headless server problem

The server has no browser (SSH only). The standard linking flow requires opening `http://localhost:8080/v1/qrcodelink?device_name=Agent` in a browser.

**The endpoint returns a raw PNG image**, not HTML. Extracting the URI from it requires a QR decoder library (`zbarimg`, `pyzbar`) which may not be installed.

### Solution — Run `signal-cli-native link` inside the container

The container ships the `signal-cli-native` binary. Run it detached so the process stays alive while the user scans:

```bash
docker exec -d signal-agent sh -c 'signal-cli-native link --name Agent > /tmp/link-uri.txt 2>&1'
sleep 3
docker exec signal-agent cat /tmp/link-uri.txt
```

**Critical**: use `-d` (detached). If the process exits before the user scans, the QR code is expired and the scan fails. Running without `-d` caused this problem on the first attempt.

### Display the QR code in the terminal

Install `qrencode` (requires root/sudo):

```bash
sudo apt-get install qrencode
```

Render the URI as a terminal QR code:

```bash
qrencode -t ANSIUTF8 "sgnl://linkdevice?uuid=...&pub_key=..."
```

**Note on terminal rendering**: `-t ANSIUTF8` uses Unicode block characters and ANSI colors. In some interfaces (e.g. Claude Code) the output is collapsed — expand it to see the QR code. The `#`-character ASCII fallback (`-t ASCII`) is harder to scan with a phone camera. ANSIUTF8 is the right format but requires the output section to be expanded.

### Scan

On the phone: Signal → **Settings → Linked Devices → Link a New Device**, then point the camera at the terminal output.

The link process will output:

```
INFO  ProvisioningManagerImpl - Received link information from +YOUR_NUMBER, linking in progress ...
Associated with: +YOUR_NUMBER
```

This links it as a secondary device on the user's own account. This is intentional — the "different color" effect comes from signal-cli-rest-api sending to +YOUR_NUMBER (Note to Self), which Signal renders on the left side as a received message.

---

## Issue: Account data written to wrong location

**Problem**: `docker exec` runs as root by default. The `signal-cli-native link` process stores account data in `/root/.local/share/signal-cli/data/accounts.json`, but the signal-cli-rest-api process runs as **user 1000** and looks in `/home/.local/share/signal-cli/`.

`curl -s http://127.0.0.1:9285/v1/accounts` returns `[]` even after successful linking.

**Fix**:

```bash
docker exec signal-agent sh -c '
  cp -r /root/.local/share/signal-cli/data/. /home/.local/share/signal-cli/data/
  chown -R 1000:1000 /home/.local/share/signal-cli/data/
'
docker restart signal-agent
```

Verify: `curl -s http://127.0.0.1:9285/v1/accounts` should return `["+YOUR_NUMBER"]`.

---

## Step 4 — Configure NanoClaw

Add env vars to `.env`:

```
SIGNAL_REST_URL=http://127.0.0.1:9285
SIGNAL_REST_ACCOUNT=+YOUR_NUMBER
```

Update the existing Signal messaging group's channel type from `signal` to `signal-rest`. The `ncl messaging-groups update` CLI does not expose `channel_type`, so update the DB directly:

```bash
node -e "
const Database = require('better-sqlite3');
const db = new Database('data/v2.db');
db.prepare(\"UPDATE messaging_groups SET channel_type='signal-rest' WHERE id='<mg-id>'\").run();
"
```

Find the messaging group ID with `node dist/cli/client.js messaging-groups list`.

Restart NanoClaw to pick up the new env vars and channel registration.

---

## Issue: User ID prefix change breaks access control

When the channel type changes from `signal` to `signal-rest`, the user's platform identity changes from `signal:+YOUR_NUMBER` to `signal-rest:+YOUR_NUMBER`. If the messaging group has `unknown_sender_policy: strict`, messages from the new identity are dropped.

The logs show:

```
MESSAGE DROPPED — unknown sender (strict policy)
userId="signal-rest:+YOUR_NUMBER" accessReason="not_member"
```

NanoClaw auto-creates a user record `signal-rest:+YOUR_NUMBER` with display name "Me", but it has no role.

**Fix**:

```bash
node dist/cli/client.js users update --id "signal-rest:+YOUR_NUMBER" --display-name "YourName"
node dist/cli/client.js roles grant --user "signal-rest:+YOUR_NUMBER" --role owner
```

---

## Issue: Old session delivering via old channel type

After the switch, the running container session may still have the old `signal` channel type baked into outbound messages written before the switch. Those replies get delivered via the old signal adapter, not signal-rest.

**Fix**: Restart the container with a wake message so it starts fresh:

```bash
node dist/cli/client.js groups restart --id <agent-group-id> \
  --message "Signal is now set up via signal-rest."
```

---

## Step 5 — Verify end-to-end

Send a test message directly via the REST API:

```bash
curl -X POST "http://127.0.0.1:9285/v2/send" \
  -H "Content-Type: application/json" \
  -d '{"message":"Test","number":"+YOUR_NUMBER","recipients":["+YOUR_NUMBER"]}'
```

Should return `{"timestamp":"..."}`. The message should appear in Signal Note to Self on the left side (different color from user's own messages).

---

## Final state summary

| Component | Value |
|-----------|-------|
| Container | `signal-agent`, `bbernhard/signal-cli-rest-api:latest`, `MODE=native` |
| Port | `127.0.0.1:9285` (localhost only) |
| Data dir | `$HOME/.local/share/signal-agent` |
| Account | `+YOUR_NUMBER` (linked as secondary device) |
| Channel type | `signal-rest` (was `signal`) |
| Env vars | `SIGNAL_REST_URL`, `SIGNAL_REST_ACCOUNT` in `.env` |
| New file | `src/channels/signal-rest.ts` |
| Modified | `src/channels/signal.ts` (exported utilities), `src/channels/index.ts` (barrel import) |
| User role | `signal-rest:+YOUR_NUMBER` granted `owner` |

---

## Tools used

| Tool | Purpose |
|------|---------|
| `docker run` | Start signal-agent container |
| `docker inspect` | Read existing Kuma signal-api config to replicate the localhost-only pattern |
| `docker exec -d` | Run `signal-cli-native link` detached inside container |
| `docker exec` | Copy account data from root to user 1000, read link URI |
| `docker restart` | Reload container after fixing account data location |
| `curl` | Test REST API, poll accounts and send endpoints |
| `qrencode -t ANSIUTF8` | Render `sgnl://` linking URI as scannable QR in terminal |
| `ncl roles grant` | Grant owner role to new `signal-rest:+YOUR_NUMBER` identity |
| `ncl groups restart` | Restart agent container with a wake message |
| `better-sqlite3` via `node -e` | Direct DB update for `channel_type` (ncl does not expose this field) |
| `pnpm run build` | Compile new TypeScript adapter |
