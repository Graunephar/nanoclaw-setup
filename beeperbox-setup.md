# Beeperbox Setup

**Project**: [hamr0/beeperbox](https://github.com/hamr0/beeperbox)

## What it is

Beeper Desktop running headless in a Docker container on the NanoClaw host (image `ghcr.io/hamr0/beeperbox:latest`). Bridges Messenger, WhatsApp, Telegram, Discord, Facebook, and LinkedIn into a single API. NanoClaw talks to it via MCP.

The container exposes a noVNC server — all setup and login is done through a browser-based VNC session, no desktop required on the host.

---

## Where it lives

| Component | Location |
|-----------|----------|
| Host files | `~/beeperbox/` (compose file + `.env`) |
| noVNC UI | `https://claws-1.tail45aefb.ts.net:6080/vnc.html` — VNC password in `.env` |
| API | `http://172.17.0.1:23373` (host and containers only — not public) |

---

## Secrets in `.env`

| Variable | Notes |
|----------|-------|
| `BEEPER_TOKEN` | Expires after 30 days. Renew in Beeper Desktop: Settings → Integrations |
| `VNC_PASSWORD` | Randomly generated. Reset by generating a new one with `openssl rand -base64 16` and restarting the container |

---

## Common operations

**Log in again**
Open the noVNC URL, authenticate with `VNC_PASSWORD`, and log in to Beeper Desktop through the browser.

**Restart the container**
```bash
cd ~/beeperbox && docker compose restart
```

**Rotate the token**
1. Open Beeper Desktop via noVNC
2. Settings → Integrations → revoke old token, generate new one
3. Update `BEEPER_TOKEN` in `.env`
4. Ask NanoClaw to re-register the MCP tool

---

## Limitations

- **Free tier cap**: 6 bridges — all slots are currently used.
- **Tailscale hostname**: The machine registered as `claws-1` after a re-login. The old `claws` machine entry needs to be deleted from the Tailscale admin panel to reclaim that name.
- **LinkedIn bridge**: Flaky — expect occasional disconnects.
