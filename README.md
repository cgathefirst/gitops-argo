# Argo CD ApplicationSet: Example Matrix Deployment

This repository contains an Argo CD `ApplicationSet` configuration used to automate the deployment of multi-environment applications from a centralized GitOps repository. 

Using the **List Generator**, this definition dynamically instantiates separate Argo CD Applications for each combination of project and cluster environment specified.

---

## 📋 Prerequisites & Requirements

To successfully deploy and run this `ApplicationSet`, your infrastructure must meet the following prerequisites:

### 1. Argo CD Installation
* **Argo CD** must be installed and running in your Kubernetes cluster.
* The ApplicationSet controller must be active (included by default in Argo CD v2.0+).
* This configuration targets the `argocd` namespace for the ApplicationSet resource itself.

### 2. Git Repository & Structure
The controller pulls configurations from the following repository:
* **Repository URL:** `https://github.com/cgathefirst/gitops-argo.git`
* **Target Revision (Branch):** `second`

For the matrix elements to synchronize correctly, your Git repository **must** have the following folder structure corresponding to the generator elements:

```text
gitops-argo/ (Branch: second)
└── app1/
    ├── k8s-dev/
    │   └── [Kubernetes manifests / Kustomize / Helm charts]
    ├── k8s-qa/
    │   └── [Kubernetes manifests / Kustomize / Helm charts]
    └── k8s-prod/
        └── [Kubernetes manifests / Kustomize / Helm charts]
