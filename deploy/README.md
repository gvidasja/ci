# Deploy action

Deploys a repo's build output to a remote host over SSH (from a GitHub-hosted
runner) and runs it as a **systemd user service of a dedicated, unprivileged
account**. CI never gets sudo: a leaked deploy key or a malicious commit
reaches only that one account — not root, not your own login account.

## One-time target setup (as root)

Per service (`myservice` = `service_name` = the account name):

```sh
useradd -m -s /bin/sh myservice      # Alpine/postmarketOS: adduser -D -s /bin/sh myservice
usermod -p '*' myservice             # no password, but not "locked" (sshd refuses locked accounts)
loginctl enable-linger myservice     # its services run without a login session and start at boot
install -d -m 700 -o myservice -g myservice ~myservice/.ssh
echo 'restrict ssh-ed25519 AAAA... ci-deploy myservice' > ~myservice/.ssh/authorized_keys
chown myservice: ~myservice/.ssh/authorized_keys && chmod 600 ~myservice/.ssh/authorized_keys
```

- Don't add the account to `sudo`/`wheel`. Use a fresh keypair per service.
- `restrict` disables port/agent/X11 forwarding and PTYs for the CI key;
  rsync and plain commands still work.
- The target needs `rsync` and `bash` (`apk add rsync bash` on Alpine).
- Keep state (e.g. a sqlite db) in the account's home **outside**
  `~/<service_name>/` — that dir is rsynced with `--delete` on every deploy.

The code lands in `~/<service_name>/`, the unit in
`~/.config/systemd/user/<service_name>.service`. A repo can ship its own
`.deploy/service.service` (a *user* unit: no `User=`,
`WantedBy=default.target`); otherwise the template runs
`bash ~/<service_name>/start.sh`. An executable `.deploy/build.sh` replaces
the default "copy all tracked files" step and must fill `dist/`.

Inspect on the target: `sudo -u myservice XDG_RUNTIME_DIR=/run/user/$(id -u myservice) systemctl --user status myservice`
(or `journalctl _SYSTEMD_USER_UNIT=myservice.service` as root).

## Public domain

The action doesn't touch nginx or TLS (that needs root). Either:

- **Reverse proxy, set up once by hand:** an nginx site proxying the domain
  to the app's localhost port, plus `certbot --nginx -d <domain>`. certbot's
  own timer renews it; nothing to redo per deploy.
- **Tunnel** (e.g. Cloudflare Tunnel): `cloudflared` on the host connects
  out, no open ports or certificates on the host — handy for a device on a
  home LAN.

## Reaching a LAN-only device (e.g. a postmarketOS phone)

Skip this whole section for a publicly reachable target (e.g. a GCE VM) —
just point `deploy_host` at its public IP and leave `wireguard_config`
empty; the action SSHes straight to it like any other host.

GitHub-hosted runners can't reach a device that only has a private LAN
address. Two options, in order of preference.

### Cloudflare Tunnel + Access (preferred)

If the device already runs `cloudflared`, route a hostname on that tunnel to
its sshd and gate it with an Access service token. The runner reaches only
that one SSH port — nothing else on the LAN — and nothing on the router is
involved.

1. **Access app first** (so the hostname is never reachable unprotected):
   Zero Trust → Access → Service Auth → create a service token. Then add a
   self-hosted application for e.g. `ssh-device.example.com` with a single
   policy: action **Service Auth**, include **Service Token** = that token.
2. **Tunnel route:** on the device's tunnel add a Public Hostname
   `ssh-device.example.com` → `ssh://localhost:22`.
3. **Secrets:** `deploy_host` = that hostname, `deploy_host_key` =
   `ssh-device.example.com ssh-ed25519 <key>` (required in this mode —
   `ssh-keyscan` can't go through the tunnel), and the token's Client ID /
   Secret as `cf_access_client_id` / `cf_access_client_secret`.

The action downloads `cloudflared` (no sudo) and uses
`cloudflared access ssh` as the SSH `ProxyCommand`. Leave `wireguard_config`
empty.

### WireGuard

If your router exposes WireGuard, have the job join it for the duration of
the deploy rather than:

- **port-forwarding SSH to the phone** — puts an always-on Linux phone
  directly on the internet, don't do this;
- **standing up a self-hosted runner** — extra host to patch, holds repo
  secrets at rest between jobs; only worth it if you need LAN access for
  more than this one job.

Set the `wireguard_config` input from a secret holding a full `wg-quick`
client config, e.g.:

```ini
[Interface]
PrivateKey = <client private key>
Address = 10.0.0.9/32
DNS = 10.0.0.1

[Peer]
PublicKey = <router's public key>
Endpoint = your-ddns-host:51820
AllowedIPs = 10.0.0.0/24   # just the LAN subnet — not 0.0.0.0/0
```

Scope `AllowedIPs` to the LAN subnet only, not `0.0.0.0/0` — the runner
should route to your phone, not tunnel *all* its traffic through your home
IP. Give this WireGuard peer its own keypair (don't reuse your personal
laptop's), so you can revoke CI's access independently.

`AllowedIPs` only limits what the *runner* routes; whoever holds the config
can change it. If your router supports it, also firewall the CI peer to just
the target's SSH port (e.g. `<target-ip>:22`), so a leaked config doesn't
open the rest of your LAN.

If you omit `wireguard_config`, the action assumes the runner already has
network access to `deploy_host` (e.g. a self-hosted runner on the LAN).

## Pinning the host key

Don't rely on `ssh-keyscan` alone (trust-on-first-use) — the action falls
back to it with a warning, but something on the network path could spoof
`deploy_host` before the real first connection. Read the key from a
channel that isn't the SSH connection itself:

```sh
# on the device
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
cat /etc/ssh/ssh_host_ed25519_key.pub
```

On GCE, the host key fingerprints are also printed to the serial console
log on first boot, so you can pull them without SSHing in blind:
`gcloud compute instances get-serial-port-output <instance>`.

Store `<deploy_host> ssh-ed25519 <key>` as the `deploy_host_key` secret.

## Inputs

| Input | Required | Notes |
|---|---|---|
| `service_name` | yes | unit name and `~/<service_name>/` dir |
| `deploy_key` | yes | SSH private key of the service account |
| `deploy_host` | yes | public IP/hostname works directly; tunnel hostname needs `cf_access_*`; LAN-only address needs `wireguard_config` |
| `deploy_host_key` | no (recommended) | pinned `known_hosts` line(s); falls back to TOFU `ssh-keyscan` with a warning |
| `deploy_user` | no | defaults to `service_name` |
| `env_file` | no | uploaded as `~/<service_name>/.env`, mode 600 |
| `wireguard_config` | no | full `wg-quick` client config; omit if the runner already reaches the host |
| `cf_access_client_id` / `cf_access_client_secret` | no | Cloudflare Access service token; SSH via `cloudflared access ssh`. Requires `deploy_host_key` |

## Example caller workflow

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: gvidasja/ci/deploy@master
        with:
          service_name: myservice
          deploy_host: ${{ secrets.DEPLOY_HOST }}
          deploy_host_key: ${{ secrets.DEPLOY_HOST_KEY }}
          deploy_key: ${{ secrets.DEPLOY_KEY }}
          env_file: ${{ secrets.ENV_FILE }}
          # LAN-only targets, one of:
          # cf_access_client_id: ${{ secrets.CF_ACCESS_CLIENT_ID }}
          # cf_access_client_secret: ${{ secrets.CF_ACCESS_CLIENT_SECRET }}
          # wireguard_config: ${{ secrets.WIREGUARD_CONFIG }}
```
