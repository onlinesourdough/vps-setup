# VPS Production Setup Template

Reusable checklist for setting up and hardening a production VPS.

This template is intentionally provider- and person-neutral. Replace placeholders like `<admin_user>`, `<server_ip>`, `<domain>` and `<app_domain>` with your own values.

> Goal: one secure VPS that can run production apps, be deployed repeatably, and be monitored without relying on ad-hoc terminal access.

## 1) Server baseline and first access

### 1.1 Provision the VPS

- [ ] Choose a current Ubuntu LTS image.
- [ ] Add at least one public SSH key during provisioning.
- [ ] Store the private SSH key securely, ideally protected by passphrase or hardware-backed auth.
- [ ] Record server metadata:
  - [ ] provider/project
  - [ ] server name
  - [ ] public IPv4
  - [ ] public IPv6, if enabled
  - [ ] region/datacenter
  - [ ] SSH key fingerprints
- [ ] Enable provider backups or take an initial snapshot before major changes.

### 1.2 First login

```bash
ssh root@<server_ip>
```

Verify identity:

```bash
whoami
hostname
cat /etc/os-release
```

### 1.3 Update OS immediately

```bash
apt update
apt upgrade -y
```

Check whether reboot is required:

```bash
ls /var/run/reboot-required
```

If required:

```bash
reboot
```

After reboot:

```bash
uname -r
apt update
apt upgrade -y
```

### 1.4 Use `tmux` for longer sessions

```bash
apt install -y tmux
tmux new -s setup
```

Why: if SSH disconnects, the setup session keeps running.

### 1.5 Basic system identity and time

```bash
hostnamectl set-hostname <server_name>
timedatectl set-timezone <timezone>   # e.g. Europe/Copenhagen or Etc/UTC
```

Verify time synchronization (accurate time matters for TLS, logs and 2FA):

```bash
timedatectl status
```

`systemd-timesyncd` is fine for a single VPS. Install `chrony` if you want more accurate/instrumented NTP.

### 1.6 Enable automatic security updates

```bash
apt install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
```

Verify:

```bash
systemctl status unattended-upgrades
cat /etc/apt/apt.conf.d/20auto-upgrades
```

Note: kernel updates still require a reboot. Check `/var/run/reboot-required` regularly (see monthly checklist).

### 1.7 Swap (if the image ships without it)

Many cloud images ship with no swap. A small swap file prevents OOM kills under memory pressure:

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
sysctl vm.swappiness=10
echo 'vm.swappiness=10' > /etc/sysctl.d/99-swappiness.conf
```

### 1.8 Know your recovery path before hardening

Before changing SSH or firewall settings, confirm you can reach the provider's web-based console (e.g. Hetzner Console → server → Console). This is your way back in if you lock yourself out. Take a snapshot before risky changes.

## 2) User accounts, SSH hardening and key hygiene

### 2.1 Create a non-root admin user

Do not use root as the normal daily login user.

```bash
adduser <admin_user>
usermod -aG sudo <admin_user>
```

Test sudo:

```bash
su - <admin_user>
sudo whoami
```

### 2.1b Optional: passwordless sudo for the admin user

If an automation agent or tooling needs to run privileged commands over SSH non-interactively, allow the admin user to sudo without a password:

```bash
echo '<admin_user> ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/<admin_user>-nopasswd
chmod 440 /etc/sudoers.d/<admin_user>-nopasswd
visudo -c
```

`visudo -c` must report `parsed OK`. Always use a drop-in file in `/etc/sudoers.d/` — a syntax error in the main sudoers file can lock you out of sudo entirely.

Trade-off, stated honestly: with key-only SSH, the sudo password only protects against escalation by someone who already has your SSH key or an open session. Removing the prompt makes the SSH private key the key to everything — acceptable for a single-admin server if the private key is passphrase-protected and stored safely; reconsider on multi-user machines.

### 2.2 Add SSH key for the admin user

```bash
install -d -m 700 -o <admin_user> -g <admin_user> /home/<admin_user>/.ssh
nano /home/<admin_user>/.ssh/authorized_keys
chmod 600 /home/<admin_user>/.ssh/authorized_keys
chown <admin_user>:<admin_user> /home/<admin_user>/.ssh/authorized_keys
```

Test from a new local terminal before closing the root session:

```bash
ssh <admin_user>@<server_ip>
sudo whoami
```

### 2.3 Harden SSH

Create a drop-in config:

```bash
nano /etc/ssh/sshd_config.d/99-hardening.conf
```

Recommended baseline:

```conf
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
```

After the admin user has been tested, prefer:

```conf
PermitRootLogin no
AllowUsers <admin_user>
```

If root access must be kept temporarily for recovery, use key-only root access:

```conf
PermitRootLogin without-password
```

If a deployment runtime (e.g. Coolify) manages the host via SSH as root from its docker network, keep root blocked from the internet but allow it key-only from that network:

```conf
PermitRootLogin no
AllowUsers <admin_user>

