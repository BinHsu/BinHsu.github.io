---
title: "Running k3s on Apple's container runtime — HA clusters, one micro-VM per node"
description: "Every local-dev Kubernetes tutorial puts all nodes on a shared kernel. Apple's container runtime breaks that assumption — one micro-VM per node, real DHCP IPs, no Docker daemon. The interesting part came when I tried to make the control plane highly available."
date: 2026-06-28 12:00:00
tags: [k3s, kubernetes, apple-silicon, virtualization, golang, etcd, haproxy]
asset_path: /assets/img/posts/2026-06-28-k3s-clusters-apple-container-k3ac
image:
  path: /assets/img/posts/2026-06-28-k3s-clusters-apple-container-k3ac/social-card.png
---

Local Kubernetes on a Mac has a quiet assumption underneath it: all nodes share one kernel.
kind, minikube, Docker Desktop's built-in cluster — they run nodes as processes or containers
on the host kernel, not as real machines. That's fine for most purposes; the Kubernetes API
looks the same either way.

Apple's [`container`](https://github.com/apple/container), shipped with macOS 26, changes that
assumption. Each Linux container is a lightweight micro-VM with its own kernel, its own
init process (`vminitd`), and a DHCP-assigned IP on Apple's vmnet. The isolation is closer
to a real machine than to a Docker container.

