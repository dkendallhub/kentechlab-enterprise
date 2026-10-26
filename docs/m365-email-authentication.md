# Microsoft 365 Email on a Lab Domain: SPF, DKIM, and DMARC

*How `kentechlab.net` went from a domain that sent no mail, to a locked-down domain, to a working Exchange Online domain with SPF, DKIM, and DMARC all passing, then back to a reject policy with reporting turned on.*

> **Sanitized for a public repo.** No tenant IDs, verification codes, passwords, mailbox addresses, or reporting addresses are included. DNS record names and their public targets are shown because anyone can look them up with `nslookup`.

---

## The goal

Get current, hands-on practice with the Exchange Online and email security side of Microsoft 365: domain verification, mail routing, and the three DNS-based checks receiving servers use to decide whether a message is genuine.

| Check | What it answers |
|---|---|
| **SPF** | Which servers are allowed to send mail for this domain? |
| **DKIM** | Was this message signed by the domain, and was it changed in transit? |
| **DMARC** | If SPF or DKIM fails, what should the receiver do, and where should it send reports? |

---

## Where the work was done

| Tool | Address | Used for |
|---|---|---|
| Microsoft 365 admin center | `admin.microsoft.com` | Tenant, domain verification, users, licenses |
| Cloudflare dashboard | `dash.cloudflare.com` | Public DNS records for `kentechlab.net`, DMARC reporting |
| Microsoft Defender portal | `security.microsoft.com` | DKIM keys and signing |
| Outlook on the web | `outlook.office.com` | Sending test mail |
| Command Prompt (`nslookup`) | Local PC | Confirming DNS changes against a public resolver |
| Gmail "Show original" | `mail.google.com` | Reading the receiver's authentication results |

---

## Phase 1: Lock down a domain that sends no mail

Before any mailbox existed, the domain had no SPF or DMARC record, which meant anyone could send mail claiming to be from `kentechlab.net` and receivers had no instructions to reject it.

| Record | Type | Value |
|---|---|---|
| `@` | TXT | `v=spf1 -all` |
| `_dmarc` | TXT | `v=DMARC1; p=reject;` |

- `-all` with no senders listed means **no server** may send for the domain.
- `p=reject` tells receivers to drop anything that fails.

**Verified** with `nslookup -type=txt kentechlab.net 1.1.1.1`. The local Pi-hole still returned "Non-existent domain" for a while because it had cached the earlier negative answer, so the check was run against Cloudflare's public resolver instead.

---

## Phase 2: Microsoft 365 tenant and domain verification

1. Started a **Microsoft 365 Business Basic** trial (monthly billing) and created the tenant's admin account on the default `onmicrosoft.com` domain.
2. **MFA** was enforced on first sign-in by Security defaults (Microsoft Authenticator).
3. Confirmed the subscription under **Billing → Your products**.
4. **Settings → Domains → Add domain** → `kentechlab.net`. Microsoft issued a `MS=ms…` TXT record; Cloudflare's one-time Domain Connect authorization added it.

**Note:** the wizard failed once with "Failed to get the options for managing DNS records." Closing the wizard, refreshing, and resuming from the Domains list fixed it. This is common on brand-new tenants.

**Design decision:** the admin account stays on `onmicrosoft.com` and has no license. If the custom domain's DNS ever breaks, admin sign-in still works, and admin rights stay separate from day-to-day mail (least privilege).

---

## Phase 3: Point mail at Exchange Online

At the "Add DNS records" step, Microsoft's automatic Cloudflare option was skipped. The records were added by hand so each one was understood, and so the existing SPF record would be **edited** instead of duplicated (two `v=spf1` records cause SPF to fail outright).

| Record | Type | Value | Purpose |
|---|---|---|---|
| `@` | MX (priority 0) | `kentechlab-net.mail.protection.outlook.com` | Delivers inbound mail to Exchange Online |
| `autodiscover` | CNAME | `autodiscover.outlook.com` | Lets Outlook find mailbox settings automatically |
| `@` | TXT (edited) | `v=spf1 include:spf.protection.outlook.com -all` | Only Microsoft 365 may send |
| `_dmarc` | TXT (temporary) | `v=DMARC1; p=none;` | Monitor-only while testing |

**Every CNAME is DNS only (grey cloud).** Cloudflare proxies CNAMEs by default, and a proxied record returns Cloudflare's addresses instead of Microsoft's target, which breaks autodiscover and DKIM validation.

**Result:** the domain showed **Healthy** and became the tenant's **Default** domain.

---

## Phase 4: Licensed mailbox

**Users → Active users → Add a user**

- Username on `@kentechlab.net`
- **Usage location: United States**, set before assigning the license (license assignment fails without it)
- **Microsoft 365 Business Basic** license
- Role: **User, no admin access**

---

## Phase 5: DKIM

**`security.microsoft.com` → Email & collaboration → Policies & rules → Threat policies → Email authentication settings → DKIM**

1. Selected `kentechlab.net` → **Create DKIM keys**. Microsoft keeps the private key and signs outbound mail; the public key is published through two CNAMEs.
2. Added both in Cloudflare, DNS only:

| Name | Type | Target |
|---|---|---|
| `selector1._domainkey` | CNAME | `selector1-kentechlab-net._domainkey.<tenant>.p-v1.dkim.mail.microsoft` |
| `selector2._domainkey` | CNAME | `selector2-kentechlab-net._domainkey.<tenant>.p-v1.dkim.mail.microsoft` |

3. Confirmed both were live before touching the portal:
   ```
   nslookup -type=cname selector1._domainkey.kentechlab.net 1.1.1.1
   nslookup -type=cname selector2._domainkey.kentechlab.net 1.1.1.1
   ```
4. The portal kept showing **CnameMissing** hours later, with a "last checked" time from before the records existed. Turning on **Sign messages for this domain with DKIM signatures** forced a recheck, and the status changed to **Valid**.

**Lesson:** confirm DNS from the outside first. Once `nslookup` showed the right targets, any remaining delay was on Microsoft's side, not DNS.

---

## Phase 6: Test and read the headers

Sent a test message from Outlook on the web to Gmail. It arrived in the **Inbox** (not spam). In Gmail, **⋮ → Show original**, the `Authentication-Results` header showed:

| Check | Result | Evidence in the header |
|---|---|---|
| SPF | **pass** | Microsoft's outbound IP listed as a permitted sender for the domain |
| DKIM | **pass** | `header.i=@kentechlab.net header.s=selector1`, signed by the domain's own key |
| DMARC | **pass** | `header.from=kentechlab.net`, policy still `p=NONE` during testing |

A Microsoft internal header further down shows `dkim=none (message not signed)`. That is Exchange's check **before** outbound signing and is expected; the `DKIM-Signature` header with `d=kentechlab.net` confirms the message was signed on the way out.

**Side finding:** Outlook for Windows refused to add the mailbox. It **found** the account (proving autodiscover works) but blocked it because **Business Basic includes web and mobile apps only**. Desktop Outlook requires Business Standard or Premium. Common help desk ticket; the fix is licensing, not DNS.

---

## Phase 7: Enforce and report

1. Changed DMARC back to `p=reject`.
2. Enabled **Cloudflare → Email → DMARC Management**, which appended an aggregate reporting address (`rua=mailto:…@dmarc-reports.cloudflare.net`) to the existing record. The change took about a minute to appear in the DNS table.

Final DMARC record shape:

```
v=DMARC1; p=reject; rua=mailto:<reporting-address>
```

Aggregate reports from Google, Microsoft, and other large receivers arrive roughly daily and show every source sending as the domain, with SPF and DKIM pass rates.

---

## Final state

| Record | Value | Status |
|---|---|---|
| MX | Exchange Online | ✅ |
| autodiscover | `autodiscover.outlook.com` | ✅ |
| SPF | Microsoft 365 only, hard fail | ✅ |
| DKIM | selector1 and selector2, signing as `kentechlab.net` | ✅ Valid |
| DMARC | `p=reject` with aggregate reporting | ✅ |

---

## Why this rollout order matters in production

On a real company domain, jumping straight to `p=reject` can block legitimate mail from senders nobody remembered: a billing platform, a marketing tool, a copier that scans to email. The safe sequence is:

1. Publish SPF and DKIM for every known sender.
2. Start DMARC at `p=none` **with reporting**.
3. Review the aggregate reports until every legitimate source passes.
4. Move to `p=quarantine`, then `p=reject`.

This lab followed the same order, just with one sender.

---

## Troubleshooting notes

| Symptom | Cause | Fix |
|---|---|---|
| Local `nslookup` says record doesn't exist after adding it | Pi-hole cached the earlier negative answer | Query `1.1.1.1` directly, or flush the Pi-hole cache and `ipconfig /flushdns` |
| Domain wizard: "domain was removed from your account" | New-tenant glitch | Close wizard, refresh, resume from Domains list |
| DKIM stuck on **CnameMissing** | Portal status not rechecked | Confirm CNAMEs with `nslookup`, then enable signing to force a recheck |
| Outlook desktop: "license does not enable access" | Business Basic has no desktop apps | Use Outlook on the web/mobile, or assign Business Standard/Premium |
| DMARC `rua` not visible right after enabling reporting | Cloudflare applies the change asynchronously | Wait a minute and refresh the DNS table |

---

## Roadmap

- [ ] Review the first DMARC aggregate reports in Cloudflare DMARC Management
- [ ] Practice message trace and a mail flow rule in the Exchange admin center (`admin.exchange.microsoft.com`)
- [ ] Repeat the DKIM and mailbox checks in PowerShell (`ExchangeOnlineManagement`: `Get-DkimSigningConfig`, `Get-Mailbox`, `Get-MessageTraceV2`)
- [ ] When the trial ends: remove MX, autodiscover, the SPF `include`, and both DKIM CNAMEs, and return SPF to `v=spf1 -all` so DNS doesn't point at a tenant that no longer exists

---

## Stack used

`Microsoft 365` · `Exchange Online` · `Microsoft Defender` · `Entra ID` · `Cloudflare DNS` · `Cloudflare DMARC Management` · `SPF` · `DKIM` · `DMARC` · `nslookup`
