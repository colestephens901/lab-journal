---
title: "Building Backup and Recovery Infrastructure with Proxmox Backup Server"
description: "Building centralized VM and container backup infrastructure with Proxmox Backup Server, NAS-backed storage, retention, verification, and recovery workflows."
pubDate: "2026-06-11"
type: "project-writeup"
tags:
  - "Proxmox Backup Server"
  - "Proxmox"
  - "Backups"
  - "Storage"
  - "Linux"
  - "Recovery"
featured: false
draft: false
---

## Overview

As my Proxmox environment grew, backups became something I wanted to treat as dedicated infrastructure rather than an occasional administrative task.

Virtual machines and Linux containers had become responsible for many important homelab services. Rebuilding an operating system is relatively straightforward, but recreating every application's configuration, data, networking, and dependencies would take considerably more time.

I deployed **Proxmox Backup Server** to centralize backups for the virtualization environment and integrate those backups with NAS-backed storage.

The project gave me practical experience with backup scheduling, storage integration, retention, verification, troubleshooting, and recovery planning.

## Goals

The backup environment was designed around several goals:

- Centralize backups for Proxmox workloads
- Protect virtual machines and Linux containers
- Store backups separately from the primary compute hosts
- Integrate backup storage with the NAS environment
- Automate recurring backup jobs
- Understand retention and pruning
- Verify backup integrity
- Develop a realistic recovery workflow
- Avoid treating RAID or snapshots as a complete backup strategy

## Architecture

The basic backup path looks like:

```text
Proxmox VE
    |
    | VM / LXC backup
    v
Proxmox Backup Server
    |
    v
Backup Datastore
    |
    v
NAS-Backed Storage
```

This separates several infrastructure responsibilities:

```text
Compute
  |
  +---- Proxmox VE

Primary Storage
  |
  +---- TrueNAS
  +---- ZFS

Backup
  |
  +---- Proxmox Backup Server
```

The separation is important because primary storage and backup storage serve different purposes.

## Why a Dedicated Backup Server

It is possible to create basic Proxmox backups without deploying PBS, but I wanted to work with a platform designed specifically for Proxmox backup workloads.

Proxmox Backup Server adds capabilities such as:

- incremental backups
- deduplication
- compression
- retention management
- integrity verification
- centralized datastore management
- restore integration with Proxmox VE

This makes the backup environment more than a folder containing periodic archive files.

## Backup Jobs

Proxmox VE can schedule backups to the PBS datastore automatically.

A simplified workflow is:

```text
Scheduled Job
     |
     v
Proxmox VE
     |
     v
Snapshot / Backup
     |
     v
Proxmox Backup Server
     |
     v
Datastore
```

Automating this process reduces the chance that backups depend on remembering to run them manually.

It also creates a predictable history of restore points.

## Incremental Backups

One of the features I wanted to understand better was incremental backup behavior.

Rather than transferring every byte of a virtual machine during every scheduled job, PBS can reuse data that already exists in the datastore.

Conceptually:

```text
First Backup
  |
  +---- Full set of required blocks


Later Backup
  |
  +---- Existing blocks reused
  |
  +---- Changed blocks transferred
```

This helps reduce backup time and storage consumption compared with repeatedly storing complete independent copies.

## Deduplication

PBS also uses deduplication to avoid storing duplicate blocks unnecessarily.

This becomes useful when multiple backups contain large amounts of identical data.

For example, several Linux virtual machines may contain many identical operating-system files.

Instead of treating every backup as completely unrelated data, deduplication allows the datastore to reuse blocks that are already present.

This helped me understand why backup-storage requirements cannot always be estimated simply by adding together the provisioned disk size of every virtual machine.

## NAS-Backed Storage

The PBS datastore is integrated with storage provided by the broader NAS environment.

This required more than simply installing Proxmox Backup Server.

The complete path depends on several layers:

```text
Proxmox Workload
       |
       v
Proxmox VE
       |
       v
Proxmox Backup Server
       |
       v
Storage Mount
       |
       v
NAS
       |
       v
Physical Storage
```

A failure at any of these layers can affect backups.

This made the project another good example of infrastructure dependencies becoming visible through troubleshooting.

## Troubleshooting Storage Mounts

One of the areas I worked through was getting storage mounted and available reliably to the backup environment.

A mount can fail for several reasons, including:

- network connectivity
- DNS
- permissions
- authentication
- export configuration
- incorrect paths
- service startup order

When a datastore is unavailable, the visible symptom may simply be a failed backup job.

The actual cause can exist much deeper in the storage path.

## Troubleshooting Authentication

Backup infrastructure also introduced another set of credentials and permissions.

Proxmox VE needs permission to access the appropriate PBS datastore.

That requires correctly configuring:

- users
- authentication
- datastore permissions
- credentials
- backup targets

A configuration can appear correct at the network layer while still failing because the requesting system does not have the correct authorization.

This reinforced the importance of separating **connectivity** problems from **authentication** problems.

