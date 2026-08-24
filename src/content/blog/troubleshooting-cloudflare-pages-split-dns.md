---
title: "Troubleshooting Split DNS with Cloudflare Pages"
description: "Diagnosing an SSL failure that only occurred on my home network and VPN after moving a public site to Cloudflare Pages."
pubDate: "2026-08-24"
type: "lab-note"
tags:
  - "DNS"
  - "Pi-hole"
  - "Cloudflare"
  - "Nginx Proxy Manager"
  - "TLS"
  - "Troubleshooting"
featured: false
draft: false
---

## The Problem

I recently deployed a public technical blog using **Astro**, **GitHub**, and **Cloudflare Pages**.

The public deployment itself worked correctly, and the custom domain was active through Cloudflare.

However, I noticed an unusual problem:

- The site worked normally over cellular data
- The site failed from my home network
- The site also failed while connected through my VPN

The browser returned:

```text
ERR_SSL_UNRECOGNIZED_NAME_ALERT
```

Because the same hostname worked from cellular, this immediately suggested that the Cloudflare Pages deployment itself was probably healthy.

The difference had to be somewhere in the network path used by local and VPN clients.

## Initial Hypothesis

My homelab uses **split DNS** for internally hosted applications.

Many internal service hostnames resolve to an internal Nginx Proxy Manager instance rather than following public DNS.

That allows services to use normal hostnames and HTTPS while keeping the traffic inside the homelab.

The simplified internal path is:

```text
Internal Client
      |
      v
Pi-hole
      |
      v
Internal Reverse Proxy
      |
      v
Self-Hosted Application
```

The new blog was different.

It was intentionally hosted on **Cloudflare Pages**, not inside the homelab.

Its correct path should be:

```text
Client
  |
  v
Public DNS
  |
  v
Cloudflare
  |
  v
Cloudflare Pages
```

I suspected my internal DNS configuration was still treating the new hostname as though it belonged to the internal reverse proxy.

## Comparing Internal and Public DNS

I started by checking the hostname using the DNS server supplied by my home network:

```powershell
nslookup blog.coleshomelab.com
```

The answer contained Cloudflare IPv6 addresses, but it also contained a **private IPv4 address belonging to my internal reverse proxy**.

I then compared that with Cloudflare's public resolver:

```powershell
nslookup blog.coleshomelab.com 1.1.1.1
```

The public result returned only Cloudflare addresses.

That was the important difference.

Conceptually:

```text
Internal DNS
blog.coleshomelab.com
        |
        +---- Cloudflare addresses
        |
        +---- Internal reverse-proxy address  <-- Problem
```

while public DNS returned:

```text
Public DNS
blog.coleshomelab.com
        |
        +---- Cloudflare addresses only
```

This explained why the behavior changed depending on which network I was using.

## Why the SSL Error Occurred

The browser requested:

```text
https://blog.coleshomelab.com
```

but internal DNS was capable of sending the connection to **Nginx Proxy Manager** instead of Cloudflare Pages.

Nginx Proxy Manager was not hosting that site.

The client therefore reached the wrong TLS endpoint.

The request path effectively became:

```text
Browser
   |
   v
Internal DNS
   |
   v
Nginx Proxy Manager
   |
   X
No matching site / certificate
```

which resulted in:

```text
ERR_SSL_UNRECOGNIZED_NAME_ALERT
```

The certificate error was therefore only the visible symptom.

The actual problem was DNS.

## Finding the Wildcard Rule

The next step was checking the Pi-hole configuration.

I inspected its custom dnsmasq settings:

```bash
sudo pihole-FTL --config misc.dnsmasq_lines
```

The resolver contained a wildcard rule equivalent to:

```text
address=/coleshomelab.com/<internal-reverse-proxy>
```

This was intentional.

It allows arbitrary internal application hostnames under the homelab domain to resolve to the reverse proxy without manually creating a DNS record for every service.

The problem was that the wildcard also matched:

```text
blog.coleshomelab.com
```

even though that hostname now belonged to an externally hosted service.

## The Fix

