# arguz-agent Helm Chart

The **Arguz Agent** chart deploys the in-cluster agents used by Arguz to collect runtime data and execute operational automations. Today it bundles:

- `Discovery-Agent` for cluster inventory, topology and runtime visibility
- `Scaling-Rules-Agent` for applying and reverting Arguz scaling templates through HPAs
- `Arguz-Node-Agent` for node-level eBPF telemetry

---

## Installation

### Install from Helm repo (GitHub Pages)

Chart version: 1.2.1

```bash
helm repo add arguz-agent https://Arguz-Labs.github.io/Arguz-Agent-Chart
helm repo update
helm upgrade --install arguz-agent arguz-agent/arguz-agent \
  --version 1.2.1 \
  -n arguz-agent \
  --create-namespace
```

### Install from local chart

Deploy the arguz-agent using Helm:

```bash
helm upgrade --install arguz-agent . \
  -n arguz-agent \
  --create-namespace
```

## Configuration

The agent requires cluster credentials to authenticate and associate the collected data with your Arguz project. These credentials can be provided either directly via CLI or through environment variables.

### Option 1: Pass credentials via CLI

```bash
helm upgrade --install arguz-agent . \
  -n arguz-agent \
  --create-namespace \
  --set-string global.credentialsSecretName=arguz-credentials \
  --set-string global.clusterCredentials.projectId=<PROJECT_ID> \
  --set-string global.clusterCredentials.clusterId=<CLUSTER_ID> \
  --set-string global.clusterCredentials.clusterToken=<CLUSTER_TOKEN>
```

### Optional: Disable the scaling rules agent

```bash
helm upgrade --install arguz-agent . \
  -n arguz-agent \
  --create-namespace \
  --set Scaling-Rules-Agent.enabled=false
```

### Scaling rules agent settings

The scaling rules agent reuses the same cluster credentials secret and additionally needs the Scaling Rules API URL. Default values:

```yaml
Scaling-Rules-Agent:
  enabled: true
  env:
    API_URL: https://api-scaling-rules.arguz.io
    LOG_LEVEL: info
    IGNORE_RESOURCES: ""
```

### Arguz Node Agent

`Arguz-Node-Agent` runs as a DaemonSet on Linux worker nodes and is enabled by
default. It reuses `global.credentialsSecretName` and the existing `CLUSTER_ID`
and `CLUSTER_TOKEN` keys to send events, traces, and metrics to Arguz Ingestion.
It does not use Cloudflare credentials or extra authentication headers.

The agent needs host PID and network namespaces, privileged eBPF access, and
host filesystem mounts for procfs, cgroups, kubelet, container runtime, tracefs,
and debugfs. Its init container may set the node-wide
`kernel.perf_event_paranoid` sysctl to `-1` so kprobes and uprobes can attach.
Install it only in a namespace and on nodes where this privileged access is
approved.

The default scheduler policy targets Linux workers and excludes control-plane
and master nodes. To include those nodes explicitly:

```yaml
Arguz-Node-Agent:
  daemonset:
    scheduleControlPlane: true
```

To disable the Node Agent for an installation:

```yaml
Arguz-Node-Agent:
  enabled: false
```

The first installation or upgrade that introduces the Node Agent requires an
administrator-run Helm upgrade. Discovery self-upgrade intentionally cannot
create new cluster-scoped RBAC or DaemonSet resources. Later upgrades can
reconcile the existing Node Agent resources through restricted name allowlists.
That bootstrap also grants Discovery the same read permissions delegated to the
Node Agent, which Kubernetes requires before it allows Discovery to update the
Node Agent ClusterRole or ClusterRoleBinding. A custom PriorityClass name still
requires an administrator bootstrap because the self-upgrade allowlist uses the
default release-derived PriorityClass name.

Container log uploading is not enabled by this chart. The agent still mounts
`/var/log` read-only for workload attribution and metrics. eBPF collection can
observe process metadata, network traffic, SQL/query data, URL paths, and stack
context; assess the data-handling policy before enabling it in production.

### Discovery agent self-upgrade

