# KenTech Lab Update — Sept 30 to Oct 4, 2026

Security hardening, private remote access, external monitoring, a Vaultwarden upgrade, Verizon Fios TV integration behind UniFi, and a VLAN segmentation design.

> **Sanitized for a public repo.** No credentials, tokens, account IDs, tunnel IDs, email addresses, MAC addresses, or public IP addresses are included. Internal addresses are RFC 1918 private ranges only.

---

## 1. Architecture at a glance

```
Internet
   │
Verizon ONT ──► UniFi Cloud Gateway Ultra (UCG Ultra) ── router, firewall, VLANs, DHCP
                   │
                   ├── Main LAN 192.168.1.0/24
                   │     ├── Synology DS224  (Docker: cloudflared, Vaultwarden, Home Assistant,
                   │     │                    Uptime Kuma, Pi-hole secondary)
                   │     ├── Synology DS218  (DSM, Plex)
                   │     ├── Raspberry Pi 5  (Pi-hole primary)
                   │     ├── TP-Link Deco mesh (household Wi-Fi, Access Point mode)
                   │     └── Verizon Fios router (TV only) ──coax/MoCA──► Fios set-top boxes
                   │                                         (192.168.10.0/24, behind the Fios router)
                   └── VLANs: Lab, Work, Gaming, Guest, Private (see section 7)

Cloudflare Tunnel (cloudflared on DS224) ──► published hostnames, protected by Cloudflare Access
Cloudflare One (WARP) ──► private access to the Main LAN from enrolled devices
GitHub Actions + cron-job.org ──► external watchdog with email alerts
```

---

## 2. Cloudflare Tunnel — published routes

| Hostname | Backend | Service type | Notes |
|---|---|---|---|
| `home.kentechlab.net` | DS224 — Home Assistant | HTTP | Behind Access |
| `nas.kentechlab.net` | DS224 — DSM | HTTPS (No TLS Verify) | Behind Access |
| `nas2.kentechlab.net` | DS218 — DSM | HTTPS (No TLS Verify) | Behind Access |
| `status.kentechlab.net` | DS224 — Uptime Kuma | HTTP | Behind Access |
| `vault.kentechlab.net` | DS224 — Vaultwarden (port 8082) | HTTP | `/admin` only behind Access |
| `pihole.kentechlab.net` | Pi 5 — Pi-hole primary | HTTPS (No TLS Verify) | **New.** Behind Access |
| `pihole2.kentechlab.net` | DS224 — Pi-hole secondary (Docker) | HTTP | **New.** Behind Access |

**Removed:** `plex.kentechlab.net`. Plex now uses Plex's own Remote Access (end-to-end encrypted, avoids routing video through Cloudflare).

**Other DNS (not tunnel):** `media.kentechlab.net` → AWS CloudFront (behind Access). The apex domain serves a Cloudflare Worker.

---

## 3. Cloudflare Access — applications and policies

**Approach:** every self-hosted admin interface sits behind an Access application with an email allowlist policy (one-time PIN login). Policies were audited and a malformed email entry was corrected in all of them; unused and duplicate policies were removed.

| Application | Destination | Policies |
|---|---|---|
| home | `home.kentechlab.net` | Allow (email list) + watchdog Service Auth |
| nas224dsm | `nas.kentechlab.net` | Allow (email list) + watchdog Service Auth |
| nas218dsm | `nas2.kentechlab.net` | Allow (email list) + watchdog Service Auth |
| status224 | `status.kentechlab.net` | Allow (email list) + watchdog Service Auth |
| pihole-pi5 | `pihole.kentechlab.net` | Allow (email list) + watchdog Service Auth |
| pihole2-ds224 | `pihole2.kentechlab.net` | Allow (email list) + watchdog Service Auth |
| media | `media.kentechlab.net` | Allow (email list) + watchdog Service Auth |
| vault-admin | `vault.kentechlab.net/admin` (path-scoped) | Allow (email list) |

**Design decisions**

- **Vaultwarden main site is not behind Access** so the Bitwarden mobile/desktop apps keep working. Only the admin panel path is protected (Access in front, admin token behind it).
- **Removed the `vpn` Access app.** Its DNS record is DNS-only (grey cloud) for WireGuard, and Access cannot protect non-proxied records.
- **Watchdog uses a dedicated Access service token** (Service Auth policy) so external checks reach the real service instead of stopping at the Access login page.

