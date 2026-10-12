# 01 · VPS access and hardening

The Contabo VPS came with Hermes pre-installed and root password login. Before running an agent that can execute commands, I locked the box down.

## 1. SSH key

On the Mac:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/hermes-contabo   # set a passphrase
ssh-copy-id -i ~/.ssh/hermes-contabo.pub root@<VPS_IP>
```

## 2. Non-root sudo user

```bash
adduser <user>
usermod -aG sudo <user>
```

**Gotcha:** `adduser admin` failed because a group named `admin` already exists on Ubuntu. Pick another name.

Copy root's key to the new user. Run this from the root session:

```bash
mkdir -p /home/<user>/.ssh
cp /root/.ssh/authorized_keys /home/<user>/.ssh/
chown -R <user>:<user> /home/<user>/.ssh
chmod 700 /home/<user>/.ssh && chmod 600 /home/<user>/.ssh/authorized_keys
```

## 3. Make `ssh` use the right key

**Gotcha:** SSH only tries default key names on its own (`id_ed25519`, `id_rsa`). A key named `hermes-contabo` is never offered, so SSH falls back to asking for the server password, which looks like the key "isn't working". Fix it with `~/.ssh/config` on the Mac:

```
Host newvps
    HostName <VPS_IP>
    User <user>
    IdentityFile ~/.ssh/hermes-contabo
    AddKeysToAgent yes
    UseKeychain yes
```

Now `ssh newvps` just works, and the passphrase is cached in the macOS Keychain.

**Gotcha:** if `ssh-copy-id` says "All keys were skipped because they already exist", the key is already installed for *that* user. Check that you targeted the right user (`root` vs `<user>`).

## 4. Lock down sshd

Keep a root session open while doing this, so you can't lock yourself out.

1. As `<user>`, confirm `sudo whoami` prints `root`.
2. In `/etc/ssh/sshd_config`, set `PermitRootLogin no` and `PasswordAuthentication no`.
3. Run `sudo systemctl restart ssh`.
4. In a **new** terminal, confirm `ssh newvps` still works. Only then close the root session.

## 5. Firewall

```bash
sudo ufw allow OpenSSH && sudo ufw enable
```

## 6. Key backup and theft plan

- Put a passphrase on every key. Check with `ssh-keygen -y -f ~/.ssh/KEY`: if it doesn't ask for a passphrase, the key is unprotected. Add one with `ssh-keygen -p -f ~/.ssh/KEY`.
- Back up private keys in a password manager. Note which server and user each one is for.
- Keep FileVault on.
- If the laptop is stolen, remove the key's line from `~/.ssh/authorized_keys` on every server, and from GitHub. The Contabo web console still works if you're locked out.