`Discovery-Agent` can use the Helm SDK to poll and upgrade its own release. It
is enabled by default for new installations. It creates release-scoped
Role/RoleBinding resources and the minimum ClusterRole/ClusterRoleBinding rules
required to update the Discovery agent's existing cluster RBAC. A catalog
assignment supplies the desired release, while a local `enabled: false` remains
an explicit opt-out for that installation/cluster.

```yaml
Discovery-Agent:
  selfUpgrade:
    enabled: false # Explicitly opt out of automatic desired-release installs.
    pollInterval: 5m
    chartRef: oci://ghcr.io/arguz-labs/arguz-agent
    timeout: 5m
    lock:
      # Empty values derive a stable name and use the release namespace.
      name: ""
      namespace: ""
    # Applied only while self-upgrade is enabled.
    rollout:
      terminationGracePeriodSeconds: 120
      preStopDelaySeconds: 30
```

The agent receives the chart reference, release identity, timing, and lock
settings through environment variables and its mounted `self-upgrade.yaml`
configuration file. `chartRef` is a normal Helm OCI/chart reference; the chart
does not impose a static registry repository allowlist.

When enabled, self-upgrades use a `RollingUpdate` strategy with `maxSurge: 1`
and `maxUnavailable: 0`. The old leader is kept alive for the configurable
`preStopDelaySeconds` through Kubernetes' native lifecycle `sleep` handler,
which does not require a shell in the distroless image. The default
`terminationGracePeriodSeconds` is 120 seconds; keep it longer than the
pre-stop delay to allow a ready replacement pod to assume leadership first.

Existing releases that persist `selfUpgrade.enabled: false` do not change
silently. Bootstrap them once with a self-upgrade-capable chart version and an
explicit value:

```bash
helm upgrade arguz-agent oci://ghcr.io/arguz-labs/arguz-agent \
  --namespace arguz-agent \
  --reuse-values \
  --set Discovery-Agent.selfUpgrade.enabled=true
```

Before enabling self-upgrade, verify the Discovery Agent ServiceAccount can
operate on the release resources it already owns. Replace the namespace or
release name when customized:

```bash
SA=system:serviceaccount:arguz-agent:arguz-agent-discovery-agent

kubectl auth can-i get secrets --as="$SA" -n arguz-agent
kubectl auth can-i create secrets --as="$SA" -n arguz-agent
kubectl auth can-i patch deployment/arguz-agent-discovery-agent --as="$SA" -n arguz-agent
kubectl auth can-i patch deployment/arguz-agent-scaling-rules-agent --as="$SA" -n arguz-agent
kubectl auth can-i create leases/arguz-agent-discovery-agent-self-upgrade --as="$SA" -n arguz-agent
kubectl auth can-i patch clusterrole/arguz-agent-discovery-agent --as="$SA"
kubectl auth can-i patch clusterrolebinding/arguz-agent-discovery-agent --as="$SA"
```

The chart intentionally does not grant unrestricted creation or deletion of
cluster-scoped RBAC resources. A future chart that introduces new cluster-scoped
resources requires an administrator bootstrap upgrade instead of allowing the
Agent to escalate its own privileges.

### Discovery agent image and health settings

Images use `repository:tag` by default. To pin an image by digest, set
`Discovery-Agent.image.digest`; it renders as `repository@digest` and takes
precedence over `tag`.

The distroless Discovery Agent serves `/healthz` on its named `health` port
(`8080`). Default readiness and liveness probes use that endpoint. The listener
defaults to `HEALTH_ADDR=0.0.0.0:8080` and can be changed with
`Discovery-Agent.env.HEALTH_ADDR`; when changing it, keep the health endpoint
reachable on the container's named port. The probes, `podSecurityContext`, and
`securityContext` can be overridden through `Discovery-Agent` values.

## Notes

- A Kubernetes secret (`credentialsSecretName`) is shared by both agents to store `PROJECT_ID`, `CLUSTER_ID` and `CLUSTER_TOKEN`.
- Ensure your cluster has the required RBAC permissions for both agents to run properly.
- It is recommended to manage credentials using environment variables or secret management systems instead of hardcoding them.
