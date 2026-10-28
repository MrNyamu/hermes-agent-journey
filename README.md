# Hermes Agent Journey

Field notes from self-hosting a [Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous Research) on a cheap VPS. I talk to it over Telegram and connect it to my tools.

The goal is to learn how agents are built, tuned and wired into real workflows. Along the way, this repo collects everything someone would need to stand up their own agent the same way, including the mistakes.

> **Status:** public and usable as a guide. Still being extended over time.

## The setup

| Piece | Choice |
|---|---|
| Server | Contabo VPS, 8 GB RAM, 100 GB disk, Hermes pre-installed |
| OS user | Non-root sudo user (`<user>`). Root SSH and password login disabled |
| Model | Claude Sonnet via my Claude account (orchestrator) |
| Chat client | Telegram bot, via the Hermes gateway (`hermes-gateway.service`, systemd user unit) |
| Tool sandbox | Docker terminal backend |
| Egress | iron-proxy firewall, allowlisted hosts only |
| GitHub | `git` + `gh` with fine-grained, per-owner tokens; protected `main`, rebase-only merges |
| Integrations | Composio CLI for SaaS tools (GitHub moved to `git`/`gh`) |
| Identity | `SOUL.md` (co-founder persona and hard lines) plus skills, with approval required for skill edits |

## Journal

1. [VPS access and hardening](docs/01-vps-access-and-hardening.md): SSH keys, a non-root user, locking down sshd, firewall
2. [Local models on 8 GB: what's realistic](docs/02-local-models.md): Carnice, llama.cpp, and why training happens elsewhere
3. [Hermes + Claude + Telegram](docs/03-hermes-telegram.md): the gateway service and how to manage it
4. [Docker, iron-proxy and Composio/GitHub](docs/04-docker-backend-and-composio.md): from "no terminal access" to the first GitHub notifications
5. [GitHub access, agent identity and guardrails](docs/05-github-identity-and-guardrails.md): per-owner tokens, SOUL.md and skills, branch protection, and two incidents
6. [How the Docker sandbox runs kanban workers](docs/06-sandbox-and-kanban-workers.md): one shared `sleep infinity` container, labels and reuse, and a night of OOM kills
7. [Troubleshooting cheatsheet](docs/troubleshooting.md): symptoms, causes and fixes, for quick lookup

## Templates

- [`templates/SOUL.example.md`](templates/SOUL.example.md): a starting point for the agent's identity
- [`templates/skills/github-workflow/SKILL.md`](templates/skills/github-workflow/SKILL.md): the GitHub routine as a Hermes skill
- [`templates/sandbox-bin/`](templates/sandbox-bin/): `ghtok`, the git credential helper and the `gh` wrapper

## Conventions for these notes

- No secrets in this repo: no keys, tokens, IPs or `.env` contents. Use placeholders like `<VPS_IP>` and `<API_KEY>`.
- Each doc says what we did, why, and what went wrong.
- Mark anything unconfirmed with **TODO**.
