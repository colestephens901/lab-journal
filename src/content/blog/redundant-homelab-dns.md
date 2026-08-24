---
title: "Building Redundant DNS Infrastructure for My Homelab"
description: "Building a redundant internal DNS environment using Pi-hole, AdGuard Home, internal records, VPN DNS, and later recursive resolution with Unbound."
pubDate: "2026-01-17"
type: "project-writeup"
tags:
  - "Pi-hole"
  - "AdGuard Home"
  - "Unbound"
  - "DNS"
  - "Networking"
  - "VPN"
featured: false
draft: false
---

## Overview

DNS became an increasingly important part of my homelab as the number of servers, applications, network segments, and remote-access services grew.

What originally began as DNS-based ad blocking evolved into infrastructure responsible for much more:

- internal hostname resolution
- DNS filtering
- redundancy
- VPN client resolution
- self-hosted application access
- split DNS
- recursive DNS resolution

I eventually built a redundant DNS environment using **Pi-hole**, **AdGuard Home**, and later **Unbound**.

The project helped me understand that DNS is not simply a background Internet service. In a self-hosted environment, it quickly becomes a core infrastructure dependency.

## Goals

The DNS environment was designed around several goals:

- Provide DNS filtering for local clients
- Avoid depending on a single DNS server
- Resolve internal application hostnames
- Support clients across multiple network segments
- Provide DNS to remote VPN clients
- Keep internal service traffic inside the homelab
- Reduce dependence on third-party recursive resolvers
- Make the DNS architecture resilient to maintenance or failure

## Early Architecture

The first major version of the environment used multiple DNS filtering systems.

A simplified layout looked like:

```text
                         Clients
                            |
               +------------+------------+
               |                         |
               v                         v
          Primary DNS                Secondary DNS
         AdGuard Home                   Pi-hole
               |                         |
               +------------+------------+
                            |
                            v
                       DNS Resolution
```

An additional Pi-hole instance on a Raspberry Pi was also used to support remote VPN clients.

This meant DNS was no longer tied to one machine.

If one resolver was unavailable for maintenance or because of a host failure, clients still had another DNS path available.

## Why Redundant DNS Matters

DNS is easy to overlook until it stops working.

An application may be completely healthy and reachable by IP address while appearing offline to users simply because its hostname cannot be resolved.

As more of my infrastructure began using DNS names, the impact of a resolver outage became increasingly significant.

Examples include:

- internal dashboards
- monitoring platforms
- password management
- Home Assistant
- reverse-proxied applications
- remote-access services

That made DNS redundancy worthwhile even in a home environment.

## Internal DNS

I prefer accessing services through hostnames instead of remembering IP addresses and port numbers.

For example, the user-facing path can be:

```text
service.example.com
        |
        v
Internal DNS
        |
        v
Reverse Proxy
        |
        v
Application
```

instead of:

```text
IP-address:port
```

This makes services easier to use while also allowing HTTPS and reverse-proxy routing to work normally.

## Split DNS

Some hostnames need to resolve differently depending on where the client is located.

For internally hosted applications, local DNS can return the private address of the internal reverse proxy.

This keeps traffic inside the network rather than unnecessarily sending it toward the public Internet.

The basic behavior looks like:

```text
Internal Client
      |
      v
Internal DNS
      |
      v
Private Reverse Proxy
      |
      v
Self-Hosted Service
```

Publicly hosted services are different.

Those hostnames need to escape the internal override and resolve through public DNS instead.

This became important as I began hosting services outside the physical homelab as well.

## Wildcard DNS

A wildcard DNS rule can be useful when many internal services share the same domain.

Instead of manually creating a record for every possible hostname, a wildcard can direct matching names toward the internal reverse proxy.

Conceptually:

```text
*.example.com
      |
      v
Reverse Proxy
```

This works well for self-hosted services, but it also introduces an important limitation:

> A wildcard rule can accidentally capture a hostname that is supposed to resolve publicly.

I encountered exactly this when a publicly hosted site worked correctly on cellular data but failed from my home network and VPN.

Public DNS returned the correct Cloudflare addresses, while internal DNS also returned the private reverse-proxy address.

The browser was therefore reaching the wrong server and producing a TLS hostname error.

The solution was to create a more-specific DNS exception for the publicly hosted hostname so it would use normal upstream resolution instead of the internal wildcard.

That troubleshooting reinforced how powerful—and potentially broad—wildcard DNS rules can be.

