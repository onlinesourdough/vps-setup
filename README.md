# VPS Production Setup Template

Reusable, provider-neutral checklist for setting up and hardening a production VPS — from first `ssh root@<server_ip>` to a monitored, backed-up, production-ready server.

Works with any VPS provider (Hetzner, DigitalOcean, Vultr, AWS Lightsail, ...). The only provider-specific concept used is a cloud/perimeter firewall and a web recovery console, which all major providers offer.

## How to use

1. Create a new repo from this template (or copy `VPS_PRODUCTION_SETUP_TEMPLATE.md` into the target project's docs).
2. Replace placeholders like `<admin_user>`, `<server_ip>`, `<domain>` and `<app_domain>` with real values.
3. Work through the checklist top to bottom — the order is intentional (recovery path before hardening, admin user before locking out root, firewall after SSH is verified).
4. Keep the filled-in copy in the project's infrastructure repo as living documentation, and use the acceptance criteria at the bottom as the definition of done.

## Contents

- `VPS_PRODUCTION_SETUP_TEMPLATE.md` — the full checklist:
  1. Server baseline and first access (updates, time, unattended-upgrades, swap, recovery path)
  2. User accounts, SSH hardening, key hygiene, Fail2ban/CrowdSec, sysctl, attack surface
  3. DNS, private admin access (VPN/Tailscale) and firewall model (incl. the Docker/UFW bypass caveat)
  4. App runtime (Coolify/Dokploy/Compose), containers, reverse proxy and TLS
  5. Deployments, tunnels, monitoring, backups, log hygiene and operations checklists

## Golden rules

- Never close your working SSH session before a new one is verified.
- The cloud firewall is the primary perimeter on Docker hosts — UFW does not protect published container ports.
- Backups are not done until a restore has been tested.
- Snapshot before every risky change.
