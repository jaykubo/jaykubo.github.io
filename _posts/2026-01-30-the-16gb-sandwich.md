---
title: "The 16GB Panini: Sealing a Subnet with Airgap-Tight Security"
date: 2026-01-30 20:00:00 +0900
categories: [Networking, Security]
tags: [technitium, raspberry-pi, dns, infrastructure, opsec]
layout: post
image:
  path: /assets/img/posts/network-sandwich.png
  alt: High Availability DNS Cluster
---

## The Symptom: A Forked Reality
It started with a subtle shift in network gravity. While I was in the process of expanding our clean `/24` infrastructure into a more robust `/23` address space to accommodate growth, a legacy contractor—operating on a 'Trust-by-Default' mindset—bridged a Cisco router into the wire. Within the hour, devices were being 'nudged' off the primary APs and into a rogue `192.168.x.x` NAT cage.

The network wasn't "down," but it was no longer mine. I was looking at a Man-in-the-Middle (MITM)–style setup where RF quirks and legacy wireless behavior nudged clients away from my clean APs and onto a Cisco-controlled pipe.

## The Response: The 16GB Panini
I didn't argue. I didn't unplug his hardware. I simply built a better reality. I deployed a High-Availability DNS Cluster as the ultimate Upstream Authority—essentially a **16GB Panini** that pressed the network layers together so tightly that unauthorized traffic had nowhere to leak.

* **Primary Node:** Raspberry Pi 5 (8GB) – The Logic Engine.
* **Secondary Node:** Raspberry Pi 4 (8GB) – The Failover Muscle.
* **The Shield:** Technitium DNS running DNS-over-TLS (DoT).

By positioning this cluster closer to the gateway, I sealed the bridge. The Cisco hardware is still there, but every bit of data it tries to resolve now passes through my filtered, encrypted, and "heat-pressed" authority.  

*All changes were confined to infrastructure I was explicitly responsible for; no third-party systems were modified or disabled.*

## The Data: 24 Hours in the Trenches
The logs from the first day confirm exactly why this was necessary. While the users enjoy a seamless connection, the "Panini" has been surgically removing the noise.

### 1. The Telemetry Blackhole (Primary Node)
The Primary Pi 5 has become a graveyard for invasive metadata. In just 24 hours, it silenced thousands of "phone home" attempts that the legacy gear was happy to ignore:

| Target Domain | Hits (Blocked) | Narrative |
| :--- | :--- | :--- |
| `self.events.data.microsoft.com` | **19,033** | OS-level telemetry effectively blackholed. |
| `teams.events.data.microsoft.com` | **4,679** | Constant data-slurping from background apps. |
| `analytics.apis.mcafee.com` | **636** | The "Security" software's own telemetry silenced. |
| `safebrowsing.charter-prod.hosted.cujo.io` | **164** | ISP-level (Spectrum/Cujo) metadata tracking stopped. |

### 2. The Self-Healing Heartbeat (Secondary Node)
The setup isn't just a filter; it’s a resilient cluster. After a brief SSL "identity crisis" (security so tight the nodes almost rejected each other), the cluster synced overnight. 

| Top Domain (Secondary) | Hits | Status |
| :--- | :--- | :--- |
| `api.internal.business.dev` | 541 | Operational (The Business) |
| `cluster-catalog.dns-cluster.node.lan` | 136 | **Synchronized** (The Cluster) |

## The Verdict: Authority via Performance
The beauty of the "Panini" is its invisibility. The contractor sees green lights on his Cisco dashboard. The users see sub-1ms DNS response times. The AV is still running, and the internet is "fast."

But the reality has shifted. I’ve effectively neutralized the invasive aspects of the legacy hardware by reclaiming Layer 7 authority. I didn’t have to win a shouting match; I just provided a version of the internet that was faster, cleaner, and more authoritative than the competing authority.

---
*Postscript: Most infrastructure conflicts aren’t won by pulling cables. They are won by being more correct than any competing system. The 16GB Panini is now the Source of Truth.*

*Also, yes, I’m Batman—the hero this network didn’t ask for, but the one who actually knows how to configure a `/23` bridge.*