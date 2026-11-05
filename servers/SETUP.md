# Server setup (one time)

Target: one Hetzner server (Ubuntu/Debian), Docker Compose behind Caddy.
Do these steps once, from your own machine. Use the credentials Hetzner gave you only for this bootstrap, then rotate or disable them.

## 1. Create the deploy user

```bash
ssh root@<server-ip>
adduser --disabled-password --gecos "" deploy
mkdir -p /home/deploy/.ssh && chmod 700 /home/deploy/.ssh
```

On your own machine, generate a dedicated deploy key (no passphrase, used only by GitHub Actions):

```bash
ssh-keygen -t ed25519 -f deploy_key -C "github-actions-deploy" -N ""
```

Install the public key on the server and keep the private key for the GitHub secret:

```bash
# on the server, as root
cat >> /home/deploy/.ssh/authorized_keys   # paste deploy_key.pub, then Ctrl-D
chmod 600 /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
```

Also add your own personal SSH key to the same file so you can log in as `deploy`.

## 2. Install Docker

```bash
curl -fsSL https://get.docker.com | sh
usermod -aG docker deploy
```

Note: the `docker` group is effectively root on the machine. That is the usual trade-off for compose deployments, so keep the deploy key secret and only in GitHub Environment secrets.

## 3. Harden SSH

Edit `/etc/ssh/sshd_config.d/99-hardening.conf`:

```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Before reloading, open a second terminal and check that `ssh deploy@<server-ip>` works. Then:

```bash
systemctl reload ssh
```

## 4. Firewall

Use a Hetzner Cloud Firewall (recommended) or ufw. Allow inbound: 22, 80, 443, and for the mailserver 25, 465, 587, 993. Drop everything else.

```bash
# ufw alternative
ufw default deny incoming
ufw allow 22,80,443,25,465,587,993/tcp
ufw allow 443/udp
ufw enable
```

Docker publishes ports around ufw rules, so the Hetzner Cloud Firewall is the more reliable choice. Only publish ports in `docker-compose.yml` that you actually want public.

## 5. Automatic security updates

```bash
apt-get install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
```

## 6. App directory and registry login

```bash
mkdir -p /opt/app && chown deploy:deploy /opt/app
```

Copy `docker-compose.yml`, `Caddyfile`, `.env.example` (as `.env`) and `backend.env.example` (as `backend.env`) from `servers/` into `/opt/app/` and edit them (domains, image names, real values).

Log the server in to GHCR as the `deploy` user, using a classic personal access token with only the `read:packages` scope (ideally from a machine/bot account):

```bash
sudo -iu deploy
echo "<PAT>" | docker login ghcr.io -u <github-username> --password-stdin
```

## 7. First start

```bash
cd /opt/app
docker compose pull
docker compose up -d
docker compose ps
```

## 8. GitHub secrets

In each project repo (or once at organization level), create the GitHub Environments `staging` and/or `production` and add these **environment secrets**:

| Secret | Value |
| --- | --- |
| `DEPLOY_SSH_KEY` | contents of the private `deploy_key` |
| `DEPLOY_HOST` | server hostname or IP |
| `DEPLOY_USER` | `deploy` |
| `DEPLOY_KNOWN_HOSTS` | output of `ssh-keyscan -t ed25519 <server-ip>` |

Check the host key fingerprint against what Hetzner's console or `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` on the server shows, so you pin the real key and not a man-in-the-middle's. Add required reviewers to the `production` environment.

Delete the local `deploy_key` files once the secret is stored.

## 9. Backups

Enable Hetzner server backups or take regular snapshots. Back up the Docker volumes (`mail_data`, `caddy_data`, and any database volumes) separately if the data matters more than the server image.

## Mailserver notes

- Hetzner blocks outbound port 25 on new accounts by default. Request the unblock in the Hetzner console.
- Set reverse DNS (PTR) for the server IP to your mail hostname (e.g. `mail.example.com`).
- Publish SPF, DKIM and DMARC DNS records for your sending domain.
- IP reputation and deliverability are the hard part of self-hosted mail. Consider a dedicated server for the mailserver so web deploys and mail problems cannot affect each other.
