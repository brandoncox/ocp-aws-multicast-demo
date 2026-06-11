# OpenShift Multicast Relay Sidecar

A portable sidecar pattern that enables IP multicast between pods running on separate nodes in AWS, without requiring changes to the application container image or cluster-level CNI configuration.

---

## The Problem

AWS VPC networking is **unicast-only**. Any IP multicast traffic that needs to cross a node boundary is silently dropped at the VPC layer. This affects any containerized application that uses multicast for peer discovery, clustering, or data distribution (JGroups, Hazelcast, Coherence, and similar frameworks).

When pods land on the **same node**, multicast works — traffic stays within the node's kernel and never touches the VPC. When pods land on **different nodes**, the traffic is dropped.

```
Node A                    AWS VPC                   Node B
┌──────────┐                                       ┌──────────┐
│  Pod 1   │──── multicast 225.x.x.x ──────✗──────│  Pod 2   │
│          │        (dropped by VPC)               │          │
└──────────┘                                       └──────────┘

Node A                    AWS VPC                   Node B
┌──────────────────┐                         ┌──────────────────┐
│ Pod 1  │ Relay   │──── unicast UDP ────────│ Relay  │  Pod 2  │
│  app   │sidecar  │   (VPC allows this)     │sidecar │   app   │
└──────────────────┘                         └──────────────────┘
```

### Why the OVN-Kubernetes Multicast Annotation Doesn't Solve This

OpenShift 4.x uses OVN-Kubernetes as its default CNI. OVN has a built-in multicast feature that can be enabled per namespace:

```yaml
annotations:
  k8s.ovn.org/multicast-enabled: "true"
```

In theory, OVN should handle cross-node replication over its Geneve overlay tunnels. In practice, on AWS-hosted OpenShift clusters (including ROSA), this does not reliably deliver multicast across nodes due to how the overlay interacts with VPC routing and security groups. The sidecar approach in this repo works regardless of CNI multicast support because it never sends multicast packets across nodes — only unicast UDP.

---

## Solution: Multicast-to-Unicast Relay Sidecar

A lightweight Python sidecar is injected alongside the application container. Because all containers in a pod share the same network namespace, the sidecar can transparently intercept and relay multicast traffic without any changes to the application image.

```
┌─────────────────────────────────────────────────────────────────┐
│  Node A                                                         │
│  ┌──────────────────────────────────────┐                       │
│  │  Pod                  shared netns   │                       │
│  │  ┌────────────┐   ┌────────────────┐ │                       │
│  │  │    app     │   │ relay sidecar  │ │                       │
│  │  │            │◀──│                │ │                       │
│  │  │ multicast  │──▶│ 1. joins group │ │                       │
│  │  │ 225.x:1234 │   │ 2. intercepts  │─┼──── unicast UDP ────▶ │
│  │  └────────────┘   │ 3. re-injects  │ │     to peer IPs       │
│  │                   └────────────────┘ │                       │
│  └──────────────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

**Outbound path** — When the app sends multicast, the sidecar receives it (shared netns), looks up peer pod IPs via a headless Service DNS query, and unicasts a copy to each peer's relay port.

**Inbound path** — When a peer relay sends a unicast packet to this pod's relay port, the sidecar re-injects it as multicast locally. The app receives it exactly as if it had come from the network directly.

**Loop prevention** — The relay's injection socket is bound to a fixed source port (`RELAY_PORT + 1`). The outbound intercept skips any packet whose source port matches, preventing re-forwarding of injected traffic.

**Peer discovery** — The relay resolves the headless Service DNS name at packet time. New pods joining are discovered automatically. No static IP configuration is required.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| OpenShift 4.20+ on AWS | ROSA or self-managed; tested on `us-east-2` |
| `oc` CLI | Logged in with `cluster-admin` or a role that can create ServiceAccounts, ConfigMaps, Services, and StatefulSets |
| Namespace to deploy into | Created in advance, or use an existing one |

Verify your cluster and login:

```bash
oc whoami
oc version
```

---

## Quick Start

### 1. Clone or download

```bash
git clone <this-repo>
cd <this-repo>
```

### 2. Deploy to a namespace

```bash
oc apply -f multicast-demo.yaml -n <your-namespace>
```

No other configuration is needed. The relay discovers its own namespace at runtime via the Kubernetes Downward API.

### 3. Verify peer discovery

```bash
# Wait for all pods to reach 2/2 Running
oc get pods -n <your-namespace> -w

