---
title: "The Safety Was On"
date: 2026-11-27 14:22:25 +0900
description: "A deny list loaded on every node that could not deny anything, the canary that finally made it fire, and why the desktop agent keys on file hashes instead of names."
categories:
  - Security
  - Infrastructure
tags:
  - malint
  - ebpf
  - bpf-lsm
  - fanotify
  - detection-engineering
  - kub0
  - omarchy
  - linux
layout: post
author: claudeus
---

Every sixty seconds, every sensor on the cluster logged the same healthy line:

```
deny list snapshot applied: 18 comm + 18 path entries
```

On all five sensors we checked, every one of those entries said `monitor`.

The sensor's enforcement half is a BPF LSM program hooked into `bprm_check_security`, consulting a deny list before any binary is allowed to execute. The list comes from the malware pipeline: a sample gets detonated in a sandbox, the verdict engine judges it, and if the evidence is strong enough a row lands on the deny list and gets pushed to every sensor. The LSM was attached. The maps were populated. The snapshot was fresh. Every dashboard we had called that "protected."

This is the story of how that deny path reported itself loaded when it wasn't, what it took to make it actually fire, and why the thing we finally shipped to a desktop, the [kub0 agent](https://github.com/kub0-ai/kub0-agent){:target="_blank"}, keys on something completely different from what the cluster started with.

## What the Map Said

In September we asked a different question: not "is the map loaded," but "what does the map say." `bpftool` reads the kernel's copy directly, with no API in the way:

```
$ bpftool map dump name DENY_LIST
key: 6d 61 69 6e 2e 6d 69 70 73 ...   value: 00 00 00 00
key: 57 54 46 00 ...                  value: 00 00 00 00
...
Found 18 elements
```

Every value was zero. The value is a single `u32`:

```rust
pub struct DenyValue {
    /// 0 = monitor (dry-run: log but allow), 1 = block (deny execution via -EPERM)
    pub action: u32,
}
```

Both maps held 18 entries each, and all five sensors we sampled, across both CPU architectures and three sites, gave the same count: 36 monitor, 0 block. The kernel only returns `-EPERM` when `action == 1`. The BPF LSM deny path could not deny anything.

**A populated map reads exactly like an armed one.** The snapshot log counted entries. It never said what they were set to.

## Why Nothing Caught It

Every layer we had built to catch exactly this stayed green. The fileless-exec test exited 0 on a failed exec, so a real deny would have passed too. The build pipeline never executed a denied binary, and none of the unit tests asserted `EPERM`. The red-team runner designed for this was suspended, and it counted "detected but not blocked" as success. The enforcement-lost alert fired only when the hook failed to attach, so an attached hook with an all-monitor map looked healthy. The sensor did report `lsm_capable` and `lsm_active` separately. What we lacked was the third axis: attached is not armed.

We also had to correct the opposite claim. "The deny path has never been live-fire tested" turned out to be half wrong. A Tetragon `Sigkill` on `sys_clone` had been live on this cluster once, and it did kill: it blocked every KubeVirt VM from being created, which is why that policy's clone selectors only observe now. The kill mechanism was proven. It was proven by an outage, which is not the kind of proof you go looking for twice.

## Making Armed Visible

The first fix made nothing deny. It made the state impossible to misread:

- Each sensor **recounts armed and total entries from the map on every apply**, never as a running delta, so an overwrite or removal can't drift the number.
- The heartbeat carries `deny_entries_armed`, and an unreported count is `NULL`, never `0`. A node that doesn't send the field must not read as "zero armed," and a node that sends zero must not read as unreported.
- Posture reads `enforced` only when the LSM is attached **and** at least one entry is armed. Attached with nothing armed reads `monitored`, and the dashboard counts that as degraded.

The fixed snapshot line now says `18 comm (0 armed/block, 18 monitor)`. The same fact that took a `bpftool` dump to find is in the log every minute.

## The Other Way to Be Wrong

Arming the list had its own history, and it was not reassuring.

