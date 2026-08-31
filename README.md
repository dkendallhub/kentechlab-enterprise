# Kentech Lab: Enterprise Infrastructure Lab

A personal enterprise infrastructure lab built to the same standard as production: identity, edge, containers, storage, and automation, wired together and actually monitored.

🌐 **Live sites:** [kentechlab.net](https://kentechlab.net) · [kentechsolution.com](https://kentechsolution.com) · [media.kentechlab.net](https://media.kentechlab.net)

---

## What this is

This repo hosts the source for the Kentech Lab landing page, a live dashboard-style site documenting a real home infrastructure environment. It's not a mockup; every system referenced (DNS redundancy, Zero Trust access, monitoring) is running and load-bearing.

## Architecture overview

| Layer | What's running |
|---|---|
| **Edge / Access** | Cloudflare Tunnel + Zero Trust Access. Internal services published securely with no open inbound ports, identity-gated (email OTP) in front of every route |
| **DNS** | Dual Pi-hole deployment (Raspberry Pi 5 primary, Raspberry Pi Zero 2 W secondary) for ad-blocking and DNS-level filtering, with manual and DHCP-level failover paths |
| **Storage** | Synology NAS (RAID array) with Hyper Backup offsite replication and snapshot retention |
| **Containers** | Docker Compose stacks for internal tooling: Uptime Kuma, Cloudflare tunnel daemon, secondary Pi-hole instance |
| **Monitoring** | Uptime Kuma tracking every service via both its public (Cloudflare) route and local-direct address, plus DNS-resolution-specific checks, not just "is the webpage up" |
| **Remote Access** | WireGuard VPN (UniFi Cloud Gateway) with Cloudflare-backed Dynamic DNS, giving full network-level access and home DNS filtering from anywhere |
| **Automation** | PowerShell scripts for backup verification and configuration drift detection |
| **Web Hosting / CDN** | AWS CloudFront + Route 53 + ACM. Registered domain (`kentechsolution.com`) with a public TLS certificate provisioned through ACM, attached as an alternate domain name on a CloudFront distribution, and routed via Route 53 alias records (no third-party redirect/shortener in the path). [Full writeup](docs/aws-hosting.md) |
| **Second Web Hosting Pattern** | AWS CloudFront + ACM + Cloudflare DNS. A second CloudFront site given a clean address by reusing an existing domain (`media.kentechlab.net`) instead of buying a new one: a real ACM certificate on the CloudFront side, with a plain unproxied CNAME in Cloudflare doing the DNS pointing. Same security result as the dedicated domain, no extra registration cost. [Full writeup](docs/media-subdomain.md) |

## Why it's built this way

Most home labs either expose services directly to the internet (risky) or hide everything behind a VPN with no redundancy (fragile). This setup deliberately separates concerns:

- **Public-facing services** go through Cloudflare Tunnel. Nothing touches the router's port forwarding, and everything sits behind an identity check before it ever reaches the origin.
- **DNS is treated as critical infrastructure**, not an afterthought. A single Pi-hole going down shouldn't take the whole network's ad-blocking and filtering with it.
- **Monitoring distinguishes edge failures from origin failures.** Every service has both a "through Cloudflare" and a "direct to the LAN" check, so an alert immediately tells you *where* the problem is, not just *that* there is one.
- **Hosting isn't locked to one provider.** The homelab runs on Cloudflare, but client/portfolio-facing sites run on AWS (CloudFront + Route 53 + ACM), matching infrastructure choice to the job rather than defaulting to a single stack everywhere.

## Tech stack

`AWS` `CloudFront` `Route 53` `ACM` `Azure` `Cloudflare` `Docker` `Synology` `UniFi` `WireGuard` `PowerShell` `Uptime Kuma`

## Local development

The site is a single static `index.html` (no build step). To preview changes locally:

```bash
docker run -p 8000:80 -v $(pwd):/usr/share/nginx/html nginx
```

Then visit `localhost:8000`.

## Deployment

Hosted on Cloudflare Pages, connected directly to this repo's `main` branch. Any push to `main` triggers an automatic rebuild and deploy, no manual publish step needed.

---

<sub>© 2026 Kentech Lab</sub>
