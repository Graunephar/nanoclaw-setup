# NanoClaw Setup Notes

Customization guides for NanoClaw. Each file documents one customization precisely enough to reproduce from scratch.

## Preferred messaging channel: Telegram

Telegram is the primary channel for talking to NanoClaw. Reasons:

- **Google Assistant dictation works** — Assistant can dictate into Telegram natively. Signal does not support this.
- **WhatsApp bridge had issues** — the Beeperbox WhatsApp bridge had reliability problems.
- **Works out of the box** — NanoClaw supports Telegram natively without any extra bridge setup.

## Guides

- [Signal separate identity](signal-separate-identity.md) — Route agent messages through a secondary Signal device so they appear in a different color (left side / received) in Note to Self.
- [Beeperbox setup](beeperbox-setup.md) — Headless Beeper Desktop in Docker bridging Messenger, WhatsApp, Telegram, Discord, Facebook, and LinkedIn into a single MCP-accessible API.
