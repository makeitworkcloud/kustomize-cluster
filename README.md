# kustomize-cluster

Kustomize configurations for OpenShift cluster workloads. Uses ArgoCD sync waves and KSOPS for secret decryption.

## Structure

```
bootstrap/          # Wave 0: ArgoCD config, Wave 1: workloads app
workloads/          # App-of-apps aggregating all workloads
├── cert-manager/
├── arc/            # GitHub Actions Runner Controller
└── ansible/        # AWX Operator
```

## Bootstrap Flow

1. Ansible installs GitOps operator and creates `gitops-bootstrap` Application
2. Wave 0: KSOPS patch, ClusterRoleBinding, wait for repo-server
3. Wave 1: `gitops-workloads` Application deploys all workloads

## Requirements

- OpenShift GitOps operator
- `sops-age-keys` secret in `openshift-gitops` namespace (for SOPS decryption)
