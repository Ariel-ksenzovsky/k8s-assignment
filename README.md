# Kubernetes Assignment – DevOps 04

This repository demonstrates basic Kubernetes operations, deployment, autoscaling, and troubleshooting in a managed cloud environment (AKS).

All resources are deployed into a dedicated namespace following least-privilege principles.

---

## 🔹 Task 1: Cluster Access & Basic Validation

### Context & Connectivity
The cluster was accessed using a valid kubeconfig and context.

```bash
copy code
kubectl config current-context
kubectl cluster-info
```

### Node Health

```bash
copy code
kubectl get nodes
```
All nodes are reachable and in Ready state.

### Namespace Listing

```bash
cpoy code
kubectl get ns
```

## 🔹 Task 2: Namespace & RBAC

### Namespace

```bash
copy code
kubectl apply -f k8s/namespace.yaml
```

Namespace created:

```bash
copy code
demo-aks-ariel
```
### Service Account & RBAC
```bash
copy code
kubectl apply -f k8s/rbac.yaml
```
RBAC is scoped to the namespace and allows **only**:

* get, list, watch

* resources: pods, services, deployments.apps

### Authorization Checks
 Allowed (namespaced):

```bash
copy code
kubectl auth can-i list pods \
  -n demo-aks-ariel \
  --as=system:serviceaccount:demo-aks-ariel:sa-demo
```
Denied (cluster-wide):

```bash
copy code
kubectl auth can-i list nodes \
  --as=system:serviceaccount:demo-aks-ariel:sa-demo
```

## 🔹 Task 3: Application Deployment

An nginx application is deployed as a Kubernetes Deployment

```bash
copy code
kubectl apply -f k8s/deployment.yaml
```
Deployment properties:

*  replicas: 2

* readiness probe

* liveness probe

* CPU & memory requests and limits

* consistent labels and selectors

### Validation

```bash
copy code
kubectl get deploy,pods -n demo-aks-ariel
```
All pods reach `2/2 Ready` and remain stable.

## Task 4: Service & Ingress Exposure

### Service

```bash
copy code
kubectl apply -f k8s/service-cluster-ip.yaml
```
A ClusterIP service exposes the Deployment internally.

### Ingress

```bash
copy code
kubectl apply -f k8s/ingress.yaml
```

NGINX Ingress Controller is used to expose the application externally.


### 🌐 How to Access

### Endpoint

The application is exposed via NGINX Ingress at:

`http://<INGRESS_EXTERNAL_IP>`

(Replace `<INGRESS_EXTERNAL_IP>` with the value from\
`kubectl get ingress -n demo-aks-ariel`)


### Example curl Command

`curl http://<INGRESS_EXTERNAL_IP>`

### Expected Response

```html
Welcome to nginx!
```

This confirms that traffic is successfully routed through the Ingress → Service → Pods.

## 🔹 Task 5: Autoscaling (HPA)

HPA Configuration

```bash
kubectl apply -f k8s/hpa.yaml
```
HPA settings:

* minReplicas: 2

* maxReplicas: 5

* CPU target: 50%


### 📈 Autoscaling Test Evidence

#### HPA Status (Metrics Populated)

The Horizontal Pod Autoscaler is active and receiving CPU metrics (not `unknown`).

`kubectl get hpa nginx-hpa -n demo-aks-ariel`

Example output:

```

NAME        REFERENCE                     TARGETS    MINPODS   MAXPODS   REPLICAS
nginx-hpa   Deployment/nginx-deployment   2%/75%     2         5         2

```

This confirms that CPU metrics are available and the HPA can calculate utilization.

* * * * *

### Replica Counts (Before / During / After Load)

**Before load:**

`Replicas: 2`

**During load (CPU utilization above target):**

`Replicas increased to 3--5`

**After load stopped (cool-down period):**

`Replicas scaled back down to 2`

Replica changes were observed using:

`kubectl get deploy nginx-deployment -n demo-aks-ariel -w`

* * * * *

### Load Generation Method

CPU load was generated from inside the cluster using a temporary pod, sending continuous HTTP requests to the internal Service.

`kubectl -n demo-aks-ariel run loadgen\
  --image=busybox:1.36\
  -it --rm -- sh`

Inside the pod:

