# Arguz Agent Helm Chart

The **Arguz Agent** is responsible for collecting and structuring runtime data from your cluster, enabling deep visibility into service behavior and interactions. It runs inside your environment and automatically gathers insights about how services communicate, helping you understand dependencies, detect issues, and analyze real system behavior without requiring manual instrumentation.

---

## Installation

Deploy the Arguz Agent using Helm:

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

## Notes

- A Kubernetes secret (`credentialsSecretName`) will be used to store the provided credentials securely.
- Ensure your cluster has the required permissions for the agent to run properly.
- It is recommended to manage credentials using environment variables or secret management systems instead of hardcoding them.
