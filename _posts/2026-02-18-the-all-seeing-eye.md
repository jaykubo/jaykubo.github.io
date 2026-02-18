---
title: "The All-Seeing Eye: 4,158 Traffic Cameras and the Art of Not Storing Garbage"
date: 2026-02-18 14:00:00 +0900
categories:
  - Infrastructure
  - Kubernetes
tags:
  - cctv
  - go
  - seaweedfs
  - cloudflare-tunnel
  - data-pipeline
  - k3s
  - tailscale
  - distributed-systems
  - claude-code
  - ai-tools
  - traefik
layout: post
image:
  path: /assets/img/posts/all-seeing-eye.png
  alt: CCTV Traffic Camera Archival Pipeline on K3s
---

There's a moment in every infrastructure project where the scope quietly doubles while you're looking the other way. This one started as "let's scrape some traffic camera metadata" and ended with 4,158 cameras across two states, a Go rewrite, SHA256→FNV hash deduplication, state persistence, and a heated argument with CalTrans District 7 about HTTP timeouts.

The body metaphor continues: if [Mimir is the eyes](https://jkubo.com/posts/prometheus-unbound/) and [LINSTOR is the skeleton](https://jkubo.com/posts/the-47tb-exorcism/), then this is the **surveillance system** — the cluster's peripheral vision, watching 4,158 intersections across Austin and California’s highway network, archiving what changes and ignoring what doesn't.

## The Python Prototype (and Why It Had to Die)

Every good Go service starts as a bad Python script.

The original implementation lived in a Jupyter notebook — because of course it did. Socrata API calls for Austin's ~788 cameras, CalTrans CCTV API for California's 12 districts. Pandas for data wrangling. Parquet for storage. The kind of stack that works beautifully on a laptop and catastrophically on a 2GB Raspberry Pi.

The problems were structural:

| Issue | Python | Go |
|-------|--------|-----|
| Runtime image | ~800MB (pip + pandas + numpy) | ~10MB (scratch + static binary) |
| Memory at rest | 256-512MB | ~32MB |
| Cold start | 8-12s (pip install every CronJob) | instant |
| Dependencies | pandas, pyarrow, minio, requests | net/http, minio-go (stdlib for the rest) |
| Deployment | CronJob (run, scrape, die) | Long-running service (scrape + serve + archive) |

The CronJob model was the real killer. Every 5 minutes: pull image → pip install → import pandas → scrape → write parquet → die. The overhead dwarfed the actual work. And on the RPi nodes with 2-4GB RAM, pandas alone ate half the available memory before a single camera was scraped.

The Go rewrite wasn't premature optimization. It was mercy.

## Architecture: One Binary to Rule Them All

The Go service does three things in one process:

1. **Scrapes** camera metadata from Austin (Socrata API) and California (CalTrans 12-district API) every 5 minutes
2. **Serves** a JSON API with camera lists, stats, filtered views, and an image proxy
3. **Archives** camera images to SeaweedFS S3 with hash-based dedup

```
┌─────────────────────────────────────────────┐
│              cctv-api (Go)                  │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
│  │ Scraper  │  │ HTTP API │  │ Archiver  │ │
│  │ (5m tick)│  │ (:8080)  │  │ (50 conc) │ │
│  └────┬─────┘  └────┬─────┘  └─────┬─────┘ │
│       │              │              │       │
│  ┌────▼──────────────▼──────────────▼─────┐ │
│  │           In-Memory Cache              │ │
│  │  austin[788]  california[3370]         │ │
│  │  hashMap[4152]  staleSet[391]          │ │
│  └────────────────────────────────────────┘ │
└────────────┬────────────────────┬───────────┘
             │                    │
     ┌───────▼───────┐   ┌───────▼────────┐
     │  CalTrans /   │   │  SeaweedFS S3  │
     │  Austin APIs  │   │  cctv-images   │
     └───────────────┘   └────────────────┘
```

The image proxy deserves special mention. Traffic camera images are hosted on government servers that (reasonably) don't set CORS headers. The kub0.ai frontend can't fetch them directly. So the API proxies: `GET /image/austin/42` → fetch from Austin's Socrata CDN → stream to client. The browser sees `api.kub0.io` as the origin. CORS solved.

## The Hash Dedup Problem

Here's the dirty secret of traffic camera archival: **most cameras don't change most of the time**.

A highway camera at 3 AM shows the same empty road for hours. A broken camera returns a static "TEMPORARILY UNAVAILABLE" JPEG forever. Without dedup, archiving 4,158 cameras every 5 minutes means:

```
4,158 cameras × 12 cycles/hour × 24 hours × ~60KB avg = ~72 GB/day
```

Seventy-two gigabytes a day of mostly identical images. On a cluster where [pool-sata is 10.9 TiB](https://jkubo.com/posts/the-47tb-exorcism/), that's a five-month countdown to full.

The fix: hash every image before uploading. If the hash matches the last known hash for that camera, skip the upload entirely.

```go
h := fnv.New64a()
h.Write(imageData)
hash := h.Sum64()

if exists && hash == lastHash {
    return false, nil // skip — camera unchanged
}
```

The initial implementation used SHA-256. Cryptographically bulletproof, completely unnecessary. We're deduplicating traffic camera JPEGs, not verifying nuclear launch codes. FNV-1a (64-bit, non-cryptographic) runs in nanoseconds and has more than enough collision resistance for our use case...for this workload.

### The Stale Camera Problem

Dedup created a secondary insight: cameras that never change are **stale**. After each archive cycle, any camera whose hash matched its predecessor gets flagged `stale: true` in the API response. The frontend filters these out of sample feeds — no more thumbnail grids full of "CAMERA OFFLINE" placeholders.

After the first full cycle: 391 of 4,158 cameras were stale. Nearly 10% of the fleet is serving static images. That's 391 cameras × 12 cycles/hour × 24 hours = **112,608 S3 uploads avoided per day**.

## The Timeout Wars

The first deployment revealed a fundamental tension: the archive process was competing with itself.

The Go service used one `http.Client` with a 30-second timeout for everything — API proxying, metadata scraping, and image archival. With 4,158 cameras and 10 concurrent workers, the initial archive cycle took **over 7 minutes**. Offline cameras would hang for the full 30 seconds before timing out, blocking workers that could be downloading active cameras.

Worse: the archive couldn't finish within the 5-minute scrape interval, so cycles stacked up. And the 50-goroutine archive saturated the pod's network, causing Kubernetes liveness probes to timeout — the kubelet killed the pod mid-archive, losing all in-flight progress.

Three fixes, applied together:

1. **Separate HTTP clients**: 30s timeout for user-facing proxy, 10s for archival. Offline cameras fail fast.
2. **Relaxed probes**: `timeoutSeconds: 5`, `failureThreshold: 5`. The pod survives network saturation.
3. **Bumped resources**: 256MB memory limit (50 concurrent image downloads need buffer space).

Result: archive cycle dropped from **7+ minutes to 2 minutes 21 seconds**.

## State Persistence: Surviving Restarts

The hash map and cumulative counters lived in memory. Every pod restart — rollout, node drain, OOM kill — reset the dedup state. The first archive cycle after restart would re-upload every single image, defeating the entire purpose.

The solution: persist state to the same S3 bucket we're archiving to.

```
cctv-images/
├── _meta/
│   └── state.json          # ~213KB: hash map + counters
├── austin/
│   └── {camera_id}/
│       └── {date}/{time}.jpg
└── california/
    └── {camera_id}/
        └── {date}/{time}.jpg
```

On graceful shutdown (SIGTERM), the server serializes its hash map (4,152 entries; excludes cameras filtered during normalization) and cumulative archive counters to `_meta/state.json`. On startup, it loads this file before the first scrape. The pod can restart as many times as it wants — dedup state survives.

The `terminationGracePeriodSeconds: 30` on the Deployment gives the shutdown hook enough time to write the state file before Kubernetes sends SIGKILL.

## The Public API

The CCTV API serves at two endpoints:

| Domain | Access | Purpose |
|--------|--------|---------|
| `cctv.kub0.xyz` | Tailnet-only | Internal dashboard + API |
| `api.kub0.io/cctv` | Public (Cloudflare Tunnel) | Public API for kub0.ai |

The public path chains through the [existing Cloudflare Tunnel](https://jkubo.com/posts/the-16gb-panini/): Cloudflare edge → cloudflared → Traefik → `strip-cctv` middleware → cctv-api. No new tunnel config needed — just a new Traefik Ingress rule.

### API Endpoints

| Path | Response |
|------|----------|
| `GET /` | HTML dashboard (browser) or JSON stats (API client) |
| `GET /stats` | Camera counts, archive stats, uptime, stale count |
| `GET /cameras/austin` | Austin cameras with stale annotations |
| `GET /cameras/california` | All California cameras |
| `GET /cameras/california/{1-12}` | Filter by CalTrans district |
| `GET /image/{region}/{id}` | Proxy live camera image (CORS bypass) |
| `GET /metrics` | Prometheus metrics (port 9090) |

The root handler uses content negotiation: `Accept: text/html` gets the embedded dashboard, everything else gets JSON stats. One endpoint, two personalities.

### The Embedded API Explorer

The dashboard at `cctv.kub0.xyz` includes a Swagger-style interactive API explorer. Each endpoint is a collapsible row with a `GET` badge. Click to expand, optionally edit parameters (district number, camera ID), and hit "Send" — the response appears inline with syntax-highlighted JSON, status code, latency, and response size. The `/image` endpoint renders the camera JPEG directly in the browser.

No external Swagger UI dependency, no OpenAPI spec file, no build step. Just 80 lines of vanilla JS embedded in a Go string constant. The same pattern was added to the ADS-B dashboard for consistency across the cluster's internal tools.

## The Frontend: Clickable Everything

The kub0.ai VISINT module evolved from a static placeholder to a fully interactive camera explorer:

- **2 API calls instead of 14**: Fetches `/cameras/austin` and `/cameras/california` in parallel via `Promise.all`, groups districts client-side. The original made 12 sequential requests (one per CalTrans district).
- **Client-side caching**: Camera data stored in a JavaScript variable, refreshed every 5 minutes. Filter clicks use cached data — instant.
- **Clickable drill-down**: Click California → districts appear. Click a district → sample grid filters to that district. Click Austin → districts hide, Austin cameras shown. Click again to deselect. The "Clear Filter" button resets to the default 4+4 sample view.
- **Stale filtering**: Sample feeds exclude cameras with `stale: true`. No more grids full of "TEMPORARILY UNAVAILABLE" placeholders.

## Lessons Learned

1. **Python is for prototypes, Go is for production.** A 10MB scratch image that starts instantly and uses 32MB of RAM isn't premature optimization — it's table stakes for a resource-constrained cluster.

2. **Hash dedup saves more than storage.** The 391 stale cameras aren't just saving disk space — they're saving network bandwidth, S3 API calls, and CPU time. Every upload avoided is a round trip that didn't happen.

3. **Separate your HTTP clients.** One timeout doesn't fit all. User-facing requests need patience (30s). Background archival needs impatience (10s). Mixing them guarantees that one starves the other.

4. **Persist your state, even if it's small.** A 213KB JSON file saved to S3 on shutdown means the difference between "seamless restart" and "re-archive 4,158 cameras from scratch."

5. **Liveness probes and background workers are enemies.** If your background work saturates the network, the kubelet can't health-check your pod. Either relax the probes or throttle the workers. We did both.

6. **CalTrans District 7 (LA) will timeout.** Always. It's the busiest district with the most cameras and the slowest API. Budget for it.

7. **Content negotiation is underrated.** Serving HTML to browsers and JSON to API clients from the same endpoint eliminates an entire nginx sidecar.

## Current State

```
Cameras:      4,158 (788 Austin + 3,370 California)
Stale:        ~391 (dedup-identified offline/static cameras)
Archive rate: ~3,800 images/cycle, 12 cycles/hour
Dedup saving: ~112,000 skipped uploads/day
Storage:      ~25-35 GB/day effective (vs ~72 GB/day naive)
Cycle time:   ~2m 21s (was 7+ minutes)
Image size:   ~10MB (scratch container)
Memory:       ~64MB typical, 256MB limit
```

The cluster's peripheral vision is online. 4,158 cameras across two states, archived with hash dedup, served over a public API, and visualized on a dashboard that loads in under a second. The Python prototype served its purpose — and then, mercifully, it was retired to `_prototype/`.

*All sources are publicly accessible traffic cameras provided by municipal and state agencies.*

Next: teaching the cluster to actually *see* what these cameras are showing. But that's a story for another post.