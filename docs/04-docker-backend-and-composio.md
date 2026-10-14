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

- **Where Composio's login lives (confirmed).** The sandbox's `/root` is a bind mount of the host folder `~/.hermes/sandboxes/docker/default/home`. Composio installs to `~/.composio/` there, and its credentials are in `user_data.json`, readable only by root. So the login **persists on the host** and survives the container being recreated. Treat that host folder as secret: keep it out of backups you share and out of this repo.
  - Check the mounts with: `docker inspect -f '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}' <container>`
  - `composio` isn't on the sandbox PATH. It lives at `~/.composio/composio`.
  - A `/restart` from Telegram reuses the running container. It was "Up 12 hours" across a gateway restart.
- **Output mount warning.** On startup the gateway warns: *"Docker backend is enabled ... but no explicit host-visible output mount ... is configured."* This only matters for sending files back over Telegram. The container already mounts `~/.hermes/cache/documents` (and other cache folders), so file delivery should work. **TODO:** confirm by asking Hermes to send a file.
- **Workspace location.** The sandbox's `/workspace` is `~/.hermes/sandboxes/docker/default/workspace` on the host. It persists, but files the agent creates are owned by `root` (the agent runs as root in the container). Run `sudo chown -R <user>:<user>` on that folder if you need to edit them.

## Wall 2: the iron-proxy egress firewall

With Docker working, the agent failed differently:

1. `proxy.enabled is true but iron-proxy is not configured` → run `hermes egress setup`
2. `iron-proxy is enabled but not running on port 9090` → run `hermes egress start`
3. Every outbound request returned **403** → the proxy was running, but its allowlist only covers the model providers

### What iron-proxy is

It's a host-side firewall for the Docker sandbox ([docs](https://hermes-agent.nousresearch.com/docs/user-guide/egress/iron-proxy)):

- The sandbox only holds **placeholder tokens**. The proxy swaps in the real API keys on the host, so a prompt-injected agent can't leak keys it never sees.
- **Default-deny allowlist.** Unlisted hosts get a 403. Private, loopback and cloud-metadata addresses are always blocked.
- It intercepts TLS using a local CA (`~/.hermes/proxy/ca.crt`). Never let `ca.key` leave the host.
- It does **not** stop data being sent to an allowlisted host. Keep the list short.

### Allowing hosts

Edit `~/.hermes/config.yaml`, **not** `proxy.yaml` (it's regenerated):

```yaml
proxy:
  enabled: true
  extra_allowed_hosts:
    - composio.dev
    - "*.composio.dev"          # wildcards match subdomains only
    - github.com
    - api.github.com
    - raw.githubusercontent.com
    - objects.githubusercontent.com
    - release-assets.githubusercontent.com   # Composio binary CDN
    - pypi.org
    - files.pythonhosted.org                 # actual pip downloads
```

```bash
hermes egress setup     # regenerate proxy.yaml
hermes egress reload    # hot-reload rules (or: hermes egress restart)
hermes egress status
```

Adding hosts is mostly a one-time cost for registries and download CDNs. The agent's web search and page fetching run host-side and aren't affected by this list.

### Start on boot ✅

The gateway refuses to create sandboxes while the proxy is down (`enforce_on_docker: true`), and Hermes doesn't start the proxy at boot. This user unit fixes that:

```ini
# ~/.config/systemd/user/hermes-egress.service
[Unit]
Description=Hermes iron-proxy egress firewall
Before=hermes-gateway.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/home/<user>/.local/bin/hermes egress start   # use the output of `which hermes`
ExecStop=/home/<user>/.local/bin/hermes egress stop

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload && systemctl --user enable hermes-egress
sudo loginctl enable-linger <user>        # start user services at boot, without logging in
loginctl show-user <user> -p Linger       # expect Linger=yes
```

**Test it before rebooting.** This hands the running proxy over to systemd:

```bash
hermes egress stop
systemctl --user start hermes-egress
systemctl --user status hermes-egress --no-pager | grep -E 'Active|code='
hermes egress status
```

Expect `Active: active (exited)` with `status=0/SUCCESS`. "Exited" is correct here: `egress start` launches the daemon and returns. `hermes egress status` should show `Listening: yes`.

**Reboot test (passed):** after `sudo reboot`, both `hermes-egress` and `hermes-gateway` came up on their own, the proxy was listening, and after `/new` in Telegram the agent reused the existing Composio login to list GitHub notifications without reinstalling anything.

## Result ✅

After a `/new` session and a retry, the Composio CLI installed, `composio login` completed, and the agent pulled 50 GitHub notifications (the first page). These included PR activity, review requests and recurring CI failures.

**Lessons:**
- The agent will **propose workarounds that weaken security**, such as "just give me a PAT", or claiming another tool has "a separate network stack". Fix the environment instead.
- Each fix revealed the next layer: Docker missing → group permissions → stale systemd groups → proxy not configured → proxy not running → allowlist. Read the *exact* error each time.

## Next

- ✅ Composio login persistence: stored on the host via the `/root` bind mount, and confirmed across a full VPS reboot.
- **TODO:** evaluate the Composio MCP route as the long-term setup.
- **Update:** GitHub was later moved off Composio to plain `git` + `gh` with per-owner tokens. Composio stays for SaaS integrations. See [05](05-github-identity-and-guardrails.md).
- ✅ Egress boot service set up and reboot-tested.
