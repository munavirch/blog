---
author: Munavir Chavody
pubDatetime: 2025-09-21T15:22:00Z
modDatetime: 2025-09-21T16:52:45.934Z
title: "Journey to Bare-Metal Kubernetes from Scratch – Part 2: The Virtualisation Layer"
slug: journey-to-bare-metal-part-2-virtualisation-layer
featured: true
draft: false
tags:
  - journey-to-bare-metal
  - bare-metal
  - kubernetes
  - homelab
  - proxmox
description: "Second part of my 'bare-metal Kubernetes journey': setting up the virtualisation layer."
---

## Table of contents

## Introduction

> Recap: In [part 1](/blog/posts/journey-to-bare-metal-part-1-building-the-rig), I've discussed my experience sourcing and assembling the hardware.

Now that the rig is up and running, it’s time for the next step: setting up the virtualization layer to run our VMs. The choice of hypervisor was pretty straightforward—since I’ll be running Linux VMs, KVM was the obvious pick. To make things easier to automate and to integrate seamlessly with Kubernetes (more on this in upcoming posts 👀), I went with Proxmox as the base OS.

### What is Proxmox?

Proxmox is an enterprise-grade solution to manage virtualisation needs. It works with 2 virtualisation stacks: KVM (full VMs) and LXC (kernel-level isolation). Proxmox also allows you to virtualise storage and networking as well, which is really useful to simulate a cloud-like setup. Now, let's get started with essential Proxmox terminologies so that you don't feel lost.

1. **PVE**: **P**roxmox **V**irtual **E**nvironment. Proxmox is actually a suite of solutions (covering virtualization, backup, clustering, etc.), and PVE is the piece focused on virtualization.
2. **Nodes**: Each physical server is called a node. Multiple nodes can be grouped into a **cluster**. In our case, we’ll stick to single-node mode.
3. **Virtual Machines**: Full VMs with their own kernel — basically like installing Windows or Ubuntu on a fresh PC.
4. **Virtualised Hardware Options**: Virtualisation option to spin up storage disk, network interfaces etc.
5. **Bridged Networking**: The most common way to connect VMs to your LAN. With a Linux bridge (e.g., `vmbr0`), VMs get an IP from your router as if they were physical devices on the network.
6. **Cloud-init**: Proxmox supports cloud-init, which means you can pass metadata like hostname, username, SSH keys, and even bootstrap scripts (user-data) into your VM at first boot.
7. **Templates & Clones**: You can turn a VM into a *template* (similar to an AMI on AWS), then quickly *clone* new VMs from it. Great for spinning up consistent environments.
8. **Snapshots and Backup**: Proxmox supports both point-in-time snapshots (useful for quick rollbacks) and full backups of VMs/containers (more on this later).

## Installing Proxmox

For installation, we need a bootable media to load the installer which will copy the OS files to the SSD we have installed and then mark it bootable (same method the PC identified installation media). We will be booting via a USB drive. Here are the steps.

1. Download [Proxmox VE ISO](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso).
2. Connect the USB, write the ISO using tools like:
    - [Rufus for Windows](https://rufus.ie/en/)
    - `dd` for Linux machines and MacOS.
3. Boot the media, proceed with the installation type of your preference and complete the installation.
4. Once installation is completed and server is up and running, check if service `pveproxy` (management UI) is running. UI will accessible via `https://<ip>:8006/` by default.

## Accessing Proxmox

### Management UI

Once the install finished and my node booted up, the first thing I did was hit `https://<ip>:8006/`. That’s the Proxmox dashboard, where you can see your entire cluster in one place. Honestly, this is where I spent most of my time in the beginning — clicking around, creating VMs, and breaking things.

![Proxmox UI](@/assets/images/proxmox-ui.png)

### CLI

The UI is great, but sooner or later you’ll want to script things. That’s where the CLI comes in. For example, I use qm whenever I want to spin up a new VM quickly, and vzdump to back things up before I do something stupid (which happens often). Proxmox provides a bunch of CLI options grouped by functionality. Below is a high level overview:

- `pvesh`: Proxmox shell wrapper for REST API.
- `pvecm`: Cluster manager: manages cluster, join/remove nodes, check quorum etc.
- `pvenode`: Manages individual node related actions like manage SSL certs, services etc.
- `qm`: QEMU VM manager. Create, configure and manage KVM virtual machines.
- `pct`: Proxmox Container Toolkit. Utility to manage LXC containers.
- `pvesm`: Storage manager. Manage storage pools, ISO template etc.
- `vzdump`: Backup utility.

There are also additional tools available like benchmarking, usage stats collector etc.

### REST API

Eventually, I wanted my cluster to feel more cloud-like, where I could script VM creation instead of clicking buttons. That’s where the REST API came in. It’s the same API the UI talks to, so anything I click on in the UI, I can eventually turn into a script. Later in this series, you’ll see how I wire this API into my Kubernetes setup to make Proxmox feel even more like a cloud provider.

## Outro

Proxmox Virtual Environment is not just a single binary. It's a collection of serivces and tools working together.

**Up Next**: We will explore more on VMs in Proxmox including templates clones and setup a basic starting point for launching our K8s nodes.