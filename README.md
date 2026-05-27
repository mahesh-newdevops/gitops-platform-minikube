# gitops-platform-minikube

Argo CD app-of-apps bootstrap repository for Minikube.

This platform repo manages these workload repositories:

- `cert-manager`: installs cert-manager and a local self-signed `ClusterIssuer`.
- `istio`: installs Istio base CRDs, `istiod`, and a NodePort ingress gateway from the official Istio Helm charts.
- `microservices-deployment`: deploys the demo application from `k8s/` into the `microservices` namespace.
- `monitoring-application`: deploys Prometheus, Loki, Grafana, and Alloy from `k8s/` into the `monitoring` namespace.
- `headlamp`: deploys the Headlamp Kubernetes UI Helm chart into the `headlamp` namespace.

## Repository Layout

```text
bootstrap/root-app.yaml   Argo CD root Application to apply once
argocd/kustomization.yaml App-of-apps entrypoint rendered by the root app
argocd/projects/          Argo CD AppProject definitions
argocd/apps/              Child Argo CD Applications for workload repos
```

## Prerequisites

- Minikube is running.
- `kubectl` is pointed at the Minikube cluster.
- Argo CD is installed in the `argocd` namespace.
- The NGINX ingress addon is enabled if you want to use the microservices ingress.

```bash
minikube start
minikube addons enable ingress
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl rollout status deployment/argocd-server -n argocd
```

## Build Local Microservice Images

The `microservices-deployment` manifests use local Minikube images with `imagePullPolicy: Never`.
Build them inside Minikube before syncing the Argo CD application:

```bash
cd ../microservices-deployment
eval $(minikube docker-env)
docker build -t frontend:latest ./frontend
docker build -t user-service:latest ./user-service
docker build -t order-service:latest ./order-service
docker build -t payment-service:latest ./payment-service
docker build -t notification-service:latest ./notification-service
```

## Bootstrap GitOps

You can bootstrap from GitHub Actions by running:

```text
Bootstrap GitOps On Minikube
```

Select:

```text
action: bootstrap
```

To remove the GitOps applications without stopping Minikube, select:

```text
action: destroy
```

To stop the Minikube cluster from the same workflow, select:

```text
action: stop
```

Apply the root application once:

```bash
cd ../gitops-platform-minikube
kubectl apply -f bootstrap/root-app.yaml
```

Argo CD will then reconcile:

- `argocd/projects/minikube-platform-project.yaml`
- `argocd/projects/minikube-workloads-project.yaml`
- `argocd/apps/cert-manager.yaml`
- `argocd/apps/cert-manager-issuers.yaml`
- `argocd/apps/istio-base.yaml`
- `argocd/apps/istiod.yaml`
- `argocd/apps/istio-ingressgateway.yaml`
- `argocd/apps/microservices-deployment.yaml`
- `argocd/apps/monitoring-application.yaml`
- `argocd/apps/headlamp.yaml`

## Check Sync Status

```bash
kubectl get applications -n argocd
kubectl get pods -n cert-manager
kubectl get clusterissuer
kubectl get pods -n istio-system
kubectl get pods -n istio-ingress
kubectl get pods -n microservices
kubectl get pods -n monitoring
kubectl get pods -n headlamp
```

## Access Argo CD

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

The default admin password can be read with:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d
```

## Access Workloads

Grafana:

```bash
kubectl port-forward -n monitoring svc/grafana 3000:3000
```

Then open:

```text
http://localhost:3000
```

Headlamp:

Headlamp is the recommended Kubernetes UI replacement for the archived Kubernetes Dashboard.

```bash
kubectl -n headlamp port-forward svc/headlamp 4466:80
```

Then open:

```text
http://localhost:4466
```

For a local Minikube admin login token:

```bash
kubectl -n headlamp create token headlamp
```

Microservices ingress:

```bash
echo "$(minikube ip) microservices.local" | sudo tee -a /etc/hosts
```

Then open:

```text
http://microservices.local
```

Istio ingress gateway:

```bash
minikube service istio-ingressgateway -n istio-ingress
```

## Cleanup

```bash
kubectl delete -f bootstrap/root-app.yaml --ignore-not-found
kubectl delete application cert-manager cert-manager-issuers istio-base istiod istio-ingressgateway microservices-deployment monitoring-application headlamp -n argocd --ignore-not-found
kubectl delete appproject minikube-platform minikube-workloads -n argocd --ignore-not-found
kubectl delete namespace cert-manager istio-system istio-ingress microservices monitoring headlamp --ignore-not-found
```
