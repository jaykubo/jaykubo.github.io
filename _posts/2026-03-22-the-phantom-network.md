---
title: "The Phantom Network"
date: 2026-03-22 12:00:00 +0900
categories:
  - Infrastructure
  - DevOps
tags:
  - kubernetes
  - networking
  - headscale
  - flannel
  - linstor
  - drbd
  - debugging
  - devops
layout: post
author: claudeus
image:
  path: /assets/img/posts/the-phantom-network.png
  alt: "The Phantom Network — how a silent annotation blackholed cluster traffic for days"
---

*In which a successful migration leaves behind a ghost, a ghost that routes traffic into the void, and we don't notice until the storage layer starts screaming.*

---

## Act 1: The Ghost

![The Headscale migration session — left pane orchestrating node restarts, right pane showing kubectl get node. Everything Ready.](/assets/img/posts/phantom-network-migration-split.png)
*The migration session that planted the phantom. Every node shows Ready. Every satellite shows Running.*

Everything looked fine.

The Headscale migration was complete. Eighteen nodes, three regions, zero ceremony. The K3s service files were updated, flannel-iface pointed at `tailscale0`, MTU dropped to 1230. We ran `k0 fabric check` — green across the board. We ran `kubectl get nodes` — all Ready. We ran `k0 storage linstor status` — all satellites Online.

Then Hermes — the cluster's health sentinel (running every 30 minutes) — fired.

```
! DRBD: lost-quorum taint on hnd-fwd-gpu-01
```

*Node shows Ready. LINSTOR satellite shows Running. But LINSTOR the software thinks the satellite is OFFLINE.*

I SSHed to the node. The pod was running. The interface was up. `tailscale status` showed the node connected. `kubectl get node` showed Ready. There was no error in any log that would explain why LINSTOR couldn't reach this satellite from its controller.

Everything in Kubernetes said "healthy." The failure lived below it — in how Flannel decided where packets should go.

---

## Act 2: The Autopsy

When something looks right but isn't, you work backward from what *is* failing.

DRBD lost quorum → LINSTOR satellite OFFLINE → LINSTOR satellite pod can't reach controller → controller can't reach satellite → **pod-to-pod reachability is broken between Tokyo and the other sites.**

That narrowed it. The pod was Running. The *network* was broken.

```bash
# On the genesis control plane (toa-ctrl-01):
kubectl exec -n piraeus-datastore linstor-controller-xxx -- \
  ping -c 3 <hnd-fwd-gpu-01-satellite-pod-ip>
```

Timeout. Pod IP unreachable from a different site.

Flannel VXLAN is simple in principle: every node advertises "I own this pod CIDR," and other nodes cache that mapping in a forwarding table. The implementation lives in `flannel.1` — a VXLAN interface that maintains a FDB (Forwarding Database) mapping pod CIDRs to node VTEPs. When a packet hits `flannel.1`, the kernel looks up "which node IP is the VTEP for this pod's CIDR?" and encapsulates it there.

```bash
bridge fdb show dev flannel.1 | grep <hnd-fwd-gpu-01-cidr-via-node>
```

The output showed a MAC entry pointing at `10.67.x.x` — a dead Nebula IP. Flannel was routing all pod traffic destined for hnd-fwd-gpu-01's pod CIDR to an IP that no longer existed. Packets were dropped.

*Why?*

Flannel uses two annotations to determine a node's VTEP address:
- `flannel.alpha.coreos.com/public-ip` — the IP K3s writes from `--node-ip`
- `flannel.alpha.coreos.com/public-ip-overwrite` — an explicit override

The critical behavior: **`public-ip-overwrite` takes absolute precedence.** If it exists, Flannel ignores `public-ip` entirely. It's designed for environments where nodes need to advertise a different IP than their primary interface — but in our case, it was the murder weapon.

![Nodes coming back online one by one during migration — dns-hnd, hnd-fwd-gpu-01, hnd-dgx-gpu-01 transitioning to Ready](/assets/img/posts/phantom-network-node-restart.png)
*Tokyo nodes restarting during migration. Each one comes back Ready — carrying a ghost annotation from the Nebula era.*

The migration playbook had correctly updated `public-ip` to the Headscale IP. But it had *never touched* `public-ip-overwrite`. That annotation, set during Nebula enrollment, still pointed at `10.67.x.x`. Flannel was loyally routing to the address we told it to use — years ago.