Rather than removing the wildcard and breaking the internal services that depended on it, I added a **more-specific DNS exception** for the blog hostname.

Conceptually:

```text
*.coleshomelab.com
        |
        v
Internal Reverse Proxy


blog.coleshomelab.com
        |
        v
Normal Upstream DNS
        |
        v
Cloudflare Pages
```

A specific hostname rule takes precedence over the broader wildcard behavior.

This allowed the existing split-DNS architecture to remain intact while letting the externally hosted blog resolve publicly.

## Redundant DNS Complication

My environment uses more than one Pi-hole resolver.

That meant fixing the rule on only one DNS server would not be enough.

If clients could query either resolver, the result could become intermittent:

```text
Query
  |
  +----> Pi-hole A ----> Correct public answer
  |
  +----> Pi-hole B ----> Internal reverse proxy
```

Depending on which resolver answered, the blog might work or fail.

I therefore applied the same exception to both Pi-hole systems.

This is an important consideration with redundant infrastructure:

> Redundancy only behaves predictably when the redundant systems have consistent configuration.

## Validation

After applying the exception, I restarted Pi-hole's DNS service and tested the hostname directly.

For the public blog hostname, I wanted:

```text
blog.coleshomelab.com
        |
        v
Public Cloudflare addresses
```

and specifically did **not** want the internal reverse-proxy address.

I then checked an internally hosted hostname to confirm that the wildcard still worked:

```text
Internal application hostname
        |
        v
Internal reverse proxy
```

This confirmed that the fix was limited to the intended hostname.

Finally, I cleared the DNS cache on my Windows client:

```powershell
ipconfig /flushdns
```

and tested the site again.

It now behaved consistently across:

- the home network
- VPN access
- cellular data

## Troubleshooting Process

The useful part of this problem was the order in which I approached it.

### 1. Compare environments

The site worked on cellular but failed at home.

That suggested the application itself was probably healthy.

### 2. Identify what changes between environments

The hostname and browser were the same.

DNS and routing were different.

### 3. Compare DNS answers

```powershell
nslookup blog.coleshomelab.com
```

versus:

```powershell
nslookup blog.coleshomelab.com 1.1.1.1
```

immediately revealed a difference.

### 4. Trace the unexpected answer

The private address led directly back to the internal reverse proxy.

### 5. Find the rule responsible

The Pi-hole wildcard explained why the hostname was being intercepted.

### 6. Make the smallest possible change

Rather than removing a working wildcard configuration, I added an exception for one hostname.

### 7. Validate both the exception and the original behavior

The public hostname needed to resolve publicly while existing internal applications still needed to resolve privately.

## What I Learned

This issue reinforced several useful troubleshooting lessons:

- A TLS error does not necessarily mean TLS is the root cause
- Comparing behavior across networks can quickly narrow the search area
- Internal and public DNS answers should be compared when split DNS is involved
- Wildcard DNS rules are convenient but can affect more hostnames than intended
- Publicly hosted services may need exceptions from internal DNS overrides
- Redundant DNS servers need consistent configuration
- Fixes should preserve working infrastructure whenever possible
- Testing the original behavior after a change is just as important as testing the fix

The most important observation was simple:

> The site worked from cellular.

That single detail made it much less likely that Cloudflare Pages or the application itself was broken and pushed the troubleshooting process toward the local network.

## Current Status

The blog now resolves through Cloudflare Pages regardless of whether the client is on the home network, connected remotely through VPN, or using a normal public connection.

Internally hosted services continue to use the existing split-DNS wildcard and reverse-proxy architecture.

The final behavior is:

```text
Self-Hosted Applications
        |
        v
Internal DNS
        |
        v
Nginx Proxy Manager


Public Blog
        |
        v
Public DNS
        |
        v
Cloudflare Pages
```

## Technologies Involved

- Pi-hole
- dnsmasq / FTL
- Split DNS
- Cloudflare DNS
- Cloudflare Pages
- Nginx Proxy Manager
- TLS
- PowerShell
- Linux
- VPN DNS