## VPN DNS

Remote access introduced another DNS requirement.

A VPN connection may successfully route packets into the homelab while internal applications still fail because their hostnames cannot be resolved.

For remote administration to behave naturally, VPN clients also need access to the appropriate DNS infrastructure.

That means remote access depends on more than the VPN tunnel itself:

```text
Remote Client
      |
      v
VPN Tunnel
      |
      +---- Routing
      |
      +---- Firewall Policy
      |
      +---- DNS
      |
      v
Internal Service
```

I configured DNS services so authorized remote clients could resolve the same internal service names used while connected locally.

## Adding Unbound

As the DNS environment evolved, I also began using **Unbound** for recursive DNS resolution.

Instead of forwarding every request to a public resolver, Unbound can perform recursive resolution itself by following the DNS hierarchy.

Conceptually:

```text
Client
   |
   v
Pi-hole
   |
   v
Unbound
   |
   +---- Root DNS
   |
   +---- TLD DNS
   |
   +---- Authoritative DNS
   |
   v
Answer
```

Pi-hole can continue providing filtering and local DNS functionality while Unbound handles recursive resolution.

This separates two responsibilities:

```text
Pi-hole
  |
  +---- Filtering
  +---- Local DNS
  +---- Client-facing resolver

Unbound
  |
  +---- Recursive resolution
```

## Troubleshooting Unbound

Adding another DNS layer also created another place where resolution could fail.

A useful troubleshooting workflow is to test each resolver directly rather than only testing from a client.

For example:

```bash
dig example.com @127.0.0.1 -p 5335
```

can test an Unbound instance directly.

Then:

```bash
dig example.com @<dns-server>
```

can test the client-facing resolver.

This helps distinguish between:

- client configuration
- Pi-hole or AdGuard behavior
- forwarding
- Unbound
- network connectivity
- upstream DNS resolution

One issue I encountered involved Unbound listening correctly on its configured port while DNS requests still timed out.

Checking the listening socket proved that the daemon was running, but it did not prove that the complete request path was working.

That reinforced another useful troubleshooting rule:

> A running service is not the same thing as a functioning service.

## Multiple Resolvers

Operating more than one DNS server introduces redundancy, but it also means configuration needs to remain consistent.

If one resolver knows about an internal hostname or split-DNS exception and another does not, client behavior can appear intermittent.

For example:

```text
Query 1
   |
   v
DNS Server A
   |
   v
Correct answer


Query 2
   |
   v
DNS Server B
   |
   v
Incorrect answer
```

From the user's perspective, the application simply seems to work sometimes and fail other times.

This means DNS changes often need to be applied across every resolver clients may use.

## Troubleshooting DNS

When something involving a hostname fails, I now test DNS separately from the application.

Useful commands include:

```bash
nslookup hostname
```

```bash
dig hostname
```

and on Windows:

```powershell
Resolve-DnsName hostname
```

I also compare internal resolution with a known public resolver when split DNS may be involved.

For example:

```powershell
nslookup hostname
```

compared with:

```powershell
nslookup hostname 1.1.1.1
```

If the answers differ, that provides immediate evidence that the problem is related to internal DNS behavior rather than the application itself.

## What I Learned

Building and maintaining the DNS environment taught me several important lessons:

- DNS is core infrastructure, not just an Internet convenience
- Redundant resolvers reduce the impact of maintenance and failures
- Internal DNS makes self-hosted services significantly easier to use
- VPN connectivity is incomplete without working DNS
- Split DNS is useful but needs carefully designed exceptions
- Wildcard records can have unintended consequences
- Multiple DNS servers need consistent configuration
- A listening service does not guarantee successful resolution
- Testing each DNS layer independently makes troubleshooting much easier
- Recursive DNS helped me better understand how public name resolution actually works

Perhaps the biggest lesson was how often an apparent application or networking problem is ultimately a name-resolution problem.

## Current Status

DNS remains a core part of my homelab architecture.

The environment has evolved beyond the original redundant filtering setup to include internal service records, VPN resolution, split DNS, wildcard behavior, and recursive resolution.

Because so many other systems depend on it, DNS is now treated as infrastructure that needs redundancy, monitoring, documentation, and deliberate troubleshooting.

## Technologies Used

- Pi-hole
- AdGuard Home
- Unbound
- DNS
- Recursive DNS
- Split DNS
- Internal DNS
- VPN DNS
- Linux
- dnsmasq / FTL