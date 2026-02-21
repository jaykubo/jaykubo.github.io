---
title: "Every Thirty Seconds: Teaching a Cluster to Listen to the Sky"
date: 2026-02-19 14:00:00 +0900
categories:
  - Data
  - Infrastructure
tags:
  - adsb
  - k3s
  - raspberry-pi
  - cloudflare-tunnel
  - traefik
  - tailscale
  - data-pipelines
  - seaweedfs
layout: post
image:
  path: /assets/img/posts/every-thirty-seconds.png
  alt: ADS-B Flight Data Pipeline Across a Global K3s Cluster
---

1090 MHz. The frequency every commercial aircraft broadcasts on, twenty-four hours a day, unencrypted, to anyone listening. Position, altitude, speed, callsign — all of it, free, in the clear. A $30 SDR dongle and a window-mounted antenna is all it takes to hear them.

Three antennas. Three regions. Three deployment models. None of them inside Kubernetes.

The cluster had a skeleton ([LINSTOR](https://jkubo.com/posts/the-47tb-exorcism/)), eyes ([Mimir + Grafana](https://jkubo.com/posts/prometheus-unbound/)), and hardened skin ([Tailscale + Technitium](https://jkubo.com/posts/the-16gb-panini/)). But every metric described the cluster's own heartbeat — CPU temperatures, pod restarts, DRBD sync rates. A hall of mirrors. Time to give this thing ears.

## The Antenna Farm

| Region | Node | Hardware | Setup |
|--------|------|----------|-------|
| Austin | aus-node-01 | RPi Zero 2W | Bare metal — dump1090-fa + lighttpd |
| Los Angeles | lax-node-01 | RPi Zero 2W | Bare metal — dump1090-fa + lighttpd |
| Tokyo | hnd-fwd-gpu-01 | Framework Desktop | Docker Compose — piaware + readsb |

ADS-B — Automatic Dependent Surveillance-Broadcast. Every commercial aircraft announces itself constantly: position, altitude, heading, speed, squawk code. `dump1090-fa` decodes the radio signals into JSON. [ADSBexchange](https://adsbexchange.com) aggregates feeds from receivers worldwide — unfiltered, unlike FlightAware's sanitized view — into the tracking maps everyone uses.

The Austin and LA feeders are RPi Zero 2Ws. 512MB of RAM. No container runtime, no orchestration, just a service that decodes radio signals and a web server that exposes JSON. Tokyo runs as a Docker Compose stack on a GPU node — the same machine that contributes 8TB of NVMe to the [storage pool](https://jkubo.com/posts/the-47tb-exorcism/) and runs K3s worker pods. SDR dongle passed through USB, data served alongside everything else.

## Bridging the Gap

Kubernetes has a pattern for absorbing external infrastructure: headless Services with manual Endpoints. No ClusterIP, no load balancing — just DNS that resolves to a Tailscale IP you hand it.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: adsb-aus
  namespace: networking
spec:
  clusterIP: None
  ports:
    - port: 8080
---
apiVersion: v1
kind: Endpoints
metadata:
  name: adsb-aus
  namespace: networking
subsets:
  - addresses:
      - ip: 100.67.12.50   # Tailscale IP
    ports:
      - port: 8080
```

Three Services, three Endpoints. Now any pod in the cluster can reach `adsb-aus.networking.svc.cluster.local:8080` and get live flight data from a Raspberry Pi in Austin. The feeders don't know they're part of Kubernetes. They just serve JSON.

## The Triple 502

Applied the Services. Created DNS records with `dns-manage.sh`. Applied Traefik Ingress with TLS via cert-manager. Hit the URLs.

502 Bad Gateway. All three regions.

Three feeders, three different reasons for silence.

**Austin and LA**: `lighttpd` — the web server that exposes dump1090-fa's JSON — was stopped. Disabled months ago to save resources on 512MB nodes. The feeders were decoding radio just fine, faithfully uploading to ADSBexchange, but the local HTTP interface was dark. `systemctl start lighttpd && systemctl enable lighttpd` on each RPi.

**Tokyo**: The piaware Docker container had port 80 not published to the host. Running, decoding, feeding — but unreachable from outside Docker's network. One line in `docker-compose.yml`: `"8080:80"`. Recreate. Tokyo came alive.

Three 502s. Three different root causes. Zero overlap. The kind of debugging that makes you question whether the universe is adversarial or just indifferent.

The original plan to reverse-proxy SkyAware's web UI died on contact — absolute asset paths like `/style.css` bypass StripPrefix entirely. ADSBexchange globe feeds replaced it. Sometimes the best UI is someone else's.

## The Path That Didn't Exist

The API design was path-based routing under a single domain:

```
adsb.kub0.xyz/{region}/data/aircraft.json
```

Traefik StripPrefix middlewares remove the region prefix. `/aus/data/aircraft.json` becomes `/data/aircraft.json` at the backend. Austin and LA serve data there natively.

Then Tokyo returned 404.

piaware wraps dump1090-fa and serves data at `/dump1090-fa/data/`, not `/data/`. Strip `/hnd`, and the request hits a path that doesn't exist. The fix is a middleware chain that adds back what piaware expects:

```
Austin/LA:  StripPrefix(/aus) ─────────────────────────> backend (/data/...)
Tokyo:      StripPrefix(/hnd) → AddPrefix(/dump1090-fa) → backend (/dump1090-fa/data/...)
```

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: prefix-hnd
  namespace: networking
spec:
  addPrefix:
    prefix: /dump1090-fa
```

Three ingress resources, each with its own middleware chain. The API consumer sees a uniform interface. The plumbing behind it is anything but.

## The Archival Pipeline

Live data is useful. Historical data is valuable.

Every 30 seconds, a CronJob reaches into all three feeders and archives raw JSON to [SeaweedFS](https://jkubo.com/posts/the-47tb-exorcism/). The schedule is `* * * * *` — every minute — but the job runs two collection cycles with `sleep 30` between them. Kubernetes CronJobs can't go sub-minute, so you do it yourself.

```bash
collect_and_upload() {
  REGION="$1"; BASE="$2"; PREFIX="$3"
  TS=$(date -u '+%Y-%m-%d/%H-%M-%S')
  for FILE in aircraft stats; do
    BODY=$(curl -sf --max-time 8 "${BASE}${PREFIX}${FILE}.json") || continue
    printf '%s' "${BODY}" | curl -sf --max-time 10 \
      --aws-sigv4 "aws:amz:us-east-1:s3" \
      --user "${KEY}:${SECRET}" \
      -X PUT --data-binary @- \
      "${S3}/adsb-raw/${REGION}/${TS}_${FILE}.json"
  done
}

collect_all    # :00
sleep 30
collect_all    # :30
```

`curl --aws-sigv4` signs requests with S3 SigV4 authentication. SeaweedFS accepts it happily — the region string is ignored. No AWS CLI, no SDK, just curl in a 5MB container image.

Six files per cycle, two cycles per minute, 17,280 JSON documents per day. Each `aircraft.json` is a snapshot of every aircraft in range. The `stats.json` tracks feeder health: messages per second, signal strength, uptime.

```
adsb-raw/
  aus/2026-02-19/
    00-00-00_aircraft.json
    00-00-30_aircraft.json
    ...
  lax/2026-02-19/...
  hnd/2026-02-19/...
```

The CronJob runs with `concurrencyPolicy: Forbid` and `activeDeadlineSeconds: 85` — stalled jobs die before the next minute ticks. `backoffLimit: 0` means a failed collection is a missed sample, not an infinite retry loop. In time-series data, a late write is worse than a gap.

The RPi Zero 2Ws also get their `dump1090-fa` stopped during `apt upgrade` — 512MB of RAM leaves no margin for both. A few minutes of missed aircraft data beats a kernel OOM that takes the whole node down.

## The Tunnel

The whole cluster lives behind Tailscale. Every domain under `kub0.xyz` resolves to a Tailscale IP. No NAT traversal, no port forwarding, no public exposure. This is the point.

But [kub0.ai](https://kub0.ai) — a Cloudflare Workers site visible to the entire internet — was fetching from endpoints that the entire internet cannot reach. Dashboard loaded fine. Data fields showed `--`. Everyone on Tailscale saw aircraft. Everyone else saw nothing.

The solution: [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/). A `cloudflared` daemon inside the cluster establishes an outbound connection to Cloudflare's edge. Public traffic enters through Cloudflare, rides the tunnel into the cluster, and exits at Traefik. No inbound ports opened. No Tailscale exposure. The private network stays private.

Split-horizon DNS crystallized: `kub0.xyz` stays internal, Technitium resolving to Tailscale IPs. A new public endpoint at `api.kub0.io` serves the same data through the tunnel. Cloudflare DNS resolves it to the tunnel. Same data, two paths, zero overlap.

```
api.kub0.io/adsb/aus/data/aircraft.json
  → Cloudflare edge
  → tunnel → cloudflared pod
  → Traefik → strip-adsb → strip-aus → adsb-aus backend
  → RPi Zero 2W in Austin
```

The path transformation chain reuses every existing middleware. `strip-adsb` peels the `/adsb` prefix, then the existing region strips and the HND `addPrefix` do the rest. Four new Ingress resources for the `api.kub0.io` host, zero changes to existing routing.

CORS was the last gate. `kub0.ai` fetches from `api.kub0.io` — different origin, blocked by default. One Traefik middleware with `accessControlAllowOriginList` for `kub0.ai` and `www.kub0.ai`, `GET` and `OPTIONS` only, 24-hour `maxAge`. The full signal path: radio waves → SDR dongle → dump1090-fa → Tailscale → headless Service → Traefik → Cloudflare Tunnel → Cloudflare edge → browser. From antenna to pixel, across two countries, in under 200ms.

### QUIC vs. WireGuard

Deployed two `cloudflared` replicas. Both failed to connect.

```
ERR Failed to dial a quic connection error="failed to dial to edge
    with quic: timeout: no recent network activity"
```

`cloudflared` defaults to QUIC — UDP on port 7844. Efficient, multiplexed, designed for exactly this use case. One problem: the cluster runs over Tailscale, which is WireGuard, which is also UDP. Every QUIC packet gets encapsulated inside a WireGuard UDP packet. Double UDP encapsulation, reduced MTU, fragmentation at every hop. The packets never make it to Cloudflare's edge.

The fix took one flag:

```yaml
args:
  - tunnel
  - --no-autoupdate
  - --protocol=http2
  - run
  - --token=$(TUNNEL_TOKEN)
```

HTTP/2 over TCP. WireGuard handles TCP fine — the encapsulation overhead is predictable and the kernel manages fragmentation properly. Both replicas connected to Cloudflare's NRT datacenter within seconds.

Thirty seconds from "everything's broken" to "everything works." The kind of fix that feels obvious in hindsight but requires understanding two layers of UDP encapsulation to diagnose.

## Lessons Learned

1. **Headless Services absorb anything.** If it speaks HTTP and has an IP, Kubernetes can adopt it. No agents, no sidecars, no awareness required from the external service.

2. **Path normalization is never uniform.** piaware serves at `/dump1090-fa/data/`, not `/data/`. When you aggregate heterogeneous backends behind a uniform API, middleware chains are your translator.

3. **Sub-minute CronJobs are a `sleep` away.** Kubernetes floors you at one-minute resolution. Two cycles with `sleep 30` in between gets you to 30-second granularity. Ugly. Works.

4. **`curl --aws-sigv4` replaces the AWS SDK.** S3-compatible storage doesn't need an SDK. One flag on a 5MB container image does the same work as a 200MB Python image with boto3.

5. **QUIC dies over WireGuard.** UDP-in-UDP encapsulation causes fragmentation and silent packet drops. Force HTTP/2 (`--protocol=http2`) when running Cloudflare Tunnel over Tailscale or any WireGuard network.

6. **Split-horizon DNS separates concerns.** Same backend, two DNS paths: internal via Tailscale (`kub0.xyz`), public via Cloudflare Tunnel (`api.kub0.io`). Neither knows about the other.

7. **Protect your weak nodes.** 512MB machines can't survive `apt upgrade` and `dump1090-fa` simultaneously. Stop the service first, upgrade, restart. A few minutes of missed data beats a kernel OOM.

8. **Absolute asset paths kill reverse proxies.** SkyAware's hard-coded `/style.css` and `/flags-tiny/` requests bypass StripPrefix entirely. If the frontend wasn't built for path-prefix hosting, don't fight it — find an alternative.

---

*The body that was built to watch itself now listens to the sky. Three antennas, three regions, 17,280 snapshots a day — archived, queryable, streaming live to anyone who asks.*

*Next: the cluster already had [eyes on 4,158 cameras](https://jkubo.com/posts/the-all-seeing-eye/). Now aircraft and cameras converge on a single map. The peripheral vision gets interesting.*
