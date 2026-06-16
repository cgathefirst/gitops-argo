# Argo CD ApplicationSet: Multi-Environment Matrix Deployment

This repository contains the configuration and operational documentation for managing multi-environment Kubernetes deployments using an Argo CD `ApplicationSet`. 

By utilizing the **List Generator**, this setup automates the lifecycle of individual applications across `dev`, `qa`, and `prod` stages while keeping the manifest configurations completely unified.

---

## 📋 Infrastructure Requirements & Prerequisites

Before applying this configuration, verify that your target environment satisfies the following prerequisites:

### 1. Cluster Prerequisites
* **Argo CD Installation:** A functional installation of Argo CD (v2.0 or higher is required, as `ApplicationSet` capabilities are bundled natively).
* **Namespace:** The target namespace `argocd` must exist and match where your Argo CD operator/controllers are actively running.
* **RBAC Privileges:** The Argo CD application controller service account requires permissions to provision new namespaces cluster-wide, since the automation policy enables namespace creation on-demand.

### 2. GitOps Repository Layout
The ApplicationSet tracks a remote repository structure. To prevent synchronization failures, your target repository must be structured to resolve the parameter paths correctly.

* **Repository URL:** `https://github.com/cgathefirst/gitops-argo.git`
* **Target Branch:** `second`

#### Expected Directory Structure:
```text
gitops-argo/                  # Root of the 'second' branch
└── app1/                     # Matches the {{project}} value
    ├── k8s-dev/              # Matches the k8s-{{cluster}} path for dev
    │   └── manifests.yaml    # Raw manifests, Kustomize, or Helm charts
    ├── k8s-qa/               # Matches the k8s-{{cluster}} path for qa
    │   └── manifests.yaml
    └── k8s-prod/             # Matches the k8s-{{cluster}} path for prod
        └── manifests.yaml
