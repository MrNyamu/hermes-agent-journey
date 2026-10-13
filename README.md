# Hermes Agent Journey

Field notes from self-hosting a [Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous Research) on a cheap VPS. I talk to it over Telegram and connect it to my tools.

The goal is to learn how agents are built, tuned and wired into real workflows. Along the way, this repo collects everything someone would need to stand up their own agent the same way, including the mistakes.

> **Status:** private while it's being filled in. It will be made public once it's a usable guide.

## The setup

| Piece | Choice |
|---|---|
| Server | Contabo VPS, 8 GB RAM, 100 GB disk, Hermes pre-installed |
| OS user | Non-root sudo user (`mrnyamu`). Root SSH and password login disabled |
| Model | Claude Sonnet via my Claude account (orchestrator) |
| Chat client | Telegram bot, via the Hermes gateway (`hermes-gateway.service`, systemd user unit) |
| Tool sandbox | Docker terminal backend |
| Egress | iron-proxy firewall, allowlisted hosts only |
| Integrations | Composio CLI → GitHub ✅ |

## Journal

1. [VPS access and hardening](docs/01-vps-access-and-hardening.md): SSH keys, a non-root user, locking down sshd, firewall
2. [Local models on 8 GB: what's realistic](docs/02-local-models.md): Carnice, llama.cpp, and why training happens elsewhere
3. [Hermes + Claude + Telegram](docs/03-hermes-telegram.md): the gateway service and how to manage it
4. [Docker, iron-proxy and Composio/GitHub](docs/04-docker-backend-and-composio.md): from "no terminal access" to the first GitHub notifications
5. [Troubleshooting cheatsheet](docs/troubleshooting.md): symptoms, causes and fixes, for quick lookup

## Conventions for these notes

- No secrets in this repo: no keys, tokens, IPs or `.env` contents. Use placeholders like `<VPS_IP>` and `<API_KEY>`.
- Each doc says what we did, why, and what went wrong.
- Mark anything unconfirmed with **TODO**.