---

## 4. Private remote access — Cloudflare One (WARP)

Goal: use the Home Assistant iPhone app away from home without exposing Home Assistant publicly.

- **CIDR route** on the tunnel: `192.168.1.0/24` (Main LAN).
- **Gateway proxy:** TCP and UDP enabled (required for private network routing). TLS inspection left off.
- **Split Tunnels (Exclude mode):** removed `192.168.0.0/16` and re-added every 192.168 range *except* `192.168.1.0/24`:
  `192.168.0.0/24`, `192.168.2.0/23`, `192.168.4.0/22`, `192.168.8.0/21`, `192.168.16.0/20`, `192.168.32.0/19`, `192.168.64.0/18`, `192.168.128.0/17`
- **Device enrollment:** policy restricted to an email allowlist (one-time PIN).
- **Home Assistant app:** internal and external URLs both point to the LAN address; reachable over cellular only while WARP is connected.

**Known trade-offs:** WARP must be connected for remote HA access; while WARP is on, phone DNS goes through Cloudflare rather than Pi-hole; remote networks that also use `192.168.1.x` will conflict.

---

## 5. Vaultwarden upgrade

**Problem:** the Bitwarden iOS app showed "We're unable to process your request." Server logs showed every request returning 200, so the server worked but its data format was too old for the current app. The running container used an orphaned (`<none>`) image.

**Fix**

1. Exported the vault (encrypted JSON) and copied the data folder with the container stopped.
2. Pulled the latest `vaultwarden/server` image.
3. Recreated the container with the same `/data` volume and environment variables.
4. Rotated the admin token (long random value, stored in the vault).
5. Moved SMTP to a dedicated Gmail app password; rotated the previous app password.
6. Port 8081 was still reserved by the old container, so the new one runs on **8082**; the tunnel route was updated to match.

**Result:** web vault, admin panel, SMTP test email, and Bitwarden iOS sync all working. Two-step login confirmed intact.

**Sharing:** created an organization with a shared collection; members join by invitation (open signups disabled) and must be confirmed by the owner.

---

## 6. Monitoring

### Internal — Uptime Kuma (DS224, Telegram alerts)

Naming convention adopted:

| Prefix | Meaning |
|---|---|
| `CFL-` | Check through Cloudflare (tunnel path) |
| `CFR-` | Public website / external resource |
| `LAN-` | Direct check on the local network |

Changes: Plex monitor converted to a LAN HTTP check of Plex's identity endpoint (`LAN-DS218: Plex`); Vaultwarden switched to its `/alive` health endpoint; added `LAN-DS224: Vaultwarden` so a failure can be pinned to the tunnel vs. the container.

### External — watchdog (separate private repo)

- Python script run by GitHub Actions; checks 8 services from outside the network using the Access service token.
- Classifies failures: **full outage** (every tunnel service down), **service down** (HTTP 502/503/504 behind a healthy tunnel), or **auth failure** (service token rejected).
- Emails on first failure, reminds hourly while down, and sends a recovery email. State is kept in a committed `state.json`.
- **Scheduling:** GitHub's built-in cron proved unreliable (one scheduled run overnight), so **cron-job.org** triggers the workflow every 30 minutes via the GitHub API (fine-grained token scoped to one repo, Actions read/write only). GitHub's cron remains as a 3-hour backup. cron-job.org notifies on trigger failure.
- Secrets live only in GitHub Actions secrets: `CF_ACCESS_CLIENT_ID`, `CF_ACCESS_CLIENT_SECRET`, `SMTP_USER`, `SMTP_PASS`, `ALERT_TO` (optional `SMTP_HOST`, `SMTP_PORT`).
- **Verified end to end:** down alert, hourly reminders, and recovery email, all from automatic runs.

---

## 7. Verizon Fios TV behind UniFi

**Problem:** after replacing the Fios router with the UCG Ultra, the Fios set-top boxes lost guide, DVR, and On Demand data. The boxes depend on the Fios router over coax (MoCA).

**Fix**

