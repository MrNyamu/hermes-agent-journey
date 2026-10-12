# 04 · Docker backend and Composio/GitHub

## Goal

Connect Hermes to GitHub through [Composio](https://composio.dev). Composio's onboarding says to send the agent this prompt:

> Install the Composio CLI: `curl -fsSL https://composio.dev/install | sh`, then run `composio login`. Then: List my GitHub notifications, or my most recently updated repos if there are none.

## The wall

Hermes replied that it had no terminal access and suggested workarounds, such as asking me for an API key. The logs showed the real cause:

```
EnvironmentConnectionError: Docker executable not found in PATH or known install locations.
hint: Install Docker (or fix PATH) and retry, or run `hermes setup terminal` to switch to Local.
```

Hermes's **terminal backend was set to Docker, but Docker wasn't installed**. Every shell and `execute_code` call failed, so the agent concluded it had no shell at all.

**Lesson:** when an agent says "I can't do X", read the gateway logs before accepting its explanation.

## Fix: install Docker

There were two options:

- **Docker backend (chosen):** commands run in a container, not directly on the host. This is the safer choice.
- **Local backend** (`hermes setup terminal`, or set `terminal.backend: local`): this is quicker, but the agent gets a real shell on the VPS as the Hermes user.

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker <user>
```

### Gotcha 1: `permission denied ... /var/run/docker.sock`

Docker was installed, but the user wasn't in the `docker` group yet, or the shell predated the change.

- `newgrp docker` asking for a password means you're **not** in the group. The docker group has no password, so nothing you type will work.
- After running `usermod`, `grep docker /etc/group` showed the user was in the group, but `groups` still didn't list it. **Groups only load at login**, so log out of SSH and back in.

### Gotcha 2: the service still has the old groups

The gateway is a systemd **user** service. Its processes inherit groups from the systemd user manager, which started before the group change. A plain `systemctl --user restart` is **not** enough. Do one of these:

```bash
sudo reboot
# or
sudo systemctl restart user@$(id -u).service
```

### Verify

```bash
grep Groups /proc/$(systemctl --user show -p MainPID --value hermes-gateway)/status   # should include the docker GID
docker ps                                                                            # empty table, no error
journalctl --user -u hermes-gateway -n 40 --no-pager -l | grep -iE 'docker|telegram'
```

When it's working, the docker GID (`988` here) is in the gateway's groups, `docker ps` returns cleanly, and Telegram reconnects.

## Known side effects of the Docker backend

- **Composio's login lives in the container.** The CLI is installed inside Hermes's sandbox container. If the container is recreated, the login may be lost. A more durable option is to add Composio as an **MCP server** in `~/.hermes/config.yaml`, so the login lives in Hermes's config.
- **Output mount warning.** On startup the gateway warns: *"Docker backend is enabled ... but no explicit host-visible output mount ... is configured."* This only matters for sending files back over Telegram. **TODO:** add a mount such as `~/.hermes/cache/documents:/output` when file delivery is needed.

## Next

1. Run `/new` in Telegram, then resend the Composio prompt.
2. Complete `composio login`. It prints a URL to approve in the browser.
3. **TODO:** record the result, and whether the login survives a gateway restart.
4. **TODO:** evaluate the Composio MCP route as the long-term setup.
