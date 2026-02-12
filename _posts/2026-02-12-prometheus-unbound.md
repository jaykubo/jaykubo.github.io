---
title: "Prometheus Unbound: Stealing Fire from a Burning Cluster"
date: 2026-02-12 12:00:00 +0900
categories: [Infrastructure, Kubernetes, Observability]
tags: [k3s, mimir, loki, grafana, minio, rustfs, observability, tailscale]
layout: post
image:
  path: /assets/img/posts/prometheus-unbound.png
  alt: Building Observability on a Geo-Distributed K3s Cluster
---

In Greek mythology, Prometheus stole fire from the gods and gave it to mortals. Zeus punished him for it — chained to a rock, an eagle eating his liver every day, regenerating every night to suffer again. The crime wasn't theft. It was giving humans the ability to see in the dark.

I've been building [kub0.ai](https://kub0.ai) in the dark.

[Post 1](/posts/the-16gb-panini/) covered the DNS authority layer. [Post 2](/posts/the-k3s-zombie-apocalypse/) covered the MTU incident that nearly collapsed the cluster — flannel.1 dying silently, etcd losing quorum, a node running pods it could no longer route. The fix was reactive: SSH in, check `lsmod`, restart services, pray. I only knew the cluster was dying because I could feel it.

This is the post about building the fire. It documents the design and deployment of a long-term observability stack (Mimir, Loki, Grafana, MinIO) on a geo-distributed K3s cluster, and the failure modes that shaped its architecture.

---

## The Titan's Bargain

The incident from Post 2 had a clean lesson buried in the chaos: **"Ready" is a lie.**

A node can report `Ready` to the Kubernetes control plane while:

- `flannel.1` doesn't exist
- Routes to `10.43.0.0/16` are missing
- CoreDNS is unreachable from every pod on that node
- etcd apply-duration is at 99ms — one millisecond from quorum collapse

The control plane doesn't know about any of this. It checks a heartbeat. The heartbeat says alive. The node says Ready. Meanwhile, pods are running blind, cut off from the cluster they think they're part of.

The only way to see this is to build sensors. That's what this post is about.

---

## Forging the Chains: The Storage Decision

Before deploying Mimir and Loki, I needed object storage. The observability stack produces a lot of data — 90-day metric retention, 90-day log retention, across 18 nodes spanning Austin, Los Angeles, Tokyo, and Torrance.

I have a dedicated NAS node (`aus-nas-01`) with four 3.6TB drives — 14TB raw. The natural choice was MinIO.

**The problem: AGPL-3.0.**

MinIO's license requires that any software *touching* MinIO in a commercial context be open-sourced under the same terms. For a startup building privacy-preserving AI infrastructure, this is a ticking clock. Better to solve it with 8 hours of data than with terabytes.

I went looking for an Apache 2.0 alternative.

### The Frontier Tax: RustFS Alpha.83

RustFS is a Rust-native S3-compatible object store. Apache 2.0 licensed. Active development — Alpha.83 was released four hours before I deployed it. That should have been a warning.

Deployment went smoothly. The pod came up. The health check passed:

```json
{"service":"rustfs-endpoint","status":"ok","version":"0.0.5"}
```

Then I pointed Mimir at it. Every store-gateway crashed with the same error:

```
failed to synchronize TSDB blocks for user anonymous: sync block:
read bucket index: Io error: rmp serde decode error:
invalid type: unit value, expected a sequence
```

Loki failed identically:

```
api error InternalError: Io error: rmp serde decode error:
invalid type: unit value, expected a sequence
```

MessagePack serialization mismatch. Grafana's tooling expects a specific binary format when reading bucket indexes from S3. RustFS Alpha.83 returns something structurally different. Both Mimir and Loki failed on first contact with real data.

S3 compatibility isn't binary — it's a spectrum. RustFS implements the API surface but not the exact serialization semantics that Grafana's stack depends on. This is the frontier tax: you pay it in time, and you learn that "S3-compatible" is a claim that requires verification, not assumption.

I rolled back to MinIO in five minutes, because the MinIO playbook was version-controlled and the rollback path was documented. Infrastructure brittleness is inversely proportional to reproducibility.

**Verdict on RustFS:** Come back at Alpha.200+. File the bug report. Contribute to the project. But don't run it under Mimir in production.

