# arguz-agent Helm Chart

The **Arguz Agent** chart deploys the in-cluster agents used by Arguz to collect runtime data and execute operational automations. Today it bundles:

- `Discovery-Agent` for cluster inventory, topology and runtime visibility
- `Scaling-Rules-Agent` for applying and reverting Arguz scaling templates through HPAs

---

## Installation

### Install from Helm repo (GitHub Pages)

Chart version: 1.0.1

```bash
helm repo add arguz-agent https://Arguz-Labs.github.io/Arguz-Agent-Chart
helm repo update
helm upgrade --install arguz-agent arguz-agent/arguz-agent \
  --version 1.0.1 \
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

## Notes

- A Kubernetes secret (`credentialsSecretName`) is shared by both agents to store `PROJECT_ID`, `CLUSTER_ID` and `CLUSTER_TOKEN`.
- Ensure your cluster has the required RBAC permissions for both agents to run properly.
- It is recommended to manage credentials using environment variables or secret management systems instead of hardcoding them.
