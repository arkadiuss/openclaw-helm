# OpenClaw Helm Chart

A Helm chart for deploying [OpenClaw](https://openclaw.ai) on Kubernetes with optional rootless Podman sandboxing.

## Overview

OpenClaw is an open-source AI agent platform. This chart deploys it as a **StatefulSet** (single replica) with persistent storage, Gateway API routing, network policies, and an optional rootless Podman sidecar for secure tool sandboxing.

### Why a StatefulSet?

OpenClaw holds in-memory WebSocket state and a Lane Queue, so it cannot be scaled horizontally. The StatefulSet with `volumeClaimTemplates` ensures stable identity and persistent storage for `~/.openclaw`.

## Prerequisites

- Kubernetes >= 1.25
- Helm >= 3
- A Gateway API implementation (e.g. NGINX Gateway Fabric) if using `gateway.enabled`

For Podman sandboxing, additional requirements apply (see [Sandboxing with Podman](#sandboxing-with-podman)).

## Quick Start

```bash
# Create namespace
kubectl create namespace openclaw

# Create the API key secret
kubectl create secret generic openclaw-api-key \
  --namespace openclaw \
  --from-literal=api-key=sk-ant-...

# Create the gateway token secret
kubectl create secret generic openclaw-gateway-token \
  --namespace openclaw \
  --from-literal=gateway-token=$(openssl rand -hex 32)

# Install the chart
helm install openclaw ./openclaw-helm \
  --namespace openclaw \
  --set openclaw.apiKeySecret=openclaw-api-key \
  --set openclaw.gatewayTokenSecret=openclaw-gateway-token \
  --set openclaw.defaultModel=claude-sonnet-4-6
```

## Onboarding

OpenClaw requires a one-time CLI onboarding before the web UI is usable. After the pod is running:

```bash
kubectl port-forward -n openclaw svc/openclaw 18789:18789

# In another terminal
curl http://localhost:18789/healthz  # verify it's up
```

Then open `http://localhost:18789` in your browser to complete the setup wizard, or use the CLI:

```bash
kubectl exec -it openclaw-0 -n openclaw -c openclaw -- node dist/index.js onboard
```

## Device Pairing

When accessing the control UI from a non-loopback address (e.g. via a gateway route), each device must be paired:

1. Open the control UI URL -- it shows "pairing required"
2. Approve the device from the pod:

```bash
kubectl exec -it openclaw-0 -n openclaw -c openclaw -- node dist/index.js devices list
kubectl exec -it openclaw-0 -n openclaw -c openclaw -- node dist/index.js devices approve <requestId>
```

Approved devices stay approved permanently. This is a one-time step per device.

## Gateway Exposure

The chart automatically configures `gateway.bind: "lan"` in `openclaw.json` so the gateway listens on `0.0.0.0:18789` instead of loopback. This is required for Kubernetes Services and health probes to reach the process.

When `gateway.enabled: true`, an HTTPRoute is created using the [Gateway API](https://gateway-api.sigs.k8s.io/):

```yaml
gateway:
  enabled: true
  parentRef:
    name: my-gateway
    namespace: gateway-ns
  hostname: openclaw.example.com
```

## Sandboxing with Podman

OpenClaw can sandbox tool execution in containers. This chart uses a **rootless Podman sidecar** instead of Docker-in-Docker, leveraging Kubernetes [User Namespaces](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/) for security.

### How it works

When `podman.enabled: true`:

1. The pod runs with `hostUsers: false` (Kubernetes User Namespaces), mapping container UIDs to unprivileged host UIDs
2. A Podman sidecar starts `podman system service` on a Unix socket
3. The OpenClaw container connects to the socket at `/run/podman/podman.sock`
4. Podman uses the **VFS storage driver** (overlay fails with emptyDir mount propagation)

### Cluster requirements

| Component | Minimum Version |
|-----------|----------------|
| Kubernetes | >= 1.33 |
| containerd | >= 2.0 |
| Linux kernel | >= 6.3 |

The node must have User Namespaces enabled. Verify with:

```bash
kubectl get node <node> -o jsonpath='{.status.features.userNamespaces}'
# Should return: true
```

### Security model

The Podman sidecar runs with these capabilities (scoped to the user namespace, not the host):

| Capability | Why |
|-----------|-----|
| `SYS_ADMIN` | Required for creating nested user/mount namespaces inside rootless containers |
| `SETUID` | Required by `newuidmap` to set up UID mappings (setuid binary) |
| `SETGID` | Required by `newgidmap` to set up GID mappings (setuid binary) |

All other capabilities are dropped. The seccomp profile is set to `Unconfined` because the default containerd profile blocks `CLONE_NEWUSER`, which is required for rootless containers.

**Why this is safe:** With `hostUsers: false`, these capabilities only apply within the pod's user namespace. On the host, the process runs as an unprivileged user (e.g. UID 524288+). `SYS_ADMIN` inside the user namespace cannot escape to the host.

### Enable sandboxing

```yaml
podman:
  enabled: true
  resources:
    limits:
      cpu: "1"
      memory: 1Gi
  storageSize: 20Gi  # ephemeral storage for container images
```

## Configuration

### Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | OpenClaw image | `ghcr.io/openclaw/openclaw` |
| `image.tag` | Image tag | Chart `appVersion` |
| `openclaw.apiKeySecret` | Secret name containing the LLM API key | `""` |
| `openclaw.apiKeySecretKey` | Key within the secret | `"api-key"` |
| `openclaw.apiKeyEnvVar` | Env var name for the key (e.g. `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`) | `"ANTHROPIC_API_KEY"` |
| `openclaw.defaultModel` | Default LLM model | `""` |
| `openclaw.gatewayTokenSecret` | Secret name containing the gateway auth token | `""` |
| `openclaw.gatewayTokenSecretKey` | Key within the secret | `"gateway-token"` |
| `openclaw.controlUiAllowedOrigins` | Allowed origins for the control UI | `[]` |
| `openclaw.env` | Additional environment variables | `[]` |
| `gateway.enabled` | Create a Gateway API HTTPRoute | `false` |
| `gateway.parentRef.name` | Parent gateway name | `""` |
| `gateway.parentRef.namespace` | Parent gateway namespace | `""` |
| `gateway.hostname` | Hostname for the HTTPRoute | `""` |
| `networkPolicy.enabled` | Create a NetworkPolicy (default-deny with DNS/HTTPS egress) | `false` |
| `networkPolicy.additionalEgress` | Additional egress rules | `[]` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `persistence.size` | PVC size | `10Gi` |
| `persistence.storageClassName` | Storage class | `""` |
| `persistence.volumeName` | Bind to a specific PV | `""` |
| `podman.enabled` | Enable rootless Podman sidecar | `false` |
| `podman.storageSize` | Ephemeral storage for container images | `10Gi` |
| `podman.resources` | Podman sidecar resource limits | `{}` |

### Example: Full production setup

```yaml
openclaw:
  apiKeySecret: openclaw-api-key
  apiKeyEnvVar: ANTHROPIC_API_KEY
  defaultModel: claude-sonnet-4-6
  gatewayTokenSecret: openclaw-gateway-token
  controlUiAllowedOrigins:
    - "https://openclaw.example.com"

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

persistence:
  size: 10Gi
```

## Upgrading

Since this chart uses a StatefulSet with `volumeClaimTemplates`, the PVC spec is immutable after creation. If you need to change storage settings:

```bash
helm uninstall openclaw -n openclaw
kubectl delete pvc openclaw-data-openclaw-0 -n openclaw
# If using a static PV, release it:
kubectl patch pv <pv-name> --type=json -p '[{"op":"remove","path":"/spec/claimRef"}]'
# Then reinstall
helm install openclaw ./openclaw-helm -n openclaw -f values.yaml
```

## Network Policy

When `networkPolicy.enabled: true`, a default-deny policy is created that allows:

- **Ingress**: Only from the gateway namespace on the service port
- **Egress**: DNS (TCP/UDP 53) and HTTPS (TCP 443) for LLM API calls

Additional egress rules can be added via `networkPolicy.additionalEgress`.

## License

MIT