`while true; do wget -q -O- http://nginx-cluster-ip/ > /dev/null; done`

This sustained traffic increased CPU usage on the nginx pods, triggering the HPA scale-up.

* * * * *

## 🔍 Task 6: Basic Observability & Debugging Runbook

This runbook uses Kubernetes-native tooling to troubleshoot issues.

### 1️⃣ Check Pod Health and Status

Verify pod state and restart behavior.

```bash

kubectl get pods -n demo-aks-ariel

```

What to look for:

```

Running → healthy

CrashLoopBackOff → container is crashing repeatedly

Pending → scheduling/resource issue

READY 0/1 → readiness failing

```

### 2️⃣ Inspect Pod Details (Readiness / Liveness)

Describe the pod to inspect probe failures and container status.

```bash

Copy code

kubectl describe pod <POD_NAME> -n demo-aks-ariel

```

What to look for:

```

Events showing:

Readiness probe failed

Liveness probe failed

Restart count increasing

Last termination reason / exit code

```

### 3️⃣ View Application Logs

Access container logs directly via Kubernetes.

```bash

Copy code

kubectl logs <POD_NAME> -n demo-aks-ariel

```

If the pod is restarting:

```bash

Copy code

kubectl logs <POD_NAME> -n demo-aks-ariel --previous

```

What to look for:

```

Application errors

Startup failures

Configuration or port binding issues

```

### 4️⃣ View Namespace Events

Events often explain failures that logs do not.

```bash

Copy code

kubectl get events -n demo-aks-ariel --sort-by=.metadata.creationTimestamp

```

What to look for:

```

Probe failures

Image pull errors

Scheduling or resource warnings

```

### 5️⃣ Debug Service Routing (Endpoints)

Confirm the Service is correctly connected to Pods.

```bash

Copy code

kubectl get svc nginx-cluster-ip -n demo-aks-ariel

kubectl get endpoints nginx-cluster-ip -n 
demo-aks-ariel

```

What to look for:

```

Endpoints should list Pod IPs

Empty endpoints → selector mismatch or pods not Ready

```

### 6️⃣ Verify Service Selector Matches Pod Labels

Ensure the Service selector matches pod labels.

```bash

Copy code

kubectl get pods -n demo-aks-ariel --show-labels

kubectl get svc nginx-cluster-ip -n demo-aks-ariel -o yaml

```

What to look for:

```

Service selector labels must exactly match pod labels

```

### 7️⃣ Debug Ingress Provisioning

Check Ingress status and controller events.

```bash

Copy code

kubectl get ingress -n demo-aks-ariel -o wide

kubectl describe ingress nginx-ingress -n demo-aks-ariel

```

What to look for:

``` 
ADDRESS populated → ingress controller is working

Events such as:

service not found

no endpoints available

ingress class mismatch

```

### 8️⃣ Check Ingress Controller Health

Verify the ingress controller is running and has an external IP.

```bash

Copy code

kubectl get pods -n ingress-nginx

kubectl get svc -n ingress-nginx

```

What to look for:

```

Controller pods in Running state

Controller Service of type LoadBalancer with EXTERNAL-IP

```

### 9️⃣ Test Service Connectivity from Inside the Cluster

Run a temporary debug pod to validate DNS and networking.

```bash

Copy code

kubectl -n demo-aks-ariel run curl

  --image=curlimages/curl:latest

  -it --rm -- sh

```

Inside the pod:

```sh

Copy code

curl -I http://nginx-cluster-ip

```

What to look for:

```

HTTP 200/301/404 → Service routing works

Timeout → Service/endpoint issue

```

### 🔟 Validate External Access (Ingress)

Test the public endpoint from outside the cluster.

```bash

Copy code

curl http://<INGRESS_EXTERNAL_IP>

```

What to look for:

```

Successful HTTP response confirms:

Ingress → Service → Pod routing works

Timeout / connection refused → ingress or service issue

```

## ✅ Failure Scenarios Covered

This runbook allows an engineer to triage:

CrashLoopBackOff → logs + pod describe

Failing readiness/liveness → pod describe + events

Service not routing → endpoints + selectors

Ingress not provisioning / no external access → ingress describe + controller checks

DNS/connectivity issues → in-cluster curl test