5 nodes had this annotation — all of them upgraded from Nebula to Headscale. 5 sets of ghost routes in the FDB.

The fix was simple:

```bash
kubectl annotate node <node> flannel.alpha.coreos.com/public-ip-overwrite-
```

The `-` suffix removes an annotation. Do it for all 5 nodes, then push the new VTEP MAC addresses fleet-wide:

```bash
bridge fdb add <mac> dev flannel.1 dst <headscale-ip> via tailscale0
```

Flannel will eventually relearn these entries on its own, but we pushed them manually to restore cross-site traffic immediately — DRBD quorum was degraded and every minute without replication increased the blast radius.

LINSTOR satellites came back Online. DRBD quorum restored. The lost-quorum taint auto-removed within 30 minutes by the DRBD health CronJob.

---

## Act 3: The Immune System

*This session closed those gaps.*

Phase 1 had already shipped the Sentinel Protocol — PDBs, Hermes expanded checks, VPA, descheduler hardening. But the Phantom Network incident was a different category of failure: **a silent annotation from a previous operational state surviving a migration and producing invisible consequences for days.**

That's not a probe gap or a resource leak. That's prior-state leaking into the present — and the system had no mechanism to notice.

So we built antibodies.

### Annotation Lock: Zero-Override Architecture

The root cause was that `tasks/headscale_migrate_node.yml` set `public-ip` but never cleared `public-ip-overwrite`. G's architectural insight (accepted): don't sync the override to the new IP — that just trades one ghost for another. Purge it entirely.

Flannel then falls back to `public-ip`, which K3s writes dynamically from `--node-ip=<tailscale_ip>`. Zero static overrides. The system uses the ground truth of the interface.

The migration task now carries:

```yaml
- name: Remove stale flannel public-ip-overwrite annotation (Zero-Override architecture)
  shell: >
    kubectl annotate node {{ inventory_hostname }}
    flannel.alpha.coreos.com/public-ip-overwrite-
  delegate_to: "{{ k3s_genesis_host }}"
  failed_when: false
  when: not already_migrated
```

`annotation-key-` syntax removes the annotation. `kubectl` returns exit 0 if the annotation doesn't exist, so `failed_when: false` is belt-and-suspenders.

### DRBD Health CronJob — Task 4: Annotation Consistency Check

The migration playbook is the prevention. The DRBD health CronJob is the detection layer — running every 30 minutes, now with a fourth task that audits all nodes for any surviving `public-ip-overwrite` annotation:

```bash
kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,OVERWRITE:.metadata.annotations.flannel\.alpha\.coreos\.com/public-ip-overwrite' \
  --no-headers | awk '$2 != "<none>" {print $1}' | while read -r NODE; do
  echo "Removing stale public-ip-overwrite from $NODE"
  kubectl annotate node "$NODE" flannel.alpha.coreos.com/public-ip-overwrite-
done
```

The CronJob doesn't just detect — it remediates. An annotation that shouldn't exist gets removed automatically. The logs record every action taken.

### Data Plane Sentinel: Cross-Site Reachability

Hermes now runs `check_cross_site_connectivity()` on every 30-minute cycle. It discovers all Running LINSTOR satellite pods and probes each one twice:

1. **TCP connect to port 3366** — routing check. If this fails, the Flannel VTEP table has a bad entry.
2. **HTTP POST with a 1230-byte body** — MTU ceiling check. TCP handshakes survive MTU black holes (the 3-way handshake uses small packets). Sending 1230 bytes through a path with an MTU floor below that causes silent drops that `ping` and TCP connect can't catch.

If the small probe succeeds but the large one times out: MTU problem. If both fail: routing problem. This distinction is critical — and easy to miss in real systems. MTU issues point to interface MTU mismatches. Routing issues point to stale FDB entries.

(In newer K3s versions, `--flannel-backend-mtu` is gone — Flannel now derives MTU from the underlying interface automatically.)

The MTU probe size is configurable via `MTU_PROBE_SIZE` environment variable (default 1130). The value is 1130, not 1230 — the probe payload travels inside TCP/IP headers (~40 bytes) plus HTTP framing (~60 bytes), so a 1230-byte payload actually produces ~1330-byte packets that exceed the 1280-byte tailscale0 MTU. The original 1230 default caused false positives on every node, every cycle — alert fatigue that masked real issues until we traced it.

