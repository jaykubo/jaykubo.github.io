---
title: "The Nerve Center: Giving a Cluster a Face It Deserves"
date: 2026-02-24 14:00:00 +0900
categories:
  - Platform
  - Observability
tags:
  - homepage
  - console
  - pulse-api
  - go
  - adsb
  - cctv
  - cloudflare-tunnel
  - cloudflare-workers
  - traefik
  - k3s
  - tailscale
  - grafana
  - kube-state-metrics
  - mimir
  - otel
  - prometheus
  - alerting
  - claude-code
  - ai-tools
layout: post
image:
  path: /assets/img/posts/nerve-center.png
  alt: Console Dashboard Redesign with Live API Widgets on K3s
---

`console.kub0.io` had a problem no amount of green status dots could fix: it looked like a random spreadsheet of services, and it told you nothing.

Eight flat groups. Identical two-column rows. "Observability" next to "Kubernetes" next to "Aviation" — as if Grafana dashboards, pod statuses, and ADS-B feeders were the same category of thing. No live data. No hierarchy. No personality. The cluster had [ears](https://jkubo.com/posts/every-thirty-seconds/), [eyes](https://jkubo.com/posts/the-all-seeing-eye/), an [intelligence map](https://jkubo.com/posts/the-overwatch-map/), an [immune system](https://jkubo.com/posts/the-immune-system/), and a [familiar](https://jkubo.com/posts/the-familiar/) — but its face was a flat list of links. Time to build a nerve center.

## The Before

```
Observability:    Grafana
Storage:          SeaweedFS S3 | SeaweedFS Filer | LINSTOR
Networking:       DNS Torrance | DNS Tokyo
Kubernetes:       Headlamp | SeaweedFS pods | Observability pods | Piraeus pods
Home:             Home Assistant
Aviation:         ADS-B Dashboard
Security:         CCTV API
Development:      JupyterLab | Forgejo
```

Eight groups, 2 columns each. Every card identical. The CCTV API — a Go service tracking 4,158 cameras across two states with hash deduplication and S3 archival — got the same visual weight as a DNS management UI. Pod statuses lived in a "Kubernetes" ghetto, divorced from the services they monitored.

And when you clicked a card? Links. Just links. No hint of what was healthy, how many cameras were tracking, how many aircraft were overhead, or whether the cluster's 18 nodes were all still breathing.

## The Redesign

The new layout borrows from Google Cloud Console's philosophy: **APIs and live data first, infrastructure second, everything else after.**

```
APIs & Intelligence:    Pulse API [18 nodes | 185 pods | 12 ns]
                        CCTV API  [4,158 cameras | 847K archived | 391 stale]
                        ADS-B API [16 aircraft | 67M messages]
                        kub0.ai

Cluster & Observability: Headlamp | Grafana [2 alerts | 47 total] | Observability pods
Storage:                 LINSTOR | SeaweedFS S3 | SeaweedFS Filer | SeaweedFS pods | Piraeus pods
Networking:              DNS Torrance | DNS Tokyo
Development:             JupyterLab | Forgejo
Home:                    Home Assistant
```

Seven groups. APIs lead. Live stats on every card that has something worth counting. Pod statuses distributed into the groups they belong to — SeaweedFS pods under Storage, Observability pods under Cluster. The old "Kubernetes" and "Aviation" and "Security" categories? Gone.

### The Widget Treatment

Homepage's `customapi` widget can pull JSON from any in-cluster endpoint and map fields to labels. CCTV already had a `/stats` endpoint — that was easy:

```yaml
widget:
  type: customapi
  url: http://cctv-api.networking.svc.cluster.local:8080/stats
  refreshInterval: 30000
  mappings:
    - field: total_cameras
      label: Cameras
      format: number
    - field: archive.total_archived
      label: Archived
      format: number
    - field: stale_cameras
      label: Stale
      format: number
```

Three numbers, refreshing every 30 seconds, visible without clicking anything. That's the gold standard. The question was: how to give every API the same treatment.

## Pulse: The Cluster's Heartbeat

The cluster had metrics (Mimir), logs (Loki), and alerts (Grafana). What it didn't have was a single endpoint that answered the simplest question: **is everything up?**

Pulse API was built to fill that gap. A Go service with read-only RBAC that polls the Kubernetes API every 30 seconds and returns a single JSON response:

```json
{
  "cluster": {
    "version": "v0.67",
    "uptime_seconds": 1125338,
    "nodes_ready": 18, "nodes_total": 18,
    "pods_running": 185, "pods_total": 192,
    "namespaces": 12
  },
  "regions": {
    "AUS": { "nodes": 8, "ready": 8, "latency_ms": 5.1 },
    "LAX": { "nodes": 5, "ready": 5, "latency_ms": 49.0 },
    "HND": { "nodes": 5, "ready": 5, "latency_ms": 172.4 }
  }
}
```

Nodes, pods, namespaces, K3s version, uptime. Plus TCP dial latency to each region's control plane — actual round-trip time from the pod to a Tailscale IP in Austin, LA, and Tokyo. Under 50ms is green. Under 100ms is yellow. Over 100ms is red (Tokyo, always red, ocean latency is physics not a bug).

Content negotiation at the root: browsers get an embedded HTML dashboard with pulsing live indicators and an API explorer. API clients get JSON. Same pattern as the [CCTV API](https://jkubo.com/posts/the-all-seeing-eye/).

```
~/Projects/pulse-api/
├── cmd/pulse-api/main.go          # Entry point, env config
├── internal/
│   ├── api/server.go              # HTTP server, dashboard HTML, Prometheus metrics
│   └── collector/
│       ├── cluster.go             # K8s polling, TCP latency probes
│       └── adsb.go                # ADS-B feeder aggregation
├── Dockerfile                     # Multi-stage, scratch, ~10MB
└── Makefile                       # buildx multi-arch (amd64 + arm64)
```

Deployed to `networking` namespace. Public at `api.kub0.io/pulse/`, internal at `pulse.kub0.xyz/`. ServiceAccount with read-only ClusterRole on nodes, pods, namespaces. Resource limits: 10m CPU, 128Mi memory. The entire service uses less RAM than a single Grafana panel.

## The ADS-B Problem

CCTV had a `/stats` endpoint from day one. Pulse was built to be a stats endpoint. ADS-B had... three dump1090 feeders returning raw aircraft arrays.

```json
{
  "now": 1771416195.7,
  "messages": 2361,
  "aircraft": [
    {"hex":"a25800","flight":"FDX1432","lat":30.38,"lon":-97.63,...},
    ...
  ]
}
```

Homepage's `customapi` can map JSON fields to labels. It cannot count array elements. There was no `total_aircraft` field — just an array you had to count yourself. Three separate endpoints (one per feeder). No deduplication (the same aircraft heard by multiple feeders appears in each response). No unified view.

The answer was already running: Pulse. It already polled the cluster every 30 seconds. Adding ADS-B aggregation was one new file.

### `collector/adsb.go`

```go
var adsbEndpoints = map[string]string{
    "AUS": "http://adsb-aus.networking.svc.cluster.local:8080/data/aircraft.json",
    "LAX": "http://adsb-lax.networking.svc.cluster.local:8080/data/aircraft.json",
    "HND": "http://adsb-hnd.networking.svc.cluster.local:8080/data/aircraft.json",
}
```

Three concurrent HTTP fetches. Per-region aircraft and message counts. Cross-feeder deduplication by hex code (ICAO transponder address). The same FedEx cargo plane heard by both Austin and LA appears once in the total.

The Pulse API response grew an `adsb` field:

```json
{
  "cluster": { ... },
  "regions": { ... },
  "adsb": {
    "total_aircraft": 16,
    "total_messages": 67184870,
    "regions": {
      "AUS": { "aircraft": 1, "messages": 2643 },
      "LAX": { "aircraft": 15, "messages": 67182227 }
    }
  }
}
```

Now ADS-B has the same widget treatment as CCTV:

```yaml
- ADS-B API:
    widget:
      type: customapi
      url: http://pulse-api.networking.svc.cluster.local:8080/
      mappings:
        - field: adsb.total_aircraft
          label: Aircraft
        - field: adsb.total_messages
          label: Messages
```

Three API cards. Three sets of live stats. Refreshing every 30 seconds. No clicking required.

---

## Proprioception

Proprioception is the sense that tells you where your hand is without looking at it. Close your eyes and touch your nose — that's proprioception. The body's internal map of itself, independent of what any external sensor reports.

Pulse was now reporting 18 nodes ready. The console displayed it proudly. The numbers looked right.

The problem was what "ready" meant.

### The Audit

A routine check of the Cluster Nodes dashboard surfaced the problem. The stat panel labelled "Total Nodes Up" was querying `up{job="kubernetes-nodes"}` — not K8s readiness, but kubelet metrics endpoint reachability. The Node Status table below it showed IP addresses and port numbers. `100.67.x.82:10250`. `100.67.x.122:10250`. Seventeen Tailscale IPs in a table called "Node Status," colored green.

That's not node status. That's "can OTel reach the kubelet's HTTP endpoint." A node can return `up=1` while its disk is full, its kubelet is degraded, and K8s is actively evicting its pods. `kube_node_status_condition{condition="Ready"}` is the metric that actually answers the question. And it didn't exist anywhere in Mimir.

kube-state-metrics wasn't deployed.

### The Fix That Wasn't

The fix is obvious: deploy kube-state-metrics, add a scrape job to OTel, add a filter to keep only the metrics we need.

```yaml
- job_name: 'kube-state-metrics'
  scrape_interval: 60s
  static_configs:
    - targets: ['kube-state-metrics.observability.svc.cluster.local:8080']
  metric_relabel_configs:
    - source_labels: [__name__]
      action: keep
      regex: kube_node_.*|kube_pod_.*|kube_deployment_.*|kube_daemonset_.*
```

Helm install. OTel config applied. Collector restarted. Wait 75 seconds for the first scrape cycle.

```
warn  internal/transaction.go:123  Failed to scrape Prometheus endpoint
  {"scrape_timestamp": 1771505174450,
   "target_labels": "{instance=\"kube-state-metrics.observability.svc.cluster.local:8080\"}"}
```

Every. Single. Cycle.

Manual curl from inside the cluster returned clean metrics immediately. `kube_node_info`, `kube_node_status_condition`, `kube_pod_info` — all present, all correct. The endpoint was healthy. The scrape job registered successfully. But every 60 seconds, `transaction.go:123` fired, and Mimir received nothing.

This took an embarrassingly long time to debug. The `transaction.go:123` warning in OTel v0.91.0 fires when the scrape succeeds (HTTP 200, full response received) but the resulting transaction contains zero data points. The working hypothesis — OTel silently dropping all metrics due to the `action: keep` filter — seemed too stupid to be true.

It was true.

OTel v0.91.0's prometheus receiver has a quirk: `metric_relabel_configs` with `source_labels: [__name__]`, `action: keep`, and a **pipe-alternation regex** drops every metric. The existing scrape jobs in this cluster use the same `source_labels: [__name__]` pattern with `action: drop` — which works fine. And the cadvisor job uses `action: keep` with a long allowlist — also fine, because that regex has no `|` operators. It's specifically the combination of `keep` + alternation that causes the silent zero-data transaction.

No error. No warning about the regex. The endpoint is reachable. The metrics exist. The filter kills them all and OTel logs a single generic warning.

The fix: remove the OTel-level filter entirely, and do the filtering at the source.

### The Ballast Problem

Default kube-state-metrics is verbose. The full output for this cluster was 2.8MB and 16,929 metric lines per scrape. The culprits:

```
kube_pod_annotations{...}         # one row per pod per annotation
kube_pod_labels{...}              # one row per pod per label
kube_configmap_annotations{...}   # every configmap, every annotation
kube_pod_tolerations{...}         # 1,293 rows for toleration entries alone
```

Annotation and label metrics exist for every resource type. They're useful in some contexts (syncing label values into dashboards). For this use case — node readiness, pod health, deployment status — they're ballast.

The Helm chart's `metricDenylist` and `collectors` values handle this at the source, before metrics are ever generated:

```yaml
collectors:
  - nodes
  - pods
  - deployments
  - daemonsets
  - statefulsets
  - namespaces
  - replicasets
  - persistentvolumeclaims
metricDenylist:
  - kube_.*_annotations
  - kube_.*_labels
  - kube_pod_tolerations
```

Filtered at the KSM level, no OTel relabeling needed. The scrape job became two lines:

```yaml
- job_name: 'kube-state-metrics'
  scrape_interval: 60s
  static_configs:
    - targets: ['kube-state-metrics.observability.svc.cluster.local:8080']
```

Next cycle: 18 `kube_node_info` series, 204 `kube_pod_info` series, 231 `kube_node_status_condition` series. All 18 nodes Ready.

The dashboard now shows K8s readiness (not kubelet reachability), node names instead of IP addresses, and a pressure conditions panel — MemoryPressure, DiskPressure, PIDPressure — per node.

### The Alert

The existing "Node Down" alert fires when `up{job="kubernetes-nodes"} == 0` — the kubelet metrics endpoint is unreachable. That's an infrastructure alarm. The network is broken, or the node is off.

"Node Not Ready" is a different signal:

```yaml
- uid: node-not-ready
  title: Node Not Ready
  data:
    - refId: A
      expr: kube_node_status_condition{condition="Ready",status="true"} == 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Node {{ $labels.node }} is NotReady"
```

A node can be reachable — metrics flowing, kubelet responding — and still be NotReady. Disk pressure. Memory pressure. A kubelet component failure. A node cordoned after a maintenance event that was never uncordoned. `up=1` doesn't catch any of those. `kube_node_status_condition == 0` does.

The `2m` grace period matters: K8s takes up to two minutes to mark a node NotReady after it stops heartbeating. By the time the metric flips to 0, the node has already been struggling. Two more minutes of sustained NotReady before the alert fires means roughly four minutes of real degradation — enough to distinguish a transient blip from a genuine failure.

`noDataState: OK` applies, same as every other rule here. The "OTel Collector Down" watchmen rule already covers the case where kube-state-metrics stops reporting entirely.

---

## The Auth Header

The old auth bar was plain text — "Signed in as **jkubo**" with a red text link. Functional. Forgettable.

The new version pulls the user's GitHub avatar (since OAuth2 is through GitHub anyway), displays it as a 28px circle next to the username, and replaces the text "Sign out" with an inline SVG log-out icon in a pill button. If the avatar fails to load (non-GitHub user, network issue), it falls back to a letter-initial circle.

```javascript
var avatarUrl = 'https://github.com/' + user + '.png?size=56';
```

Small touch. But when every service in the cluster gets its identity from GitHub OAuth, showing the GitHub face makes the connection literal.

## The Iframe Experiment (and Why It Failed)

The original plan included embedded iframes — the Intel Map and CCTV Explorer as live dashboards directly in the console. Two problems killed it instantly:

**X-Frame-Options.** kub0.ai returns `SAMEORIGIN` via Cloudflare Workers. The console at `console.kub0.io` is a different origin. The browser blocks the embed. Fixable by updating the Worker's response headers, but it opens a framing vector.

**Value proposition.** The CCTV API's content negotiation already serves a full HTML explorer when you click the card. The Intel Map is a full-viewport Leaflet experience that doesn't make sense in a 400px iframe. Both are better as click-through destinations than cramped embeds.

The "Live Dashboards" group was removed entirely. The API cards link to their explorers. The stats show what matters at a glance. If you want the full experience, click through.

## The Shell Gets Real

Pulse powered the console. It also fixed a lie.

The kub0.ai shell module — a terminal UI on the site — had a secret: every number was fake. `PING` showed random latency to fictional nodes. `STATUS` reported hardcoded strings. `NETSTAT` counted packets by multiplying seconds since the LLC founding date by 1,000. Uptime was days since July 17, 2020.

It looked convincing. It meant nothing.

| Before | After |
|--------|-------|
| UPLINK_UPTIME: `4Y 216D` (LLC founding) | CLUSTER_UPTIME: `13D 0H` (oldest node) |
| ACTIVE_SENSORS: `03` (hardcoded) | ACTIVE_NODES: `18/18 NODES` (real) |
| ENCRYPTION_LEVEL: `AES-256` (hardcoded) | RUNNING_PODS: `186 PODS` (real) |

The shell now fetches from pulse-api on load and every 30 seconds after. `PING` shows real TCP latency, color-coded green/yellow/red. `STATUS` prints the actual K3s version, node count, uptime, and per-region breakdown. The fake data generators — `updateLatency()`, the founding-date uptime math, the random jitter intervals — all deleted.

One rule: if the API is unreachable, every cluster command prints `CLUSTER_UNREACHABLE`. No fake fallback. No invented numbers. If the data isn't real, you see nothing.

---

From antenna to terminal: radio waves hit SDR dongles in three countries, aircraft data flows through [the ADS-B pipeline](https://jkubo.com/posts/every-thirty-seconds/), [4,158 cameras](https://jkubo.com/posts/the-all-seeing-eye/) archive to SeaweedFS, and the cluster's own vitals — 18 nodes, 186 pods, 12 namespaces, three regions — pulse through the same tunnel to the same terminal.

The body has a face now. The numbers are real. And for the first time, it knows whether its own limbs are responding.

That's a different kind of knowing.
