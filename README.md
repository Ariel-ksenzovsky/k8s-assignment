🌐 How to Access
----------------

### Endpoint

The application is exposed via NGINX Ingress at:

`http://<INGRESS_EXTERNAL_IP>`

(Replace `<INGRESS_EXTERNAL_IP>` with the value from\
`kubectl get ingress -n demo-aks-ariel`)

* * * * *

### Example curl Command

`curl http://<INGRESS_EXTERNAL_IP>`

### Expected Response

`<!DOCTYPE html>
  <html>
  <head>
    <title>Welcome to nginx!</title>
  </head>
  <body>
    <h1>Welcome to nginx!</h1>
  </body>
  </html>``

This confirms that traffic is successfully routed through the Ingress → Service → Pods.


📈 Autoscaling Test Evidence
----------------------------

### HPA Status (Metrics Populated)

The Horizontal Pod Autoscaler is active and receiving CPU metrics (not `unknown`).

`kubectl get hpa nginx-hpa -n demo-aks-ariel`

Example output:

`NAME        REFERENCE                     TARGETS    MINPODS   MAXPODS   REPLICAS
nginx-hpa   Deployment/nginx-deployment   2%/75%     2         5         2`

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