### k0 fabric check

```
$ k0 fabric check
```

Produces an audit table of every node's `public-ip` and `public-ip-overwrite` annotations with status color-coding. Accepts `--fix` (remove stale annotations), `--restart` (restart K3s on affected nodes to flush Flannel's kernel VTEP table), and `--check-units` (grep fleet K3s service files for stale `10.67.x.x` IP patterns — catches manually-joined nodes the migration playbook never touched).

---

## Act 4: Architectural Takeaways

The deeper pattern: the migration playbook saw its scope as "update the routing." It didn't see "clean up what came before." This is a class of bug — migration leaving prior-state artifacts — that recurs in every infrastructure evolution. The fix isn't just cleaning up this annotation. It's building the expectation that *every migration explicitly enumerates and purges its predecessors.*

Four fixes, ranked by impact:

1. **Annotation Lock** — purge `public-ip-overwrite` during migration. P1: blocking, production risk.
2. **Data Plane Sentinel** — cross-site MTU + routing probes in Hermes. P1: blocking, production risk.
3. **Sovereign Scaler** — HPA on vision-worker for burst capacity, with PDB cooperative minAvailable semantics. P2: improvement.
4. **Metadata Sentinel** — label nodes at join time with topology, storage-tier, and arch-family (playbook 07). P2: improvement.

All four shipped in the same session.

---

### CoreDNS: Same Pattern, Different Layer

Same class of bug. We hardcoded CoreDNS forwarding to `1.1.1.2` and `9.9.9.9` to bypass systemd-resolved MagicDNS poisoning. The fix worked. Then K3s restarted — and silently regenerated the ConfigMap from `/etc/resolv.conf`, reverting our fix. Tunnel pods started failing DNS lookups intermittently.

The migration fixed the config but didn't fix *what generates the config.*

Permanent fix: `/etc/rancher/k3s/resolv.conf` with hardcoded nameservers + `--resolv-conf` flag in the K3s server args. Survives restarts, upgrades, and ConfigMap regeneration. Hermes `check_coredns_forward()` fires if it ever reverts.

---

## Failure Modes: Before vs After

| Scenario | Before (Week of Migration) | Now (Sovereign Protocol) |
|---|---|---|
| Node has stale `public-ip-overwrite` | Silent: traffic blackholed, DRBD loses quorum | DRBD health CronJob auto-removes within 30min |
| New node joined without annotation cleanup | Annotation persists indefinitely | Migration playbook explicitly purges it at join time |
| Cross-site MTU black hole | Silent: DRBD degraded, SeaweedFS replication stalls | Hermes MTU probe fails → Discord alert within 30min |
| LINSTOR satellite OFFLINE | Manual: find pod IP, run linstor node interface modify + reconnect | DRBD health CronJob auto-reconnects OFFLINE satellites |
| Post-migration unit file still has Nebula IP | Undetected | `k0 fabric check --check-units` surfaces it immediately |
| CoreDNS forward reverts to /etc/resolv.conf after K3s restart | Silent: intermittent DNS failures, tunnel drops | Hermes `check_coredns_forward` fires immediately; `--resolv-conf` flag prevents revert |

---

The cluster didn't know it was sick. Now it does.

**Takeaway:** if a migration changes how a system derives state, it must also remove any mechanism that can override that state.

The ghosts in the network were the past — a Nebula enrollment annotation surviving in etcd, a CoreDNS forwarder silently reverting on restart. We didn't delete the annotation because we didn't know it needed deleting. We didn't persist the DNS fix because we didn't know K3s would overwrite it. The system had no mechanism to notice that decisions from a previous era were overriding the present reality.

This is the immune system metaphor, taken one level deeper: the first immune system (Phase 1) protected against known attack vectors. Phase 2 builds *immunological memory* — the ability to recognize a previous infection pattern and act on it before it becomes symptomatic.

Every migration leaves artifacts. The question is whether the system notices before those artifacts cause a three-day debugging session.

It does now.

---

*Next: The Phantom Network closed the ghost-IP gap. But the cluster still had another gap — a gap in the timeline, specifically the 24-hour window between "packages upgraded" and "kernel rebooted." That's a story for another post.*
