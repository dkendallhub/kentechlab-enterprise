# Kentech Lab — Enterprise Infrastructure Lab

A personal enterprise infrastructure lab built to the same standard as production: identity, edge, containers, storage, and automation — wired together and actually monitored.

🌐 **Live site:** [kentechlab.net](https://kentechlab.net)

---

## What this is

This repo hosts the source for the Kentech Lab landing page — a live dashboard-style site documenting a real home infrastructure environment. It's not a mockup; every system referenced (DNS redundancy, Zero Trust access, monitoring) is running and load-bearing.

## Architecture overview

| Layer | What's running |
|---|---|
| **Edge / Access** | Cloudflare Tunnel + Zero Trust Access — internal services published securely with no open inbound ports, identity-gated (email OTP) in front of every route |
| **DNS** | Dual Pi-hole deployment (Raspberry Pi 5 primary, Raspberry Pi Zero 2 W secondary) for ad-blocking and DNS-level filtering, with manual and DHCP-level failover paths |
| **Storage** | Synology NAS (RAID array) with Hyper Backup offsite replication and snapshot retention |
| **Containers** | Docker Compose stacks for internal tooling — Uptime Kuma, Cloudflare tunnel daemon, secondary Pi-hole instance |
| **Monitoring** | Uptime Kuma tracking every service via both its public (Cloudflare) route and local-direct address, plus DNS-resolution-specific checks — not just "is the webpage up" |
| **Remote Access** | WireGuard VPN (UniFi Cloud Gateway) with Cloudflare-backed Dynamic DNS, giving full network-level access and home DNS filtering from anywhere |
| **Automation** | PowerShell scripts for backup verification and configuration drift detection |

## Why it's built this way

Most home labs either expose services directly to the internet (risky) or hide everything behind a VPN with no redundancy (fragile). This setup deliberately separates concerns:

- **Public-facing services** go through Cloudflare Tunnel — nothing touches the router's port forwarding, and everything sits behind an identity check before it ever reaches the origin.
- **DNS is treated as critical infrastructure**, not an afterthought — a single Pi-hole going down shouldn't take the whole network's ad-blocking and filtering with it.
- **Monitoring distinguishes edge failures from origin failures** — every service has both a "through Cloudflare" and a "direct to the LAN" check, so an alert immediately tells you *where* the problem is, not just *that* there is one.

## Tech stack

`AWS` `Azure` `Cloudflare` `Docker` `Synology` `UniFi` `WireGuard` `PowerShell` `Uptime Kuma`

## Local development

The site is a single static `index.html` (no build step). To preview changes locally:

```bash
docker run -p 8000:80 -v $(pwd):/usr/share/nginx/html nginx
```

Then visit `localhost:8000`.

## Deployment

Hosted on Cloudflare Pages, connected directly to this repo's `main` branch. Any push to `main` triggers an automatic rebuild and deploy — no manual publish step.

---

<sub>© 2026 Kentech Lab</sub>
