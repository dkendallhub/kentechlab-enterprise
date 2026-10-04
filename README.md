# Kentech Lab: Enterprise Infrastructure Lab

A personal enterprise infrastructure lab built to the same standard as production: identity, edge, containers, storage, network segmentation, and automation, wired together and actually monitored.

🌐 **Live sites:** [kentechlab.net](https://kentechlab.net) · [kentechsolution.com](https://kentechsolution.com) · [media.kentechlab.net](https://media.kentechlab.net)

---

## What this is

This repo hosts the source for the Kentech Lab landing page, a live dashboard-style site documenting a real home infrastructure environment. It's not a mockup; every system referenced (DNS redundancy, Zero Trust access, monitoring, segmentation) is running and load-bearing.

📄 **Latest changes:** [October 2026 lab update](docs/2026-10-lab-update.md): Zero Trust private access, external watchdog, Vaultwarden upgrade, Fios TV integration, and VLAN segmentation.

## Architecture overview

| Layer | What's running |
|---|---|
| **Edge / Access** | Cloudflare Tunnel + Zero Trust Access. Internal services published with no open inbound ports, identity-gated (email OTP) in front of every admin route. Path-scoped protection where apps need direct access (e.g. password manager admin panel only) |
| **Private Remote Access** | Cloudflare One (WARP) with a private CIDR route and Split Tunnel rules, giving enrolled devices access to the home LAN without publishing services, used for the Home Assistant mobile app |
| **Network / Segmentation** | UniFi Cloud Gateway Ultra routing isolated VLANs for Lab, Work, Gaming, Guest, and Private traffic (single NAT, documented address plan with reserved ranges) |
| **DNS** | Dual Pi-hole deployment (Raspberry Pi 5 primary, Docker on Synology secondary) for ad-blocking and DNS-level filtering, both published behind Zero Trust Access |
| **Storage** | Synology NAS (RAID array) with Hyper Backup offsite replication and snapshot retention |
| **Containers** | Docker on Synology: Vaultwarden (self-hosted password manager), Home Assistant, Uptime Kuma, Cloudflare tunnel daemon, secondary Pi-hole |
| **Monitoring (internal)** | Uptime Kuma tracking 32 services, tiered by real-world impact (Tier 1 public-facing vs Tier 2 internal), with keyword content checks, DNS-resolution checks, LAN vs. tunnel checks, and Telegram alerts. [Full writeup](docs/telegram-alerts.md) |
| **Monitoring (external)** | Python watchdog on GitHub Actions, triggered every 30 minutes by an external scheduler, authenticating through Zero Trust with a service token. Classifies full outages vs. single-service failures vs. auth failures, with hourly reminder and recovery emails |
| **Automation** | PowerShell scripts for backup verification and configuration drift detection |
| **Web Hosting / CDN** | AWS CloudFront + Route 53 + ACM. Registered domain (`kentechsolution.com`) with a public TLS certificate provisioned through ACM, attached as an alternate domain name on a CloudFront distribution, and routed via Route 53 alias records. [Full writeup](docs/aws-hosting.md) |
| **Second Web Hosting Pattern** | AWS CloudFront + ACM + Cloudflare DNS. A second CloudFront site given a clean address by reusing an existing domain (`media.kentechlab.net`) instead of buying a new one, with a real ACM certificate and a plain unproxied CNAME in Cloudflare, protected by Zero Trust Access. [Full writeup](docs/media-subdomain.md) |

## Why it's built this way

Most home labs either expose services directly to the internet (risky) or hide everything behind a VPN with no redundancy (fragile). This setup deliberately separates concerns:

- **Public-facing services** go through Cloudflare Tunnel. Nothing touches the router's port forwarding, and everything sits behind an identity check before it ever reaches the origin.
- **Private services stay private.** Home Assistant is reached through Zero Trust device enrollment (WARP), not a public hostname.
- **Traffic is segmented by trust level.** Lab experiments, work devices, gaming, and guests each live on their own isolated VLAN, so a problem in one can't reach the rest.
- **DNS is treated as critical infrastructure**, not an afterthought. A single Pi-hole going down shouldn't take the whole network's filtering with it.
- **Monitoring watches from inside and outside.** Uptime Kuma checks services from the LAN; an external watchdog checks from the internet, so a full outage (power, internet, or the NAS itself) still produces an alert.
- **Hosting isn't locked to one provider.** The homelab runs on Cloudflare, but client/portfolio-facing sites run on AWS (CloudFront + Route 53 + ACM), matching infrastructure choice to the job.

## Roadmap

- Second `cloudflared` connector on the Raspberry Pi 5 for tunnel redundancy
- UniFi access point to deliver the segmented networks over Wi-Fi
- WireGuard on the UniFi gateway with Dynamic DNS as a backup path independent of the NAS

## Tech stack

`AWS` `CloudFront` `Route 53` `ACM` `Azure` `Cloudflare` `Zero Trust` `Docker` `Synology` `UniFi` `VLANs` `Pi-hole` `Vaultwarden` `Home Assistant` `GitHub Actions` `PowerShell` `Python` `Uptime Kuma`

## Local development

The site is a single static `index.html` (no build step). To preview changes locally:

```bash
docker run -p 8000:80 -v $(pwd):/usr/share/nginx/html nginx
```

Then visit `localhost:8000`.

## Deployment

Hosted on Cloudflare, connected directly to this repo's `main` branch. Any push to `main` triggers an automatic rebuild and deploy, with no manual publish step.

---

<sub>© 2026 Kentech Lab</sub>
