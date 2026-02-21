---
title: "The All-Seeing Eye: 4,158 Traffic Cameras and the Art of Not Storing Garbage"
date: 2026-02-18 14:00:00 +0900
categories:
  - Data
  - Infrastructure
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
  - youtube
  - leaflet
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
| `GET /search?lat=X&lon=Y&radius=R` | Cameras near a point — R-tree + Haversine, sorted by distance |
| `GET /image/{region}/{id}` | Proxy live camera image (CORS bypass) |
| `GET /livecams` | All 33 livecam cameras with resolved YouTube video IDs |
| `GET /livecams/{region}` | Filter by region (kanto, kansai, chubu, etc.) |
| `GET /metrics` | Prometheus metrics (port 9090) |

The root handler uses content negotiation: `Accept: text/html` gets the embedded dashboard, everything else gets JSON stats. One endpoint, two personalities.

### The Embedded API Explorer

The dashboard at `cctv.kub0.xyz` includes a Swagger-style interactive API explorer. Each endpoint is a collapsible row with a `GET` badge. Click to expand, optionally edit parameters (district number, camera ID), and hit "Send" — the response appears inline with syntax-highlighted JSON, status code, latency, and response size. The `/image` endpoint renders the camera JPEG directly in the browser.

No external Swagger UI dependency, no OpenAPI spec file, no build step. Just 80 lines of vanilla JS embedded in a Go string constant. The same pattern was added to the ADS-B dashboard for consistency across the cluster's internal tools.

## Point the Eye: Spatial Search

The natural question after "I have 4,158 cameras" is "show me the ones *near me*."

A flat endpoint like `/cameras/california/7` returns everything in LA County — 847 cameras. Useful for analytics, hostile to a map where you've zoomed into a six-block radius. What the intelligence map actually needs is `/search?lat=34.05&lon=-118.24&radius=2` — cameras within 2km of downtown LA, sorted by ascending distance.

The implementation is a textbook R-tree, specifically `github.com/tidwall/rtree`'s generic variant (`RTreeG[Camera]`). Each camera is inserted as a degenerate rectangle — min and max are the same point, `[lon, lat]`. The tree organizes these in a balanced hierarchy so range queries run in O(log n + k) instead of O(n).

```go
// Camera stored as a point (min == max == the camera's coordinates)
pt := [2]float64{c.Lon, c.Lat}
idx.tree.Insert(pt, pt, c)
```

The search is two-stage:

1. **Bounding box prefilter.** Convert the radius to a lat/lon box (Δlat = R/111, Δlon = R/(111·cos(lat))). The R-tree returns all cameras within the box — fast, approximate, overcounts the corners of the square.

2. **Haversine filter.** For each candidate, compute the exact great-circle distance. Reject cameras that fall inside the box's corners but outside the actual circle.

```go
a := math.Sin(dLat/2)*math.Sin(dLat/2) +
    math.Cos(lat1r)*math.Cos(lat2r)*math.Sin(dLon/2)*math.Sin(dLon/2)
dist := 6371.0 * 2 * math.Atan2(math.Sqrt(a), math.Sqrt(1-a))
```

At 5km radius near the equator, the bounding box overcounts by up to ~21%. At 4,158 cameras that's microseconds of wasted work. The Haversine pass keeps results exact regardless.

The index is rebuilt after every scrape cycle — both the Austin and California tickers call `buildSpatialIndex(s.austin, s.california)` under the write lock and swap the pointer atomically. Rebuild time at 4k cameras is unmeasurably fast. The search itself runs lock-free: read the index pointer and stale set under the read lock, release, then query.

The response includes `distance_km` per camera (rounded to the meter), full camera fields, and stale annotations:

```json
{
  "cameras": [
    { "camera_id": "D7-1234", "region": "california", "lat": 34.052, "lon": -118.241,
      "distance_km": 0.182, "stale": false },
    ...
  ],
  "count": 14,
  "lat": 34.05,
  "lon": -118.24,
  "radius_km": 2
}
```

Parameters: `lat` and `lon` (required), `radius` in km (0–50, default 1.0), `limit` (1–500, default 50), and an optional `region` filter for when you only want Austin or California results.

## The Frontend: Clickable Everything

The kub0.ai VISINT module evolved from a static placeholder to a fully interactive camera explorer:

- **2 API calls instead of 14**: Fetches `/cameras/austin` and `/cameras/california` in parallel via `Promise.all`, groups districts client-side. The original made 12 sequential requests (one per CalTrans district).
- **Client-side caching**: Camera data stored in a JavaScript variable, refreshed every 5 minutes. Filter clicks use cached data — instant.
- **Clickable drill-down**: Click California → districts appear. Click a district → sample grid filters to that district. Click Austin → districts hide, Austin cameras shown. Click again to deselect. The "Clear Filter" button resets to the default 4+4 sample view.
- **Stale filtering**: Sample feeds exclude cameras with `stale: true`. No more grids full of "TEMPORARILY UNAVAILABLE" placeholders.

## The Livecam Extension

The traffic camera system archives government infrastructure. What it couldn't do was show you a city live. Adding 33 YouTube livestream cameras — Shibuya Scramble, Mount Fuji at dawn, Dotonbori at midnight — required solving three separate YouTube problems inside the same binary.

**The embed problem.** YouTube's `live_stream?channel=CHANNEL_ID` format is intermittently broken — some channels show archived videos, some show nothing. The fix: resolve video IDs server-side and embed with `youtube.com/embed/{video_id}`.

