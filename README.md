# wlow-charts

Helm charts for deploying [wlow](https://github.com/wlow/wlow-core) on Kubernetes.

## Charts

| Chart | Description |
|-------|-------------|
| `charts/wlow` | Control plane + microVM runner |

## Install

```sh
helm repo add wlow https://wlow-io.github.io/wlow-charts
helm install wlow wlow/wlow \
  --namespace wlow \
  --create-namespace \
  --set nats.url=nats://nats.nats:4222 \
  --set image.repository=ghcr.io/wlow/wlow-runtime \
  --set image.tag=latest
```

For microVM runners, also set your OCI registry:

```sh
helm install wlow wlow/wlow \
  --set runner.registry=ghcr.io/your-org/wlow-artifacts \
  --set runner.snapshotRegistry=ghcr.io/your-org/wlow-snapshots \
  --set runner.imagePullAuth.secretName=oci-pull-auth
```

## Configuration

See `charts/wlow/values.yaml` for all available options and their defaults.

Key values:

| Value | Default | Description |
|-------|---------|-------------|
| `image.repository` | `ghcr.io/wlow-io/wlow-runtime` | Runtime image |
| `image.tag` | Chart `appVersion` | Image tag |
| `nats.url` | `nats://nats.nats:4222` | NATS server URL |
| `controlPlane.replicaCount` | `1` | Control plane replicas |
| `runner.kind` | `Deployment` | `Deployment` or `DaemonSet` |
| `runner.runtimes` | `microvm,snapshot` | Runtimes to serve |
| `runner.kvm.enabled` | `true` | Enable KVM node selection |
| `runner.registry` | `""` | OCI registry for artifacts |
| `mcp.enabled` | `false` | Deploy the MCP server |
| `buildkit.enabled` | `false` | Deploy BuildKit |
| `nats.cluster.enabled` | `false` | Deploy an embedded NATS cluster |

### Enable the MCP server (AI agent integration)

```yaml
mcp:
  enabled: true
  service:
    type: ClusterIP   # or LoadBalancer to expose externally
```

Port-forward and connect from Cursor / Claude Desktop:

```sh
kubectl -n wlow port-forward svc/wlow-mcp 8088:8088
# Add http://localhost:8088/mcp to your MCP client config
```

See [wlow-core docs/mcp.md](https://github.com/wlow-io/wlow-core/blob/main/docs/mcp.md) for the full tool reference.

### Enable BuildKit (required for `wlow push --runtime microvm`)

```yaml
buildkit:
  enabled: true
  rootless: true          # false = privileged mode (faster builds)
  persistence:
    enabled: true
    size: 50Gi
```

Port-forward for local use:

```sh
kubectl -n wlow port-forward svc/wlow-buildkit 1234:1234
export BUILDKIT_HOST=tcp://127.0.0.1:1234
wlow push --id my-proc --runtime microvm ...
```

### Embedded NATS cluster (dev/staging)

For production, use the [official NATS Helm chart](https://github.com/nats-io/k8s/tree/main/helm/charts/nats) directly.

```yaml
nats:
  cluster:
    enabled: true
    replicaCount: 3
    jetstream:
      fileStorage: 20Gi
      memStorage: 2Gi
```

### Process/WASM-only runner (no KVM)

```yaml
runner:
  runtimes: "process,wasm"
  kvm:
    enabled: false
```

### DaemonSet mode (one runner per KVM node)

```yaml
runner:
  kind: DaemonSet
```

## KVM node setup

Label nodes with KVM access:

```sh
kubectl label node <node-name> wlow.io/kvm=true
kubectl taint node <node-name> wlow.io/kvm=true:NoSchedule
```

See the [KVM setup guide](https://github.com/wlow/wlow-core/blob/main/docs/runner-setup.md) for GCP, AWS, and Linux workstation instructions.