### The Pragmatic Choice

MinIO is AGPL. That concern is real. But I deferred it — the commercial risk is theoretical today, and the operational risk of running Alpha software is concrete.

One architectural decision I made during the migration attempt proved its value: I renamed the storage mounts from `/mnt/minio-disk{1,2,3,4}` to `/mnt/blob-disk{1,2,3,4}`. Generic names. When I swapped storage backends, the filesystem paths didn't encode the vendor name. When I swapped back, same story.

This also taught me something I almost missed: **filesystem ownership is state that persists across container restarts.** RustFS runs as UID 10001. MinIO runs as UID 1000. After chowning the disks for RustFS and then rolling back, MinIO couldn't write anything. The pods showed `1/1 Running`. The post-job retried 16 times before I caught it.

A running pod with `EACCES` at the syscall level is harder to debug than a crash.

---

## Fire, Finally: The Stack Deployment

With MinIO stable, I deployed the full observability stack via Ansible playbooks — idempotent, version-controlled, reproducible.

### Components

**Cert-Manager** handles TLS certificate automation across the cluster. Foundation layer — everything else depends on it.

**MinIO** provides 14TB of S3-compatible object storage across four dedicated NAS disks. Distributed mode, four nodes, all on `aus-nas-01`.

**Mimir** is Grafana's horizontally-scalable Prometheus backend. It handles metrics storage with 90-day retention. The deployment spans 20 pods across three availability zones (zone-a, zone-b, zone-c), with a Kafka-based ingest pipeline decoupling write throughput from storage latency.

**Loki** handles log aggregation at the same 90-day retention. 21 pods in a write/read/backend split architecture, plus 11 canary pods distributed across every node in the cluster — one per node, continuously probing end-to-end log pipeline health.

**OpenTelemetry Collector** is the unified collection layer. It scrapes Kubernetes metrics from the API server and running pods, then routes them to Mimir (metrics) and Loki (logs). Two replicas for HA.

**Grafana** is the visualization layer. NodePort 31035, ClusterIP internally, pre-loaded with Mimir and Loki datasources.

### The Data Flow

```
Kubernetes API / Pod Metrics
         |
         v
  OTel Collector (2x HA)
    /            \
   v              v
Mimir           Loki
(metrics)       (logs)
   \              /
    v            v
        MinIO
     (14TB NAS)
         |
         v
      Grafana
```

### Deployment Challenges Worth Documenting

**Service naming matters.** The Mimir endpoint isn't `mimir-nginx` — it's `mimir-gateway`. The RustFS service was `rustfs-svc`, not `rustfs`. DNS failures that look like network issues are often just wrong service names in configuration.

**Jobs are immutable.** Kubernetes Job specs cannot be updated in place. When I changed the bucket creation job to point at RustFS and then back to MinIO, I had to delete the old Job before reapplying. `kubectl delete job -n observability create-mimir-buckets` before every config change.

**Pod anti-affinity nearly blocked NAS deployment.** The MinIO Helm chart defaults assume nodes are spread across failure domains. All four NAS disks are on the same physical machine. I had to explicitly disable pod anti-affinity for the NAS deployment.

**Stale SQLite state broke Grafana.** After a failed deployment left an old PVC with a Grafana SQLite database, the new deployment couldn't run migrations against it:

```
no such column: dashboard_provisioning.check_sum
```

The fix: delete the PVC entirely and let Grafana initialize fresh. A reminder that persistent storage persists failures, not just data.

---

## The Zombie Returns

I thought the incident was over.

While the observability stack was coming up, one pod kept crash-looping: `mimir-store-gateway-zone-c`. Every restart landed on `aus-nas-01`. The logs showed a familiar signature:

```
lookup mimir-gossip-ring.observability.svc.cluster.local on 10.43.0.10:53:
read udp: i/o timeout
```

The node was `Ready`. Tailscale was active. MTU was 1280. But pods on that node couldn't reach CoreDNS at `10.43.0.10`.

```bash
ip link show flannel.1
# Device "flannel.1" does not exist.

ip route | grep 10.43
# (silence)
```

No VXLAN interface. No route to the service network. A node running six pods, all of them cut off from cluster DNS, in a state the control plane couldn't see.

The fix was `sudo systemctl restart k3s` — not `k3s-agent`, because this was a control plane node running the server process. Flannel.1 reappeared. Routes were restored. Store-gateway-zone-c came up clean.