Match Address <runtime_network>,127.0.0.1,::1
    PermitRootLogin prohibit-password
    AllowUsers <admin_user> root
```

Find the runtime's source network by checking accepted logins: `journalctl -u ssh | grep "Accepted publickey for root"`. Verify the runtime still works afterwards (e.g. trigger a deployment or test its SSH path directly).

Validate and reload:

```bash
sshd -t
systemctl reload ssh
sshd -T | grep -E 'passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|permitrootlogin|maxauthtries|allowusers'
```

### 2.4 SSH key hygiene

For every key in `authorized_keys`:

- [ ] Known owner.
- [ ] Known device or system.
- [ ] Clear comment.
- [ ] Still needed.
- [ ] Old devices removed.

Example comments:

```text
admin-laptop-2026-01-15
admin-workstation-2026-02-03
deploy-system-local-management-key
```

Backup before editing:

```bash
cp -a ~/.ssh/authorized_keys ~/.ssh/authorized_keys.backup-$(date -u +%Y%m%dT%H%M%SZ)
```

### 2.5 Brute-force protection (Fail2ban or CrowdSec)

A public VPS sees thousands of failed SSH attempts per day. Key-only auth stops them, but banning repeat offenders reduces noise and load, and protects any future weak spot.

Option A — Fail2ban (simple, battle-tested):

```bash
apt install -y fail2ban
```

Create `/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 4
# Never ban: localhost, deployment runtime docker network, VPN/Tailscale range
ignoreip = 127.0.0.1/8 ::1 <runtime_network> 100.64.0.0/10

[sshd]
enabled = true
backend = systemd
```

Note: on Ubuntu 24.04+, `backend = systemd` is required because there is no `/var/log/auth.log` journal file by default in all setups; verify the jail actually reads logs.

```bash
systemctl enable --now fail2ban
fail2ban-client status sshd
```

Unban a mistakenly banned IP:

```bash
fail2ban-client set sshd unbanip <ip>
```

Option B — CrowdSec (community blocklists, broader detection):

- [ ] Install the CrowdSec security engine.
- [ ] Install a remediation component (firewall bouncer) — without it, nothing is blocked.
- [ ] Verify decisions block traffic: `cscli decisions list`.

Either way:

- [ ] Document the chosen tool and unban procedure in the runbook.
- [ ] Whitelist your own admin IP/VPN range if static.

### 2.6 Kernel network hardening (sysctl)

Create `/etc/sysctl.d/99-hardening.conf`:

```conf
# Ignore ICMP redirects and source-routed packets
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_source_route = 0

# Log and drop spoofed/martian packets
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.log_martians = 1

# SYN flood protection
net.ipv4.tcp_syncookies = 1

