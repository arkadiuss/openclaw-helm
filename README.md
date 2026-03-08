# openclaw-helm

A Helm chart for running [OpenClaw](https://openclaw.ai) on Kubernetes -- with rootless Podman sandboxing, headless browser support, Gateway API routing, and network policies baked in.

Built for homelabs. Single node, single replica, no compromises on security (yes, we know it's OpenClaw -- that's exactly why).

## Why another Helm chart for OpenClaw?

There are a couple of community charts out there already. This one exists because none of them solve sandboxing properly.

Most charts either skip sandboxing entirely or fall back to Docker-in-Docker with `--privileged`. This chart uses **rootless Podman with Kubernetes User Namespaces** -- no privileged containers, no host Docker socket mounts, no root on the host. If you care about running OpenClaw on a box that also runs your other homelab services, this is the chart for you.

It also handles the operational details that OpenClaw on Kubernetes needs but the official docs don't cover: gateway bind configuration, auth token injection, control UI origins, trusted proxy awareness, headless browser for the browser tool, health probes that actually work, and network policies.

## Why Kubernetes? OpenClaw doesn't officially support it.

You're right -- OpenClaw is designed as a single-instance service with in-memory WebSocket state and a Lane Queue. It can't scale horizontally, and the official docs point you toward Docker Compose or bare metal.

But if you're already running Kubernetes for your homelab (and you're the kind of person who does), you get some things for free that are annoying to set up otherwise:

- **Persistent storage** that survives container restarts without bind-mount gymnastics
- **Gateway API routing** with TLS termination through your existing ingress infrastructure
- **Network policies** that restrict OpenClaw to DNS + HTTPS egress only -- no phone-home surprises
- **Secret management** through native Kubernetes Secrets (or whatever you've wired into them)
- **Health checks** and automatic restarts when the gateway process dies

This chart deploys OpenClaw as a **StatefulSet** (single replica) with `volumeClaimTemplates` for stable identity and persistent `~/.openclaw` storage. It's not pretending to be a distributed system -- it's just a well-managed single instance.

## Why rootless Podman instead of Docker-in-Docker?

OpenClaw sandboxes tool execution in containers. The official setup uses Docker, which means either:

1. **Docker-in-Docker (DinD)** -- requires `--privileged`, giving the container full root access to the host. On a homelab box running your other services? No thanks.
2. **Mounting the host Docker socket** -- even worse. Any container OpenClaw spawns can talk to your host Docker daemon.

This chart takes a different approach: a **rootless Podman sidecar** using Kubernetes [User Namespaces](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/) (`hostUsers: false`).

The Podman sidecar runs with `SYS_ADMIN`, `SETUID`, and `SETGID` capabilities -- but scoped to the pod's user namespace, not the host. On the host, the process is just an unprivileged user (e.g. UID 524288+). It can create nested namespaces for rootless containers without being able to touch anything on the host.

<details>
<summary>The full security story</summary>

| Capability | Why it's needed |
|-----------|----------------|
| `SYS_ADMIN` | Creating nested user/mount namespaces inside rootless containers |
| `SETUID` | `newuidmap` is a setuid binary that maps UIDs into the user namespace |
| `SETGID` | `newgidmap` does the same for GIDs |

All other capabilities are dropped (`drop: ALL`). The seccomp profile is `Unconfined` because containerd's default profile blocks `CLONE_NEWUSER`, which rootless containers need.

**Why `Unconfined` is acceptable here:** With `hostUsers: false`, the container's root is mapped to an unprivileged host UID. Even without seccomp filtering, the process cannot perform privileged operations on the host. The user namespace is the security boundary, not seccomp.

Podman uses the **VFS storage driver** instead of overlay. Overlay requires remounting with `MS_PRIVATE` propagation, which fails on `emptyDir` volumes. VFS is slower but works reliably.

</details>

### Cluster requirements for Podman

| Component | Minimum Version | Why |
|-----------|----------------|-----|
| Kubernetes | >= 1.33 | User Namespaces GA |
| containerd | >= 2.0 | User Namespace support in CRI |
| Linux kernel | >= 6.3 | `idmap` mounts for user namespaces |

Verify your node supports it:

```bash
kubectl get node <node> -o jsonpath='{.status.features.userNamespaces}'
# Should return: true
```

## Quick Start

```bash
# Create namespace and secrets
kubectl create namespace openclaw

kubectl create secret generic openclaw-api-key \
  --namespace openclaw \
  --from-literal=api-key=sk-ant-...

kubectl create secret generic openclaw-gateway-token \
  --namespace openclaw \
  --from-literal=gateway-token=$(openssl rand -hex 32)

# Install
helm install openclaw ./openclaw-helm \
  --namespace openclaw \
  --set openclaw.apiKeySecret=openclaw-api-key \
  --set openclaw.gatewayTokenSecret=openclaw-gateway-token \
  --set openclaw.defaultModel=claude-sonnet-4-6
```

## Getting Started

### 1. Device pairing

When you access the control UI from a non-loopback address (e.g. through your gateway route), OpenClaw requires device pairing. This is a security feature -- every browser/device must be explicitly approved.

1. Open your OpenClaw URL -- you'll see "pairing required"
2. Approve it:

```bash
kubectl exec -it openclaw-0 -n openclaw -c openclaw -- openclaw devices list
kubectl exec -it openclaw-0 -n openclaw -c openclaw -- openclaw devices approve <requestId>
```

This is a one-time step per device. Approved devices stay approved.

See the [OpenClaw device pairing docs](https://docs.openclaw.ai/channels/pairing) for more details.

### 2. You're done

Open the control UI and start building.

## How config management works

OpenClaw stores its runtime config at `~/.openclaw/openclaw.json`. It reads *and writes* to this file at runtime (device pairings, sessions, etc.), so we can't just mount a ConfigMap over it.

This chart uses an **init container** that seeds the config on first boot only:

1. If `openclaw.json` doesn't exist on the PV, copy the managed config (gateway bind, rate limiting, allowed origins, trusted proxies) and inject the auth token
2. If it already exists, do nothing -- your runtime state is preserved

To re-seed after changing values (e.g. `trustedProxies`, `controlUiAllowedOrigins`), delete the PVC and let it recreate, or `kubectl exec` in and edit the file directly.

The `checksum/config` pod annotation triggers a restart whenever the ConfigMap changes, so your init container always runs with the latest managed config.

## How the gateway exposure works

OpenClaw's gateway binds to `127.0.0.1` by default -- fine for local use, but means Kubernetes Services and health probes can't reach it.

This chart seeds `openclaw.json` with `gateway.bind: "lan"` (listen on `0.0.0.0:18789`) on first boot. It also injects the gateway auth token, control UI allowed origins, trusted proxies, and rate limiting into the initial config.

When `gateway.enabled: true`, an [HTTPRoute](https://gateway-api.sigs.k8s.io/) is created:

```yaml
gateway:
  enabled: true
  parentRef:
    name: my-gateway
    namespace: gateway-ns
  hostname: openclaw.example.com
```

### Trusted proxies

If you're running behind a reverse proxy or ingress controller, OpenClaw needs to know which IPs to trust for `X-Forwarded-For` header processing. Use the pod CIDR (not specific pod IPs, which change):

```yaml
openclaw:
  trustedProxies:
    - "10.8.0.0/24"
```

## Browser sidecar

OpenClaw's browser tool needs a Chromium instance to connect to via [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) (CDP). This chart runs a **headless Chromium sidecar** using [`chromedp/headless-shell`](https://github.com/chromedp/docker-headless-shell) -- a minimal headless Chrome image built for CDP automation.

When `browser.enabled: true`, the sidecar exposes CDP on `localhost:9222` within the pod. The managed config seeds OpenClaw with `attachOnly: true` and a `cdpUrl` pointing to the sidecar -- so it never tries to find or launch a local browser binary.

The browser runs as a non-root user (UID 999) with all capabilities dropped. A tmpfs mount at `/dev/shm` (default 1Gi) provides the shared memory Chromium needs for rendering.

```yaml
browser:
  enabled: true
  resources:
    limits:
      cpu: "1"
      memory: 1Gi
  shmSize: 2Gi  # increase if pages are complex
```

## Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | OpenClaw image | `ghcr.io/openclaw/openclaw` |
| `image.tag` | Image tag | Chart `appVersion` |
| `openclaw.apiKeySecret` | Secret containing the LLM API key | `""` |
| `openclaw.apiKeySecretKey` | Key within the API key secret | `"api-key"` |
| `openclaw.apiKeyEnvVar` | Env var name (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.) | `"ANTHROPIC_API_KEY"` |
| `openclaw.defaultModel` | Default LLM model | `""` |
| `openclaw.gatewayTokenSecret` | Secret containing the gateway auth token | `""` |
| `openclaw.gatewayTokenSecretKey` | Key within the gateway token secret | `"gateway-token"` |
| `openclaw.controlUiAllowedOrigins` | Allowed origins for the control UI CORS | `[]` |
| `openclaw.trustedProxies` | Trusted proxy CIDRs for X-Forwarded-For processing | `[]` |
| `openclaw.env` | Additional environment variables | `[]` |
| `gateway.enabled` | Create a Gateway API HTTPRoute | `false` |
| `gateway.parentRef.name` | Parent gateway name | `""` |
| `gateway.parentRef.namespace` | Parent gateway namespace | `""` |
| `gateway.hostname` | Hostname for the route | `""` |
| `networkPolicy.enabled` | Default-deny NetworkPolicy (DNS + HTTPS egress only) | `false` |
| `networkPolicy.additionalEgress` | Extra egress rules | `[]` |
| `persistence.enabled` | Enable persistent storage for `~/.openclaw` | `true` |
| `persistence.size` | PVC size | `10Gi` |
| `persistence.storageClassName` | Storage class | `""` |
| `persistence.volumeName` | Bind to a specific PersistentVolume | `""` |
| `podman.enabled` | Enable rootless Podman sidecar for sandboxing | `false` |
| `podman.storageSize` | Ephemeral storage for pulled container images | `10Gi` |
| `podman.resources` | Podman sidecar resource limits | `{}` |
| `browser.enabled` | Enable headless Chromium sidecar for browser tool | `false` |
| `browser.image.repository` | Browser image | `chromedp/headless-shell` |
| `browser.image.tag` | Browser image tag | `"latest"` |
| `browser.resources` | Browser sidecar resource limits | `{}` |
| `browser.shmSize` | Shared memory size for Chromium (`/dev/shm` tmpfs) | `1Gi` |

### Example: full setup with sandboxing and browser

```yaml
openclaw:
  apiKeySecret: openclaw-api-key
  apiKeyEnvVar: ANTHROPIC_API_KEY
  defaultModel: claude-sonnet-4-6
  gatewayTokenSecret: openclaw-gateway-token
  controlUiAllowedOrigins:
    - "https://openclaw.example.com"
  trustedProxies:
    - "10.200.0.0/24"

gateway:
  enabled: true
  parentRef:
    name: shared-gateway
    namespace: gateway-ns
  hostname: openclaw.example.com

networkPolicy:
  enabled: true

podman:
  enabled: true
  resources:
    limits:
      cpu: "2"
      memory: 2Gi
  storageSize: 20Gi

browser:
  enabled: true
  resources:
    limits:
      cpu: "1"
      memory: 1Gi
```
## License

MIT