## Retention

Keeping every backup forever would eventually consume all available storage.

PBS therefore requires a retention strategy.

Retention policies can preserve useful recovery points while allowing older backups to be removed automatically.

The exact policy can vary depending on the workload, but the underlying goal is to balance:

- recovery history
- storage capacity
- backup frequency
- importance of the workload

This introduced me to thinking about backup storage as a resource that requires planning rather than simply a destination for unlimited data.

## Pruning

Pruning removes backup snapshots that no longer need to be retained according to policy.

That is different from immediately reclaiming every piece of storage associated with those backups because deduplicated chunks may still be referenced by other snapshots.

Understanding that relationship helped me better understand how a deduplicated backup datastore behaves internally.

## Verification

A backup existing in the interface does not automatically prove that every stored block remains usable.

Verification jobs provide a way to check backup data for integrity.

This is an important distinction:

```text
Backup completed
      !=
Backup proven recoverable
```

Verification provides another layer of confidence, but it still does not replace testing the restore process itself.

## Backup Is Not Recovery

One of the most important lessons from this project is that **backup and recovery are related but separate concepts**.

A successful backup job answers:

> Did the system store the backup?

A recovery test answers:

> Can I actually use it when the original system is gone?

A complete backup strategy therefore needs to consider:

- how backups are created
- where they are stored
- how long they are retained
- whether they are verified
- how a workload would be restored
- what infrastructure must exist before restoration is possible

## Restore Workflow

One advantage of PBS integration with Proxmox VE is that backed-up workloads can be restored through the virtualization platform.

The conceptual recovery path is:

```text
PBS Datastore
      |
      v
Selected Backup
      |
      v
Proxmox VE
      |
      v
Restored VM / LXC
      |
      v
Application Validation
```

The final step matters.

A virtual machine successfully booting does not necessarily mean the application inside it is healthy.

After a restore, I still need to validate things such as:

- networking
- DNS
- storage mounts
- application services
- reverse-proxy access
- monitoring
- dependent services

## RAID Is Not Backup

The storage environment already includes disk redundancy, but that does not eliminate the need for PBS.

RAID can help protect against certain disk failures.

It does not inherently protect against:

- accidental deletion
- a bad configuration change
- application corruption
- administrator error
- ransomware
- loss of the entire system

This is why I treat:

```text
Redundancy
```

and:

```text
Backup
```

as separate parts of the infrastructure design.

## Snapshots Are Not Backup Either

Virtualization and ZFS both provide snapshot functionality, but snapshots are also not equivalent to an independent backup.

Snapshots are extremely useful for short-term rollback and change management.

However, they may still depend on the same underlying storage that contains the live workload.

PBS gives me a separate recovery mechanism.

## Monitoring Backups

A scheduled backup system should not quietly fail for weeks without being noticed.

Backup status is therefore something I want visible alongside the rest of the homelab.

Useful signals include:

- last successful backup
- failed jobs
- datastore capacity
- backup-server availability
- verification status

This allows backup failures to become actionable infrastructure events instead of something discovered only during an emergency.

## Failure Scenarios

Thinking about recovery requires considering what could actually fail.

### Single VM Failure

Restore the affected workload from PBS.

### Application Misconfiguration

Restore an earlier backup or recover the required configuration/data.

### Proxmox Node Failure

Rebuild or replace the compute host and restore workloads from backup.

### Primary Storage Failure

Recover using storage redundancy where appropriate and use backups for data that requires restoration.

### Backup Server Failure

Rebuild the PBS service while ensuring the underlying backup datastore remains protected and accessible.

Thinking in scenarios made the architecture more useful than simply checking whether scheduled jobs showed green.

## What I Learned

Building the backup environment taught me several important infrastructure lessons:

- A backup system should be separate from the workloads it protects
- RAID does not replace backup
- Snapshots do not replace independent backup
- Automated backups reduce reliance on human memory
- Retention policies are part of capacity planning
- Deduplication changes how backup storage is consumed
- Verification provides confidence in stored backup data
- Mount and authentication problems can appear as backup failures
- Recovery planning matters as much as backup scheduling
- A restored VM still requires application-level validation

The biggest change was moving from thinking:

> "I have backups."

to asking:

> "What exactly happens if this system disappears tomorrow?"

That question makes backup design much more practical.

## Current Status

Proxmox Backup Server is now part of the core infrastructure supporting my virtualization environment.

It provides centralized backup workflows for Proxmox workloads and integrates with NAS-backed storage as part of a broader strategy involving primary storage, redundancy, monitoring, and recovery.

As the homelab continues to evolve, I also continue refining retention, verification, monitoring, and restore procedures rather than treating backup configuration as a one-time task.

## Technologies Used

- Proxmox Backup Server
- Proxmox VE
- Linux
- NAS storage
- NFS
- Backup scheduling
- Incremental backups
- Deduplication
- Retention
- Pruning
- Verification
- Disaster recovery