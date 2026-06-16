# Argo CD ApplicationSet: Multi-Environment Matrix Deployment

This repository contains the configuration and operational documentation for managing multi-environment Kubernetes deployments using an Argo CD `ApplicationSet` and a custom `AppProject`. 

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
```

---

## 🛠️ ApplicationSet Architecture

The `ApplicationSet` controller uses a generator pattern to scale application deployments. This specific manifest employs a **List Generator**, which acts as an array iterator passing parameters into a single reusable application template.

### Defined Variables
The list generator passes two custom variables: `{{project}}` and `{{cluster}}`. 

```yaml
generators:
  - list:
      elements:
        - project: app1
          cluster: dev
        - project: app1
          cluster: qa
        - project: app1
          cluster: prod
```

### Parameter Matrix & Resolution
When compiled by the controller, the template expands into three unique Argo CD `Application` resources:

| Application Name | Git Source Path (`path`) | Cluster Destination (`server`)   | Target Namespace (`namespace`) |
| :--------------- | :----------------------- | :------------------------------- | :----------------------------- |
| **`app1-dev`**   | `app1/k8s-dev/`          | `https://kubernetes.default.svc` | `app1-dev`                     |
| **`app1-qa`**    | `app1/k8s-qa/`           | `https://kubernetes.default.svc` | `app1-qa`                      |
| **`app1-prod`**  | `app1/k8s-prod/`         | `https://kubernetes.default.svc` | `app1-prod`                    |

---

## 🔒 Security & Project Isolation (`AppProject`)

To secure these deployments, we define a dedicated `AppProject` named `my-project`. This resource acts as a logical security boundary within Argo CD to restrict where and what this team can deploy.

### Key Policies Enforced:
* **Source Repository Whitelisting:** Restricts deployments exclusively to code coming from `https://github.com/cgathefirst/gitops-argo.git`.
* **Destination Control:** Allows applications inside this project to deploy only to the local cluster (`https://kubernetes.default.svc`) while permitting deployment to any namespace (`*`) to accommodate the dynamic `{{project}}-{{cluster}}` namespaces created by the ApplicationSet.
* **Source Namespaces:** Allows resources to be generated from any source namespace (`*`).

> ⚠️ **Note:** To connect this security policy to your deployment, ensure your `ApplicationSet` template specification uses `project: my-project` instead of `project: default`.

---

## ⚙️ Core Configuration Deep Dive

### 1. Source & Destination Specs
* **Dynamic Paths:** The `path: '{{project}}/k8s-{{cluster}}/'` guarantees that each environment isolates its own configuration files.
* **Local Cluster Targeting:** The destination server is mapped to `https://kubernetes.default.svc`. This targets the local cluster where Argo CD itself is installed.

### 2. Automated Synchronization & Guardrails
To enforce strict GitOps principles and prevent configuration drift, the `syncPolicy` incorporates the following production guardrails:

* **Automated Sync (`automated`):** The controller listens for webhooks or polls the repository, immediately executing an update when changes hit the `second` branch.
* **Prune (`prune: true`):** If a resource or object is deleted from the Git repository, Argo CD will automatically strip it from the Kubernetes cluster to maintain an exact match.
* **Self-Heal (`selfHeal: true`):** If an engineer manually alters or tweaks a resource using `kubectl edit` or `kubectl apply` directly in the cluster, the controller will instantly overwrite it back to the state declared in Git.
* **Namespace Creation (`CreateNamespace=true`):** Prevents deployment blockages. If the target namespaces (`app1-dev`, `app1-qa`, `app1-prod`) do not exist prior to application execution, Argo CD creates them automatically.

---

## 🚀 Deployment Steps

### Step 1: Create the AppProject (Security Boundary)
Save the project configuration as `project.yaml` and apply it first. This ensures the environment boundaries exist before the applications try to reference it.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: my-project
  namespace: argocd
spec:
  description: Allow access to a specific Git repository and branch
  sourceRepos:
    - https://github.com/cgathefirst/gitops-argo.git
  destinations:
    - namespace: '*'
      server: https://kubernetes.default.svc
  sourceNamespaces:
    - '*'
```

Apply the file:
```bash
kubectl apply -f project.yaml -n argocd
```

### Step 2: Save the ApplicationSet Configuration
Save your application set definition locally as `applicationset.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: example-applicationset
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - project: app1
            cluster: dev
          - project: app1
            cluster: qa
          - project: app1
            cluster: prod
  template:
    metadata:
      name: '{{project}}-{{cluster}}'
    spec:
      project: my-project # Linked to our custom AppProject boundary
      source:
        repoURL: https://github.com/cgathefirst/gitops-argo.git
        targetRevision: second
        path: '{{project}}/k8s-{{cluster}}/'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{project}}-{{cluster}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Step 3: Apply the ApplicationSet Manifest
Execute the following `kubectl` command to feed the definition to your cluster:

```bash
kubectl apply -f applicationset.yaml -n argocd
```

### Step 4: Verify Application Generation
The generation happens near-instantly. Verify that the parent `ApplicationSet` is active and that the child `Application` objects have been instantiated within your project scope:

```bash
# Verify the ApplicationSet controller picked up the config
kubectl get applicationset -n argocd

# Check that the 3 target applications were generated
kubectl get applications -n argocd
```

### Step 5: Monitor Sync Status
You can tail the status of your newly generated child applications using the Argo CD CLI or standard `kubectl`:

```bash
argocd app list
# OR
kubectl describe app app1-dev -n argocd
```
