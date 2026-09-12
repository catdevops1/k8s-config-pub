# Flannel Overlay Recovery — master-node

**Cluster:** 5-node bare-metal Kubernetes (Ubuntu 24.04, k8s v1.35.0, Flannel VXLAN)
**Applies to:** master-node (Dell Inspiron 15, USB Ethernet adapter)
**Last incident:** 2026-08-27 → detected 2026-09-11 (15 days undetected)

---

## Symptom

ArgoCD apps intermittently show:

```
ComparisonError: Failed to load target state: failed to generate manifest
for source 1 of 2: rpc error: code = Unknown desc = failed to get git
client for repo https://github.com/<org>/<repo>.git
```

This message is misleading. The repo is reachable and the credentials are
valid. A hard refresh may succeed once and fail on the next attempt, which
makes it look like a transient GitHub problem.

Other signs:

- Pods on worker nodes cannot reach pods on master-node
- The ArgoCD UI may still look entirely green — it serves cached
  comparisons when Redis is unreachable, so "Synced" can be days or
  months stale
- Repository settings page shows the repo as `Successful` (that is a
  cached credential check, not a live render test)

---

## Root cause

`flannel.1` — the VXLAN overlay device — is missing on master-node.

master-node uses a USB Ethernet adapter (`enxXXXXXXXXXXXX`); this laptop
has no built-in NIC. If the adapter is unplugged or drops, the kernel
removes the underlying device and `flannel.1` goes with it, along with
every route pointing through it.

**flanneld does not recover on its own.** Its reconcile loop is driven by
the Kubernetes node watch. When the watch dies with the network, the loop
stops. The process stays alive, the container stays `Running`, the
readiness probe keeps passing — and nothing ever recreates the interface.

From the flanneld log at the time of the 2026-08-27 incident:

```
W0827 17:53:48  watch of *v1.Node ended with: http2: client connection lost
W0827 17:53:50  failed to list *v1.Node: dial tcp 10.96.0.1:443:
                connect: network is unreachable
```

...and then the log simply stops. No recovery messages for 15 days.

---

## Diagnosis

### 1. Is it actually Redis, not Git?

```bash
kubectl -n argocd logs deploy/argocd-repo-server --tail=100 | grep 6379
```

Look for `dial tcp <redis-clusterip>:6379: i/o timeout`. ArgoCD checks its
Redis cache *before* reaching out to Git, so a Redis timeout surfaces as a
Git error. If you see this, the problem is the network, not the repo.

### 2. Is the overlay device present?

```bash
ip link show flannel.1
```

`Device "flannel.1" does not exist` confirms it.

```bash
ip route | grep 10.244
```

Healthy output has one route per peer CIDR via `flannel.1`, plus the local
subnet on `cni0`:

```
10.244.0.0/24 dev cni0 proto kernel scope link src 10.244.0.1
10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink
10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink
10.244.3.0/24 via 10.244.3.0 dev flannel.1 onlink
10.244.7.0/24 via 10.244.7.0 dev flannel.1 onlink
```

Only the `cni0` line present → the overlay is gone.

### 3. Confirm the direction (optional but decisive)

```bash
for N in node02 node03 node04 master-node; do
  echo "=== $N ==="
  kubectl run nettest-$N --image=busybox:1.36 --restart=Never --rm -i \
    --overrides="{\"spec\":{\"nodeName\":\"$N\"}}" -- \
    sh -c "nc -zvw2 <master-node-pod-ip> 6379"
done
```

Expected pattern when broken: works from master-node (same-node traffic
goes over `cni0` and never touches the overlay), times out from every
worker.

> Note: ArgoCD's own pods are distroless and have no `nc`. Use a busybox
> pod pinned to a node, not `kubectl exec` into the repo-server.

---

## Fix

```bash
kubectl -n kube-flannel get pods -o wide     # find the master-node pod
kubectl -n kube-flannel delete pod <master-node-flannel-pod>
```

flanneld recreates the VXLAN device unconditionally at startup. It reuses
the VTEP MAC stored in the node annotation
(`flannel.alpha.coreos.com/backend-data`), so peer nodes do not need to
re-learn it and no restart is required on the workers.

Recovery is immediate — the device and routes are back within seconds of
the new pod starting.

---

## Verify

Do not trust a green dashboard. Verify each layer.

