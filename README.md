# Akuity Technical Support Engineer Take-Home Challenge

## Overview

This repository contains my solution for the Akuity Technical Support Engineer Take-Home Challenge.

The solution demonstrates:

* Argo CD self-management using GitOps
* App-of-apps deployment pattern
* Argo CD repo-server customization using Helm v3.7.0
* Prometheus and Grafana monitoring for Argo CD
* ServiceMonitor resources for metrics collection
* Sync wave ordering for bootstrap dependencies
* Operational validation and troubleshooting procedures

---

## Challenge Requirement Mapping

| Requirement                       | Implementation                    |
| --------------------------------- | --------------------------------- |
| Argo CD manages its own lifecycle | `apps/argocd.yaml`                |
| Deploy Prometheus and Grafana     | `apps/monitoring.yaml`            |
| Monitor Argo CD metrics           | `apps/argocd-servicemonitor.yaml` |
| Replace Helm with v3.7.0          | `argocd/helm-3.7-config.yaml`     |
| App-of-apps sync wave = -10       | `bootstrap/app-of-apps.yaml`      |
| Dashboards                        | Grafana dashboards                |
| Alerts                            | Prometheus alerting rules         |

---

## Architecture

```text
Git Repository
      |
      v
+----------------+
| App Of Apps    |
+----------------+
      |
      +----------------------+
      |                      |
      v                      v

+-------------+      +----------------+
| Argo CD     |      | Monitoring     |
| Self Managed|      | Stack          |
+-------------+      +----------------+
                            |
                            +--> Prometheus
                            +--> Grafana
                            +--> ServiceMonitors
                            +--> Alert Rules
```

---

## Repository Structure

```text
.
├── apps/
│   ├── argocd.yaml
│   ├── monitoring.yaml
│   ├── argocd-servicemonitor.yaml
│   └── kustomization.yaml
│
├── argocd/
│   ├── helm-3.7-config.yaml
│   └── kustomization.yaml
│
├── bootstrap/
│   └── app-of-apps.yaml
│
└── README.md
```

---

## Solution Details

### App-of-Apps Pattern

The repository uses an app-of-apps architecture.

The bootstrap Application creates a root Application which manages all child applications stored in the `apps/` directory.

This approach provides:

* centralized management
* consistent GitOps workflows
* scalable onboarding of additional applications

---

### Sync Wave Configuration

The root application includes:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-10"
```

Using a sync wave of `-10` ensures the root application is reconciled before child applications are processed.

This guarantees correct bootstrap ordering.

---

### Argo CD Self Management

Argo CD manages its own configuration through Git.

Changes committed to the `argocd/` directory are automatically reconciled back to the running Argo CD installation.

Benefits:

* configuration drift prevention
* full auditability
* Git-based change history
* repeatable deployments

---

### Helm 3.7 Override

The challenge requires replacing the Helm binary bundled with Argo CD.

The repo-server Deployment is patched using:

* initContainer
* shared volume
* custom Helm binary

Installed version:

```text
Helm v3.7.0
```

Validation:

```bash
kubectl exec -it deploy/argocd-repo-server -n argocd -- helm version
```

Expected output:

```text
version.BuildInfo{
  Version:"v3.7.0"
}
```

---

### Monitoring

Monitoring is implemented using kube-prometheus-stack.

Components deployed:

* Prometheus
* Grafana
* Prometheus Operator

Additional ServiceMonitor resources are created for:

* argocd-server
* argocd-repo-server
* argocd-application-controller

---

### Dashboards

Grafana dashboards provide visibility into:

* Application health
* Sync status
* Reconciliation performance
* Repository operations
* API metrics

---

### Alerts

Prometheus alerting rules monitor:

* Application sync failures
* Degraded application health
* Missing scrape targets
* Repo-server availability

These alerts provide early warning of operational issues.

---

## Deployment

### Prerequisites

* Kubernetes cluster
* kubectl
* Internet access
* Argo CD installed

Recommended:

* Kind
* 4 CPU
* 8 GB RAM

---

### Install Argo CD

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for readiness:

```bash
kubectl wait \
--for=condition=Available \
deployment/argocd-server \
-n argocd \
--timeout=300s
```

---

### Deploy Bootstrap Application

```bash
kubectl apply -f bootstrap/app-of-apps.yaml
```

Argo CD will automatically deploy:

* Argo CD self-managed instance
* Monitoring stack
* ServiceMonitors
* Alert rules

---

## Validation

### Verify Applications

```bash
kubectl get applications -n argocd
```

Expected:

```text
app-of-apps            Synced Healthy
argocd-self-managed    Synced Healthy
monitoring             Synced Healthy
```

---

### Verify Helm Version

```bash
kubectl exec -it deploy/argocd-repo-server -n argocd -- helm version
```

Expected:

```text
v3.7.0
```

---

### Verify ServiceMonitors

```bash
kubectl get servicemonitors -n monitoring
```

---

### Verify Prometheus Targets

```bash
kubectl port-forward svc/prometheus-operated \
-n monitoring \
9090:9090
```

Navigate to:

```text
http://localhost:9090/targets
```

Expected:

```text
argocd-server
argocd-repo-server
argocd-application-controller
```

Targets should be UP.

---

### Access Grafana

```bash
kubectl port-forward svc/prometheus-grafana \
-n monitoring \
3000:80
```

Retrieve password:

```bash
kubectl get secret prometheus-grafana \
-n monitoring \
-o jsonpath="{.data.admin-password}" \
| base64 -d
```

Access:

```text
http://localhost:3000
```

---

## Troubleshooting Approach

### Application Not Syncing

Check:

```bash
kubectl describe application <app> -n argocd
```

Review:

* sync errors
* repository access
* destination configuration

---

### Repo-Server Issues

Check:

```bash
kubectl logs deploy/argocd-repo-server -n argocd
```

Verify:

* Helm binary installation
* manifest generation
* repository access

---

### Monitoring Issues

Check:

```bash
kubectl get servicemonitors -n monitoring

