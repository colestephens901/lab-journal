---
title: "Building an Internal Reverse Proxy Platform with Nginx Proxy Manager"
description: "Deploying Nginx Proxy Manager as a centralized reverse proxy for self-hosted applications, then expanding it with internal DNS, HTTPS, and Cloudflare DNS validation."
pubDate: "2026-01-18"
type: "project-writeup"
tags:
  - "Nginx Proxy Manager"
  - "Reverse Proxy"
  - "DNS"
  - "HTTPS"
  - "TLS"
  - "Cloudflare"
featured: false
draft: false
---

## Overview

As the number of self-hosted applications in my homelab increased, accessing everything by IP address and port quickly became difficult to manage.

A single application might be reachable through an address such as:

```text
server-address:8080
```

while another used a completely different host and port.

That works technically, but it becomes inconvenient as the environment grows.

I deployed **Nginx Proxy Manager** to create a centralized reverse-proxy layer for the homelab.

Instead of remembering individual addresses and ports, services could be accessed using consistent hostnames while the reverse proxy handled the connection to the correct backend.

The initial reverse-proxy deployment became operational on January 18, 2026. The platform was later expanded with internal DNS, Let's Encrypt certificates, and Cloudflare DNS validation so HTTPS could become the standard way I accessed internal services.

## Goals

The reverse-proxy platform was designed around several goals:

- Replace raw IP-and-port URLs with memorable hostnames
- Centralize application routing
- Reduce the amount of client-side configuration required
- Provide one place to manage HTTPS certificates
- Integrate with internal DNS
- Support applications running across multiple servers and containers
- Avoid exposing unnecessary application ports directly to users
- Make self-hosted services easier to document and maintain

## Initial Architecture

Before using a reverse proxy, accessing applications looked approximately like this:

```text
Browser
   |
   +----> Server A : Port 8080
   |
   +----> Server B : Port 3000
   |
   +----> Server C : Port 8123
   |
   +----> Server D : Port 9000
```

The user needed to know both the server address and the application port.

After deploying Nginx Proxy Manager, the architecture became:

```text
Browser
   |
   v
DNS
   |
   v
Nginx Proxy Manager
   |
   +----> Application A
   |
   +----> Application B
   |
   +----> Application C
   |
   +----> Application D
```

The client only needs to know the application's hostname.

Nginx Proxy Manager determines which internal service should receive the request.

## Hostname-Based Routing

HTTP and HTTPS requests include the hostname the client is attempting to reach.

The reverse proxy uses that hostname to select a backend.

Conceptually:

```text
service-a.example.com
          |
          v
    Reverse Proxy
          |
          v
    Application A
```

while:

```text
service-b.example.com
          |
          v
    Reverse Proxy
          |
          v
    Application B
```

Both hostnames can resolve to the same reverse-proxy system while still reaching completely different applications.

This allowed me to separate the public-facing service name from the application's actual location.

## Internal DNS

The reverse proxy became much more useful once it was combined with internal DNS.

Instead of clients resolving self-hosted application names to external addresses, internal DNS can direct those hostnames toward the reverse proxy inside the homelab.

The request path becomes:

```text
Internal Client
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

This keeps traffic for internal applications inside the network.

It also means an application can move to a different backend system without requiring every client to learn a new address.

Only the reverse-proxy configuration needs to change.

## Separating Names From Infrastructure

One of the most useful architectural benefits of a reverse proxy is abstraction.

A hostname represents the **service**, not necessarily the physical machine running it.

For example:

```text
monitoring.example.com
```

does not tell the user:

- which virtualization node hosts the application
- whether it runs in a VM or container
- what port it uses
- whether the backend later moves

That information becomes an implementation detail behind the reverse proxy.

This makes infrastructure changes less disruptive to users.

## Adding HTTPS

The first working version of the reverse-proxy platform focused on getting hostname-based routing operational.

HTTPS was added afterward.

Once more applications were being accessed through stable DNS names, I wanted internal services to use normal TLS connections rather than relying on plain HTTP.

That introduced several additional components:

```text
Client
  |
  v
DNS
  |
  v
HTTPS
  |
  v
Nginx Proxy Manager
  |
  v
Backend Application
```

The reverse proxy terminates the TLS connection and then forwards the request to the application.

This provides one centralized location for managing certificates across many services.

## Let's Encrypt

I use **Let's Encrypt** certificates so supported services can present trusted certificates rather than browser warnings from self-signed certificates.

This made the internal environment behave more like production web infrastructure.

Instead of accepting certificate warnings, clients can validate the certificate normally.

That provides experience with concepts such as:

- certificate issuance
- certificate renewal
- hostname validation
- TLS termination
- certificate chains
- reverse-proxy configuration

## Cloudflare DNS Validation

Many of the applications behind the reverse proxy are intended to remain internal and are not directly reachable from the public Internet.

That creates a challenge for normal HTTP-based certificate validation.

Rather than exposing internal applications simply to obtain certificates, I configured certificate issuance using a **Cloudflare DNS challenge**.

The general process is:

```text
Nginx Proxy Manager
        |
        v
Let's Encrypt
        |
        v
Cloudflare DNS Validation
        |
        v
