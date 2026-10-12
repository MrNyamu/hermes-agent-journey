# 03 · Hermes + Claude + Telegram

Hermes runs on the VPS with Claude Sonnet as its model. I talk to it through a Telegram bot.

**TODO:** write up the exact setup steps: the model/provider config, creating the bot with @BotFather, the gateway setup command, and restricting the bot to my Telegram user ID.

## How it runs

The Telegram connection is the **Hermes gateway**, installed as a **systemd user service** under my account. It is *not* a system-wide service, so `systemctl list-units` won't show it.

```bash
systemctl --user status hermes-gateway
systemctl --user restart hermes-gateway
journalctl --user -u hermes-gateway -n 50 --no-pager -l
```

Config lives in `~/.hermes/config.yaml`.

Healthy startup logs end with:

```
[Telegram] Connected to Telegram (polling mode)
```

## Sessions remember failures

A Hermes chat session carries its history. If a tool failed earlier in the session, the agent will keep insisting it can't do the thing, even after you've fixed the cause. It's reasoning from the old error, not re-checking.

**Fix:** start a fresh session in Telegram with `/new` (or `/reset`), then retry. Or tell it explicitly that the cause is fixed and ask it to try again.

## Watch for duplicate installs

`pgrep -af hermes` showed **two** installs:

- `/home/<user>/.hermes`: runs `gateway run`, the one Telegram uses
- `/root/.hermes`: runs as root, serving the `dashboard`

These have separate configs and keys. Edit the one the gateway uses.

**TODO:** decide whether to remove the root copy, or move the dashboard to the user install.

## Context for the agent

To give Hermes context about other projects, use one of these:

- paste a summary in Telegram
- copy docs to `~/context/` on the VPS and ask it to read them
- clone the repo using a narrow-scope token

Never put `.env` files, secrets or customer data on the box or in Telegram. Normal Telegram chats are not end-to-end encrypted.