This is the same zombie state from Post 2, on a different node, 24 hours later, surfacing during the construction of the sensors designed to detect it. The irony is precise: I was building Prometheus while the cluster was doing exactly what Prometheus was meant to catch.

The MTU-induced packet fragmentation that kills flannel's VXLAN interface doesn't announce itself. The node stays Ready. The pods stay Running. Nothing alerts. You only find out when a pod tries to resolve a cluster service and times out.

That changes now.

---

## What the Fire Reveals

The observability stack is operational. Here's what I can now see that we couldn't before:

**Mimir is receiving metrics.** 58+ metric families flowing from OTel Collector into MinIO-backed long-term storage. Traefik ingress metrics, cert-manager certificate lifecycle, Go runtime stats from every component.

**Loki canaries are probing every node.** 11 canary pods — one per cluster node — continuously pushing log entries end-to-end through the Loki pipeline. If a canary goes silent, a node is having trouble reaching the Loki write path. This is the early warning system for exactly the kind of DNS failure we just survived.

**What we can't see yet** (the next phase):

- `flannel.1` existence per node — a missing interface should page immediately
- Route coverage of `10.43.0.0/16` — no route means no cluster DNS
- etcd apply-duration per member — alert at 95ms, not 100ms
- MTU verification on `tailscale0` per node — drift kills the bridge
- DNS resolution latency from each node to CoreDNS

These are the specific gaps the incident exposed. The dashboards will be built around them.

---

## Lessons

**On alpha software:** "S3-compatible" is a claim, not a guarantee. Verify against your actual workload before committing. RustFS will be excellent software — the Rust foundation is solid and the development velocity is real. Alpha.83 just isn't there yet for Mimir/Loki's serialization expectations.

**On filesystem state:** Containers are ephemeral. The filesystems they mount are not. UID ownership, stale metadata, leftover `.minio.sys` directories — all of it persists across pod restarts and even across complete stack redeployments. When something inexplicably fails after what should have been a clean redeploy, check the disk.

**On naming:** Generic mount names (`/mnt/blob-disk*`) cost nothing and save hours. Vendor-specific names encode assumptions that break when assumptions change.

**On Ready:** Kubernetes Ready status is a heartbeat check, not a health check. A node can be Ready while flannel is dead, DNS is unreachable, and every pod on it is running blind. The control plane doesn't know what it doesn't measure.

---

## Side Quest: kubo.xyz

While researching distributed storage alternatives, I discovered that IPFS has a Go implementation called **Kubo**. I own **kubo.xyz**.

The naming overlap with kub0 is not lost on us. Future exploration: content-addressed storage for immutable cluster state snapshots, artifact pinning, and distributed config management. IPFS as infrastructure layer, not just file sharing.

---

## What's Next

The sensors are online. The dashboards are empty. The next phase:

1. **node-exporter DaemonSet** — complete node-level metrics (CPU, memory, disk, network per interface including `tailscale0` and `flannel.1`)
2. **Flannel health dashboard** — interface existence, route coverage, MTU verification per node
3. **etcd latency alerting** — apply-duration tracking with 95ms threshold
4. **Loki canary alerting** — silence detection per node
5. **Post 4** — "The Dashboard" — building the sensor grid from the incident spec

The cluster is no longer a black box. The fire is lit. Now we learn what it illuminates.

---

*The observability stack is running. Mimir has metrics. Loki has logs. Grafana has datasources. The zombie node was fixed — by restarting the right service this time. Prometheus is unbound. The eagle will have to find something else to eat.*

---

**Stack Summary**

| Component | Version | Pods | Status |
|-----------|---------|------|--------|
| MinIO | 2024-12-18 | 4/4 | Running |
| Mimir | 3.0.1 | 20/20 | Running |
| Loki | 3.6.5 | 21/21 | Running |
| Grafana | 12.3.1 | 1/1 | Running |
| OTel Collector | 0.91.0 | 2/2 | Running |
| Cert-Manager | 1.13.3 | 3/3 | Running |

**Cluster:** 18 nodes — Austin (7), Los Angeles (6), Tokyo (3), Torrance (1), plus GPU forward nodes in each region. K3s + Flannel + etcd HA over Tailscale mesh, MTU 1280.
