---
author: Munavir Chavody
pubDatetime: 2025-09-16T15:22:00Z
modDatetime: 2025-09-16T16:52:45.934Z
title: "Journey to Bare-Metal Kubernetes from Scratch – Part 1: Building the Rig"
slug: journey-to-bare-metal-part-1-building-the-rig
featured: true
draft: false
tags:
  - journey-to-bare-metal
  - bare-metal
  - kubernetes
  - homelab
description: "I'm starting an exciting journey to build my own Kubernetes cluster on bare metal, from scratch!"
---

## Table of contents

## The Motivation

I’ve always prioritised using managed services at work, mostly to align with organisational needs for reduced operational overhead. It makes complete sense from a business perspective. But from an engineering standpoint, I’ve often felt that relying too heavily on managed services isn’t all that technically challenging (of course, managed services come with their own complexities). Still, they don’t always push you to learn the internals of a technology or to understand how cloud providers actually make these services reliable.

One thing my recent job hunt taught me is that market trends increasingly emphasise on-premises expertise. Out of all the companies where I interviewed, three explicitly considered on-prem management skills a big plus—the emphasis tended to grow with the scale of the organisation.

So, getting hands-on with bare-metal experience won’t just help me uncover my “unknown unknowns”, but it also meets a real demand in the market. To me, that’s a pretty compelling motivation to kick off a side project.

## What are We Doing?

The primary goal is to simulate the bare-metal experience as closely as possible, while still keeping it affordable. It wouldn’t make much sense if the cost of buying hardware could instead get you multiple months of compute in a cloud environment to set up a (non-managed) cluster. The extras here are the feel of bare metal, exposure to virtualization, and of course the thrill of building the PC.

So, the rough plan would look like:

1. Source the parts and build the PC.
2. Setup the virtualisation layer via Proxmox.
3. Launch VMs to act as K8s nodes.
4. Start running applications in K8s.
5. Try to figure out what to do with all the compute 😁🤔

Let’s see where this rabbit hole leads!

## Step 1: Research the Hardware

Let's state the requirement to figure out the bare minumun hardware we want to buy. Requirement is to **run 3-4 VMs with 1-2 cores, 4-6 GiB of memory and 50-100GiB of storage per VM**. Roughly, we would need:

1. CPU with 12 cores.
2. 32GiB of memory.
3. 500GiB of storage (NVMe preferred, we'll add additional SATA SSDs later on for backups and other stuff).

Starting with CPU, 12 physical core is atleast gonna cost ~20-25K INR, which is way too much for the budget I'm aiming for, so we'll look at 12 threads instead of 12 cores. The cheapest I could find with 12 threads is 10th gen i5 (didn't look at AMD 🤭, don't know why!). This is gonna cost around 7-8K INR which is a huge saving! But let's wait and see how things play out.

Figuring out the board was a bit tricky B560M looked like a better option with 2 M2 slots and 3200MHz RAM speed, but was not widely available. So had to fallback to H510M which has a single M2 slot and RAM speed limited to 2900MHz. 

For other components, we'll just grab the cheapest option available from reputable brands. Here's the shopping list I ended up with (with links to where I bought them):

1. CPU: [Intel Core i5-10400F](https://www.flipkart.com/intel-core-i5-10400f-2-9-ghz-upto-4-3-lga-1200-socket-6-cores-12-threads-mb-smart-cache-desktop-processor/p/itmf52924285f8dc) - ₹7,500/-
2. Motherboard: [Asus Prime H510M-E R2.0 M-ATX Motherboard](https://mdcomputers.in/product/asus-prime-h510m-e-r2-0-motherboard) - ₹5,800/-
3. RAM: [Crucial Pro 32GB 3200MHz CL22 DDR4 RAM](https://mdcomputers.in/product/crucial-ram-pro-32gb-ddr4-cp32g4dfra32a) - ₹7,350/-
4. Storage: [Western Digital 500GB Blue SN5000 PCIe 4.0 NVMe M.2 Internal SSD](https://computechstore.in/product/western-digital-500gb-sn5000-ssd/) - ₹3,299/-
5. PSU: [DeepCool PL550D 550 Watt Atx 3.0 80 Plus Bronze Power Supply (R-PL550D-FC0B-IN-V2)](https://www.pcstudio.in/product/deepcool-pl550d-550-watt-atx-3-0-80-plus-bronze-power-supply/) - ₹3,340/-
6. Cabinet: [Ant Esports Si11 (M-ATX) Mini Tower Cabinet](https://computechstore.in/product/ant-esports-si11-cabinet-black/) - ₹1,799/-

Most of the orders were smooth except for the motherboard (B560M was out of stock and had to order H510M afterwards) and delivery mess-up by DTDC (inter-Bengaluru took 10 days! 😬).

We kept the budget under ₹30K, which I think is a very affordable. My initial assumption was *that if half the cost of hardware can get us 6 months of cloud run time for three 2 vCPU and 8 GiB of memory, it doesn't make any sense to purchase the hardware*. Here's the data by Grok for how long we can run for across cloud for 15K rupees.

<div class="overflow-x-auto">

|Platform|Instance Type|Hourly Cost per Instance (USD)|Hourly Cost for 3 Instances (USD)|Hourly Cost for 3 Instances (INR)|Max Runtime (hours)|Max Runtime (days) |
|---|---|---|---|---|---|---|
|AWS|t3.medium|0.0512|0.1536|12.90|1,162|48.4|
|Google Cloud|e2-standard-2|0.067|0.201|16.88|889|37.0|
|Microsoft Azure|Standard_D2s_v3|0.0768|0.2304|19.37|774|32.3|
|Oracle Cloud|VM.Standard.E2.2|0.031|0.093|7.81|1,921|80.0|
|DigitalOcean|Basic Droplet|0.0292|0.0876|7.36|2,039|85.0|
|Linode (Akamai)|Shared 8 GB|0.0292|0.0876|7.36|2,039|85.0|
|Vultr|Cloud Compute 2 vCPU/8 GB|0.06|0.18|15.12|992|41.3|

</div>

P.S.: These numbers are purely for the reference of readers and I haven't cross checked it. I was in it too deep 🤣.

## Step 2: The Assembly

It was a long wait for the parts to arrive. Finally, after around 2 weeks, everything showed up. Time for assembly! It took me a bit to figure out the screws and placements, but after about 3–4 hours, the build was done.

![Assembly in-process](@/assets/images/rig-assembly.jpg)
<div class="text-center text-gray-700">Assembly in-progress</div>

Now, the moment of truth: powering it on. The PSU came with a 16A plug but my socket was 6A, so I had to get a connector first. Switched it on and… BOOM — fans spinning! Honestly, I was expecting a few hiccups, but it went surprisingly smooth.

![Assembly in-process](@/assets/images/rig-final.jpg)
<div class="text-center text-gray-700">Look at that thing of beauty!</div>

## What's Next?

[Next, we'll the setting up Proxmox and launch a sample VM](/blog/posts/journey-to-bare-metal-part-2-virtualisation-layer). We'll also explore Proxmox cluster (first, I'll have to learn what I'm doing 🤣).

From a hardware standpoint, I'm still lacking a UPS and automated shutdowns. But it's fine for now. We'll get those once we get our K8s cluster up and running.