# Ignore broadcast ICMP (smurf protection)
net.ipv4.icmp_echo_ignore_broadcasts = 1
```

Apply and verify:

```bash
sysctl --system
sysctl net.ipv4.tcp_syncookies
```

Note: skip `send_redirects=0`/`rp_filter` changes only if the box acts as a router/VPN gateway and you know you need them.

### 2.7 Reduce attack surface

- [ ] List listening services: `ss -tulpn`.
- [ ] Disable services you do not use (examples, verify first):

```bash
systemctl disable --now avahi-daemon 2>/dev/null || true
```

- [ ] Verify AppArmor is active (default on Ubuntu):

```bash
aa-status --enabled && echo "AppArmor OK"
```

- [ ] Confirm `unattended-upgrades` is running (see 1.6).

## 3) DNS, private access and firewall model

### 3.1 DNS records

DNS is mainly for readability, routing and certificates. SSH via domain is not automatically safer than SSH via IP; security comes from SSH keys, disabled password login, firewall/VPN and host key verification.

Suggested records:

| Type | Name | Target | Purpose |
|---|---|---|---|
| `A` | `app.<domain>` | VPS IPv4 | Public app |
| `AAAA` | `app.<domain>` | VPS IPv6 | Public app over IPv6, if used |
| `A` | `admin.<domain>` | VPS IPv4 | Admin UI, private/allowlisted only |
| `A` | `status.<domain>` | VPS IPv4 | Status page / uptime dashboard |
| `A` | `vps.<domain>` | VPS IPv4 | Human-friendly SSH/server alias |

Optional local SSH config:

```sshconfig
Host production-vps
  HostName vps.<domain>
  User <admin_user>
  IdentityFile ~/.ssh/<private_key_file>
```

Then connect with:

```bash
ssh production-vps
```

### 3.2 Private admin access

Recommended approach for admin-only services:

- [ ] Use a VPN/mesh network such as Tailscale or WireGuard.
- [ ] Keep SSH and admin UIs reachable only via VPN or trusted IP allowlist.
- [ ] Use private DNS/MagicDNS where available.

Typical private services:

- SSH
- app runtime dashboard
- metrics dashboard
- browser-based server admin UI
- database admin tools

### 3.3 Firewall baseline

Use the provider/cloud firewall (e.g. Hetzner Cloud Firewall, AWS Security Groups) as the first perimeter — it filters traffic before it reaches the server. Use a host firewall (UFW/nftables) as a second layer for defense-in-depth.

Critical Docker caveat: Docker publishes ports via its own iptables/DNAT rules (`DOCKER` chains and `docker-proxy`), which bypass UFW `INPUT` rules entirely. A container started with `-p 8080:80` is reachable from the internet even if `ufw deny 8080` is set. Therefore:

- Treat the cloud firewall as the primary perimeter for Docker hosts.
- Never rely on UFW alone to protect published Docker ports.
- Prefer not publishing ports at all (use reverse proxy + internal networks / `expose:`).
- If a port must be published for local use, bind it to loopback: `127.0.0.1:port:port`.
- For host-level container ingress policy, put rules in the `DOCKER-USER` iptables chain (Docker evaluates it before its own accept rules), or use the `ufw-docker` tool.
- Verify exposure from the outside, not just on the host: `nmap <server_ip>` from another machine, or an online port scanner.

Public inbound:

| Port | Purpose |
|---|---|
| `80/tcp` | HTTP redirect / certificate challenge |
| `443/tcp` | HTTPS |

Private or allowlisted inbound:

| Port | Purpose |
|---|---|
| `22/tcp` | SSH |
| app runtime admin port | Coolify/Dokploy/etc. |
| reverse proxy dashboard port | Traefik/Caddy/Nginx admin, if enabled |
| metrics dashboard port | Netdata/Prometheus/etc. |
| OS admin dashboard port | Cockpit/etc. |

Host firewall baseline (UFW), as second layer behind the cloud firewall:

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp    # or restrict: ufw allow from <admin_ip> to any port 22 proto tcp
ufw limit 22/tcp    # built-in SSH rate limiting
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
ufw status verbose
```

