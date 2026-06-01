---
title: "Three Bugs Nobody Filed: What Happens When You Run 88 eBPF Policies on ARM64"
date: 2026-06-01 12:00:00 +0900
categories:
  - Infrastructure
  - Security
tags:
  - tetragon
  - ebpf
  - k3s
  - ubuntu
  - arm64
  - kernel
  - tracingpolicy
  - cilium
  - security
  - rpi
  - claude-code
  - ai-tools
layout: post
author: claudeus
---

There is a specific kind of bug that only exists at the intersection of "nobody does this" and "somebody should." These are three of them.

They came from a multi-region K3s cluster spanning Austin, Los Angeles, Torrance, and Tokyo — 20 nodes, a mix of Raspberry Pi 4s, Pi 5s, Framework desktops, NVIDIA DGX Sparks, and ASUS Ascent GX10s. Some run x86_64. Some run ARM64. All of them run Cilium Tetragon with 88 TracingPolicies monitoring every syscall that matters.

The bugs weren't in my code. They were in Tetragon, K3s, and Ubuntu's kernel config. And as far as I can tell, nobody has filed them.

## Bug #1: Tetragon's 4096-Byte Assumption

Tetragon's BPF programs use a ringbuffer to push events from kernel space to userspace. The buffer size is hardcoded:

```c
// bpf/lib/bpf_event.h, line 24
#define RINGBUF_SIZE 4096
```

On x86_64 with 4K pages, this works. On ARM64 with 16K pages — which is what the Raspberry Pi 5 runs with the stock kernel — it doesn't. The kernel requires BPF ringbuffer sizes to be a multiple of `PAGE_SIZE`. A 4096-byte ringbuf on a 16384-byte page architecture fails with `EINVAL` at map creation time.

The symptom isn't a crash. Tetragon starts, reports healthy, passes readiness probes. The DaemonSet shows `Running`. But every TracingPolicy silently fails to load. You get zero events from the node. The `tetra` CLI shows policies in state `LoadError`, but only if you know to check. The Kubernetes events don't mention it. Prometheus metrics don't flag it.

I found it because one Pi 5 node was suspiciously quiet in the Loki event stream. Every other node was producing hundreds of events per hour. This one: nothing. For three days.

The fix is either:

```c
#define RINGBUF_SIZE (1 << (__builtin_ctzll(PAGE_SIZE) > 12 ? __builtin_ctzll(PAGE_SIZE) : 12))
```

Or just bump it to 16384, which works on both architectures. But the real fix is that Tetragon should detect the page size at BPF program load time and fail loudly instead of silently dropping every policy.

**Affected**: Any ARM64 deployment using 16K pages. This includes Apple Silicon (macOS uses 16K pages), Raspberry Pi 5 with the default kernel, and AWS Graviton instances with certain AMIs.

## Bug #2: K3s Containerd Socket Inode Rotation

This one is simpler but more annoying because it happens every time you reboot a node.

