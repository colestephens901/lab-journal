---
title: "Self-Hosting NetBird for Remote Homelab Access"
description: "Deploying a self-hosted NetBird environment to evaluate an open-source alternative for secure remote access to my homelab."
pubDate: 2026-08-21
updatedDate: 2026-08-22
type: project-writeup
tags:
  - NetBird
  - VPN
  - Networking
  - Linux
  - DNS
  - Nginx
featured: true
draft: false
---

## Overview

Remote access is an important part of my homelab because I regularly need to administer systems while away from home.

I had already been using Tailscale successfully, but I wanted to explore a more self-hosted approach and better understand the components involved in building a WireGuard-based remote-access environment.

For this project, I deployed **NetBird** and configured it to provide access to internal homelab networks without routing all client Internet traffic through my home connection.

## Goals

The project had several goals:

- Deploy the NetBird management platform inside the homelab
- Secure the web interface with HTTPS
- Allow remote clients to reach internal infrastructure
- Preserve normal Internet routing on remote devices
- Support internal DNS resolution
- Restrict access through NetBird policies
- Create a dedicated routing peer for homelab networks
- Avoid exposing internal management interfaces directly to the Internet

## Architecture

The deployment uses two primary Linux systems.

The first hosts the NetBird management services and web dashboard.

The second acts as a dedicated routing peer between NetBird clients and selected homelab networks.

The public-facing management interface is placed behind **Nginx Proxy Manager**, which handles HTTPS and forwards traffic to the NetBird services.

The resulting traffic flow is approximately:

```text
Remote Client
     |
     v
NetBird WireGuard Tunnel
     |
     v
Routing Peer
     |
     v
Authorized Homelab Networks
````

The management plane follows a separate path:

```text
Internet
    |
    v
Cloudflare DNS
    |
    v
Nginx Proxy Manager
    |
    v
NetBird Management Services
```

## Reverse Proxy and TLS

The NetBird management interface needed to be reachable securely from outside the network.

I configured a dedicated hostname through Cloudflare DNS and routed it through Nginx Proxy Manager.

Nginx Proxy Manager handles the public TLS certificate and forwards traffic internally to the NetBird deployment.

Because NetBird uses services that rely on gRPC in addition to normal HTTP traffic, the reverse-proxy configuration required additional attention compared with a basic web application.

I validated the public endpoint using `curl` and confirmed that the hostname was presenting a valid Let's Encrypt certificate and communicating over HTTP/2.

## Dedicated Routing Peer

Rather than using the NetBird management server itself as the primary router, I deployed a separate Linux virtual machine to act as the routing peer.

This keeps routing responsibilities separate from the management plane and makes the design easier to understand and troubleshoot.

The routing peer advertises only the internal networks that remote clients actually need to access.

This allows remote devices to reach homelab services without turning the VPN into a full Internet gateway.

## Split Routing

One of the most important design requirements was avoiding unnecessary VPN traffic.

I did not want ordinary Internet browsing, software downloads, or speed tests to travel through the homelab simply because NetBird was connected.

Instead, I configured routes only for the internal networks that needed to be reachable.

The intended behavior is:

```text
Homelab traffic
      |
      +----> NetBird tunnel

Internet traffic
      |
      +----> Normal local Internet connection
```

This provides remote administrative access while preserving the performance of the client's current network connection.

## Internal DNS

Routing packets into the homelab was only part of the problem.

I also wanted remote clients to continue using internal service names rather than remembering addresses.

That required configuring NetBird DNS behavior so internal homelab domains could resolve correctly while ordinary DNS traffic continued to behave normally.

I validated the configuration from both Linux and Android clients and confirmed that internal services could be resolved while connected remotely.

This became especially important because many of the services in the environment are accessed through internal reverse-proxy hostnames rather than direct IP addresses.

## Access Policies

NetBird provides policy-based access controls between peers and networks.

Rather than allowing every connected device unrestricted access, I created policies for my administrative devices and the networks they were allowed to reach.

This gives the remote-access environment another layer of control beyond simply possessing VPN credentials.

The goal is to treat remote access as part of the network security design rather than as a universal bypass around it.

## Problems Encountered

### Routing Peer Remained in a Connecting State

During the initial deployment, the routing peer did not immediately behave as expected.

The dashboard showed connectivity issues even though portions of the configuration appeared correct.

I worked through the problem by validating each layer independently:

* peer connectivity
* routing configuration
* DNS resolution
* reverse-proxy behavior
* policy assignment
* client-side routes

Testing individual components prevented the troubleshooting process from turning into random configuration changes.

### Internal DNS Required Separate Attention

A working VPN tunnel did not automatically mean internal applications were usable.

Clients could potentially reach internal networks while still failing to resolve the hostnames used to access services.

I configured dedicated DNS behavior and validated it using command-line tools such as:

```bash
resolvectl
```

and:

```bash
nslookup
```

This helped confirm whether failures were caused by routing or name resolution.

### HTTPS and gRPC Behind the Reverse Proxy

The management interface also required more than a simple HTTP reverse proxy.

I verified the public endpoint with:

```bash
curl -vkI https://example.domain
```

and checked the certificate subject, issuer, HTTP protocol, and returned status.

This was useful for separating TLS and reverse-proxy problems from NetBird application problems.

## Validation

After configuration, I tested the environment from multiple remote clients.

Validation included:

* connecting from a Linux laptop
* connecting from an Android phone
* resolving internal service hostnames
* reaching services on authorized internal networks
* confirming normal Internet traffic was not being routed through the homelab
* verifying HTTPS on the management interface
* confirming routing and DNS policies applied to the intended devices

Testing from multiple clients was important because VPN behavior can differ between desktop and mobile operating systems.

## What I Learned

The most useful part of this project was seeing how many separate systems are involved in something that initially sounds as simple as "connect to my home network remotely."

A functional remote-access platform depends on several layers working together:

* authentication
* WireGuard tunnels
* routing
* DNS
* firewall policy
* reverse proxying
* TLS
* client configuration

A problem in any one of those layers can make the overall VPN appear broken.

Separating the troubleshooting process into connectivity, routing, DNS, and application-access tests made the deployment much easier to reason about.

## Current Status

The NetBird deployment successfully demonstrated self-hosted remote access to selected homelab networks and provided useful hands-on experience with VPN routing and split DNS.

I continue to evaluate remote-access solutions based on reliability, maintainability, security, and how well they support administrative work from outside the network.

## Technologies Used

* NetBird
* WireGuard
* Debian Linux
* Nginx Proxy Manager
* Cloudflare DNS
* Let's Encrypt
* DNS
* Network routing
* Access-control policies

```
```
