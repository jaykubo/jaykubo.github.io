---
title: "Dead Reckoning: Querying the Archive"
date: 2026-02-26 00:00:00 +0900
categories:
  - Data
  - Platform
tags:
  - adsb
  - go
  - seaweedfs
  - s3
  - timeline
  - rate-limiting
  - cloudflare-workers
layout: post
image:
  path: /assets/img/posts/dead-reckoning.png
  alt: ADS-B Historical Timeline Scrubber on the Intelligence Map
---

17,280 snapshots per day. Every aircraft over three continents. Every altitude, heading, and squawk code, archived every thirty seconds to SeaweedFS, indefinitely, with no expiry policy. The pipeline from [Every Thirty Seconds](https://jkubo.com/posts/every-thirty-seconds/) had been running for weeks.

The archive was write-only.

A perfect memory with no recall. The data existed — structured neatly under `adsb-raw/{region}/{date}/{HH-MM-SS}_aircraft.json` — but there was no read path. No API. No way to ask: *what was over Tokyo at 14:00 yesterday?* The cluster remembered everything and could tell you nothing.

Dead reckoning: the old navigation technique for computing current position from a previously known point, using elapsed time and course. Not GPS. Not landmarks. Just math applied to history. The archive was the known position. The API would be the math.

## The Read Path

The archival pipeline had already decided the key format. Objects in SeaweedFS look like this:

```
adsb-raw/aus/2026-02-19/14-00-00_aircraft.json
adsb-raw/lax/2026-02-19/14-00-30_aircraft.json
adsb-raw/hnd/2026-02-19/14-01-00_aircraft.json
```

Bucket: `adsb-raw`. Keys: `{region}/{date}/{HH-MM-SS}_{file}.json`. The timestamp format is lexicographically sortable — `HH-MM-SS` orders chronologically without parsing. The archive is naturally indexed.

Two new endpoints. First, list available timestamps for a region and date:

```
GET /history/{region}/{date}
→ { "region": "aus", "date": "2026-02-19", "count": 2880, "timestamps": ["00-00-00", "00-00-30", ...] }
```

Second, fetch the raw snapshot at a specific time:

```
GET /history/{region}/{date}/{time}
→ raw dump1090-fa JSON, pass-through, no re-serialization
```

The history client is a thin wrapper around minio-go. `ListTimestamps` lists objects under `{region}/{date}/`, filters keys that end in `_aircraft.json`, strips the filename to extract the timestamp, and returns a sorted slice. `GetSnapshot` fetches the exact key with a 15-second context timeout — SeaweedFS is in-cluster and fast, but S3 fetches shouldn't block indefinitely.

One critical detail, which also destroyed thirty minutes when wiring up the [CCTV archival pipeline](https://jkubo.com/posts/the-all-seeing-eye/): minio-go wants `host:port`, not `http://host:port`. Feed it a URL with a scheme prefix and it silently routes through TLS negotiation and returns connection refused. The error message does not mention the scheme. The endpoint for SeaweedFS is `seaweedfs-filer.seaweedfs.svc.cluster.local:8333`, no prefix. Every time this happens, twenty minutes disappear.

Caching: listing objects under a prefix is a SeaweedFS round-trip. Do it on every request and the scrubber is unusably slow. Today's listings are volatile — a 14:00 timestamp list is stale by 14:01. Past dates are immutable forever. Two TTLs: one minute for today, one hour for any prior date. One `map[string]cachedList` keyed by `"{region}/{date}"`, with an expiry field per entry. No Redis, no external dependencies. One pod, one process, one map.

## The Scrubber

The intel map at [kub0.ai/overwatch](https://kub0.ai/overwatch) had always been live — a snapshot of now, refreshing every five seconds. History mode changes that.

A HISTORY button in the layers panel. Click it and the live refresh stops. A scrubber bar slides up from the bottom of the viewport: date picker on the left, range slider across the center, a timestamp label, step-backward and step-forward buttons flanking the slider, play/pause, and a LIVE button to return.

Loading a date fetches all three regions in parallel:

```javascript
async function loadHistoryDate() {
    const date = document.getElementById('scrub-date').value;
    const [aus, lax, hnd] = await Promise.all([
        fetch(`${FLIGHTS_API}/history/aus/${date}`).then(r => r.json()),
        fetch(`${FLIGHTS_API}/history/lax/${date}`).then(r => r.json()),
        fetch(`${FLIGHTS_API}/history/hnd/${date}`).then(r => r.json()),
    ]);
    const all = [...(aus.timestamps||[]), ...(lax.timestamps||[]), ...(hnd.timestamps||[])];
    histMaster = [...new Set(all)].sort();
    document.getElementById('scrub-slider').max = histMaster.length - 1;
}
```

Three parallel fetches produce three timestamp arrays. Merge, deduplicate, sort. The slider ranges over the full union timeline — every 30-second tick across all three regions from midnight to the current moment.

Drag the slider and the map fetches snapshots from all three regions for the nearest available timestamp. ADS-B feeders in three countries don't share a clock — Austin and LA are on the same CronJob cycle; Tokyo runs its own, offset by a few seconds. Binary search finds the closest match per region:

```javascript
function nearestTs(arr, target) {
    let lo = 0, hi = arr.length - 1;
    while (lo < hi) {
        const mid = (lo + hi + 1) >> 1;
        if (arr[mid] <= target) lo = mid; else hi = mid - 1;
    }
    return lo >= 0 ? arr[lo] : null;
}
```

For each slider position, three parallel S3 fetches retrieve the nearest snapshot per region. The responses merge by ICAO hex code — the same deduplication logic as the live layer. A 150ms debounce prevents flooding S3 while dragging. The play button advances one step every two seconds.

### The First Test

Click HISTORY. The live aircraft disappeared. Every registration, every type code, every enrichment field the aggregator had accumulated over hours of operation — gone. The map showed raw feeder data only: hex codes and positions, callsigns where the radio heard them. The aircraft became anonymous.

History snapshots are raw dump1090-fa JSON. They contain what the antenna heard at that instant. They don't contain the `r` (registration) and `t` (type) fields that the aggregator accumulates by cross-referencing airplanes.live's global database over time. An archive from 14:00 is a photograph. The live layer is a map that has been annotated over time.

The bug was one line in `loadHistoryDate()`:

```javascript
scrubToIndex(histMaster.length - 1);  // auto-jump to latest snapshot
```

The moment you entered history mode, it automatically fetched the latest available snapshot and replaced the live enriched data with unenriched archive data. Not intentional. Just the natural consequence of "arm the scrubber, and also go to the most recent position."

The fix: arm the slider to the correct range, don't fetch. Keep the live aircraft on screen until the user explicitly drags. The scrubber is ready; the map is unchanged. The user initiates the first seek.

Forty minutes of confusion. One line removed.

## RJTT, Again

The rate limiting bug from [The Intelligence Map](https://jkubo.com/posts/the-overwatch-map/) was back. Or rather — it had never left. The fix had been committed. The Docker image had not been rebuilt.

The new pod started to deploy the history endpoint pulled the old image. The retry loop existed in the repository and nowhere else.

The symptom was the same: Tokyo coverage with hex codes and callsigns but no registration, no type code. Austin: enriched. LA: enriched. Tokyo: 429, 429, 429 in the logs.

### The Sleep That Wasn't

The specific bug was one line. In `fetchGlobal()`, the sleep was placed after success, not after failure:

```go
list, err := a.fetchGlobalPoint(url)
if err != nil {
    slog.Warn("airplanes.live fetch failed", "url", url, "error", err)
    continue   // ← skips the sleep
}
// ...process list...
time.Sleep(5 * time.Second)  // ← only reached on success
```

LA 429 → `continue` → Tokyo fires immediately. Zero delay. Every cycle repeated the burst before any proxy's rate window could clear. The fix:

```go
if err != nil {
    slog.Warn("airplanes.live fetch failed", "url", url, "error", err)
    time.Sleep(15 * time.Second)  // sleep even on failure
    continue
}
```

The previously undeployed retry loop also got built into the image this time. Sleep on every exit path, not just success. A commit is not a deployment — the retry loop was correct, reviewed, and committed for the entire lifetime of the previous pod. It ran on zero replicas. Build the image; deploy the image. Both steps have to happen.

---

*17,280 snapshots per day. The cluster has been writing them since the antenna went up. Now it can read them.*

*Drag the scrubber to any point in the past and the coverage rings fill with whatever was overhead at that moment. The overnight cargo run from LAX. The morning rush into HND. The rare hour when AUS had nothing but a Cessna holding at 3,500 feet. All of it, retrievable, queryable, scrollable.*

*The cluster developed a long-term memory. The intelligence map learned to look backward.
