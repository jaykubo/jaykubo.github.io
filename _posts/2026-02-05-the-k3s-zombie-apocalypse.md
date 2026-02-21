---
title: "The K3s Zombie Apocalypse: A Global Tale of Pings and Packets"
date: 2026-02-05 21:00:00 +0900
categories:
  - Infrastructure
  - Kubernetes
tags: [k3s, tailscale, networking, distributed-systems, infrastructure, raspberry-pi, gpu-computing]
layout: post
image:
  path: /assets/img/posts/zombie-apocalypse.png
  alt: Global Kubernetes Cluster Resource
---

In the digital dark corners of my network, a dream was stirring. A dream of a single, mighty Kubernetes brain, humming across continents. From the neon glow of Tokyo to the dusty plains of Austin, from the relentless hum of LA’s data centers to my quiet corner in Torrance – a distributed computing empire. 

My weapon? **K3s**. My enemy? The cruel mistress of networking: **Latency**, and her undead horde of **Zombie Kernel Modules**.

You, dear reader, might glance at a perfectly healthy `kubectl get nodes` output and think, *"Ah, serenity. The gentle hum of distributed compute."* You would be wrong. Behind that placid façade lies a battlefield, strewn with the digital corpses of failed agents and the ghostly whispers of a daemon that just wouldn't stay down.

### Chapter 1: The Gathering Storm – A Tale of Too Many Pings

My initial vision was simple: a cluster. A beautiful, unified cluster. I had my Control Planes (valiant Raspberry Pis scattered across Austin, LA, and Torrance), and then the agents. Oh, the agents! The mighty NVIDIA DGX Spark in Tokyo, two **Framework Desktop (FWD)** GPU behemoths in Austin, a Beelink SER4 beast in LA, and a swarm of Raspberry Pi workers scattered like digital breadcrumbs across the globe. Each stood ready, Tailscale VPN their digital umbilical cord to the mothership.

I unleashed Ansible, my automation cavalry, with a confident `ansible-playbook`. The expectation? A symphony of changed states.

The reality? A cacophony of **"FAILED - RETRYING"**. It was less a symphony, more a death metal concert where half the band kept tripping over their mic cables. Nodes in Tokyo and LA would hit the same wall: *"Service did not take the steps required by its unit configuration."*

```text
Feb 05 12:38:21 hnd-dgx-gpu-01 k3s[24077]: Error: failed to parse kubelet flag: unknown flag: --node-status-update-cache-ttl
Feb 05 12:38:21 hnd-dgx-gpu-01 systemd[1]: k3s-agent.service: Failed with result 'protocol'.
Feb 05 12:38:21 hnd-dgx-gpu-01 systemd[1]: Failed to start k3s-agent.service - Lightweight Kubernetes.
```

### Chapter 2: The Zombie Apocalypse – The Daemon That Wouldn't Die

The real horror show started with the cleanup. I thought I could just "kill" the failing processes and start fresh. I was wrong. This was a digital plague. 

Every time I issued a `kill -9` or a `systemctl stop`, the K3s agent would claw its way back to life seconds later. It was a relentless loop of respawning—haunted by "Zombie Kernel Modules."

The `journalctl` logs revealed the source of the infection: `modprobe: FATAL: Module br_netfilter not found`. My kernel had updated in the night, but the system hadn't rebooted. The running kernel was a "Zombie"—walking around, performing basic tasks, but unable to access its own nervous system (the networking modules). Because the modules on disk didn't match the kernel in memory, K3s couldn't route a single packet, but its systemd unit was programmed to keep trying... forever.

I spent three hours in the trenches performing digital surgery:
* `sudo apt install linux-modules-extra-raspi`
* Targeted reboots to lay the "living dead" kernels to rest.
* Manually purging the CNI interfaces that refused to unbind.



---

### Chapter 3: The MTU Massacre – A Tragic Tale of Fragmentation

But even with the zombies exorcised, the whispers of instability remained. The nodes would join, then occasionally blink out. The connection across the Pacific, while active, felt... fragile. 

Then it hit me: **MTU (Maximum Transmission Unit)**. 

