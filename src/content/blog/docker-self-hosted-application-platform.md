---
title: "Building a Docker-Based Self-Hosted Application Platform"
description: "Building and maintaining a Docker Compose environment for self-hosted applications, infrastructure utilities, monitoring, and internal services."
pubDate: "2026-08-11"
type: "project-writeup"
tags:
  - "Docker"
  - "Docker Compose"
  - "Linux"
  - "Self-Hosting"
  - "Reverse Proxy"
  - "Monitoring"
featured: false
draft: false
---

## Overview

As my homelab expanded, Docker became one of the primary ways I deployed and maintained applications that did not require an entire dedicated virtual machine.

Instead of manually installing every application directly into a Linux operating system, I began using **Docker Compose** to define services, storage, networking, dependencies, and configuration in a repeatable format.

Over time, this grew into a larger self-hosted application platform supporting infrastructure utilities, monitoring tools, media services, password management, search, notification systems, and personal projects.

The useful part of the project has not simply been running containers. It has been learning how to operate a group of interconnected services reliably.

## Goals

The Docker environment was built around several goals:

- Standardize application deployment
- Keep applications isolated from one another
- Make services easier to rebuild and migrate
- Store configuration in repeatable Compose files
- Separate persistent data from disposable containers
- Centralize reverse-proxy access
- Monitor application availability
- Reduce the need for dedicated virtual machines
- Learn practical Linux and container troubleshooting

## Why Docker

Many applications in the lab do not need their own full operating system.

Docker provides a useful middle ground between installing everything directly on one Linux host and creating a separate virtual machine for every service.

A typical deployment looks like:

```text
Linux Host
    |
    v
Docker Engine
    |
    +-------------------+
    |                   |
    v                   v
Container A         Container B
    |                   |
    v                   v
Persistent Data     Persistent Data
```

The container itself can be recreated, while important application data is stored separately.

That distinction makes upgrades, rebuilding, and troubleshooting much easier.

## Docker Compose

I use **Docker Compose** to define multi-container applications and their configuration.

Instead of relying on a collection of manually entered `docker run` commands, a Compose file provides a repeatable description of the deployment.

A simplified application might look like:

```yaml
services:
  application:
    image: example/application:latest
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - ./data:/data
```

Real deployments usually contain additional configuration such as:

- environment variables
- Docker networks
- health checks
- dependencies
- bind mounts
- named volumes
- restart policies

Keeping that information in configuration files makes the deployment easier to understand later.

## Application Categories

The Docker environment has grown to support several different types of workloads.

### Infrastructure Utilities

Some containers provide supporting services for the rest of the environment.

Examples include:

- reverse-proxy services
- internal search
- notification systems
- synchronization tools
- administration interfaces

### Media Services

Containerized applications also provide parts of the media environment.

This allows supporting services to be deployed independently while still sharing the storage and networking they require.

### Monitoring

Monitoring tools are another important part of the platform.

Rather than finding out a service is down only when I try to use it, monitoring systems track service availability and infrastructure health.

### Personal Applications

Docker also gives me a convenient way to deploy applications and projects that I build or experiment with myself.

This allows the homelab to function as both infrastructure and a development environment.

## Persistent Storage

One of the first important Docker concepts I had to understand was that containers should generally be treated as replaceable.

Important application state should not depend on the writable filesystem inside a running container.

Persistent data is therefore stored separately through mechanisms such as:

- bind mounts
- Docker volumes
- network storage

A simplified model looks like:

```text
Container
    |
    +---- Application Runtime
    |
    +---- Configuration Mount
    |
    +---- Persistent Data
```

This allows the container image to be replaced during an upgrade without intentionally replacing the application's data.

## Networking

Docker introduces another network layer into the environment.

A connection to a self-hosted application may pass through several components:

```text
Client
  |
  v
DNS
  |
  v
Reverse Proxy
  |
  v
Linux Host
  |
  v
Docker Network
  |
  v
Application Container
```

That means an application appearing unavailable does not automatically mean the application itself has failed.

The problem could exist in:

- DNS
- routing
- firewall policy
- the reverse proxy
- Docker port publishing
- a Docker network
- container health
- application configuration

This made container troubleshooting another good example of why infrastructure should be diagnosed layer by layer.

## Reverse Proxy Integration

Many self-hosted services are accessed through consistent hostnames rather than remembering individual addresses and port numbers.

A reverse proxy sits in front of those services and forwards requests to the appropriate backend.

The basic path is:

```text
service.example.com
        |
        v
       DNS
        |
        v
 Reverse Proxy
        |
        v
Docker Service
```

This provides a cleaner user experience while also allowing HTTPS certificates and web access to be managed centrally.

