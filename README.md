# openclaw-helm

A Helm chart for running [OpenClaw](https://openclaw.ai) on Kubernetes: config seeding, Gateway API routing, network policies, a headless browser sidecar, and first-class support for [NVIDIA OpenShell](https://github.com/NVIDIA/OpenShell) sandboxing.

Built for homelabs. Single node, single replica, and biased toward the secure option wherever there's a choice (yes, we know it's OpenClaw - that's exactly why).

## Why another Helm chart for OpenClaw?

There are a couple of community charts already. This one exists because running OpenClaw on Kubernetes has a set of operational details the official docs don't cover, and because it takes sandboxing seriously enough to have found out what actually works.

The details: OpenClaw reads *and writes* its own config file at runtime, so a mounted ConfigMap is not an option. It binds to loopback by default, so Services and probes can't reach it. It needs a browser it can attach to over CDP. It needs to know which proxies to trust for `X-Forwarded-For`. None of that is hard, all of it is fiddly, and this chart has it wired up.

The sandboxing: OpenClaw executes model-directed shell commands, and on a box that also runs your other homelab services you want a real boundary around that. See [Sandboxing](#sandboxing) below - including an honest account of which options in this chart actually work today.

## Why Kubernetes? OpenClaw doesn't officially support it.

Correct - OpenClaw is a single-instance service with in-memory WebSocket state and a Lane Queue. It can't scale horizontally, and the official docs point you at Docker Compose or bare metal.

But if you already run Kubernetes, a few things come for free that are annoying otherwise:

- **Persistent storage** that survives restarts without bind-mount gymnastics
- **Gateway API routing** with TLS termination through your existing ingress
- **Network policies** scoping what the agent can reach
- **Secret management** through native Kubernetes Secrets
- **Health checks** and automatic restarts when the gateway process dies

This chart deploys OpenClaw as a **StatefulSet** (single replica) with `volumeClaimTemplates` for stable identity and persistent `~/.openclaw` storage. It isn't pretending to be a distributed system - it's a well-managed single instance.

## Sandboxing

Two options, and they are not equivalent. Read this before choosing.

### OpenShell (`openshellCli` + `openshellPolicy`) - the one that works

[NVIDIA OpenShell](https://github.com/NVIDIA/OpenShell) is a separate, self-hosted sandbox runtime with a first-party OpenClaw plugin (`@openclaw/openshell-sandbox`, `backend: "openshell"`). Its Kubernetes compute driver provisions one sandbox pod per agent/session through the [agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox) controller, created on first tool use and pruned when idle. Isolation is enforced by an in-pod supervisor at the kernel boundary (Landlock filesystem and process policy, network policy at FQDN/port/L7/binary granularity), with gVisor or Kata available via RuntimeClass.

That's a materially stronger boundary than any container-capability arrangement, and it's the option to pick if the threat you care about is prompt injection turning into arbitrary code execution.

**You stand OpenShell up yourself.** It's a second control plane - its own gateway StatefulSet, the agent-sandbox CRDs and controller, a certgen Job, RBAC, per-sandbox workspace PVCs - and neither project publishes a resource-footprint baseline, so profile it on your own cluster first. Wiring all of that into this chart would make it a different project.

What this chart does supply is the two pieces that have to live next to the OpenClaw process:

**`openshellCli.enabled`** sideloads the `openshell` CLI that the plugin shells out to. The published OpenClaw image doesn't include it, so an initContainer downloads the pinned release and verifies its sha256 before extracting - provisioning time, not agent runtime, and it fails closed on a checksum mismatch.

**`openshellPolicy.enabled`** renders a sandbox policy into a ConfigMap and mounts it where the CLI reads it at sandbox-create time. Three things it does beyond the plumbing:

- **A `filesystem_policy` baseline that works.** OpenShell's implicit default omits `/dev/null`, which `git` opens on every invocation - so git fails inside the sandbox with a permission error that points nowhere near the real cause. Landlock is an allowlist and ignores permission bits, so `ls` will cheerfully show `/dev/null` as `crw-rw-rw-` while every open fails. Supply your own `network_policies` and the baseline merges in underneath. This should go away once upstream ships a default that works.
- **Policy changes roll the pod.** The rendered ConfigMap is hashed into the pod template. It's a `subPath` mount, which never picks up a ConfigMap update in place, so without the hash a policy edit would take effect at the next unrelated restart and not before.
- **The mount path is checked.** Anything under the workspace root fails at template time, for the same reason [`workspaceSeed`](#workspace-seeding) exists.

```yaml
openshellCli:
  enabled: true

openshellPolicy:
  enabled: true
  policy:
    network_policies:
      forge_git_http:
        name: Git over HTTP
        endpoints:
          - host: forge.internal
            port: 3000
            protocol: rest
            rules:
              - allow: { method: GET, path: /**/info/refs* }
              - allow: { method: POST, path: /**/git-upload-pack }
        binaries:
          - path: /usr/bin/git
```

Note the `binaries` list: policy entries are scoped to the binaries they name. If you debug a denial with `curl` or `dig`, those are different binaries and will be denied no matter what the policy says about the destination - which proves nothing about whether the allowed binary works.

Point the plugin at the mounted path in `extraConfig`:

```yaml
openclaw:
  extraConfig:
    agents:
      defaults:
        sandbox:
          mode: "all"
          backend: "openshell"
    plugins:
      entries:
        openshell:
          enabled: true
          config:
            command: "/opt/openshell/bin/openshell"
            policy: "/etc/openclaw/openshell-policy.yaml"
```

### Rootless Podman (`podman.enabled`) - isolation design is sound, wiring is not

This chart ships a rootless Podman sidecar using Kubernetes [User Namespaces](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/) (`hostUsers: false`) - no privileged container, no host Docker socket, no root on the host. As an isolation design that is a real improvement on the Docker-in-Docker and socket-mount patterns other charts use.

**It does not currently work end to end with the published OpenClaw image**, for two reasons worth stating plainly rather than leaving you to discover:

1. The sidecar sets `OPENCLAW_SANDBOX` and `OPENCLAW_DOCKER_SOCKET` on the openclaw container. Nothing in the OpenClaw gateway process reads either one - they're consumed by `scripts/docker/setup.sh` on the Docker Compose install path, which never runs here. The gateway resolves a container engine through `DOCKER_HOST` and the docker context instead.
2. The published image ships without a `docker` or `podman` CLI unless built with `OPENCLAW_INSTALL_DOCKER_CLI=1`, which the release images are not. There is nothing for the gateway to shell out to.

So `podman.enabled: true` gets you a working rootless Podman daemon in the pod that OpenClaw never talks to. Closing the gap means pointing `DOCKER_HOST` at the socket and supplying an image with a container CLI; neither is wired up here, and neither has been tested, so the chart doesn't claim it. Use OpenShell above, or treat this as the starting point for that work.

<details>
<summary>The isolation design, for when it is wired up</summary>

| Capability | Why it's needed |
|-----------|----------------|
| `SYS_ADMIN` | Creating nested user/mount namespaces inside rootless containers |
| `SETUID` | `newuidmap` is a setuid binary that maps UIDs into the user namespace |
| `SETGID` | `newgidmap` does the same for GIDs |

All other capabilities are dropped (`drop: ALL`). The seccomp profile is `Unconfined` because containerd's default profile blocks `CLONE_NEWUSER`, which rootless containers need.

**Why `Unconfined` is acceptable here:** with `hostUsers: false`, the container's root maps to an unprivileged host UID. Even without seccomp filtering the process cannot perform privileged operations on the host. The user namespace is the boundary, not seccomp.

Podman uses the **VFS storage driver** rather than overlay. Overlay requires remounting with `MS_PRIVATE` propagation, which fails on `emptyDir` volumes. VFS is slower but works.

Cluster requirements:

| Component | Minimum Version | Why |
|-----------|----------------|-----|
| Kubernetes | >= 1.33 | User Namespaces GA |
| containerd | >= 2.0 | User Namespace support in CRI |
| Linux kernel | >= 6.3 | `idmap` mounts for user namespaces |

```bash
kubectl get node <node> -o jsonpath='{.status.features.userNamespaces}'
# Should return: true
```

</details>

## Quick Start

```bash
kubectl create namespace openclaw

kubectl create secret generic openclaw-api-key \
  --namespace openclaw \
  --from-literal=api-key=sk-ant-...

kubectl create secret generic openclaw-gateway-token \
  --namespace openclaw \
  --from-literal=gateway-token=$(openssl rand -hex 32)

helm install openclaw ./openclaw-helm \
  --namespace openclaw \
  --set openclaw.apiKeySecret=openclaw-api-key \
  --set openclaw.gatewayTokenSecret=openclaw-gateway-token
```

Then set a model through `extraConfig` (see [Choosing a model](#choosing-a-model)) and pair your first device.

### Device pairing

Accessing the control UI from a non-loopback address - which is to say, through your gateway route - requires device pairing. Every browser must be explicitly approved.

```bash
kubectl exec -it openclaw-0 -n openclaw -c openclaw -- openclaw devices list
kubectl exec -it openclaw-0 -n openclaw -c openclaw -- openclaw devices approve <requestId>
```

One-time per device. See the [OpenClaw pairing docs](https://docs.openclaw.ai/channels/pairing).

## How config management works

OpenClaw stores its runtime config at `~/.openclaw/openclaw.json` and writes to it at runtime - device pairings, sessions, installed plugins. A mounted ConfigMap would be overwritten by the application, so this chart uses an **init container that seeds the file on first boot only**:

1. If `openclaw.json` doesn't exist on the PV, write the managed config (gateway bind, rate limiting, allowed origins, trusted proxies), deep-merge `openclaw.extraConfig` over it, and inject the auth token.
2. If it already exists, leave it completely alone.

### Known limitation: config is not updated after the first boot

**Changing values does not re-seed an existing install.** The `checksum/config` annotation restarts the pod when the ConfigMap changes, so the init container runs again - but it hits the same "does the file exist" check and does nothing. A `helm upgrade` that changes `extraConfig`, `trustedProxies`, or `controlUiAllowedOrigins` will appear to succeed and change nothing about how OpenClaw runs.

Until that is fixed, applying a config change to a live install means doing it by hand:

```bash
# edit in place
kubectl exec -it openclaw-0 -n openclaw -c openclaw -- openclaw config set <path> <value>
kubectl rollout restart statefulset/openclaw -n openclaw
```

or deleting the PVC and letting it re-seed from scratch, which also discards device pairings, sessions, and installed plugins.

Treat the values file as the source of truth and reconcile by hand when the two drift. This is being worked on - a future version will reconcile the managed config on every start rather than only on the first one, without clobbering the state OpenClaw owns.

### Choosing a model

Set it in `extraConfig`, using the provider-prefixed form:

```yaml
openclaw:
  extraConfig:
    agents:
      defaults:
        model:
          primary: "anthropic/claude-opus-5"
          fallbacks:
            - "anthropic/claude-sonnet-5"
```

`openclaw.defaultModel` sets an `OPENCLAW_MODEL` environment variable that the current gateway does not read. It's kept for compatibility and does nothing; use `extraConfig`.

### Workspace seeding

To put read-only reference material into the agent's workspace, use `workspaceSeed` rather than a volume mount:

```yaml
workspaceSeed:
  - name: reference
    configMap: my-reference-docs
    subPath: reference
```

The init container copies the ConfigMap's contents into `~/.openclaw/workspace/<subPath>` as plain files on every pod start.

**Never mount a ConfigMap or Secret under the workspace path directly.** A mirror-mode sandbox deletes and recreates the entire workspace on every sync. A mount point cannot be deleted, so the delete aborts partway, the restore never runs, and unrelated workspace content is destroyed along with it. `workspaceSeed` exists specifically to avoid that, and `openshellPolicy.mountPath` is validated against the same rule.

## How the gateway exposure works

OpenClaw's gateway binds to `127.0.0.1` by default, which means Services and health probes can't reach it. This chart seeds `gateway.bind: "lan"` (listen on `0.0.0.0:18789`) on first boot, along with the auth token, control UI allowed origins, trusted proxies, and rate limiting.

With `gateway.enabled: true` an [HTTPRoute](https://gateway-api.sigs.k8s.io/) is created:

```yaml
gateway:
  enabled: true
  parentRef:
    name: my-gateway
    namespace: gateway-ns
  hostname: openclaw.example.com
```

### Trusted proxies

Behind a reverse proxy or ingress controller, OpenClaw needs to know which IPs to trust for `X-Forwarded-For`. Use the pod CIDR, not individual pod IPs:

```yaml
openclaw:
  trustedProxies:
    - "10.8.0.0/24"
```

## Network policy

`networkPolicy.enabled: true` creates a default-deny policy for the OpenClaw pod: ingress only from the gateway namespace (when `gateway.enabled`), egress only to DNS and port 443, plus whatever you add in `additionalEgress`.

Be clear-eyed about what that buys you. Port 443 is unrestricted by destination, so this stops lateral movement inside the cluster and stops non-HTTPS egress - it is not an allowlist and it will not stop an agent from reaching an arbitrary host on the internet. The DNS rule is likewise unscoped by destination, because the resolver's namespace and labels vary by distribution. If you need real egress allowlisting, either narrow both rules for your cluster with `additionalEgress` and a `to` selector, or let OpenShell's L7 policy do it at the sandbox boundary, which is where the untrusted code actually runs.

```yaml
networkPolicy:
  enabled: true
  additionalEgress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: forge
      ports:
        - protocol: TCP
          port: 3000
```

## Browser sidecar

OpenClaw's browser tool needs a Chromium instance to attach to over [CDP](https://chromedevtools.github.io/devtools-protocol/). With `browser.enabled: true`, a headless [`chromedp/headless-shell`](https://github.com/chromedp/docker-headless-shell) sidecar exposes CDP on `localhost:9222` in the pod, and the managed config seeds `attachOnly: true` with a `cdpUrl` pointing at it - so OpenClaw never tries to find or launch a browser binary itself.

The browser runs as a non-root user (UID 999) with all capabilities dropped. A tmpfs at `/dev/shm` provides the shared memory Chromium needs.

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

Values that feed `openclaw.json` - `extraConfig`, `trustedProxies`, `controlUiAllowedOrigins` - only take effect on a fresh PV. See [the limitation above](#known-limitation-config-is-not-updated-after-the-first-boot) before changing one on a running install.

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Ignored; the StatefulSet is always single-replica | `1` |
| `image.repository` | OpenClaw image | `ghcr.io/openclaw/openclaw` |
| `image.tag` | Image tag | Chart `appVersion` |
| `image.pullPolicy` | Image pull policy | `Always` |
| `imagePullSecrets` | Image pull secrets | `[]` |
| `nameOverride` / `fullnameOverride` | Name overrides | `""` |
| `openclaw.apiKeySecret` | Secret containing the LLM API key | `""` |
| `openclaw.apiKeySecretKey` | Key within the API key secret | `"api-key"` |
| `openclaw.apiKeyEnvVar` | Env var name (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, ...) | `"ANTHROPIC_API_KEY"` |
| `openclaw.defaultModel` | Sets `OPENCLAW_MODEL`, which the gateway does not read - use `extraConfig` | `""` |
| `openclaw.gatewayTokenSecret` | Secret containing the gateway auth token | `""` |
| `openclaw.gatewayTokenSecretKey` | Key within the gateway token secret | `"gateway-token"` |
| `openclaw.controlUiAllowedOrigins` | Allowed origins for the control UI CORS | `[]` |
| `openclaw.trustedProxies` | Trusted proxy CIDRs for `X-Forwarded-For` | `[]` |
| `openclaw.env` | Additional environment variables (full env var specs) | `[]` |
| `openclaw.extraConfig` | `openclaw.json` fragment merged into the first-boot seed | `{}` |
| `service.type` / `service.port` | Service type and port | `ClusterIP` / `18789` |
| `gateway.enabled` | Create a Gateway API HTTPRoute | `false` |
| `gateway.parentRef.name` / `.namespace` | Parent gateway reference | `""` |
| `gateway.hostname` | Hostname for the route | `""` |
| `networkPolicy.enabled` | Default-deny NetworkPolicy (see caveats above) | `false` |
| `networkPolicy.additionalEgress` | Extra egress rules | `[]` |
| `persistence.enabled` | Persistent storage for `~/.openclaw` | `true` |
| `persistence.size` | PVC size | `10Gi` |
| `persistence.accessModes` | PVC access modes | `[ReadWriteOnce]` |
| `persistence.storageClassName` | Storage class | `""` |
| `persistence.volumeName` | Bind to a specific PersistentVolume | `""` |
| `podman.enabled` | Rootless Podman sidecar (not wired to the gateway - see above) | `false` |
| `podman.image.*` | Podman image, tag, pull policy | `quay.io/podman/stable:latest` |
| `podman.storageSize` | Ephemeral storage for pulled container images | `10Gi` |
| `podman.resources` | Podman sidecar resources | `{}` |
| `browser.enabled` | Headless Chromium sidecar for the browser tool | `false` |
| `browser.image.*` | Browser image, tag, pull policy | `chromedp/headless-shell:latest` |
| `browser.resources` | Browser sidecar resources | `{}` |
| `browser.shmSize` | Shared memory for Chromium (`/dev/shm` tmpfs) | `1Gi` |
| `openshellCli.enabled` | Sideload the `openshell` CLI via an initContainer | `false` |
| `openshellCli.version` | OpenShell release tag to fetch | `"0.0.106"` |
| `openshellCli.checksum` | sha256 of the release asset, verified before extraction | see `values.yaml` |
| `openshellCli.installPath` | Where the CLI lands in the container | `/opt/openshell/bin` |
| `openshellCli.image.*` / `.resources` | initContainer image and resources | `curlimages/curl:8.11.1` / `{}` |
| `openshellPolicy.enabled` | Render and mount an OpenShell sandbox policy | `false` |
| `openshellPolicy.mountPath` | Where the policy is mounted; rejected under the workspace root | `/etc/openclaw/openshell-policy.yaml` |
| `openshellPolicy.policy` | Policy document, merged over the chart's baseline | see `values.yaml` |
| `workspaceSeed` | ConfigMaps copied into the workspace as plain files | `[]` |
| `volumes` / `volumeMounts` | Additional volumes and mounts for the openclaw container | `[]` |
| `livenessProbe` / `readinessProbe` | Probe config; set to `null` to disable | `/healthz` / `{}` |
| `resources` | OpenClaw container resources | `{}` |
| `serviceAccount.create` / `.name` / `.annotations` | ServiceAccount handling | `false` / `""` / `{}` |
| `podSecurityContext` / `securityContext` | Pod and container security contexts | `{}` / non-root UID 1000 |
| `podAnnotations` / `podLabels` | Extra pod metadata | `{}` |
| `nodeSelector` / `tolerations` / `affinity` | Scheduling | `{}` / `[]` / `{}` |

Note: the pod always runs with `automountServiceAccountToken: false`. `serviceAccount.name` sets the identity but nothing in the pod can use it until that changes.

### Example: OpenShell sandboxing, browser, and a gateway route

```yaml
image:
  tag: "2026.9.4"
  pullPolicy: IfNotPresent

openclaw:
  apiKeySecret: openclaw-api-key
  apiKeyEnvVar: ANTHROPIC_API_KEY
  gatewayTokenSecret: openclaw-gateway-token
  controlUiAllowedOrigins:
    - "https://openclaw.example.com"
  trustedProxies:
    - "10.200.0.0/24"
  extraConfig:
    agents:
      defaults:
        model:
          primary: "anthropic/claude-opus-5"
        sandbox:
          mode: "all"
          backend: "openshell"
    tools:
      elevated:
        enabled: false
      exec:
        host: "sandbox"

gateway:
  enabled: true
  parentRef:
    name: shared-gateway
    namespace: gateway-ns
  hostname: openclaw.example.com

networkPolicy:
  enabled: true

openshellCli:
  enabled: true

openshellPolicy:
  enabled: true
  policy:
    network_policies: {}

browser:
  enabled: true
  resources:
    limits:
      cpu: "1"
      memory: 1Gi
```

`tools.exec.host: "sandbox"` is worth copying: the `auto` default silently falls back to running on the gateway pod whenever the sandbox runtime is unavailable, which is exactly when you least want it to. And `tools.elevated.enabled: false` matters because elevated exec leaves the sandbox regardless of which backend you configured.

## License

MIT