```bash
# 1. Device exists and is up
ip link show flannel.1
# expect: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450

# 2. Routes rebuilt — one per peer CIDR
ip route | grep 10.244

# 3. Forwarding table — one entry per peer node
bridge fdb show dev flannel.1
# expect 4 entries mapping VTEP MACs to 192.168.4.101-.104

# 4. Traffic actually flows
kubectl run nettest --image=busybox:1.36 --restart=Never --rm -i \
  --overrides='{"spec":{"nodeName":"node02"}}' -- \
  nc -zvw2 <master-node-pod-ip> 6379
# expect: "open"

# 5. ArgoCD stopped timing out
kubectl -n argocd logs deploy/argocd-repo-server --tail=50 | grep 6379
# no output = recovered

# 6. ArgoCD is genuinely re-rendering, not replaying cache
kubectl -n argocd logs deploy/argocd-repo-server --tail=50 | grep -i "cache"
# "manifest cache hit/miss" lines require Redis to answer

# 7. Storage is intact before doing any further work
kubectl -n longhorn-system get volumes.longhorn.io
kubectl -n longhorn-system get replicas.longhorn.io \
  -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeID,STATE:.status.currentState
```

---

## Blast radius — what breaks and what doesn't

### Unaffected

- **Worker-to-worker pod traffic.** Only master-node's device was missing;
  the workers' overlay between themselves stayed intact.
- **The control plane.** kubelet, scheduler, and controller-manager reach
  the API server at `192.168.4.100:6443` — the node's real LAN address,
  not a pod IP. That traffic never touches the overlay.
- **Ingress.** Cloudflare Tunnel, Envoy Gateway and MetalLB all run on
  workers. External traffic to hosted sites is unaffected.
- **Pods on master-node reaching the API** via `10.96.0.1`, which kube-proxy
  DNATs to the host address.

### Affected

- **ArgoCD repo-server → Redis** (Redis runs on master-node). Degrades to
  "no cache" rather than visible failure — ArgoCD keeps serving stale
  comparisons, so apps can show `Synced` against a target state they have
  not regenerated in months.
- **Longhorn CSI sidecars on master-node → managers on workers.** No
  practical impact here because master-node has `allowScheduling: false`
  and hosts zero replicas.
- **netdata-parent on master-node ← children on workers.** Metrics from
  every other node stop arriving. The monitoring system was itself a
  casualty of the outage it should have reported.

---

## Prevention

The reason this went undetected for 15 days is not a missing cron job. It
is that every health signal in the chain reports **liveness, not function**:

"comparison 6 months stale" — that was the vault app specifically, and it wasn't caused by this incident. The vault app hadn't re-rendered since March because its source never changed, which is normal ArgoCD caching. Conflating it with the partition overstates the impact. Better as comparison served from stale cache.

"Longhorn | all volumes healthy | control plane isolated" — the volumes genuinely were healthy, and you verified that afterward. The row implies Longhorn was misreporting, which it wasn't. I'd either drop the row or reword to something like CSI sidecars on master-node unreachable in the Actual column, since the misleading part was that nothing surfaced the isolation.

| Signal | Reported | Actual |
|---|---|---|
| flannel pod | `Running`, `1/1 Ready`, 0 restarts | reconcile loop dead |
| master-node | `Ready` | severed from pod network |
| ArgoCD apps | `Synced`, `Healthy` | comparison 6 months stale |
| Longhorn | all volumes healthy | control plane isolated |

Checks worth adding, weakest to strongest:

1. **Interface presence** (catches this exact failure)

   ```bash
   ip link show flannel.1 >/dev/null 2>&1 || <alert>
   ```

2. **Cross-node pod reachability** (catches this *and* every other cause of
   the same symptom) — a pod on a worker connecting to a known pod IP on
   master-node.

3. **ArgoCD Redis connectivity** — alert on `6379` timeouts in the
   repo-server log rather than waiting to notice a ComparisonError.

Physical: the USB Ethernet adapter is a single point of failure for the
control plane's entire network path. Secure it against being bumped loose.

Also confirm the laptop does not suspend on lid close:

```bash
grep -E '^#?Handle(LidSwitch|LidSwitchExternalPower)' /etc/systemd/logind.conf
# both should be: ignore
```

---

## Related gotchas discovered during this incident

- **`ip link show flannel.1` returning a device is not proof traffic
  flows.** Routes and FDB entries must also be present, and reachability
  must be tested from a worker.
- **ArgoCD "Synced" means "the last comparison found no diff"** — not
  "Git currently matches the cluster." On an app whose source has not
  changed, that badge can be months stale. Force a hard refresh
  (`kubectl -n argocd annotate app <name> argocd.argoproj.io/refresh=hard
  --overwrite`) before relying on it prior to any change.
- **The Repositories settings page connection status is cached** and can
  show `Successful` while renders are failing.
- **`kubectl exec` into ArgoCD pods for network testing does not work** —
  distroless images have no `nc`, and `bash`'s `/dev/tcp` may be compiled
  out, producing false negatives. Use a busybox pod pinned to the node.