kubectl get targets
```

Verify:

* labels
* port names
* Prometheus discovery

---

### Cluster Resource Issues

Check:

```bash
kubectl top nodes
kubectl top pods -A
```

Look for:

* Pending pods
* OOMKills
* CPU starvation

---

# Part 2 – GitOps Flow Walkthrough

## End-to-End Synchronization Flow

```text
Developer
    |
    v
Git Repository
    |
    v
Application Controller
    |
    v
Repo Server
    |
    v
Rendered Manifests
    |
    v
Kubernetes API Server
    |
    v
Cluster State Updated
```

### Step 1

A developer pushes changes to Git.

### Step 2

The Application Controller detects a repository change.

### Step 3

The Application Controller requests manifest generation from Repo Server.

### Step 4

Repo Server:

* clones repository
* renders Kustomize
* renders Helm charts
* produces Kubernetes manifests

### Step 5

Application Controller compares:

* desired state
* live cluster state

### Step 6

Differences are sent to the Kubernetes API Server.

### Step 7

Kubernetes reconciles workloads.

### Step 8

Application health and sync status are updated in Argo CD.

---

## Component Responsibilities

### Git Repository

Source of truth.

### Argo CD Server

Provides UI, API, RBAC and user interaction.

### Application Controller

Performs reconciliation and synchronization.

### Repo Server

Generates manifests.

### Kubernetes API Server

Applies resource changes to the cluster.

---

## OCI Repository Support

Argo CD supports OCI artifact registries.

Flow:

```text
OCI Registry
      |
      v
Repo Server
      |
      v
Manifest Generation
      |
      v
Application Controller
      |
      v
Cluster
```

Examples:

* OCI Helm charts
* Artifact Registry
* Amazon ECR
* GHCR

OCI repositories eliminate the need for traditional Helm chart repositories while preserving versioning and immutability.

---

## Assumptions

* Argo CD exists before bootstrap Application deployment.
* Internet connectivity is available.
* Git repository is accessible by Argo CD.
* Helm 3.7 compatibility is required.
* Monitoring requirements are limited to challenge scope.

---

## Future Improvements

* Custom repo-server image containing Helm 3.7
* SSO integration
* External Secrets integration
* Network Policies
* Additional alerting
* Long-term monitoring retention
* Multi-cluster GitOps support

---

## Conclusion

This solution demonstrates:

* GitOps best practices
* Argo CD self-management
* App-of-apps architecture
* Helm runtime customization
* Observability with Prometheus and Grafana
* Operational troubleshooting methodology
* Understanding of Argo CD internals and GitOps workflows

## Output Screenshots
1. Pods health
<img width="511" height="127" alt="Pods health" src="https://github.com/user-attachments/assets/96a11bf2-88e5-4e92-bc0f-505b06498cf2" />

2. Monitor services
<img width="442" height="184" alt="Monitor services" src="https://github.com/user-attachments/assets/5be85963-9e6e-4df2-ae55-fc366eade0a2" />

3. Prometheus rules
<img width="676" height="522" alt="Prometheus rules" src="https://github.com/user-attachments/assets/1f226ad5-d3aa-49f4-bc88-e7e99d8fb5d5" />

4. Argocd version
<img width="970" height="63" alt="Argocd version" src="https://github.com/user-attachments/assets/87c88d5b-c18c-4cf5-a7b7-1104b56149f0" />

5. Sync wave
<img width="595" height="112" alt="Sync wave" src="https://github.com/user-attachments/assets/92f99889-fcba-46a1-8841-7954d63d5ada" />

6. Argo CD UI
<img width="1452" height="693" alt="Argo CD UI" src="https://github.com/user-attachments/assets/e4a5c019-7020-4f9a-9cf6-5a049a7636fd" />

7. Prometheus targets expand
<img width="1460" height="733" alt="Prometheus targets expand" src="https://github.com/user-attachments/assets/7febbd28-68a5-49d7-b11b-fdc4cb1605de" />

8. Grafana dashboards
<img width="1470" height="810" alt="Grafana dashboards" src="https://github.com/user-attachments/assets/bf7138bb-5e27-4988-acbb-a7ab868c6b7b" />

9. Prometheus targets collapse
<img width="1460" height="812" alt="Prometheus targets collapse" src="https://github.com/user-attachments/assets/9317ce5d-d68c-489c-9296-350e3c4f4a96" />

10. Grafana explore
<img width="1466" height="807" alt="Grafana explore" src="https://github.com/user-attachments/assets/0e2b227a-23dd-42e5-9a5b-b2099bf04c32" />

11. Grafana prometheus dashboard
<img width="1464" height="787" alt="Grafana prometheus dashboard" src="https://github.com/user-attachments/assets/adad70a0-688d-4bdf-913e-e73299117d37" />

12. Grafana datasource
<img width="1466" height="818" alt="Grafana datasource" src="https://github.com/user-attachments/assets/40f50108-4b71-4ac0-ac0b-491563368031" />