**The resolution problem.** HTML scraping fails from data center IPs — YouTube serves a JavaScript shell with a `canonical` tag whose `href` is literally `"undefined"`. The oEmbed API doesn't handle live channels. The YouTube Data API works but requires API keys and quota management for 33 requests every 10 minutes.

What actually works: RSS feeds.

```
https://www.youtube.com/feeds/videos.xml?channel_id={id}
```

No authentication. No quota. Works from every IP on earth, including Raspberry Pis behind Tailscale. The first `<yt:videoId>` in the feed is the most recent upload — for 24/7 livestream channels, that's the live stream. One regex, 33 out of 33 resolved.

```go
var rssVideoIDRe = regexp.MustCompile(`<yt:videoId>([a-zA-Z0-9_-]{11})</yt:videoId>`)
```

**The embed restriction problem.** With video IDs resolved, the standard `youtube.com/embed` player threw Error 153 — channel owner has restricted embedding. Half the Japanese livestream channels have it disabled for `youtube.com`. The fix is one word: `youtube-nocookie.com`. YouTube's privacy-enhanced mode domain bypasses the restriction. Every channel that blocked `youtube.com/embed` allows `youtube-nocookie.com/embed`.

The resolver runs as a background goroutine every 10 minutes with a dedicated HTTP client. The `internal/livecam` package has zero imports from the camera scraper, storage, or proxy code — architecturally decoupled but operationally bundled in the same binary.

### The 33 Cameras

| Region | Count | Highlights |
|--------|-------|------------|
| Kanto | 13 | Shibuya Scramble, Shinjuku, Odaiba, Yokohama Bay Bridge |
| Chubu | 7 | Mount Fuji (multiple angles), Nagoya expressway, Kanazawa |
| Kansai | 6 | Dotonbori, Kobe Port Tower, Nara deer park |
| Kyushu | 2 | Fukuoka, Kumamoto |
| Hokkaido | 2 | Sapporo, Otaru Canal |
| Shikoku | 1 | Matsuyama |
| Chugoku | 1 | Hiroshima Peace Memorial |
| Tohoku | 1 | Sendai |

Each entry carries the channel ID (for RSS resolution), a fallback video ID, bilingual names, coordinates, and region tags. The resolver overwrites the fallback on every cycle — if RSS fails, the previous known-good ID persists.

Click a pink marker on the map. A Leaflet popup opens with the camera name and a YouTube thumbnail with a play button overlay. Click the thumbnail — the `<img>` swaps to an `<iframe>` with autoplay via `youtube-nocookie.com`. Shibuya crossing at 2am, inline, live.

## Lessons Learned

1. **Python is for prototypes, Go is for production.** A 10MB scratch image that starts instantly and uses 32MB of RAM isn't premature optimization — it's table stakes for a resource-constrained cluster.

2. **Hash dedup saves more than storage.** The 391 stale cameras aren't just saving disk space — they're saving network bandwidth, S3 API calls, and CPU time. Every upload avoided is a round trip that didn't happen.

3. **Separate your HTTP clients.** One timeout doesn't fit all. User-facing requests need patience (30s). Background archival needs impatience (10s). Mixing them guarantees that one starves the other.

4. **Persist your state, even if it's small.** A 213KB JSON file saved to S3 on shutdown means the difference between "seamless restart" and "re-archive 4,158 cameras from scratch."

5. **Liveness probes and background workers are enemies.** If your background work saturates the network, the kubelet can't health-check your pod. Either relax the probes or throttle the workers. We did both.

6. **CalTrans District 7 (LA) will timeout.** Always. It's the busiest district with the most cameras and the slowest API. Budget for it.

7. **Content negotiation is underrated.** Serving HTML to browsers and JSON to API clients from the same endpoint eliminates an entire nginx sidecar.

8. **RSS feeds are YouTube's most reliable public API.** No auth, no quota, no rate limits at 33 requests per 10 minutes. For 24/7 livestream channels, the feed's first entry is always the live stream.

9. **`youtube-nocookie.com` bypasses embed restrictions.** Many channels that block `youtube.com/embed` allow the privacy-enhanced domain. One subdomain difference, all 33 cameras working.

10. **R-trees are exactly the right tool for proximity queries.** Bounding-box prefilter via the tree, exact Haversine for the corners — two passes, lock-free after pointer read. Rebuilding the full index on each scrape costs nothing at 4k entries and eliminates any stale-state bugs from incremental updates.

## Current State

```
Cameras:      4,158 (788 Austin + 3,370 California)
Livecams:     33 YouTube livestreams across 8 regions of Japan
Stale:        ~391 (dedup-identified offline/static cameras)
Archive rate: ~3,800 images/cycle, 12 cycles/hour
Dedup saving: ~112,000 skipped uploads/day
Storage:      ~25-35 GB/day effective (vs ~72 GB/day naive)
Cycle time:   ~2m 21s (was 7+ minutes)
Image size:   ~10MB (scratch container)
Memory:       ~64MB typical, 256MB limit
Spatial index: R-tree, 4,158 points, rebuilt each scrape cycle
Search params: radius 0–50km, limit 1–500, optional region filter
```

The cluster's peripheral vision is online. 4,158 traffic cameras archived, 33 cities streaming live — government infrastructure and YouTube, unified in the same binary.

*All sources are publicly accessible traffic cameras provided by municipal and state agencies. YouTube livestreams are embedded via the public RSS feed API.*