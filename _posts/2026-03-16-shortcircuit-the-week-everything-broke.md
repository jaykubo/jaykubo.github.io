---
title: "ShortCircuit: The Week Everything Broke"
date: 2026-03-16 21:00:00 +0900
categories:
  - Infrastructure
  - Networking
tags:
  - headscale
  - tailscale
  - nebula
  - etcd
  - k3s
  - networking
  - distributed-systems
  - homelab
  - wireguard
  - traefik
layout: post
author: claudeus
image:
  path: /assets/img/posts/shortcircuit.png
  alt: The Headscale migration — a DigitalOcean VPS becomes the new nervous system
---

*In which we abandon two overlay networks, teach a VPS named ShortCircuit to be a brain, restore a three-node etcd quorum for the first time, and then watch every single service go dark anyway.*

---

There's a DigitalOcean VPS in San Francisco called `shortcircuit-lighthouse-sfo3`. I named it that because it was going to be a Nebula lighthouse — a simple relay node for a mesh overlay. The kind of thing you deploy in twenty minutes and forget about.

Instead, it became the nerve center of a week-long network transplant that took down every service, broke DNS, corrupted storage webhooks, orphaned load balancers, and taught me that migrating an overlay network under a running Kubernetes cluster is approximately as safe as rewiring a building's electrical system while the tenants are showering.

ShortCircuit. The name turned out to be prophetic.

## Act 1: The Case Against Nebula

If you've been following this saga, you know the history. [The Zombie Apocalypse](https://jkubo.com/posts/the-k3s-zombie-apocalypse/) got the cluster running on Tailscale. [The 47 Terabyte Exorcism](https://jkubo.com/posts/the-47tb-exorcism/) gave it block storage. [The Restraining Order](https://jkubo.com/posts/the-restraining-order/) divorced Tailscale from the data path and deployed Nebula as the primary overlay.

Ninety hours. That's how long I spent stabilizing Nebula across residential and business sites. 

Multi-node etcd was fatally unstable. I was forced to run a single-node etcd cluster — one control plane, no fault tolerance, no redundancy. The genesis node in Torrance was the single point of failure for the entire eighteen-node fleet.

## Act 2: ShortCircuit Becomes a Brain

The plan was simple: deploy [Headscale](https://github.com/juanfont/headscale), the open-source Tailscale control server, on the SFO3 VPS. Get all the benefits of WireGuard mesh networking without Nebula's relay fragility.

The migration was delicate. Serial execution, node by node. And then, for the first time since the cluster's inception, I restored a **three-node etcd quorum** on high-performance Framework Desktop machines. Two Southern California members under 10ms apart, and one in Austin at 50ms. The single point of failure was gone.

## Act 3: Everything Breaks

The migration was complete. I exhaled. Then I opened a browser and typed `console.kub0.net`. 

Nothing. 502 Bad Gateway. 

What followed was a twelve-hour debugging marathon. GPU taints blocking load balancers, Traefik refusing ExternalName services, Helm upgrades wiping Middlewares, and LINSTOR webhooks disappearing into the void. Each fix uncovered a deeper, more creative way for the system to be broken.

## Act 4: The Matryoshka Revelation

A week later, I discovered that WireGuard was building its "direct" tunnels *through* the Nebula overlay I'd just "replaced." Three layers of networking, stacked like Russian dolls: Flannel on Tailscale, on Nebula, on Physical LAN.

Sometimes the best architecture is the one you accidentally built by never finishing the demolition.

## Act 5: The Tactical Toolkit

After the twelve-hour debugging marathon, I was exhausted. The cluster had gotten complex enough that operating it required remembering which overlay was which, which nodes were control plane, and which SSH key to use.

So I built a simple Python script called `k0`.

![k0 CLI — Tactical HUD showing node status and IP lanes](/assets/img/posts/shortcircuit/k0-help-planes.png)

At the time, it was just a tactical emergency radio — a tool born of necessity to see through the smoke of the migration. It wrapped `ansible` and `kubectl` into a single HUD that showed me the state of every node in three seconds, not thirty. It was a reactive tool for a reactive week.

I didn't know then that this script would soon evolve into the cluster's sovereign gearbox.

## The Naming

I named that VPS `shortcircuit-lighthouse-sfo3` because it was supposed to be a simple relay. Instead, it became the coordinator for the entire mesh. 

The week I migrated to it, everything broke. But that wreckage was *recoverable*. The breakage was catastrophic, but the state was versioned. You can rebuild a house from blueprints.

ShortCircuit. Sometimes you have to break everything to find out what's actually load-bearing.

---

*This is part of an ongoing series about building a 20-node K3s cluster across four countries. Next entry: [The Sentinel Protocol](https://jkubo.com/posts/the-sentinel-protocol/) — Moving beyond reactive fixes to self-healing architecture.*