# Check that all pods see each other
oc logs -l app=multicast-demo -c app -n <your-namespace> --prefix | grep "Known peers"
```

Expected output (one line per pod):

```
[pod/multicast-demo-0] Known peers alive are: multicast-demo-1, multicast-demo-2
[pod/multicast-demo-1] Known peers alive are: multicast-demo-0, multicast-demo-2
[pod/multicast-demo-2] Known peers alive are: multicast-demo-0, multicast-demo-1
```

### 4. Verify relay activity

```bash
oc logs -l app=multicast-demo -c multicast-relay -n <your-namespace> --prefix | tail -20
```

Expected output:

```
[pod/multicast-demo-0] relay: forwarded 16b to ['10.129.0.x', '10.130.0.x']
[pod/multicast-demo-1] relay: injected 16b from 10.130.0.x into 225.1.2.3:1234
```

### 5. Confirm cross-node placement

```bash
oc get pods -n <your-namespace> -o wide
```

If all pods land on the same node by chance, delete one and let it reschedule until you have pods on at least two nodes. Cross-node traffic is when the relay is doing real work.

---

## Deploy to Multiple Namespaces

Because namespace is never hardcoded in the YAML, the same file deploys cleanly to any namespace:

```bash
oc apply -f multicast-demo.yaml -n team-a
oc apply -f multicast-demo.yaml -n team-b
```

Each deployment is fully isolated — the headless Service DNS name resolves only within its own namespace, so pods in `team-a` never receive relayed traffic from `team-b`.

---

## Configuration Reference

All relay behaviour is controlled via environment variables on the `multicast-relay` sidecar container. Edit the `StatefulSet` env section to override defaults.

| Variable | Default | Description |
|---|---|---|
| `MULTICAST_GROUP` | `225.1.2.3` | Multicast group address your app joins |
| `MULTICAST_PORT` | `1234` | UDP port your app sends and receives on |
| `RELAY_PORT` | `14446` | Unicast port used between relay sidecars |
| `PEER_SERVICE_NAME` | `multicast-peers` | Name of the headless Service used for peer discovery |
| `POD_NAMESPACE` | *(Downward API)* | Injected automatically — do not set manually |
| `POD_IP` | *(Downward API)* | Injected automatically — do not set manually |

`INJECT_SRC_PORT` is always `RELAY_PORT + 1` (computed in the script, not configurable). Ensure it does not conflict with ports your application uses.

---

## Adapting to Your Own Application

To use this relay with a different container image:

**Step 1** — Replace the app container image:

```yaml
containers:
- name: app
  image: your-org/your-image:tag   # replace mjelen/demo-multicast
```

**Step 2** — Find your application's multicast group and port. If you don't know them, deploy with just the app container and inspect the running process:

```bash
# Decode port from hex (field 3 in /proc/net/udp is local_address:port)
oc exec <pod> -- cat /proc/net/udp | awk 'NR>1 {print $2}' | sort -u

# Or read strings from the binary to find embedded addresses
oc exec <pod> -- strings /path/to/binary | grep -E '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+:[0-9]+'
```

Port values in `/proc/net/udp` are hexadecimal. Convert with:

```bash
printf '%d\n' 0x04D2   # example: 0x04D2 → 1234
```

**Step 3** — Update the env vars on the relay sidecar:

```yaml
- name: MULTICAST_GROUP
  value: "239.x.x.x"   # your app's multicast group
- name: MULTICAST_PORT
  value: "45678"        # your app's multicast port
```

**Step 4** — Update the headless Service port to match:

```yaml
ports:
- name: app
  port: 45678           # match MULTICAST_PORT
  protocol: UDP
```

**Step 5** — Deploy and verify as described in Quick Start.

---

## How Peer Discovery Works

The relay uses a Kubernetes **headless Service** (a Service with `clusterIP: None`) for peer discovery. A headless Service returns all backing pod IPs as DNS A records instead of a single virtual IP.

```bash
# From inside any relay container:
python3 -c "import socket; print(socket.gethostbyname_ex('multicast-peers.<namespace>.svc.cluster.local'))"
# ('multicast-peers...', [], ['10.129.0.90', '10.130.0.43', '10.130.0.44'])
```

The relay queries this DNS name on every packet, so pod scale-up and scale-down are handled automatically without restarts or reconfiguration.

---

## Cleanup

Remove all resources from a namespace:

```bash
oc delete -f multicast-demo.yaml -n <your-namespace>
```

---

## Troubleshooting

**Pods are 1/2 — relay sidecar is not starting**

```bash
oc describe pod multicast-demo-0 -n <namespace>
oc logs multicast-demo-0 -c multicast-relay -n <namespace>
```

Common cause: the `python:3.11-slim` image pull failed. Verify your cluster has access to Docker Hub or mirror the image to an internal registry.

**Relay logs show `peer lookup failed`**

The headless Service DNS name is not resolving. Verify the Service exists and its selector matches the pod labels:

```bash
oc get service multicast-peers -n <namespace>
oc get endpoints multicast-peers -n <namespace>
```

**App logs still show no peers after relay is running**

Confirm the relay is configured with the correct `MULTICAST_GROUP` and `MULTICAST_PORT` for your app. Use the port-finding steps in [Adapting to Your Own Application](#adapting-to-your-own-application) to verify.

**Relay logs show forwarded packets but app still sees no peers**

The relay is sending but the app may not be receiving re-injected multicast. Check whether `IP_MULTICAST_LOOP` is disabled on the app's socket. If the app explicitly disables loopback, injected packets will not be delivered to it within the same netns.

**All pods land on the same node and multicast works without the relay**

This is expected — same-node multicast uses the kernel bridge and never hits the VPC. Force cross-node placement with a pod anti-affinity rule:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: multicast-demo
        topologyKey: kubernetes.io/hostname
```