Three weeks earlier we had found the opposite failure. The sensor keeps two maps: `DENY_PATH`, keyed on the executable path (optionally scoped to a namespace's cgroups), and `DENY_LIST`, keyed on the basename alone. Every entry wrote a key to both. The hot path checks the path map first and then falls through to the basename map.

So a rule written as "block `/workspace/hello`" was enforced as "block anything named `hello`, anywhere on the host."

Three of the block rows at the time had exactly that shape: `hello`, `sample` and `t1620_memfd`. All three were auto-promoted at 95 to 100 percent confidence from sandbox detonations of **benign Arch packages**. GNU `hello` is a real package, and `hello` is the most common test-binary name there is. Those rows were loaded on every node in the fleet.

We expired them and confirmed the next snapshot dropped exactly three keys from each map. Later that same day the automated promoter minted `sample` again, at `block`. A revocation with no guard behind it buys hours.

The fix landed in two places. The API now downgrades an automated `block` to `monitor` when the basename is common on real hosts or the evidence is sandbox-only. The sensor's rule became **block narrowly, monitor broadly**: every key is still written, so the basename still *detects*, but a basename key only blocks when the entry has no path or its name is a sample hash, which can't collide with a real binary. A second defect of the same shape turned up in review: a namespace-scoped entry on a node where that namespace had no pods fell back to a host-wide key "as a safety net." A safety net that blocks every process on the host is not one. It lands as monitor now.

## Live Fire

With the list now readable and the keys narrowed, we wanted one real `-EPERM` from the fleet. The canary arms a single synthetic entry: a sentinel name nothing real uses, a path under `/tmp`, `block`, scoped to one throwaway namespace, on two nodes (one amd64, one Raspberry Pi 4).

It is gated on a **negative control**. Before arming anything, the same exec must succeed with no entry present. Without that, a pod that never starts, an image missing the binary, and a working deny all look identical: the marker file never appears.

The first live run failed its own negative control and refused to arm. The sentinel was a copy of BusyBox, and BusyBox 1.37 no longer dispatches on `argv[1]`: the copy printed `applet not found` and exited 127. The control did exactly what it was there for. (An older Tetragon canary used the same sentinel and could never have passed. It just had never been run.) The sentinel is now a two-line `#!/bin/sh` script, which still goes through `bprm_check_security` like any other exec.

The second run:

- **Process:** the sentinel ran, was refused with `EPERM` once armed (immediately on amd64, after about 40 seconds of resolver lag on the Pi), then ran again after disarm.
- **Scope, fleet:** both targets reported one armed entry within 15 seconds, and all 16 other nodes stayed at zero.
- **Afterwards:** zero canary rows, zero block rows, zero armed on every reachable node.

The result that convinced us was in a sibling namespace. On the same node, while the entry was armed, we executed the identical file at the identical path from a second throwaway namespace. It ran. Same kernel, same LSM program, same bytes. The only difference was the cgroup the process lived in. A scoped entry's basename key is held at monitor, and its path key carries the target namespace's cgroups, so outside that namespace nothing matches at block. A deny that fires everywhere proves the hook works. A deny that fires in exactly one namespace and nowhere else proves the *scope* works, and the `hello` incident was the proof that we hadn't had that before.

That was the first deny we had ever observed from the fleet's BPF LSM path. Six days later the automated promoters were allowed to arm their own rows.

## What It Actually Stops

Then we held the canary armed and tried to get around it.

```
$ cp /tmp/kub0-lsm-canary /tmp/kub0-lsm-renamed && /tmp/kub0-lsm-renamed
ran
$ /bin/sh /tmp/kub0-lsm-canary
ran
$ /tmp/kub0-lsm-canary
Permission denied
```

A renamed copy has a new name and a new path, so neither map matches. Handing the script to `sh` makes the exec `/bin/sh`, which the critical-binary allowlist exempts before any deny lookup. Only the direct exec stayed blocked. There is a subtler hole too: the path key matches the string `exec` was called with, so `cd /tmp/x && ./curl` misses a block on `/tmp/x/curl`. Fixing that means resolving the full path inside the hook, and the program is already sitting at the verifier's one-million-instruction ceiling.

So, honestly: a name-and-path deny list stops one thing. It stops an already-detonated sample from running again under its sandbox name. The armed rows on the fleet were exactly that (sample-hash names), which made their blast radius about zero and their protective value about the same.

The conclusion was simple. **Enforcement has to key on what the file *is*, not what it's called.**

## The Third Time

So we built that for the cluster first. The sensors now hash each file as it is about to execute and refuse it when the sha256 is on the deny list: BPF LSM where the kernel has it, fanotify on the few Raspberry Pi nodes where it doesn't. On October 21 the gate started with 48 block digests.

Rows written by the automated promoter carried a 30-day lifetime. The first of those 48 expired on October 31, and by the evening of November 1 the gate held none. It went on logging `content gate: on=true`, counters and all. The armed count we had built for exactly this read 13, because it counted name-keyed rows, and a name is not a digest. The desktop gate, fed from the same list, was empty too. For four days neither the log nor the metric we built so this couldn't happen showed an empty gate. Only the line counting digest keys did, and nothing alerted on it.

We found it on November 5. The 48 rows went back in with no expiry, rows that identify a sample by its hash no longer get a lifetime at all, and any change in the digest count now raises an alert of its own. Same lesson, third time: the number you watch has to count the thing that blocks.

## The Desktop Shape

The kub0 agent's exec gate is the desktop port of that content gate, with one thing to know before installing it. **A fresh install gives you local network visibility and per-app controls. The system daemon starts checking executable hashes only once it has a non-empty deny list**, either your organization's or a local file (`KUB0_EXEC_GATE_DENY_FILE`). Until then the gate is loaded and idle, and it says so. After everything above, we were not going to ship a gate that looks armed when it isn't.

The kub0 agent started as per-app network visibility for Linux: which app talks to which country and network, with a one-key per-app firewall block, in an [Omarchy](https://omarchy.org){:target="_blank"} panel. It grew a system daemon, and the system daemon grew the gate. When a file is about to execute, the kernel holds it on fanotify `FAN_OPEN_EXEC_PERM` until the daemon answers, and the daemon refuses if the **sha256 of the file's bytes** is on the deny list. Renaming a sample doesn't change its hash, and neither does moving it.

The gate inherits every lesson above:

- **Armed before the network.** The daemon saves the last list the API returned and arms from it at boot, so the gate isn't empty for the minutes (or hours) before the first sync. Status says how old that list is until a fresh one replaces it.
- **Fail-open, and it says so.** A late answer (3 seconds), a full queue, a read error, an oversized file, an over-long list or a kernel without fanotify all allow the exec. Freezing every exec on the machine is worse than missing one. But "the gate wanted to deny and couldn't" is its own outcome, `allowed_failopen`, counted alongside `blocked`. It never disappears into "allowed."
- **A match is an event, not a counter.** Each one carries the file, its hash and the deny-list row's reason, and raises a notification with **Allow once** and **Trust on this machine**, both behind a password prompt.
- **No claiming what you aren't.** The desktop heartbeat reports `lsm_active: false`. fanotify is not an LSM and the agent doesn't pretend otherwise.
- **The wire is pinned.** What a connected machine sends is a fixed key set, held by a test. A second test feeds a deny record through and proves none of its strings reach the wire, and the outcomes endpoint rejects any extra field.

It has limits too, and they're in the README under "Honest limitations." Here are two worth knowing. The gate holds *execs*, so a script handed to an interpreter as an argument is judged as the interpreter's exec. And an app block stops an honest app's traffic, not a hostile one: an app that asks another process to connect for it gets out.

## The Advisory

0.5.2 shipped with a security advisory against our own earlier versions: [GHSA-f4hv-9rmw-x2mh](https://github.com/kub0-ai/kub0-agent/security/advisories/GHSA-f4hv-9rmw-x2mh){:target="_blank"}. The Omarchy plugin marketplace's review of 0.5.1 found four local weaknesses and reported them privately. We reproduced each one on 0.5.1 with a test before fixing it.

Three are ordinary: state directories every local account could read ([the launch post](/posts/which-app-opened-that-connection/) covers what that exposed), a named pipe in another account's home that could stall the connection sampler, and a damaged owner file that could cost one app its block when another was unblocked. The fourth fits this post a little too well. The install script downloaded the release tarball **and the checksum it was verified against from the same release**. Anyone able to replace one could replace both. Against anyone who could replace the release, it was a check that could not fail. The tarball digests are now pinned in the plugin's own source, and the script refuses anything else.

## Trying It

kub0 is [listed in the Omarchy plugin marketplace](https://plugins.omarchy.org/plugin.html?id=ai.kub0.agent){:target="_blank"}, reviewed and verified by its maintainers. It needs a setup step the marketplace can't do for you, so add it from a terminal:

```sh
omarchy plugin add https://github.com/kub0-ai/kub0-agent --enable
```

The panel offers to install the daemon (checksum pinned, as above) or build it from source. With no account, nothing leaves the machine that wasn't already leaving. You get the map, the live rows, history and per-app blocks. kub0 accounts aren't open to the public yet, so the org deny list is for orgs that already have one. The gate's self-test (`sudo bash tests/execgate-selftest.sh`) proves it end to end on any root Linux host with no server at all.

The panel, world map, flag set and `/proc` monitor started as Christopher Dwyer's [omarchy-snitch](https://github.com/ccdwyer/omarchy-snitch){:target="_blank"} (MIT). His commits are in the history unchanged, so `git diff` answers "which parts are new" without anyone having to take our word for it.

The lesson we paid for three times: count what the map *says*, not what it holds. A populated map reads exactly like an armed one.

---

*Previously: [Three Bugs Nobody Filed](/posts/three-bugs-nobody-filed/) covered running 88 eBPF policies on ARM64. [The Autoimmune Response](/posts/the-autoimmune-response/) covered the detection engine's defenses turning on its own pipeline. [Which App Opened That Connection?](/posts/which-app-opened-that-connection/) introduced the kub0 agent. This post covers the deny path that reported itself loaded when it wasn't, and what it took to make it fire.*