1. Isolated the Fios router and changed its LAN to `192.168.10.1/24` so it no longer overlaps the Main LAN.
2. Found the Fios router's **DHCP server was disabled** (the cause of clients getting 169.254 addresses); enabled it with range `.100–.200` and a 24-hour lease.
3. Disabled its Wi-Fi.
4. Disabled the **WAN Coax Link** so internet comes in over Ethernet and coax is used only for the TV boxes (MoCA LAN stays on).
5. Connected the Fios router's **WAN port** to the UniFi network (never a LAN port) and gave it a fixed IP in UniFi.
6. Added a MoCA-rated splitter where the router and a set-top box share a coax outlet.

**Result:** set-top boxes reconnected. Pi-hole query logs showed the router's Verizon lookups being allowed, ruling out DNS blocking.

---

## 8. Network segmentation (UniFi VLANs)

All networks are routed by the UCG Ultra (single NAT — no double NAT). Each VLAN ID matches the third octet of its subnet.

| Network | Subnet | VLAN | DHCP | Purpose / policy |
|---|---|---|---|---|
| Main | `192.168.1.0/24` | 1 | `.100–.200` | Trusted devices and servers |
| WireGuard VPN | `192.168.2.0/24` | — | — | UCG Ultra WireGuard clients (reserved) |
| Teleport VPN | `192.168.3.0/24` | — | — | UniFi Teleport (reserved — discovered via overlap error) |
| **Lab** | `192.168.4.0/24` | 4 | `.100–.200` | Home lab and DR testing. Isolated. DHCP Guarding off (to allow test DHCP servers) |
| **Work** | `192.168.5.0/24` | 5 | `.100–.200` | Employer laptop. Isolated. Auto DNS. DHCP Guarding on |
| *Reserved* | `192.168.6–8.0/24` | 6–8 | — | Future: Media, IoT/Smart Home, Cameras |
| **Gaming** | `192.168.9.0/24` | 9 | `.100–.200` | Gaming devices and game server. Isolated. Client isolation off |
| Verizon TV | `192.168.10.0/24` | — | — | Behind the Fios router (not a UniFi VLAN) |
| **Guest** | `192.168.11.0/24` | 11 | `.100–.200` | UniFi Guest Network (built-in isolation, internet only) |
| **Private** | `192.168.12.0/24` | 12 | `.100–.200` | Untrusted browsing and experimenting. Isolated, strictest filtering |

**Conventions:** `.1` gateway; `.2–.99` fixed/reserved addresses; `.100–.200` DHCP. Auto-Scale disabled so ranges are fixed. IPv6 off for now.

**Delivery:** Wi-Fi VLANs require a UniFi access point (the Deco mesh cannot tag VLANs). Planned: one UniFi U7 Lite on a single trunk port (Main native, all VLANs tagged), up to four SSIDs (or PPSK). Until then, VLANs are reachable via UCG Ultra ports assigned per network.

**Game server plan:** wired to a Gaming port with a fixed address; prefer private access (WireGuard/Tailscale/ZeroTier) over port forwarding; UPnP, if used, limited to the Gaming network.

---

## 9. Security hygiene

- Corrected malformed allowlist entries across Access policies.
- Removed unused/duplicate Access policies and an Access app that could not apply.
- Protected the Vaultwarden admin panel with a path-scoped Access app; rotated the admin token.
- Rotated an SMTP app password and a GitHub token after partial exposure during troubleshooting.
- Kept every secret in a password vault or GitHub Actions secrets — never in code.

---

## 10. Roadmap

- [ ] Add the Raspberry Pi 5 as a second `cloudflared` connector (tunnel redundancy; distinguishes DS224 failure from internet outage). Blocked on recovering the Pi's system login.
- [ ] Update container images one at a time (`cloudflared` first, once the second connector exists; then Uptime Kuma; then Pi-hole).
- [ ] Install the U7 Lite, adopt it, and create SSIDs for Lab, Work, Guest, and Private.
- [ ] WireGuard on the UCG Ultra plus Dynamic DNS (backup path independent of the DS224).
- [ ] Cloudflare Gateway DNS policy blocking security-risk categories for WARP clients.
- [ ] Rename Uptime Kuma monitors to the `CFL-`/`CFR-`/`LAN-` convention; remove paused duplicates.
- [ ] Before March 2028: update the GitHub API version header used by the cron-job.org trigger.
