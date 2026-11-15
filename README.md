# Argo CD GitOps Application on Amazon EKS

This project deploys an NGINX application to Amazon EKS with Argo CD. It
includes a standard Deployment with an HPA and an optional KEDA deployment
with a `ScaledObject`.

## Repository

```text
https://github.com/Shyamnair488/argocd-app
```

The commands in this guide are written for Windows PowerShell. Each command
is intentionally on one line so it can be pasted without shell continuation
errors.

## Prerequisites

Install and authenticate these tools before starting:

- AWS CLI
- `eksctl`
- `kubectl`
- Helm
- Git
- Argo CD CLI (optional)

Verify the tools:

```powershell
aws --version
eksctl version
kubectl version --client
helm version
git --version
```

Clone the repository if it is not already available locally:

```powershell
git clone https://github.com/Shyamnair488/argocd-app.git; Set-Location argocd-app
```

## 1. Create the EKS Cluster

```powershell
eksctl create cluster --name mycluster --region us-east-1 --node-type t3.small --nodes 2 --nodes-min 2 --nodes-max 2 --with-oidc --managed
```

This creates the `mycluster` EKS cluster with two managed nodes and OIDC
enabled for AWS IAM roles for service accounts.

## 2. Configure and Verify kubectl

```powershell
aws eks update-kubeconfig --name mycluster --region us-east-1
kubectl config current-context
kubectl get nodes
kubectl get namespaces
```

The nodes must show `Ready` before continuing.

## 3. Install Argo CD

Create the namespace:

```powershell
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
```

Install Argo CD:

```powershell
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for the Argo CD components:

```powershell
kubectl wait --for=condition=Available deployment --all -n argocd --timeout=10m
kubectl get pods -n argocd
```

## 4. Install KEDA

The KEDA manifests require the `ScaledObject` custom resource definition.
Install KEDA before syncing this repository, otherwise Argo CD cannot
discover `keda.sh/v1alpha1`.

```powershell
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm upgrade --install keda kedacore/keda --namespace keda --create-namespace --wait --timeout 5m
kubectl get crd scaledobjects.keda.sh
kubectl get pods -n keda
```

## 5. Access the Argo CD UI

Start port forwarding in a separate PowerShell window:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open [https://localhost:8080](https://localhost:8080). A browser certificate
warning is expected because this is a local port-forward.

The Argo CD username is:

```text
admin
```

Retrieve the generated password with the Windows PowerShell command below:

```powershell
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")))
```

Do not commit or share the generated password.

## 6. Create the Argo CD Application

In the Argo CD UI, select **New App** and use:

```text
Application name: nginx-app
Project: default
Repository URL: https://github.com/Shyamnair488/argocd-app
Revision: main
Path: .
Cluster URL: https://kubernetes.default.svc
Namespace: default
```

Enable automated sync if desired. KEDA must be installed before creating or
syncing this application because the repository includes a `ScaledObject`.

## 7. Sync and Verify

Select **Create**, open the application, and select **Sync**.

Verify the deployed resources:

```powershell
kubectl get deployments -n default
kubectl get pods -n default
kubectl get services -n default
kubectl get hpa -n default
kubectl get scaledobject -n default
```

The standard resources are defined in [`k8s/`](k8s):

| File | Resource | Purpose |
| --- | --- | --- |
| `deployment.yaml` | `nginx` Deployment | Standard NGINX workload |
| `service.yaml` | `nginx-service` Service | LoadBalancer access |
| `hpa.yaml` | `nginx-hpa` HPA | CPU-based autoscaling |
| `keda-deployment.yaml` | `nginx-keda` Deployment | Separate KEDA workload |
| `scaleobject.yaml` | `nginx-cpu-scaler` ScaledObject | KEDA CPU autoscaling |

The standard Deployment and KEDA Deployment intentionally have different
names and selectors so they do not overwrite each other. The Service targets
only the standard `nginx` Deployment.

## 8. Test GitOps

Edit `k8s/deployment.yaml`, for example by changing `replicas`, then commit
and push:

```powershell
git add .
git commit -m "Scale NGINX deployment"
git push origin main
```

Argo CD detects the Git change and reports the application as `OutOfSync`
until automated or manual synchronization applies it.

## Optional Local Monitoring

Start the pinned Prometheus, Grafana, and Node Exporter containers:

```powershell
docker compose up -d
```

Check their status:

```powershell
docker compose ps
```

Stop them:

```powershell
docker compose down
```

## Cleanup

Remove the Argo CD port-forward by closing its PowerShell window.

Delete the EKS cluster when the lab is complete:

```powershell
eksctl delete cluster --name mycluster --region us-east-1
```

This deletes the cluster and its managed AWS resources. Confirm the cluster
name and region before running the command.

## Troubleshooting

### Argo CD cannot find `keda.sh/v1alpha1`

Install KEDA and verify the CRD:

```powershell
kubectl get crd scaledobjects.keda.sh
kubectl get pods -n keda
```

Then retry the Argo CD sync.

### Pods are not ready

Inspect the workload and events:

```powershell
kubectl describe deployment nginx -n default
kubectl get events -n default --sort-by=.lastTimestamp
```

### Service has no external address

`LoadBalancer` provisioning depends on the cloud provider and can take a few
minutes:

```powershell
kubectl get service nginx-service -n default --watch
```
