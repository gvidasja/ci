# Deploy action

Deploys a repo's build output + a systemd unit (and, optionally, nginx +
certbot when a public `domain` is set) to a remote device over SSH, from a
GitHub-hosted runner.

## Reaching a LAN-only device (e.g. a postmarketOS phone)

Skip this whole section for a publicly reachable target (e.g. a GCE VM) —
just point `deploy_host` at its public IP and leave `wireguard_config`
empty; the action SSHes straight to it like any other host.

GitHub-hosted runners can't reach a device that only has a private LAN
address. Since your router already exposes WireGuard, have the job join it
for the duration of the deploy rather than:

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

## Locking down sudo on the target

The deploy user needs passwordless sudo for exactly the commands this
action runs — not full sudo. On the target device (adjust `myservice`/paths
to match your `service_name`/`domain`):

```
# /etc/sudoers.d/ci-deploy-myservice
gvidasja ALL=(root) NOPASSWD: \
  /usr/bin/install -m 644 /tmp/deploy-config/service.service /etc/systemd/system/myservice.service, \
  /bin/systemctl daemon-reload, \
  /bin/systemctl enable --now myservice, \
  /bin/systemctl restart myservice
```

Only add the nginx/certbot lines if you're actually using a `domain` for
this service:

```
  /usr/bin/install -m 644 -C /tmp/deploy-config/nginx.conf /etc/nginx/sites-available/example.com.conf, \
  /usr/sbin/nginx -t, \
  /bin/systemctl reload nginx, \
  /usr/bin/test -e /etc/letsencrypt/live/example.com/fullchain.pem, \
  /usr/bin/certbot --nginx -d example.com *
```

`certbot --nginx` is the broadest line here (it can rewrite nginx config
files via its plugin), so the action now checks
`/etc/letsencrypt/live/<domain>/fullchain.pem` first and only invokes
certbot when there's no certificate yet — subsequent deploys skip it
entirely and just reload nginx. Renewal is handled by certbot's own
installed timer, not by this action.

On postmarketOS (Alpine-based) you'll first need `apk add openssh rsync
sudo`, and, if you want nginx/certbot, `apk add nginx certbot
certbot-nginx`. Debian/GCE images ship openssh/rsync already; nginx/certbot
are `apt install nginx certbot python3-certbot-nginx`.

## Inputs

| Input | Required | Notes |
|---|---|---|
| `service_name` | yes | |
| `domain` | no | Leave empty for LAN-only devices — nginx/certbot are skipped entirely. |
| `port` | no | default `12345` |
| `deploy_key` | yes | SSH private key |
| `deploy_host` | yes | public IP/hostname works directly; LAN-only address needs `wireguard_config` first |
| `deploy_host_key` | no (recommended) | pinned `known_hosts` line(s); falls back to TOFU `ssh-keyscan` with a warning |
| `deploy_user` | no | default `gvidasja` |
| `deploy_email` | required if `domain` set | for certbot |
| `env_file` | no | uploaded as `.env`, mode 600 |
| `wireguard_config` | no | full `wg-quick` client config; omit if the runner already has LAN access |

## Example caller workflow

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: gvidasja/ci/deploy@main
        with:
          service_name: myservice
          deploy_host: 10.0.0.9
          deploy_host_key: ${{ secrets.DEPLOY_HOST_KEY }}
          deploy_key: ${{ secrets.DEPLOY_KEY }}
          wireguard_config: ${{ secrets.WIREGUARD_CONFIG }}
          env_file: ${{ secrets.APP_ENV }}
          # domain/deploy_email omitted — this is a LAN-only deploy
```

For a publicly reachable target (e.g. GCE), drop `wireguard_config`, point
`deploy_host` at the public IP, and add `domain`/`deploy_email` for nginx +
certbot.