Certificate Issued
```

Let's Encrypt verifies control of the domain through DNS rather than connecting directly to the internal application.

This allows internal-only services to receive trusted certificates without making those services publicly accessible.

## Why DNS Challenge Was Useful

The DNS challenge fits the homelab architecture well because public accessibility and certificate validity become separate concerns.

An application can remain:

```text
Private network only
```

while still using:

```text
https://service.example.com
```

with a trusted certificate.

This is particularly useful for administrative services that I do not want exposed directly to the Internet.

## Reverse Proxy as Shared Infrastructure

Once many applications depend on the same reverse proxy, the proxy itself becomes important infrastructure.

If one application becomes unavailable, the problem may be specific to that service.

If many unrelated applications become unavailable at the same time, a shared component becomes a much stronger suspect.

Possible shared dependencies include:

- DNS
- Nginx Proxy Manager
- the reverse-proxy host
- networking
- storage
- firewall policy

This changes the troubleshooting process.

Instead of diagnosing every failed application separately, I first look for the common layer connecting them.

## Troubleshooting Request Flow

When a proxied application does not work, I test the request path in stages.

### 1. Test DNS

Confirm that the hostname resolves to the expected destination.

Examples include:

```bash
nslookup service.example.com
```

or:

```bash
dig service.example.com
```

### 2. Test the Backend Directly

If possible, bypass the reverse proxy and connect directly to the application.

For example:

```bash
curl http://backend-host:port
```

If direct access fails, the reverse proxy is probably not the root cause.

### 3. Test the Reverse Proxy

If the backend works directly, test the hostname through Nginx Proxy Manager.

```bash
curl -I https://service.example.com
```

### 4. Inspect TLS

When HTTPS is involved, I check whether the expected certificate is actually being served.

A useful test is:

```bash
curl -vkI https://service.example.com
```

This can show information such as:

- certificate subject
- certificate issuer
- negotiated HTTP protocol
- returned HTTP status

### 5. Check Proxy Configuration

If DNS, TLS, and the backend all appear healthy, I review:

- forwarding hostname
- forwarding port
- HTTP versus HTTPS backend scheme
- WebSocket requirements
- advanced proxy configuration
- application-specific headers

This layered approach prevents unnecessary changes to unrelated parts of the environment.

## Applications With Special Requirements

Not every application behaves like a basic HTTP website.

Some services require additional features such as:

- WebSockets
- HTTP/2
- gRPC
- custom headers
- larger upload limits
- longer connection timeouts

This became especially noticeable when proxying more complex infrastructure applications.

A proxy host can appear correctly configured while one application still fails because that application expects protocol behavior beyond normal HTTP forwarding.

Learning to recognize those differences has been an important part of operating the reverse-proxy platform.

## Internal and Public Services

As the environment evolved, not every hostname using the same parent domain remained internally hosted.

Some applications are now hosted on external platforms.

That means internal DNS cannot simply assume that every hostname belongs to the homelab.

For example:

```text
Internal application
        |
        v
Internal DNS
        |
        v
Nginx Proxy Manager
```

while another hostname may need:

```text
Publicly hosted application
        |
        v
Public DNS
        |
        v
External hosting platform
```

This required adding exceptions to internal wildcard DNS behavior so publicly hosted services could resolve normally.

It reinforced the relationship between DNS and reverse-proxy architecture.

## Security Considerations

Using a reverse proxy does not automatically make an application secure.

The proxy simplifies access and TLS management, but security still depends on the rest of the design.

Important considerations include:

- whether the service should be publicly reachable at all
- firewall policy
- authentication
- application security
- software updates
- certificate management
- DNS configuration
- network segmentation

For administrative interfaces, I generally prefer keeping the service private and reaching it through trusted local or VPN access rather than exposing it directly to the Internet.

## Service Migration

Another major advantage of hostname-based access is that backend services can move.

Suppose an application originally runs on:

```text
Host A
```

and is later migrated to:

```text
Host B
```

Users can continue accessing:

```text
service.example.com
```

Only the reverse-proxy target needs to change.

That makes infrastructure upgrades and migrations less disruptive.

## Monitoring

Because the reverse proxy sits in the path to many applications, I also monitor proxied services separately.

This helps distinguish between several failure types.

For example:

```text
Application monitor fails
Reverse proxy monitor succeeds
        |
        v
Likely application/backend problem
```

compared with:

```text
Many application monitors fail
Reverse proxy monitor also fails
        |
        v
Likely shared infrastructure problem
```

Monitoring therefore becomes another troubleshooting signal rather than simply an uptime percentage.

## What I Learned

Building the reverse-proxy platform taught me several important infrastructure concepts:

- DNS names can abstract applications away from physical hosts
- Reverse proxies simplify access to many internal services
- HTTPS certificate management can be centralized
- DNS validation allows private services to receive trusted certificates
- A shared proxy becomes critical infrastructure
- Application failures should be diagnosed across the entire request path
- Not every web application uses the same protocol requirements
- Internal DNS and reverse proxies are tightly connected
- Wildcard DNS needs exceptions when some services are externally hosted
- Trusted HTTPS does not require exposing every application publicly

The project also helped me understand web infrastructure as a chain of dependencies rather than simply typing a URL into a browser.

## Current Status

Nginx Proxy Manager remains the central reverse-proxy platform for many self-hosted services in my homelab.

The platform has evolved from its original HTTP proxy deployment into an infrastructure service combining:

- internal DNS
- hostname-based routing
- HTTPS
- Let's Encrypt
- Cloudflare DNS validation
- application-specific proxy configuration
- monitoring

It provides a consistent access layer for services running across different virtual machines, containers, and physical systems while allowing the backend infrastructure to continue changing independently.

## Technologies Used

- Nginx Proxy Manager
- Nginx
- Reverse proxying
- DNS
- Split DNS
- HTTPS
- TLS
- Let's Encrypt
- Cloudflare DNS
- Linux
- Docker