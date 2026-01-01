# Vault Fauxprise

[![License: MPL 2.0](https://img.shields.io/badge/License-MPL%202.0-brightgreen.svg)](https://opensource.org/licenses/MPL-2.0)

Kubernetes CronJobs for [HashiCorp Vault](https://developer.hashicorp.com/vault) operations usually reserved for _enterprise_ installations, such as [automated snapshots](https://developer.hashicorp.com/vault/docs/sysadmin/snapshots/automate) and [root credential rotation](https://developer.hashicorp.com/vault/docs/secrets/aws#schedule-based-root-credential-rotation).

## Features

- **Automated Vault Snapshots**: Scheduled backups with configurable retention
- **Root Credential Rotation**: Automatic rotation of root credentials for auth backends (AWS, Azure, etc.)
- **ConfigMap-Driven Config**: Clean separation of configuration from deployment
- **Multi-Environment Support**: Kustomize examples for dev/prod
- **GitOps Friendly**: All config version-controlled and auditable

## Configuration

### Common ConfigMap (`common-configmap.yaml`)

Shared across all features:
- `VAULT_ADDR`: Vault server URL
- `VAULT_AUTH_PATH`: Kubernetes auth mount path

### Snapshot ConfigMap (`snapshot/configmap.yaml`)

Snapshot-specific settings:
- `VAULT_ROLE`: Kubernetes role for authentication
- `SNAPSHOT_DIR`: Directory to store snapshots
- `SNAPSHOT_RETENTION_DAYS`: Days to keep snapshots
- `SNAPSHOT_FILENAME_PREFIX`: Prefix for snapshot files
- `SNAPSHOT_FORMAT`: Archive format (tgz)

### Token Rotation ConfigMap (`token-rotation/configmap.yaml`)

Root credential rotation settings:
- `VAULT_ROLE`: Kubernetes role for authentication
- `ROTATION_PATHS`: Comma-separated list of rotation endpoints (e.g., `/auth/aws/config/rotate-root`, `/auth/azure/rotate-root`)

## Deployment

### Prerequisites

- Kubernetes cluster
- `kubectl` installed
- `kustomize` installed (or use `kubectl -k`)
- Vault server accessible from the cluster
- Vault Kubernetes auth method configured with appropriate roles and policies

<details>
<summary><b>Deploy with Kustomize</b></summary>

### Using Kustomize

Create your own overlay by copying the examples and customizing to your environment:

```bash
mkdir -p k8s/overlays/my-production
cd k8s/overlays/my-production
```

Create a `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: hashicorp-vault

resources:
  - https://github.com/edeckers/vault-fauxprise/tree/v1.0.0/k8s/base

patches:
  - path: common-config-patch.yaml
  - path: snapshot-config-patch.yaml
  - path: snapshot-cronjob-patch.yaml
  - path: snapshot-pvc-patch.yaml
  - path: token-rotation-config-patch.yaml
  - path: token-rotation-cronjob-patch.yaml

labels:
  - pairs:
      environment: production
```

Create your patches (see [`k8s/examples`](k8s/examples) for inspiration):

```yaml
# common-config-patch.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vault-fauxprise-common
data:
  VAULT_ADDR: "https://vault.example.com"
```

Deploy:

```bash
kubectl apply -k k8s/overlays/my-production
```

This will deploy:
- **Snapshot CronJob**: Runs daily at 2 AM, saves Raft snapshots with configurable retention
- **Token Rotation CronJob**: Runs every 6 hours, rotates root credentials for configured auth backends (AWS, Azure, etc.)

</details>

<details>
<summary><b>Deploy with ArgoCD</b></summary>

### Using ArgoCD

Create your Kustomize overlay as described above, commit it to your GitOps repository, then create an Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vault-fauxprise
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/your-gitops-repo
    path: k8s/overlays/production  # Path to your overlay
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: hashicorp-vault
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

ArgoCD will continuously sync your configuration and automatically deploy:
- **Snapshot CronJob**: Daily backups at 2 AM with retention cleanup
- **Token Rotation CronJob**: Root credential rotation every 6 hours for configured auth methods

</details>

## Customization

### Changing Configuration

Edit the appropriate ConfigMap or create a patch in your overlay:

```yaml
# examples/production/snapshot-config-patch.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vault-snapshot-config
data:
  SNAPSHOT_RETENTION_DAYS: "30"  # Override for production
```

### Changing Schedule

Patch the CronJob schedule:

```yaml
# examples/production/snapshot-cronjob-patch.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vault-snapshot
spec:
  schedule: "0 */6 * * *"
```

## Storage

Snapshots are stored in a `PersistentVolumeClaim`. Adjust the storage class and size in `k8s/base/snapshot/pvc.yaml` based on your cluster configuration.

## Security Considerations

- Each feature uses its own ServiceAccount for Vault Kubernetes authentication
- Vault authentication via Kubernetes auth method
- Root credentials are rotated automatically for configured auth backends

## License

[MPL-2.0](LICENSE)