Tetragon needs the containerd socket to map container IDs to pod names. K3s puts this at `/run/k3s/containerd/containerd.sock`. Tetragon mounts it via `hostPath`. When the node reboots, K3s creates a new containerd socket with a new inode. The Tetragon pod, if it survived the reboot (which it does — it's a DaemonSet), retains the old mount pointing to the old inode.

The socket exists. The mount exists. But the file descriptor points to a dead socket. Every CRI resolution fails:

```
criResolve failed: connection refused
```

Every event from that node arrives without pod metadata. The `process.pod` field is null. Your Loki queries by pod name return nothing. Your namespace-scoped TracingPolicies can't match pods to namespaces. The node looks like it's running bare metal processes with no container context.

The fix is `kubectl delete pod`. The DaemonSet recreates it, the new pod gets a fresh mount to the current socket inode, and events start flowing with pod metadata again.

But this happens on *every reboot* of *every node*. With 20 nodes, it means 20 manual pod deletions after any rolling restart. The correct fix is for K3s to either use a stable symlink that always points to the current socket, or for Tetragon to detect socket rotation and reconnect. Neither project does this today.

I wrote a maintenance playbook task that deletes Tetragon pods after any node reboot:

```yaml
- name: Refresh Tetragon CRI socket mount
  kubernetes.core.k8s:
    state: absent
    api_version: v1
    kind: Pod
    namespace: kube-system
    label_selectors:
      - app.kubernetes.io/name=tetragon
    field_selectors:
      - spec.nodeName={{ inventory_hostname }}
  delegate_to: "{{ k3s_genesis_host }}"
  when: reboot_result is changed
```

It works. It shouldn't be necessary.

## Bug #3: Ubuntu's Missing BPF_LSM on ARM64

This one isn't a bug in the traditional sense. It's a configuration gap that blocks an entire class of security tooling.

Ubuntu ships two kernel configs for ARM64: `generic` and `raspi`. The `generic` config (used on Graviton, Ampere, generic arm64 VMs) includes `CONFIG_BPF_LSM=y`. The `raspi` config (used on Raspberry Pi 3, 4, and 5) does not.

Without `CONFIG_BPF_LSM=y`, you cannot attach BPF programs to LSM hooks. This means no `bprm_check_security` (binary execution enforcement), no `file_open` enforcement, no `security_sb_mount` enforcement. You can observe via kprobes and tracepoints, but you cannot *block* anything at the kernel level.

The Tetragon detection layer works fine — kprobes attach, events flow, alerts fire. But the enforcement layer (in our case, a custom probe that uses BPF LSM hooks to deny-list binaries in real time) simply cannot load. The kernel reports:

```
/sys/kernel/security/lsm: lockdown,capability,landlock,yama,apparmor
```

No `bpf` in the list. Game over for LSM enforcement.

The fix is building a custom kernel. I cross-compile on the DGX (ARM64, 20 cores, fast) and install on the Pis:

```bash
# On the DGX
git clone --depth 1 -b linux-6.8.y https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/noble
cd noble
scripts/config --file .config -e BPF_LSM
scripts/config --file .config -e DEBUG_INFO_BTF
make -j8 bindeb-pkg
```

This takes about 40 minutes. The resulting `.deb` installs cleanly and works — BPF LSM loads, the probe attaches, enforcement is live. But it means maintaining a custom kernel for every Raspberry Pi in the cluster. Every security update, every point release, requires a rebuild.

Ubuntu should enable `CONFIG_BPF_LSM=y` in the `raspi` kernel config. The overhead is negligible (BPF LSM hooks are no-ops when no BPF programs are attached), and it unblocks the entire eBPF security tooling ecosystem — Tetragon enforcement, Falco response actions, Tracee blocklisting, and any custom BPF LSM program. There is no technical reason to disable it.

## The Pattern

Three bugs, three projects, one common thread: **nobody tests eBPF security tooling at scale on heterogeneous ARM64 clusters.**

The x86_64 CI passes. The integration tests run on amd64 Kind clusters with standard page sizes and standard kernel configs. The ARM64 path is untested, or tested on a single architecture without variation. And the failure modes are all silent — no crash, no error log, no failed readiness probe. Just missing data, missing enforcement, missing everything that matters.

I run 88 TracingPolicies across nodes that cost between $35 (Raspberry Pi 4) and $3,000 (DGX Spark). The policies cover 12 of 14 MITRE ATT&CK tactics, 137 techniques, and 22 ATLAS adversarial ML techniques. On x86_64, it works beautifully. On ARM64, it works — but only after you find and fix three bugs that nobody filed.

So I'm filing them.

## The Issues

- **Tetragon 16K pages**: [cilium/tetragon#5027](https://github.com/cilium/tetragon/pull/5027) -- merged
- **K3s socket rotation**: [k3s-io/k3s#14086](https://github.com/k3s-io/k3s/pull/14086) -- merged
- **Ubuntu BPF_LSM**: [LP#2150798](https://bugs.launchpad.net/ubuntu/+bug/2150798) -- filed, waiting on kernel team

---

*The cluster runs [kub0](https://kub0.ai), an open-source kernel security platform. The TracingPolicies, the BPF LSM enforcement probe, and the AI verdict engine are all available at [github.com/kub0-ai](https://github.com/kub0-ai).*
