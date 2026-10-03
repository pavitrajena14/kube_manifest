# kube_manifest

Kubernetes manifests for the **ToDo app**, kept in their own repository so they can be deployed with a GitOps workflow (ArgoCD). This repo holds only the *desired state* of the application in the cluster. It contains no application source code, no Dockerfile and no infrastructure code.

## Table of contents

- [Overview](#overview)
- [Repository structure](#repository-structure)
- [How it fits into the GitOps pipeline](#how-it-fits-into-the-gitops-pipeline)
- [Manifest reference](#manifest-reference)
  - [deployment.yaml](#deploymentyaml)
  - [service.yaml](#serviceyaml)
- [Prerequisites](#prerequisites)
- [Deploying](#deploying)
  - [Option 1: ArgoCD (recommended)](#option-1-argocd-recommended)
  - [Option 2: kubectl (manual)](#option-2-kubectl-manual)
- [Verifying the deployment](#verifying-the-deployment)
- [Updating the application version](#updating-the-application-version)
- [Scaling and rollbacks](#scaling-and-rollbacks)
- [Cleanup](#cleanup)
- [Troubleshooting](#troubleshooting)
- [Possible improvements](#possible-improvements)

## Overview

The repository defines two Kubernetes objects:

| Object | Name | Purpose |
| --- | --- | --- |
| `Deployment` | `myapp` | Runs 2 replicas of the ToDo app container image. |
| `Service` (`LoadBalancer`) | `myapp-service` | Exposes the pods to the internet on port 80 through a cloud load balancer. |

The application is a containerised web app (the image is served on container port `80`, which is typical for an nginx-served React build) published on Docker Hub as `pavijena14/todo-app`.

## Repository structure

```text
kube_manifest/
├── README.md
└── manifest/
    ├── deployment.yaml   # Deployment: 2 replicas of pavijena14/todo-app
    └── service.yaml      # Service: LoadBalancer on port 80 -> container port 80
```

## How it fits into the GitOps pipeline

This repo is the "config repo" half of a typical two-repo GitOps setup:

```text
 ┌──────────────┐   push    ┌───────────────┐  build/test/push  ┌────────────┐
 │ App source   │ ────────► │  CI pipeline  │ ────────────────► │ Docker Hub │
 │ repo         │           │  (e.g. CircleCI)                  │ todo-app:N │
 └──────────────┘           └──────┬────────┘                   └────────────┘
                                   │ commits new image tag
                                   ▼
                          ┌──────────────────┐   watches   ┌───────────┐   syncs   ┌─────────────┐
                          │ kube_manifest    │ ◄────────── │  ArgoCD   │ ────────► │ Kubernetes  │
                          │ (this repo)      │             │           │           │ cluster/EKS │
                          └──────────────────┘             └───────────┘           └─────────────┘
```

1. A change to the application code triggers the CI pipeline.
2. CI builds and pushes a new image to Docker Hub, tagged with the build number (for example `build-6`).
3. CI (or a person) updates the `image:` line in `manifest/deployment.yaml` and commits it here.
4. ArgoCD detects the new commit and syncs the cluster so it matches Git.

Git is the single source of truth: if the cluster drifts from what is in this repo, ArgoCD can report it and correct it.

## Manifest reference

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  replicas: 2
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: pavijena14/todo-app:build-6
          ports:
            - containerPort: 80
```

| Field | Value | Explanation |
| --- | --- | --- |
| `kind` | `Deployment` | Manages a ReplicaSet and gives rolling updates and rollbacks. |
| `metadata.name` | `myapp` | Name of the Deployment. |
| `spec.replicas` | `2` | Two identical pods run at all times, so one pod can fail or be replaced without downtime. |
| `spec.selector.matchLabels` | `app: myapp` | Tells the Deployment which pods it owns. Must match the pod template labels. |
| `template.metadata.labels` | `app: myapp` | Label placed on every pod. The Service uses this same label to find its pods. |
| `containers[].name` | `myapp` | Container name inside the pod. |
| `containers[].image` | `pavijena14/todo-app:build-6` | Docker Hub image. The tag (`build-6`) identifies the exact build being deployed. |
| `containers[].ports[].containerPort` | `80` | Port the application listens on inside the container. |

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp

  type: LoadBalancer
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
```

| Field | Value | Explanation |
| --- | --- | --- |
| `kind` | `Service` | Gives the pods a stable network endpoint. |
| `metadata.name` | `myapp-service` | Name of the Service. |
| `spec.selector` | `app: myapp` | Routes traffic to any pod carrying this label (the pods created by the Deployment). |
| `spec.type` | `LoadBalancer` | Asks the cloud provider to provision an external load balancer (on AWS EKS, an ELB/NLB). |
| `ports[].port` | `80` | Port exposed by the Service and the load balancer. |
| `ports[].protocol` | `TCP` | Transport protocol. |
| `ports[].targetPort` | `80` | Port on the pod that receives the traffic. Matches `containerPort`. |

**Traffic flow:** `Internet -> Load balancer :80 -> myapp-service :80 -> one of the myapp pods :80`

## Prerequisites

- A running Kubernetes cluster (for example AWS EKS, or a local one such as minikube, kind or k3s).
- `kubectl` configured to talk to that cluster (`kubectl config current-context` should show the right cluster).
- For the GitOps route: ArgoCD installed in the cluster.
- A cluster that can provision `LoadBalancer` Services (cloud clusters do this automatically; local clusters need something like `minikube tunnel` or MetalLB).

## Deploying

### Option 1: ArgoCD (recommended)

Create an ArgoCD `Application` that points at this repository:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: todo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/pavitrajena14/kube_manifest.git
    targetRevision: main
    path: manifest
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply it and let ArgoCD do the rest:

```bash
kubectl apply -f application.yaml
```

With `automated` sync, every new commit to `main` is rolled out automatically. `prune` removes resources deleted from Git, and `selfHeal` reverts manual changes made directly in the cluster.

You can also create the application from the ArgoCD CLI:

```bash
argocd app create todo-app \
  --repo https://github.com/pavitrajena14/kube_manifest.git \
  --path manifest \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
```

### Option 2: kubectl (manual)

```bash
git clone https://github.com/pavitrajena14/kube_manifest.git
cd kube_manifest
kubectl apply -f manifest/
```

## Verifying the deployment

```bash
# Pods should reach Running with READY 1/1 (two of them)
kubectl get pods -l app=myapp

# Deployment status
kubectl get deployment myapp

# Service and its external address
kubectl get svc myapp-service
```

The `EXTERNAL-IP` (or hostname on AWS) of `myapp-service` can take a few minutes to appear while the load balancer is created. Once it is populated, open it in a browser:

```text
http://<EXTERNAL-IP-or-hostname>
```

Useful follow-ups:

```bash
kubectl describe deployment myapp
kubectl logs -l app=myapp --tail=50
kubectl rollout status deployment/myapp
```

## Updating the application version

Change the image tag in `manifest/deployment.yaml`:

```yaml
image: pavijena14/todo-app:build-7
```

Commit and push to `main`:

```bash
git add manifest/deployment.yaml
git commit -m "Deploy todo-app build-7"
git push origin main
```

- With ArgoCD, the cluster picks up the change automatically (or after a manual **Sync** if auto-sync is off).
- With plain `kubectl`, run `kubectl apply -f manifest/` again.

Kubernetes performs a rolling update: new pods are started and become ready before old pods are removed, so the app stays available.

## Scaling and rollbacks

**Scale** by changing `spec.replicas` in Git (preferred under GitOps). A quick temporary change is possible with:

```bash
kubectl scale deployment myapp --replicas=3
```

Note that ArgoCD's `selfHeal` will revert this to the value in Git.

**Roll back** by reverting the commit in Git, which ArgoCD then syncs:

```bash
git revert <commit-sha>
git push origin main
```

Outside GitOps you can use `kubectl rollout undo deployment/myapp`.

## Cleanup

```bash
kubectl delete -f manifest/
```

If ArgoCD manages the app, delete the `Application` instead (with prune/cascade enabled it removes the managed resources). Deleting the `LoadBalancer` Service also removes the cloud load balancer, which stops its charges.

## Troubleshooting

| Symptom | Likely cause | What to check |
| --- | --- | --- |
| Pods in `ImagePullBackOff` / `ErrImagePull` | Wrong image name or tag, or a private image without credentials | `kubectl describe pod <pod>`; confirm the tag exists on Docker Hub. |
| Pods in `CrashLoopBackOff` | App fails at startup | `kubectl logs <pod>`; check the container actually listens on port 80. |
| Service `EXTERNAL-IP` stuck on `<pending>` | No load balancer provider (local cluster) or cloud permissions/quotas | Use `minikube tunnel`, MetalLB, or `kubectl port-forward svc/myapp-service 8080:80`. |
| Service has no endpoints | Selector does not match pod labels | `kubectl get endpoints myapp-service`; labels must be `app: myapp`. |
| ArgoCD shows `OutOfSync` | Git and cluster differ | Click **Sync**, or enable automated sync. Check the diff in the UI. |
| Browser cannot reach the app | Load balancer still provisioning, or security group blocks port 80 | Wait a few minutes; verify inbound rules on the load balancer. |

## Possible improvements

The manifests are intentionally minimal. For a more production-ready setup, consider:

- **Resource requests and limits** on the container so the scheduler and autoscaler behave predictably.
- **Liveness and readiness probes** (for example an HTTP GET on `/`) so traffic only reaches healthy pods.
- **A dedicated namespace** instead of `default`.
- **Rolling update strategy** settings (`maxSurge`, `maxUnavailable`) made explicit.
- **Standard labels** such as `app.kubernetes.io/name` and `app.kubernetes.io/version`.
- **Ingress + TLS** (for example an AWS ALB Ingress with an ACM certificate) instead of exposing a plain HTTP LoadBalancer.
- **Security context** (run as non-root, read-only root filesystem) if the image supports it.
- **Pinning the image by digest** for fully reproducible deployments.
- **HorizontalPodAutoscaler** to scale replicas with load.

## Author

Maintained by [pavitrajena14](https://github.com/pavitrajena14).