The reverse proxy itself is still another infrastructure dependency, so when multiple applications fail at once, it is one of the first shared components worth checking.

## HTTPS

Using hostnames for internal services also allowed me to use normal HTTPS connections instead of accessing everything through raw HTTP addresses and ports.

That introduced experience with:

- DNS records
- certificate issuance
- TLS
- reverse-proxy configuration
- hostname validation

It also made the environment behave more like services I would expect to administer outside a home network.

## Container Administration

I use web-based administration tools where they are useful, but I also work directly with Docker from the command line.

Some of the commands I regularly use while troubleshooting include:

```bash
docker ps
```

```bash
docker compose ps
```

```bash
docker logs <container>
```

```bash
docker inspect <container>
```

and:

```bash
docker compose up -d
```

The web interface may make routine management easier, but the command line is usually more useful when something is actually broken.

## Updating Services

Containerized applications make software updates relatively straightforward, but updates still need to be treated carefully.

A typical update workflow involves:

1. reviewing the application or image changes
2. checking the current deployment
3. confirming persistent data is protected
4. pulling the updated image
5. recreating the container
6. checking logs
7. validating application functionality
8. confirming monitoring returns to normal

The fact that a newer image exists does not automatically mean every service should be blindly updated.

## Monitoring

As the number of applications increased, monitoring became increasingly important.

The environment uses service-health monitoring so I can quickly see whether important applications are available.

Monitoring is useful not only for outages, but also for identifying patterns.

If several unrelated applications become unavailable simultaneously, that suggests checking a shared dependency such as:

- the Docker host
- storage
- DNS
- networking
- the reverse proxy

rather than troubleshooting every application independently.

## Troubleshooting Containers

One of the most useful parts of running a large number of services has been developing a consistent troubleshooting workflow.

When a containerized application is unavailable, I generally work through the stack in order.

### 1. Is the host available?

Verify the Linux system itself is reachable.

### 2. Is Docker running?

Confirm the Docker daemon is healthy.

### 3. Is the container running?

```bash
docker ps
```

### 4. What do the logs show?

```bash
docker logs <container>
```

### 5. Is the application listening internally?

Test the application directly on the Docker host if possible.

### 6. Is the port published correctly?

Confirm the container's internal port and host-side port mapping.

### 7. Does the reverse proxy reach it?

If direct access works but the hostname does not, move outward to the proxy.

### 8. Does DNS resolve correctly?

Only after verifying the application stack do I treat DNS as the likely problem.

This prevents me from changing several unrelated parts of the infrastructure at once.

## Rebuilding Services

One of the biggest advantages of containerizing applications is that I can rebuild a service from its configuration instead of relying on an installation that only exists because of manual steps performed months earlier.

A properly documented service should make it possible to understand:

- which image is used
- which ports are required
- where the persistent data lives
- what environment configuration is necessary
- which networks it uses
- what other services it depends on

This makes Docker Compose files useful as both deployment configuration and infrastructure documentation.

## Service Separation

As the lab grew, I also learned that putting every container onto one enormous Docker host is not automatically the best design.

Different workloads have different requirements.

For example:

- infrastructure services may deserve greater isolation
- media applications can have different storage requirements
- monitoring should ideally remain available when an application host fails
- experimental applications should not unnecessarily affect critical utilities

That has led the Docker environment to evolve alongside the rest of the homelab rather than remaining one monolithic server.

## What I Learned

Operating Docker as an application platform taught me considerably more than how to start containers.

Some of the most important lessons were:

- Containers should generally be replaceable
- Persistent data needs to be designed separately from the container
- Docker Compose makes deployments easier to reproduce
- Container networking adds another troubleshooting layer
- Reverse proxies and DNS become shared infrastructure dependencies
- Logs are often more useful than a management interface when diagnosing failures
- Monitoring becomes increasingly important as the number of services grows
- A service is easier to maintain when its configuration is documented
- Not every application belongs on the same host

Most importantly, Docker changed the way I approach application deployment.

Instead of thinking primarily about installing software onto a server, I now think about the complete service definition: runtime, configuration, storage, networking, dependencies, monitoring, and recovery.

## Current Status

Docker Compose remains a major part of my self-hosted environment.

The specific applications and hosts continue to change, but the underlying deployment model remains useful for infrastructure utilities, application services, monitoring, and personal projects.

As the environment grows, I continue refining how containers are separated, monitored, backed up, upgraded, and documented.

## Technologies Used

- Docker
- Docker Compose
- Linux
- Portainer
- Dockge
- Nginx Proxy Manager
- Reverse proxying
- HTTPS / TLS
- DNS
- Persistent volumes
- Container networking
- Service monitoring