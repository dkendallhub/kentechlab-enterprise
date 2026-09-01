# Tiered Monitoring and a Daily Telegram Status Digest

*How the lab's uptime monitoring got organized by real-world impact, and how it now reports itself automatically every morning.*

## The problem

Uptime Kuma was already tracking every service in the lab, but everything sat in one flat list. A Pi-hole web admin page being down looked exactly as urgent as the public Kentech Solutions website being down, even though one is a personal convenience and the other is something a stranger or client could actually see fail. There was also no way to check on things without manually opening the dashboard, nothing came to me.

## What I did, in plain terms

**1. Split every monitor into two tiers.**
Tier 1 covers anything public facing, the actual domains someone would type or click (root and www versions of each site), and the DNS records behind them. If DNS breaks, the site is down for a visitor even if the server behind it is perfectly healthy, so DNS checks count as Tier 1 too. Tier 2 covers everything else: home lab tools, admin panels, and the raw CloudFront origin checks that matter for diagnosis but aren't themselves visitor facing.

**2. Cleaned up naming so the source is obvious at a glance.**
Monitors now use two prefixes: `CFR-` for anything served through AWS CloudFront, and `CFL-` for anything running through Cloudflare Tunnel or Cloudflare Pages. Before this, both used the same `CF-` prefix, which made it easy to mix up which platform was actually involved when scanning the list quickly.

**3. Tagged everything to match.**
Every monitor got a `Tier1` or `Tier2` tag in Uptime Kuma, so the dashboard can be filtered by urgency, not just by name.

**4. Built a script that reports the whole picture every morning, unprompted.**
Uptime Kuma only notifies on a status change, up to down or back again, it has no built in way to send a routine "here's everything, nothing needed" summary. So a small Python script runs on a schedule, pulls the live status of every monitor from Uptime Kuma's status page API, and posts a formatted report straight to Telegram: Tier 1 problems called out first if any exist, then the full list grouped by tier underneath.

## A few real problems along the way, and how they got solved

- **Cloudflare Access blocked the script entirely at first.** The public status page domain sits behind Cloudflare's Zero Trust login wall, which is correct for a human visitor but stops a script cold, it just gets served a login page instead of data. Fix: since the script runs on the same local network as Uptime Kuma, it talks to it directly over the LAN instead of going out through the public domain, skipping Cloudflare entirely for this internal use case.
- **Telegram's Markdown parser choked on the monitor names.** Names with underscores (`UGC_Router.local`) broke Telegram's formatting parser, since underscores are treated as italic markers. Fix: the message is sent as plain text instead of formatted Markdown, more reliable than trying to escape every special character correctly.
- **Uptime Kuma's bulk monitor import has a real bug for DNS-type monitors specifically**, it doesn't set a required field for that type, causing every DNS monitor in a batch to fail on import. Those three had to be added one at a time through the normal UI instead of the bulk JSON import used for everything else.

## Why it matters

The tiering means a glance at the morning message answers the actual question that matters: is anything a stranger could see broken right now, or is it just something on my own network. The script itself is a small, transferable pattern too, pulling structured status data from an API and pushing a formatted summary to a chat platform on a schedule, useful well beyond this specific lab.

## Stack used

`Uptime Kuma` (monitoring, tagging, status page API) · `Python 3` (script) · `Telegram Bot API` (delivery) · `Synology Task Scheduler` (cron)