[@saiyam1814's `kiac`](https://github.com/saiyam1814/kiac) was first to prove Kubernetes
works on this substrate, using kubeadm, single control plane. I wanted to see the k3s path,
and specifically what happens when you try HA. [My Talos sibling project]({% post_url 2026-06-15-a-native-talos-provider-for-apple-container %})
covers the immutable-OS take on the same substrate — including the DHCP story and the
micro-VM recipe in depth — so I'll keep the substrate background brief here. This post
is the k3s take, and the HA path is where it gets its own story.

The result is [`k3ac`](https://github.com/BinHsu/aegis-apple-container-provisioner-k3s)
("k3s + apple container") — a standalone Go launcher that provisions k3s clusters as
per-node micro-VMs on apple/container. v0.6.0, all hardware gates green.

![k3ac HA architecture on Apple Silicon]({{ page.asset_path }}/architecture.png)
*Single-node default uses sqlite embedded in the k3s server VM. HA mode (shown) adds a
3-node etcd quorum and an haproxy L4 API load balancer — all as micro-VMs on the same
vmnet, each with its own DHCP IP.*

## Why k3s, and why a freestanding launcher

k3s fits the apple/container substrate well. It's a self-contained binary that embeds its
own containerd; it doesn't expect systemd or a pre-existing container runtime socket. It
boots under `vminitd` — k3ac's gate G1. Flannel's `host-gw` backend gives clean pod-to-pod
routing across micro-VMs on the same vmnet (gate G2, 0% packet loss both ways).

There is no upstream interface to implement. The Talos sibling satisfies
`siderolabs/talos/pkg/provision.Provisioner` — a clean Go interface that let the out-of-tree
implementation compile straight against the real Talos provisioner machinery. k3s has no
equivalent. There is no plugin loader, no extension point, no merge path. k3ac is a
freestanding launcher that execs the `container` CLI to boot k3s nodes as micro-VMs. The
exec-the-CLI pattern is carried over verbatim from the Talos sibling; the freestanding shape
is different because the upstream is different.

## The DHCP constraint, and why it hits etcd harder in HA mode

Both the Talos sibling and k3ac share the same substrate constraint: vmnet assigns each
node's IP by DHCP. There is no static-IP option. After a cold restart — stopping and
restarting the `container` daemon, or the VMs themselves — DHCP may hand out different
addresses.

For single-node k3s on sqlite this is handled cleanly. k3ac uses FQDN naming via Apple's
container DNS: each node is `<cluster>-server-1.<domain>`, the kubeconfig endpoint is the
FQDN, and the API server cert covers the FQDN as a SAN (not the IP). After a DHCP shift
the FQDN still resolves; the kubeconfig stays valid. Gate G5 verifies this.

For a multi-server HA control plane, the same DHCP constraint runs directly into etcd's
internals. And that is where the embedded-etcd path dies.

## The embedded-etcd dead end

k3s supports two HA control-plane paths:

- **`--cluster-init`** (embedded etcd, peer-to-peer between server nodes). This is what the
  k3s HA quickstart shows first. It needs no extra infrastructure.
- **`--datastore-endpoint`** (pluggable external datastore; server nodes are stateless against
  it). This needs an external etcd or SQL database.

The natural instinct is `--cluster-init`. I tested it. It does not survive a cold restart on
this substrate.

Here is the mechanism: embedded etcd's peer membership is **IP-bound**. Each member stores
the other members' peer URLs as `https://<IP>:2380`. When you cold-restart all three server
VMs, vmnet DHCP hands out new addresses. The etcd members come up and try to contact their
peers at the old IPs. Nothing answers. Quorum never forms. The cluster is permanently stuck;
you destroy and recreate.

This is not a k3ac bug or a k3s bug. It is a fundamental collision between etcd's membership
model and a DHCP substrate. etcd stores IP addresses as peer URLs because that is how the
Raft membership protocol works — changing a member's peer URL requires a membership
reconfiguration operation, not just a restart. The cluster doesn't know that the IP changed;
it just sees an unreachable peer.

The [ADR](https://github.com/BinHsu/aegis-apple-container-provisioner-k3s/blob/main/docs/ADR/0002-ha-via-external-datastore-not-embedded-etcd.md)
documents the verification sequence: bring up embedded etcd HA, cold-restart all server VMs,
observe the quorum failure, confirm the peer URLs in etcd's member list contain the old IPs.
Gate G9 is the hardware verification of the external-datastore fix.

The fix is `--datastore-endpoint`, addressed by FQDN. An external etcd cluster reached by
FQDN (not IP) has no IP-bound membership problem. The k3s server nodes reconnect to the
same FQDNs after a DHCP shift and resume normally. k3ac auto-provisions this when you pass
`-servers N` with no explicit `-datastore-endpoint`.

## What the HA design looks like

One command provisions the full stack:

```sh
go run ./cmd/k3ac -name hav -servers 3 -agents 1
```

The provisioner runs this sequence:

**1. Generate a TLS bundle on the host.** Pure stdlib `crypto/x509` — no external tooling. One
CA, one server cert per etcd member (SANs: member FQDN + `localhost` + `127.0.0.1`, never the
DHCP IP), one k3s client cert. The FQDN SANs keep TLS valid across DHCP shifts because no IP
is ever in a SAN.

**2. Provision a 3-node etcd quorum.** Three micro-VMs — `hav-etcd-{1,2,3}` — running
`quay.io/coreos/etcd:v3.5.16`. The TLS bundle is delivered via the same host bind-mount mechanism
used for kubeconfig delivery (see below). Each member's FQDN is registered in Apple's
container DNS. The provisioner waits for each member's client port to become reachable before
proceeding.

**3. Provision an haproxy API load balancer.** One more micro-VM — `hav-api.<domain>` — running
haproxy in `mode tcp`. TCP passthrough; it does not terminate TLS. The kubeconfig endpoint is
`https://<cluster>-api.<domain>:6443`. The API server cert covers that SAN. Adding or removing
a server node rewrites the haproxy config in place.

**4. Bring up server nodes.** Each wired to the external etcd via `--datastore-endpoint`, with
`--tls-san` covering both the server's own FQDN and the LB address.

**5. Bring up agent nodes** against the LB endpoint (`K3S_URL=https://<cluster>-api.<domain>:6443`).

The 3-node etcd quorum (rather than a single-VM datastore) tolerates losing one member and
keeps serving. Member count is configurable via `-datastore-members` (must be odd and ≥ 3).
You can also bring your own datastore: `-datastore-endpoint 'postgres://user:pass@db.aegis:5432/kine'`
skips the auto-provisioning entirely.

Single-node default skips all of this. Embedded sqlite, no etcd, no haproxy, no extra VMs.

## Two gotchas the substrate adds regardless of HA

Two constraints bite before the HA question even comes up. Both are shared with the Talos
sibling, but worth naming here because the failure modes are non-obvious.

**The `container cp` vsock fault.** The obvious way to retrieve the kubeconfig from inside a VM
is `container cp`. It doesn't work reliably during k3s cold boot. k3s unpacks a large bundle
of embedded container images on first start; this saturates the guest (`vminitd`) during the
unpack. The saturation faults the vsock channel that `container cp` rides over. The cp is killed
externally by the platform — verified against a 180 s timeout that the kill arrives well
before, not at. A faulted vsock also wedges `container stop` and
`container rm` for two or more minutes afterward — hardware-verified.

k3ac avoids the guest agent entirely for the critical path. The host cluster state directory
is bind-mounted into the VM at create time. k3s writes its kubeconfig to that bind-mount via
`--write-kubeconfig /mnt/k3s-out/k3s.yaml --write-kubeconfig-mode 0644`. The provisioner
polls `os.Stat` for the file on the host side — no vsock, no `container exec`, no `container
cp`. Readiness is a direct TLS dial from the host to `https://<server-fqdn>:6443`. The full
analysis is in
[ADR-0001](https://github.com/BinHsu/aegis-apple-container-provisioner-k3s/blob/main/docs/ADR/0001-kubeconfig-delivery-via-host-bind-mount.md).

**The silent apiserver OOM.** At tight server memory the Kubernetes API server can be
OOM-killed with no crash log — the VM stays up but the cluster never becomes ready.
Steady-state k3s usage is around 620 MB, but the boot window needs headroom above that;
the default is 2 GB for this reason. The failure mode is completely silent — no explicit
error, just a cluster that never becomes ready. The Talos sibling has the same constraint;
it's a Kubernetes-on-micro-VM floor, not a k3s-specific one.

## Day-2 operations, hardware-verified

A design goal that separates k3ac from a spike script: the day-2 operations work and are
hardware-verified on Apple Silicon. All destructive verbs require `-force`; without it the
command prints the plan and exits non-zero — a dry-run guard.

**Snapshot and restore.** `-snapshot` saves the managed etcd datastore to a host-side file
(timestamped, under `_out/clusters/<cluster>/snapshots/`). `-restore` rebuilds all etcd
member data volumes from the snapshot. Snapshot is safe; restore requires `-force`.

**Rolling upgrade and rollback.** `-upgrade <cluster> -image rancher/k3s:v1.33.0-k3s1 -force` replaces
nodes one at a time: servers first, then agents. Each node is stopped, reprovisioned at the
new image, and waits for Kubernetes readiness before the next one proceeds. After a DHCP IP
change the node re-registers to the cluster by FQDN — the FQDN endpoint handles the
re-registration transparently. `-rollback` returns to the image pinned at the last upgrade.

**Cert and token rotation.** `-rotate-certs` regenerates the entire TLS bundle (CA + all
member/client certs) and redelivers it to all etcd members and k3s servers. `-rotate-token`
generates a new `K3S_TOKEN` and re-registers all servers and agents against it. Both require
`-force`.

These are verified at gates G15–G17 in
[`docs/VERIFICATION.md`](https://github.com/BinHsu/aegis-apple-container-provisioner-k3s/blob/main/docs/VERIFICATION.md)
— first-person runbook with exact commands, not just unit assertions.

## What it doesn't promise

k3ac is a proven spike, not maintained tooling. The honest limits:

**No upstream merge path.** k3s has no `pkg/provision.Provisioner`-style interface. There is
nothing to contribute upstream. k3ac will not become a k3s CLI subcommand; it is a
freestanding launcher that lives here.

**Clusters do not survive a host reboot.** Apple's container DNS domain (`sudo container system
dns create <domain>`) does not survive a macOS restart. Neither does the vmnet DNS
registration. After a full host reboot you recreate from scratch (~4–5 minutes for HA). After
a cold-restart of just the VMs (stopping and restarting nodes without rebooting the host),
`-stop` and `-start` preserve the cluster data — FQDN addressing handles the DHCP shift.

**This is not for production.** Same scope as k3s's own embedded-mode documentation: local
development, CI validation, ephemeral test environments. The HA design makes the cluster
resilient to VM restarts, not to host reboots, which is the right tradeoff for the use case.

## The differentiator from the Talos sibling

The [Talos sibling]({% post_url 2026-06-15-a-native-talos-provider-for-apple-container %})
uses Talos's embedded etcd. It does not hit the IP-bound membership problem the same way,
because the Talos provider's DHCP reconciliation runs before etcd ever starts — the endpoint
is patched to the real DHCP address before the node boots with its config. The embedded etcd
binds the correct IP from the start.

k3s's HA path doesn't have that seam. k3s's `--cluster-init` forms the cluster after boot,
and by the time you restart the cluster, the peer URLs are already baked with the first-boot
IPs. There's no pre-boot config patch to hook into. That's why k3ac's HA path is external
datastore rather than embedded etcd — and why it needs the extra etcd VMs and haproxy that
the Talos sibling doesn't.

| Aspect | Talos sibling | k3s (k3ac) |
|---|---|---|
| Datastore | Embedded etcd inside control-plane nodes | External 3-node etcd quorum + haproxy LB |
| Extra VMs for HA | None | 3 etcd members + 1 haproxy LB |
| DHCP handling | Pre-boot config patch (endpoint is the real IP) | FQDN addressing throughout; IP is never in a peer URL or SAN |
| Day-2 ops scope | Full Talos lifecycle (via `talosctl`) | Snapshot/restore, rolling upgrade, cert/token rotation (k3ac verbs) |

The k3s lane costs more VMs for HA. It also delivers something the Talos lane currently
doesn't: a full day-2 ops suite hardware-verified on this substrate.

---

Code: [github.com/BinHsu/aegis-apple-container-provisioner-k3s](https://github.com/BinHsu/aegis-apple-container-provisioner-k3s).
~9,480 LOC Go, 4-job CI (lint / race-detector tests / govulncheck / gitleaks), ADR trail,
hardware verification log.

---

*This site is the lab side of my work. The polished portfolio lives at [binhsu.org](https://binhsu.org).*
