# Reusing an Existing Domain for a Second CloudFront Site

*How `media.kentechlab.net` got a clean, secure address without buying a new domain.*

## The problem

Same starting point as the `kentechsolution.com` project: a site running on AWS CloudFront, only reachable through the default, ugly auto-generated URL. This time though, the site didn't need its own dedicated domain. It made more sense to hang it off a domain I already owned, `kentechlab.net`, as a subdomain instead.

## What I did, in plain terms

**1. Reused an existing domain instead of buying a new one.**
Since I already own `kentechlab.net` and manage its DNS through Cloudflare, I didn't need to register anything new or pay any extra domain fee. I just needed a subdomain, `media.kentechlab.net`, to point at the CloudFront distribution.

**2. Registered the subdomain on the CloudFront side.**
Just like with `kentechsolution.com`, the CloudFront distribution needed to be told to accept traffic for this specific hostname, and it needed a matching TLS certificate to secure it. That was set up as an alternate domain name on the distribution with a certificate issued through AWS Certificate Manager, the same core step as before, just applied to a subdomain of an existing domain rather than a brand new one.

**3. Pointed the subdomain at CloudFront using Cloudflare DNS.**
Since DNS for `kentechlab.net` lives in Cloudflare rather than Route 53, I added a CNAME record there: `media.kentechlab.net` pointing at the CloudFront distribution's default address. This record is set to DNS only (not proxied), meaning Cloudflare is purely acting as the phone book entry here. It's not intercepting, caching, or terminating traffic; it just tells browsers where to go, and from there the browser connects straight to CloudFront and gets the real, AWS-issued certificate.

**4. Verified it end to end.**
This is the step worth calling out honestly: at first glance, the certificate looked odd, it was showing as issued by NordVPN rather than Amazon. That turned out to be a red herring. My VPN's Threat Protection feature was locally intercepting and re-signing HTTPS traffic on my own machine, which meant I was looking at a certificate NordVPN generated for me, not the real one the rest of the world would see. Turning the VPN off and reloading the page showed the actual certificate, issued by Amazon, correctly covering `media.kentechlab.net`. Good reminder that "it works on my machine" can mean something different than expected when a VPN or DNS filter sits in the middle.

## Why it's a good pattern

This is the cheaper sibling of the `kentechsolution.com` setup. When a new project doesn't need its own standalone brand, hanging it off an existing domain as a subdomain gets the exact same result, a clean address, a real certificate, no third-party redirect, without paying for a second domain registration. The tradeoff is just that the URL carries the parent domain's name (`media.kentechlab.net`) instead of standing fully on its own, which is fine for an internal or supporting project and less ideal for something meant to be its own independent brand.

## Stack used

`AWS CloudFront` (hosting, alternate domain name, TLS certificate) · `AWS Certificate Manager` (certificate) · `Cloudflare DNS` (CNAME record, unproxied)
