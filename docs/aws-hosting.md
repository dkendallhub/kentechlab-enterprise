# Setting Up a Real Domain for a CloudFront Site

*How `kentechsolution.com` went from a raw CloudFront URL to a proper, trusted web address.*

## The problem

I had a site running on AWS CloudFront (Amazon's content delivery network), but it was only reachable through the default URL AWS generates automatically, something like `d3kjtgg39op14v.cloudfront.net`. That works fine technically, but it's not something you'd ever want to put on a resume or hand to a client. It doesn't look trustworthy, it's hard to remember, and it doesn't say anything about who owns the site.

For a while I patched over this with a URL shortener (TinyURL), which redirects a short link to the long CloudFront address. That's an okay quick fix, but it's not a real solution. The browser still eventually shows the ugly CloudFront URL, the redirect is one more thing that can break or get rate limited, and it doesn't give visitors any of the trust signals a real domain does, like a matching padlock and certificate in the address bar.

The actual fix was to put a real domain in front of the CloudFront distribution, the same way any production website does it.

## What I did, in plain terms

**1. Bought the domain.**
I registered `kentechsolution.com` through Amazon Route 53 (AWS's domain name service), for about $16/year. Before landing on that name I checked several close variants. The exact name I originally wanted, `kentechsolutions.com` (plural), was already taken, so I compared alternatives on price and how clean they'd look written down, and picked the singular version instead of settling for something like `.biz`, which tends to read as spammy for a site meant to represent professional work.

**2. Got a security certificate.**
Every legitimate website today needs to run over HTTPS. That's what puts the padlock icon in the browser bar and encrypts traffic between the visitor and the site. To get that padlock, you need a TLS/SSL certificate that proves you actually control the domain. I requested one for free through AWS Certificate Manager (ACM), which is AWS's built in tool for handling this. It covers both `kentechsolution.com` and `www.kentechsolution.com` under a single certificate.

**3. Proved I owned the domain.**
Certificates aren't handed out just because you typed a name in a box. AWS needs proof you actually control that domain, and it does this by asking you to create a specific DNS record (basically a small piece of verification text) inside the domain's settings. Since my domain's DNS was already managed in Route 53, AWS was able to create that verification record automatically with one click, and the certificate was issued within a few minutes.

**4. Connected the domain to CloudFront.**
With the certificate ready, I went into the CloudFront distribution's settings and added `kentechsolution.com` as what AWS calls an "alternate domain name." Essentially, this tells CloudFront "also answer to this name, and here's the certificate to use when you do." This is the step that actually links the pretty domain to the real infrastructure serving the site.

**5. Pointed the domain at the site.**
The last piece is DNS routing, the part of the internet's address book that tells a browser typing `kentechsolution.com` where to actually go. I created what's called an alias record in Route 53, which routes any request for the domain straight to the CloudFront distribution. I did this for both the root domain and the `www` version, so both work identically.

## Why it matters

None of this is complicated once you've done it, but it reflects a real, transferable skill: understanding how a domain name, a TLS certificate, and a CDN actually fit together to serve a trusted site, rather than just clicking "buy domain" and hoping it works. It's the same basic pattern used behind almost every production website, just done here for a portfolio scale project.

It also quietly fixed a real limitation. The site no longer depends on a third party redirect service to look presentable, which means one less thing that could break, rate limit, or change pricing later.

## Stack used

`AWS Route 53` (domain registration and DNS), `AWS Certificate Manager` (TLS certificate), `AWS CloudFront` (CDN and hosting)