Warning: confirm SSH access for the admin user works before `ufw enable`, and keep an existing session open while testing. Remember the Docker caveat above — UFW does not protect published container ports.

Checklist:

- [ ] Only `80/443` are open to the world.
- [ ] SSH is restricted to VPN or trusted IPs.
- [ ] Admin dashboards are not public.
- [ ] IPv6 firewall rules match IPv4 rules.
- [ ] No Docker-published ports are unintentionally public (verify with external `nmap`).
- [ ] App still works after firewall changes.
- [ ] A new SSH session still works after firewall changes.

## 4) App runtime, containers, reverse proxy and TLS

### 4.1 Runtime choice

Common options:

| Runtime | Use when | Notes |
|---|---|---|
| Coolify | You want UI-based deployments, reverse proxy and app management | Good general-purpose VPS PaaS |
| Dokploy | You want a lightweight Docker-based deployment UI | Similar category to Coolify |
| Docker Compose | You want minimal moving parts and full control | More manual ops |
| Kubernetes | You need cluster-level orchestration | Usually too complex for one VPS |

### 4.2 Containerization

Prefer immutable app images over compiling directly on the production server.

Typical pattern:

- [ ] Build image in CI or locally.
- [ ] Push image to a registry.
- [ ] Deploy image via runtime or Docker Compose.
- [ ] Keep secrets in runtime/env secret store, not in git.
- [ ] Keep databases and internal services off the public internet.

Install Docker using the official instructions for the OS, then verify:

```bash
systemctl status docker
docker version
docker compose version
```

Optional, but security-sensitive:

```bash
usermod -aG docker <admin_user>
```

Note: membership in the `docker` group is effectively root-equivalent.

### 4.3 Reverse proxy

Expose the reverse proxy publicly, not every app container.

Common options:

- Traefik
- Caddy
- Nginx
- Runtime-managed reverse proxy

Important Docker rule:

- Avoid unnecessary `ports:` mappings in app services.
- Prefer internal networks and labels/routing through the reverse proxy.
- If a service must bind locally, bind to `127.0.0.1`, not `0.0.0.0`.

### 4.4 TLS / HTTPS

Every public app should use HTTPS.

Checklist:

- [ ] DNS points to the server or tunnel endpoint.
- [ ] Reverse proxy provisions certificates automatically.
- [ ] Certificates auto-renew.
- [ ] HTTP redirects to HTTPS.
- [ ] No public app is HTTP-only.

Test:

```bash
curl -I https://<app_domain>
```

## 5) Deployments, tunnels, monitoring and operations

### 5.1 Cloud tunnel / WAF option

For a more locked-down model, use a tunnel provider such as Cloudflare Tunnel:

- VPS opens outbound connection to tunnel provider.
- Users hit the tunnel provider, not the VPS IP directly.
- Public app can be served without opening app ports inbound.
- Web Application Firewall can sit in front of the app.
- Origin IP exposure is reduced.

Typical flow:

1. Add domain to DNS/tunnel provider.
2. Create tunnel.
3. Install tunnel daemon on VPS.
4. Route hostname to local service, e.g. `http://localhost:3000`.
5. Configure WAF and access rules.

### 5.2 Private deployments

If the app runtime dashboard/API is private, public Git webhooks may not reach it.

Options:

- [ ] Use runtime pull-based deployments.
- [ ] Use CI runner connected to VPN/mesh network.
- [ ] Use Tailscale GitHub Action or equivalent to reach private deployment API.
- [ ] Trigger deployments via API over the private network.

Goal: avoid exposing deployment/admin services just to receive webhooks.

### 5.3 Monitoring and alerts

Minimum recommended stack:

| Tool category | Purpose |
|---|---|
| Provider console | Server state, firewall, snapshots, graphs |
| App runtime UI | Deploy logs, container health, app status |
| Uptime monitor | External checks and alerts |
| Metrics dashboard | CPU/RAM/disk/network/container metrics |
| Log viewer | App/system logs |

Common tools:

- Uptime Kuma for uptime checks and notifications.
- Netdata for server metrics.
- Cockpit for browser-based OS overview, only behind VPN/allowlist.
- Prometheus/Grafana/Loki for more advanced monitoring and logs.

Alerts should go to at least one active channel:

- [ ] email
- [ ] Slack/Discord/Teams
- [ ] SMS/push for critical production systems

Important: if the uptime monitor runs on the same VPS, it may not alert when the whole VPS is down. Use an external monitor for critical apps.

### 5.4 Backups and restore

Backups are not complete until restore is tested.

Checklist:

- [ ] Provider snapshots/backups enabled.
- [ ] Manual snapshot before risky changes.
- [ ] Database backups automated.
- [ ] App volume backups automated.
- [ ] Secrets/config backup documented.
- [ ] Restore process documented.
- [ ] Restore test completed or scheduled.
- [ ] Retention policy defined.

### 5.5 Log hygiene

- [ ] Configure Docker log rotation so container logs cannot fill the disk. In `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "20m",
    "max-file": "3"
  }
}
```

Applies to new containers after `systemctl restart docker` (plan restart with runtime, e.g. Coolify).

- [ ] Verify journald disk limits: `journalctl --disk-usage`, cap with `SystemMaxUse=` in `/etc/systemd/journald.conf` if needed.

### 5.6 Periodic security audit

- [ ] Run Lynis for a host audit baseline:

```bash
apt install -y lynis
lynis audit system
```

Review the hardening index and warnings; treat it as a guide, not a compliance mandate.

- [ ] Review failed/successful logins periodically: `lastb | head`, `last | head`.
- [ ] Re-run an external port scan after infrastructure changes.

### 5.7 Monthly operations checklist

- [ ] Review uptime incidents.
- [ ] Review CPU/RAM/disk trends.
- [ ] Review app/container health.
- [ ] Check pending package updates.
- [ ] Check `/var/run/reboot-required`.
- [ ] Verify backups completed.
- [ ] Review SSH keys.
- [ ] Review firewall rules.
- [ ] Verify TLS certificate renewal.
- [ ] Verify Fail2ban/CrowdSec is active and banning (`fail2ban-client status sshd`).
- [ ] Check deployment and rollback procedure still works.

## Optional next-level hardening

Not required for a solid baseline, but worth considering per server:

- [ ] Tailscale/WireGuard so SSH and admin UIs are never public at all.
- [ ] Cloudflare Tunnel/WAF in front of public apps.
- [ ] SSH 2FA (TOTP via `libpam-google-authenticator`) for sensitive environments.
- [ ] `auditd` for forensic-grade audit logging.
- [ ] Off-site backups (e.g. restic to object storage) in addition to provider snapshots.
- [ ] CIS/STIG automation (e.g. ansible-lockdown) — test on staging first; strict profiles can break running services.
- [ ] Changing the SSH port: reduces log noise only, it is not hardening. If done, update firewall rules and all clients/CI.

## Production-ready acceptance criteria

- [ ] Admin user exists and daily login is not root.
- [ ] SSH is key-only.
- [ ] Password SSH login is disabled.
- [ ] Root SSH login is disabled or explicitly temporary.
- [ ] Brute-force protection (Fail2ban/CrowdSec) is active.
- [ ] Automatic security updates are enabled; reboot procedure for kernel updates exists.
- [ ] Time synchronization is active.
- [ ] Firewall exposes only required public ports, verified from outside.
- [ ] No Docker-published port bypasses the intended firewall model.
- [ ] Admin dashboards are private/allowlisted.
- [ ] Public apps use HTTPS with auto-renewing certificates.
- [ ] Apps deploy from reproducible images or documented compose/runtime config.
- [ ] Secrets are not committed to git.
- [ ] Backups exist and restore is documented.
- [ ] Monitoring and alerts exist.
- [ ] Log rotation prevents disk exhaustion.
- [ ] Runbook exists for common incidents.
- [ ] Provider recovery console access is confirmed.
