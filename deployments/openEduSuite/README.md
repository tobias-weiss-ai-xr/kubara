# openEduSuite Deployment

This directory contains the ArgoCD Application definitions for the openEduSuite
platform — a GitOps-managed Kubernetes deployment for the educational suite at
Philipps-Universität Marburg (HRZ).

## Purpose

These manifests are used to test kubara's `bootstrap` command. They define the
ArgoCD app-of-apps pattern: once kubara bootstraps ArgoCD onto a cluster, ArgoCD
pulls the actual workload manifests from the GitLab repositories referenced by
these Applications.

## Architecture

```
                    ┌─────────────────────────────────────────┐
                    │           ArgoCD AppProject: scs         │
                    │  (allows all repos + namespaces)         │
                    └────────────────┬────────────────────────┘
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           │                         │                         │
     ┌─────▼─────┐           ┌───────▼───────┐         ┌───────▼───────┐
     │ scs-infra │           │ openedusuite  │         │ openedusuite-  │
     │           │           │               │         │ idp            │
     │ ansible-  │           │ openEduSuite- │         │ openEduSuite-  │
     │ baremetal-│           │ hrz/k8s/      │         │ hrz/k8s/       │
     │ k3s/k8s   │           │ openedusuite  │         │ openedusuite-  │
     └───────────┘           └───────────────┘         │ idp            │
                             ┌───────────────┐         └───────────────┘
                             │ openedusuite- │
                             │ staff         │
                             │ openEduSuite- │
                             │ hrz/k8s/      │
                             │ openedusuite- │
                             │ staff         │
                             └───────────────┘
                             ┌───────────────┐
                             │ openedusuite- │
                             │ students      │
                             │ openEduSuite- │
                             │ hrz/k8s/      │
                             │ openedusuite- │
                             │ students      │
                             └───────────────┘
                             ┌───────────────┐         ┌───────────────┐
                             │ operaton      │         │ operaton-cnpg │
                             │ openEduSuite- │         │ openEduSuite- │
                             │ hrz/k8s/      │         │ hrz/k8s/      │
                             │ operaton/     │         │ operaton      │
                             └───────────────┘         └───────────────┘
                             ┌───────────────┐         ┌───────────────┐
                             │ security-     │         │ security-     │
                             │ kyverno       │         │ policies      │
                             │ openEduSuite- │         │ openEduSuite- │
                             │ hrz/k8s/      │         │ hrz/k8s/      │
                             │ security/     │         │ security/     │
                             │ kyverno       │         │ policies      │
                             └───────────────┘         └───────────────┘
                             ┌───────────────┐
                             │ next-mailserver│
                             │ weissto/       │
                             │ next-mailserver│
                             └───────────────┘
```

## Applications

| App | Namespace | Source Repo | Path | Auto-Sync |
|-----|-----------|-------------|------|-----------|
| `scs-infra` | default | ansible-baremetal-k3s | `k8s` | ✅ prune+selfHeal |
| `openedusuite` | openedusuite | openEduSuite-hrz | `k8s/openedusuite` | ✅ prune+selfHeal |
| `openedusuite-idp` | openedusuite | openEduSuite-hrz | `k8s/openedusuite-idp` | ❌ manual |
| `openedusuite-staff` | openedusuite-staff | openEduSuite-hrz | `k8s/openedusuite-staff` | ✅ prune+selfHeal |
| `openedusuite-students` | openedusuite-students | openEduSuite-hrz | `k8s/openedusuite-students` | ✅ prune+selfHeal |
| `operaton` | openedusuite | openEduSuite-hrz | `k8s/operaton/overlay` | ✅ prune+selfHeal |
| `operaton-cnpg` | openedusuite | openEduSuite-hrz | `k8s/operaton` | ✅ prune+selfHeal |
| `security-kyverno` | kyverno | openEduSuite-hrz | `k8s/security/kyverno` | ✅ prune+selfHeal |
| `security-policies` | default | openEduSuite-hrz | `k8s/security/policies` | ✅ prune+selfHeal |
| `next-mailserver` | openedusuite | weissto/next-mailserver | `k8s/scs-k3s` | ✅ prune+selfHeal |

## Source Repositories

- **openEduSuite-hrz**: `ssh://git@gitlab.hrz.uni-marburg.de/hrz/kubernetes/openedusuite/openedusuite.git`
- **ansible-baremetal-k3s**: `https://gitlab.hrz.uni-marburg.de/hrz/kubernetes/ansible-baremetal-k3s.git`
- **next-mailserver**: `ssh://git@gitlab.hrz.uni-marburg.de/weissto/next-mailserver.git`

## Usage with kubara

This branch is for testing kubara's `bootstrap` command. To use:

```bash
# 1. Initialize kubara config
kubara init

# 2. Edit config.yaml to point to your cluster

# 3. Bootstrap ArgoCD
kubara bootstrap <cluster-name>

# 4. Apply the ArgoCD apps
kubectl apply -k deployments/openEduSuite/argocd/
```

## Notes

- The ArgoCD apps reference GitLab repositories (not this branch). ArgoCD pulls
  the actual workload manifests from those repos after bootstrap.
- `openedusuite-idp` has auto-sync **disabled** (manual sync only) to avoid
  Galera-STS conflicts with the `openedusuite` app.
- The `cmp-helmfile` ConfigMap enables helmfile-based rendering in ArgoCD.
- All apps use `CreateNamespace=true` and `ApplyOutOfSyncOnly=true`.
