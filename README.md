# Jenkins on Kubernetes (K8s)

A fully production‑grade, declarative, Kubernetes‑native deployment of Jenkins using plain manifests. This project demonstrates an enterprise DevOps setup including persistent storage, RBAC, Configuration‑as‑Code (JCasC), plugin bootstrap, secure credential handling, and multiple access patterns (NodePort/Ingress).

This repository is designed as a **portfolio‑ready, real‑world implementation** suitable for demonstrating Kubernetes expertise in CI/CD, platform engineering, and infrastructure automation roles.

---

# 1. Architecture Overview

## 1.1 High-Level Architecture

```
+-------------------------------------------------------------+
|                         Kubernetes Cluster                   |
|                                                             |
|  +----------------------+        +-----------------------+   |
|  |  Jenkins StatefulSet |<------>|  Persistent Volume    |   |
|  |  (Controller)        | PVC    |  (10Gi / RWO)         |   |
|  +----------+-----------+        +-----------------------+   |
|             |                                          |
|     Service (NodePort / ClusterIP)                     |
|             |                                          |
|      +------+-------+                                 |
|      |   Ingress    |  (Optional)                     |
|      +------+-------+                                 |
|             |                                          |
|        External Access                                 |
+-------------------------------------------------------------+
```

### Key Components

* **StatefulSet** – Ensures stable network identity and persistent storage
* **PersistentVolumeClaim (PVC)** – Binds storage for Jenkins home directory
* **ConfigMaps** – Store JCasC configuration and plugin list
* **Secrets** – Secure storage of admin credentials
* **ServiceAccount + RBAC** – Controls Jenkins access inside the namespace
* **Service (NodePort)** – Primary access method
* **Ingress (Optional)** – Domain‑based access using an ingress controller

This layout reflects enterprise-grade deployment patterns used in large-scale DevOps environments.

---

# 2. Repository Structure

```
jenkins-on-k8s/
│
├── k8s/
│   ├── namespace.yaml
│   ├── rbac.yaml
│   ├── secrets-admin.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── statefulset.yaml
│   ├── config/
│   │   ├── plugins.txt
│   │   └── casc-jenkins.yaml
│
└── README.md
```

---

# 3. Prerequisites

* Kubernetes cluster (Minikube, Kind, GKE, AKS, EKS, etc.)
* kubectl configured
* StorageClass available (default works for Minikube)
* (Optional) Ingress controller enabled

```
minikube addons enable ingress
```

---

# 4. Deployment Guide

All commands assume you are inside the project root.

## 4.1 Step 1: Create Namespace

```
kubectl apply -f k8s/namespace.yaml
```

## 4.2 Step 2: Apply RBAC

```
kubectl apply -f k8s/rbac.yaml
```

## 4.3 Step 3: Apply Admin Secret

```
kubectl apply -f k8s/secrets-admin.yaml
```

## 4.4 Step 4: Create ConfigMaps

```
kubectl create configmap jenkins-plugins -n jenkins \
  --from-file=k8s/config/plugins.txt \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create configmap jenkins-casc -n jenkins \
  --from-file=k8s/config/casc-jenkins.yaml \
  --dry-run=client -o yaml | kubectl apply -f -
```

## 4.5 Step 5: Apply StatefulSet

```
kubectl apply -f k8s/statefulset.yaml
```

## 4.6 Step 6: Deploy Service

```
kubectl apply -f k8s/service.yaml
```

## 4.7 Step 7 (Optional): Deploy Ingress

```
kubectl apply -f k8s/ingress.yaml
```

---

# 5. Accessing Jenkins

## Option A: NodePort (recommended for local clusters)

```
minikube service jenkins -n jenkins --url
```

Or manually:

```
http://<node-ip>:30080
```

## Option B: Port-Forwarding

```
kubectl -n jenkins port-forward svc/jenkins 8080:8080
```

Access at: `http://localhost:8080`

## Option C: Ingress

Add to `/etc/hosts`:

```
<minikube-ip>  jenkins.local
```

Access at: `http://jenkins.local`

---

# 6. Credentials

```
Username: admin
Password: admin1234
```

(Defined in secrets-admin.yaml)

For real deployments, update this secret.

---

# 7. Key Features

* Completely declarative Jenkins deployment
* Zero manual configuration through **Configuration as Code (JCasC)**
* Automatic plugin installation using `jenkins-plugin-cli`
* Persistent storage via StatefulSet + PVC
* Namespace-scoped RBAC for improved security
* Optional ingress-based access
* Enterprise-grade folder structure
* Real pipeline seeded automatically (`hello-kubernetes`)

---

# 8. Troubleshooting

### Pod stuck in `Pending`

Check PVC binding:

```
kubectl -n jenkins get pvc
```

Ensure a StorageClass exists:

```
kubectl get storageclass
```

### Port-forward fails (port already in use)

Use another local port:

```
kubectl -n jenkins port-forward svc/jenkins 9080:8080
```

### Plugins not installing

Cluster must allow outbound traffic or proxy. Validate logs:

```
kubectl -n jenkins logs statefulset/jenkins -c install-plugins
```

### JCasC not loading

Confirm ConfigMap is mounted:

```
kubectl -n jenkins describe pod -l app=jenkins
```

---

# 9. Cleanup

```
kubectl delete -n jenkins -f k8s/ingress.yaml --ignore-not-found
kubectl delete -n jenkins -f k8s/service.yaml
kubectl delete -n jenkins -f k8s/statefulset.yaml
kubectl delete configmap jenkins-plugins -n jenkins
kubectl delete configmap jenkins-casc -n jenkins
kubectl delete -n jenkins -f k8s/secrets-admin.yaml
kubectl delete -n jenkins -f k8s/rbac.yaml
kubectl delete -f k8s/namespace.yaml
```

---

# 10. Recommended Enhancements

These can be implemented in future revisions:

* Helm chart packaging
* Jenkins dynamic Kubernetes agents
* Prometheus/Grafana monitoring
* TLS-enabled Ingress
* GitOps (ArgoCD) automation
* Backup/restore strategy using Velero

---

# 11. License

This repository is available under the MIT License.
