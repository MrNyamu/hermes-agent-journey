# Troubleshooting cheatsheet

| Symptom | Cause | Fix |
|---|---|---|
| SSH asks for the server password despite the key | Key has a non-default filename and isn't offered | `IdentityFile` in `~/.ssh/config`, or `ssh -i` ([01](01-vps-access-and-hardening.md)) |
| `ssh-copy-id`: "All keys were skipped" | Key is already installed for that user | Check you targeted the right user |
| `adduser admin` fails | An `admin` group already exists | Use another username |
| Agent says "no terminal/shell access" | Docker backend configured, Docker missing | Install Docker, or switch the backend to local ([04](04-docker-backend-and-composio.md)) |
| `permission denied ... docker.sock` | User not in the `docker` group, or old session | `usermod -aG docker <user>`, then log out and back in |
| `newgrp docker` asks for a password | Not actually in the group | Fix with `usermod`. There is no password to enter |
| Gateway still can't use Docker after the fix | User systemd manager has stale groups | `sudo reboot` or `sudo systemctl restart user@$(id -u).service` |
| Agent keeps refusing after a fix | The session remembers the earlier failure | `/new` in Telegram, then retry |
| `systemctl list-units` doesn't show Hermes | It's a **user** service | `systemctl --user ...` |
| `systemctl status` output gets cut off / pager | Pager plus line truncation | Press `q`. Use `--no-pager -l` |
| `proxy.enabled is true but iron-proxy is not configured` | Egress proxy enabled, never set up | `hermes egress setup` |
| `iron-proxy is enabled but not running on port 9090` | Proxy not started, e.g. after a reboot | `hermes egress start`. To make it permanent, set up the `hermes-egress` user service ([04](04-docker-backend-and-composio.md)) |
| Sandbox gets 403 for every site | Host not on the allowlist | Add it to `proxy.extra_allowed_hosts`, then `hermes egress setup && hermes egress reload` |
| Agent asks for a GitHub PAT or API key in chat | It's working around a broken environment | Fix the environment. Don't paste secrets into Telegram |
| "Stream stale for 180s" in logs | Model API stream stalled | Usually transient; retry. **TODO:** watch whether it recurs |
| llama-server killed | Out of memory on 8 GB | Smaller quant, or `-c` at 8k or less ([02](02-local-models.md)) |

## Useful one-liner: gateway health

```bash
{ echo "== groups"; grep Groups /proc/$(systemctl --user show -p MainPID --value hermes-gateway)/status
  echo "== docker"; docker ps 2>&1 | head -3
  echo "== config"; grep -n -A6 '^terminal' ~/.hermes/config.yaml
  echo "== logs"; journalctl --user -u hermes-gateway -n 40 --no-pager -l | grep -iE 'docker|telegram|error|warn'
} 2>&1
```