Standard Ethernet MTU is 1500 bytes. Tailscale's WireGuard (adds ~80 bytes of overhead) tunnel reduces this to 1280 bytes. **Flannel VXLAN** adds another ~50 bytes of overhead. Without adjustment, Flannel would try to send 1450-byte packets through a 1280-byte tunnel, causing **fragmentation**.

Over hundreds of milliseconds, across oceans, fragments get lost. Fragmented packets are dropped packets. Dropped packets are "blips."



The solution was brutal but necessary: an **MTU of 1230** for Flannel. By significantly undershooting the fragmentation threshold, I ensured every packet—no matter how many headers we stacked on it—would slide through the trans-Pacific tunnel in one piece. No more digital shrapnel littering the network.

---

### Chapter 4: The Council of Five – Solving the Trans-Pacific Brain Drain

The final hurdle was the "Brain Lag." My cluster's initial architecture was technically a quorum, but geographically lopsided. I started with three Control Plane nodes: **Austin (1)**, **LA (1)**, and **Torrance (1)**. 

The glaring omission? **Tokyo had zero control planes.** Every time a worker in Tokyo needed to report status or pull a new pod spec, it had to "phone home" to Texas across 7,000 miles of undersea fiber. When that trans-Pacific link saw even a millisecond of jitter, the Austin master would panic: *"Tokyo is silent! It must be dead! Evict the pods!"* This was the source of my phantom "blips."



To stop the madness, I promoted two more nodes to the brain trust: **Tokyo (+1)** and an additional **Austin (+1)** node, bringing us to a **5-node distributed etcd quorum**. 
> Quorum math: 5 nodes = need 3 for consensus = survives 2 failures. Can lose an entire region and stay operational.

By placing a Control Plane node directly in the Haneda/Tokyo region, the local workers now speak to a **local master**. The "Council of Five" maintains global consensus over Tailscale, but the day-to-day survival of a node in Japan is no longer at the mercy of a trans-Pacific ping.

---

### The Aftermath: A Stable, Sane Global Brain

Today, my `kubectl get nodes` shows a glorious tableau of **Ready**. We survived the Zombie Module uprising, tamed the beast of MTU fragmentation, and decentralized the hive mind. 

```text
======================================
    KUBERNETES GLOBAL CLUSTER REPORT
======================================
NODES: 18 total (5 control planes, 13 workers)
RESOURCES: 232 cores / 750GB RAM
STORAGE:   0TB total ephemeral (Orchestration Pending)

COMPUTE FLEET:
  Framework Desktops: 5
  DGX Spark Units:    1
  GPUS: 5 AMD AI Max+, 1 NVIDIA GB10

GEOGRAPHIC DISTRIBUTION:
  US-Central (Austin): 8 nodes
  US-West (LA/Torrance): 5 nodes
  Asia-NE (Tokyo):     5 nodes
======================================
```

#### Current Cluster State:

| NAME | STATUS | ROLES | AGE | VERSION | INTERNAL-IP | OS-IMAGE | KERNEL-VERSION |
|:---|:---|:---|:---|:---|:---|:---|:---|
| hnd-ctrl-01 | Ready | control-plane | 12h | v1.34.3 | 100.67.10.6 | Ubuntu 24.04 | 6.7.0-raspi |
| hnd-dgx-gpu-01 | Ready | worker | 11h | v1.34.3 | 100.67.10.70 | Ubuntu 24.04 | 6.7.0-nvidia |
| hnd-fwd-gpu-01 | Ready | worker | 11h | v1.34.3 | 100.67.10.71 | Ubuntu 24.04 | 6.17.0-generic |
| aus-ctrl-01 | Ready | control-plane | 12h | v1.34.3 | 100.67.20.6 | Ubuntu 24.04 | 6.7.0-raspi |
| aus-sec-01 | Ready | control-plane | 12h | v1.34.3 | 100.67.20.66 | Kali Linux¹ | 6.7.0-raspi |
| aus-fwd-gpu-01 | Ready | worker | 12h | v1.34.3 | 100.67.20.70 | Ubuntu 24.04 | 6.17.0-generic |
| aus-fwd-gpu-02 | Ready | worker | 12h | v1.34.3 | 100.67.20.71 | Ubuntu 24.04 | 6.17.0-generic |
| aus-nas-01 | Ready | worker | 12h | v1.34.3 | 100.67.20.27 | Ubuntu 24.04 | 6.7.0-raspi |
| aus-node-02 | Ready | worker | 11h | v1.34.3 | 100.67.20.37 | Ubuntu 24.04 | 6.7.0-raspi |
| aus-node-03 | Ready | worker | 12h | v1.34.3 | 100.67.20.38 | Ubuntu 24.04 | 6.7.0-raspi |
| aus-node-04 | Ready | worker | 12h | v1.34.3 | 100.67.20.39 | Ubuntu 24.04 | 6.7.0-raspi |
| lax-ctrl-01 | Ready | control-plane | 12h | v1.34.3 | 100.67.30.6 | Ubuntu 24.04 | 6.7.0-raspi |
| lax-fwd-gpu-01 | Ready | worker | 11h | v1.34.3 | 100.67.30.70 | Ubuntu 24.04 | 6.17.0-generic |
| lax-ser4-gpu-01 | Ready | worker | 11h | v1.34.3 | 100.67.30.71 | Ubuntu 24.04 | 6.7.0-generic |
| lax-node-01 | Ready | worker | 11h | v1.34.3 | 100.67.30.31 | Ubuntu 24.04 | 6.7.0-raspi |
| lax-node-02 | Ready | worker | 11h | v1.34.3 | 100.67.30.32 | Ubuntu 24.04 | 6.7.0-raspi |
| toa-ctrl-01 | Ready | control-plane | 12h | v1.34.3 | 100.67.40.6 | Ubuntu 24.04 | 6.7.0-raspi |
| toa-fwd-gpu-01 | Ready | worker | 11h | v1.34.3 | 100.67.40.70 | Ubuntu 24.04 | 6.17.0-generic |

> **Note on Network Topology:** Internal IPs have been remapped to the `100.67.x.y` range. This is a deliberate nod to the "67" trend.
>
> **¹The Kali "Identity Crisis":** You might notice `aus-sec-01` identifies as Kali Linux despite a vanilla Ubuntu 24.04 base install. This is a common "OS hijacking" side effect when you add the Kali Rolling repositories (for security tools) and run a `dist-upgrade`. It's essentially Ubuntu wearing a Kali leather jacket now.

We survived the Zombie Module uprising and tamed the beast of MTU fragmentation. There were no lost lives, thankfully, but plenty of lost sleep. So, the next time you see a Kubernetes cluster running smoothly across oceans, remember the blood, sweat, and MTU settings that went into that serenity.

---

### Appendix: The Global Compute Fleet (Actual Specs)

For the hardware enthusiasts, here is the verified breakdown of the silicon powering this distributed brain. No "standard" builds here—this is a mix of cutting-edge AI silicon and battle-hardened ARM nodes.

| Node Name | Hardware / Host | CPU Architecture | GPU / Accelerator | RAM |
| :--- | :--- | :--- | :--- | :--- |
| **hnd-dgx-gpu-01** | NVIDIA DGX Spark (A.7) | 20-Core (Cortex-X925/A725) | NVIDIA GB10 | 128 GB |
| **hnd-fwd-gpu-01** | Framework (AMD Ryzen AI Max+) | 32-Core Ryzen AI Max+ 395 | Radeon 8060S | 128 GB |
| **lax-ser4-gpu-01**| Beelink SER4 | 8-Core Ryzen 7 4700U | Radeon Vega | 16 GB |
| **toa-ctrl-01** | Raspberry Pi 5 | 4-Core BCM2712 | Broadcom VC7 | 8 GB |
| **hnd-ctrl-01** | Raspberry Pi 4 | 4-Core BCM2711 | Broadcom VC5 | 8 GB |


### The Final Tally
Our global resource pool now sits at a staggering:
* **18 Total Nodes**
* **232 Logic Cores**
* **~750GB Distributed RAM**
* **Storage Tiers:**
  - `pd-premium` (NVMe): 3x geo-replicated for etcd/databases
  - `pd-standard` (SSD): 2x replicated for application data
  - `pd-archive` (NAS): Single-copy for logs, backups, object storage