# Nginx Health Monitoring

A Kubernetes deployment example that runs Nginx with high availability, self-healing, and zero-downtime rolling updates.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step 1: Deployment](#step-1-deployment)
- [Step 2: Service](#step-2-service)
- [Step 3: ConfigMap](#step-3-configmap)
- [Operations](#operations)
- [Cleanup](#cleanup)
- [Concepts Summary](#concepts-summary)

---

## Overview

This project demonstrates production-style workload patterns on Kubernetes:

| Feature | Description |
|--------|-------------|
| **High availability** | Multiple replicas spread across the cluster |
| **Self-healing** | Failed or deleted pods are replaced automatically |
| **Rolling updates** | New versions deploy with zero downtime |
| **Scaling** | Scale replica count up or down on demand |

Single-pod workloads are not suitable for production. This guide uses a **Deployment** (Step 1), a **Service** (Step 2), and a **ConfigMap** (Step 3) to run, expose, and configure Nginx in a production-like way.

---

## Prerequisites

- A running Kubernetes cluster (e.g. [kind](https://kind.sigs.k8s.io/), minikube, or cloud provider)
- `kubectl` configured to talk to your cluster

Verify access:

```bash
kubectl cluster-info
```

---

## Step 1: Deployment

### Architecture

```
                    ┌─────────────────────────────────────┐
                    │  Deployment (nginx-health)          │
                    │  replicas: 3                        │
                    └─────────────────┬───────────────────┘
                                      │
                                      │ manages
                                      ▼
                    ┌─────────────────────────────────────┐
                    │  ReplicaSet                         │
                    │  selector: app=nginx-health         │
                    └─────────────────┬───────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
                    ▼                 ▼                 ▼
              ┌──────────┐     ┌──────────┐     ┌──────────┐
              │  Pod 1   │     │  Pod 2   │     │  Pod 3   │
              │  nginx   │     │  nginx   │     │  nginx   │
              │  :80     │     │  :80     │     │  :80     │
              └──────────┘     └──────────┘     └──────────┘
```

The Deployment declares the desired state (3 replicas). The ReplicaSet ensures exactly 3 Pods with matching labels exist. Each Pod runs one Nginx container on port 80.

### Manifest: `manifest/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-health
  labels:
    app: nginx-health
    environment: dev

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx-health

  template:
    metadata:
      labels:
        app: nginx-health

    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
```

### Apply the Deployment

From the project root:

```bash
kubectl apply -f manifest/deployment.yaml
```

Expected output:

```
deployment.apps/nginx-health created
```

### Verify Deployment

**Check Deployment status:**

```bash
kubectl get deployments
```

| NAME         | READY | UP-TO-DATE | AVAILABLE |
|-------------|-------|-------------|-----------|
| nginx-health | 3/3   | 3           | 3         |

**Check Pods:**

```bash
kubectl get pods -l app=nginx-health
```

Example output:

| NAME                              | READY | STATUS  | RESTARTS | AGE |
|-----------------------------------|-------|---------|----------|-----|
| nginx-health-6cc58544c4-bsw85     | 1/1   | Running | 0        | 11m |
| nginx-health-6cc58544c4-kghvv     | 1/1   | Running | 0        | 12m |
| nginx-health-6cc58544c4-r6rxk     | 1/1   | Running | 0        | 11m |

**Concepts in this step:** Deployment, ReplicaSet, replicas, selector, Pod template, `kubectl get deployments`, `kubectl get pods`.

---

## Step 2: Service

### Architecture

```
    Client Pod                    Service                      Backend Pods
   (curl-test)              (ClusterIP :80)
        │                         │
        │  http://nginx-health-   │
        │  service:80              │   selector:
        │ ───────────────────────►│   app=nginx-health
        │                          │         │
        │                          │    ┌────┼────┬────────┐
        │                          │    │    │    │        │
        │                          └────┼────┼────┼────────┼───►  Pod 1 (nginx:80)
        │                               │    │    │        │
        │                               └────┼────┼────────┼───►  Pod 2 (nginx:80)
        │                                    │    │        │
        │                                    └────┼────────┼───►  Pod 3 (nginx:80)
        │                                         │        │
        ◄─────────────────────────────────────────┴────────┘
              response (load-balanced to one of the Pods)
```

The Service gets a stable ClusterIP and DNS name (`nginx-health-service`). Traffic to the Service is load-balanced to Pods that match `selector: app=nginx-health`. Pod IPs can change; the Service name and port stay the same.

### What is a Service?

A **Service** is a Kubernetes object that exposes your Pods as a stable network endpoint. Pods get new IPs when they restart or scale; a Service gives you a **stable DNS name and virtual IP** that always routes traffic to the healthy Pods behind it.

### Why is a Service important?

| Without a Service | With a Service |
|-------------------|----------------|
| You must track each Pod’s IP, which changes on restart. | One stable name (e.g. `nginx-health-service`) and port. |
| Other apps break when Pods are replaced. | Traffic is sent only to running Pods that match the selector. |
| No built-in load balancing across replicas. | Kubernetes load-balances across all matching Pods. |

**In short:** Deployments manage *which* Pods run; Services define *how* other workloads (and optionally users) reach them over the network.

### Manifest: `manifest/service.yaml`

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-health-service
  labels:
    app: nginx-health

spec:
  type: ClusterIP
  selector:
    app: nginx-health
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

- **`type: ClusterIP`** — Internal-only IP; reachable from inside the cluster (default).
- **`selector`** — Must match the Pod labels from the Deployment so the Service sends traffic to those Pods.
- **`port`** — Port the Service listens on. **`targetPort`** — Port the app listens on inside the Pod (here, 80).

### Apply the Service

```bash
kubectl apply -f manifest/service.yaml
```

Expected output:

```
service/nginx-health-service created
```

### Verify the Service

**List Services:**

```bash
kubectl get svc nginx-health-service
```

Example output:

| NAME                   | TYPE        | CLUSTER-IP    | EXTERNAL-IP | PORT(S) | AGE |
|------------------------|-------------|---------------|-------------|---------|-----|
| nginx-health-service   | ClusterIP   | 10.96.xx.xx   | \<none\>    | 80/TCP  | 1m  |

**Inspect endpoints (Pods behind the Service):**

```bash
kubectl get endpoints nginx-health-service
```

You should see one endpoint per running Pod (e.g. 3 addresses). If a Pod is not Ready, it will not appear here.

### Verify from another Pod (curl the Service)

Confirm that traffic to the Service name reaches the Nginx Pods. Run a temporary Pod and `curl` the Service:

```bash
kubectl run curl-test --rm -it --restart=Never --image=curlimages/curl:latest -- curl -s -o /dev/null -w "%{http_code}" http://nginx-health-service:80
```

- **`http://nginx-health-service:80`** — Inside the same namespace, the hostname `nginx-health-service` resolves to the Service ClusterIP; port 80 is the Service port.
- Expected: `200` (HTTP 200 from Nginx). The Pod exits when the command finishes.

**Alternative: shell into a Pod and curl:**

```bash
# Start a debug Pod that stays running
kubectl run debug-shell --rm -it --restart=Never --image=curlimages/curl:latest -- sh

# Inside the Pod, run:
curl -s -o /dev/null -w "%{http_code}\n" http://nginx-health-service:80
# Expected: 200

curl -s http://nginx-health-service:80 | head -5
# Expected: HTML from Nginx welcome page

# Exit the Pod
exit
```

If you get **200** and Nginx HTML, the Service is correctly routing traffic to the matching Pods.

**Concepts in this step:** Service, ClusterIP, selector, port and targetPort, Endpoints, stable DNS name, `kubectl get svc`, `kubectl get endpoints`, verifying connectivity from another Pod.

---

## Step 3: ConfigMap

### Architecture

```
    Client (browser/curl)              Service                    Deployment
            │                          │                              │
            │  http://.../health.html  │   selector:                  │
            │ ────────────────────────►│   app=nginx-health           │
            │                          │ ──────────────────────────►  │
            │                          │                              │
            │                          │         ┌─────────┬─────────┬─────────┐
            │                          │         │         │         │         │
            │                          │         ▼         ▼         ▼         │
            │                          │        Pod 1     Pod 2     Pod 3      │
            │                          │         │         │         │         │
            │                          │         └─────────┴────┬────┴─────────┘
            │                          │                        │
            │                          │                        │ volumeMount
            │                          │                        ▼
            │                          │                ┌───────────────┐
            │                          │                │  ConfigMap    │
            │                          │                │  health.html  │
            │                          │                └───────────────┘
            │                          │
            ◄──────────────────────────┴──  response: Status: OK
```

The ConfigMap holds the `health.html` content. Each Pod mounts it at `/usr/share/nginx/html/health.html`, so the same health page is served by every replica without rebuilding the image.

### What is a ConfigMap?

A **ConfigMap** stores non-sensitive configuration data (files, env vars, or key-value pairs). Pods mount it as a volume or inject it as environment variables. Config stays separate from the container image so you can change it without rebuilding.

### Why use a ConfigMap for the health page?

| Without ConfigMap | With ConfigMap |
|-------------------|----------------|
| Health page baked into image; change requires rebuild. | Update `health.html` by editing ConfigMap and rolling Pods. |
| Config and app tightly coupled. | Config and app decoupled; same image, different config per env. |
| No single source of truth for config. | One ConfigMap shared by all replicas. |

**Production note:** ConfigMaps are for non-sensitive data (app config, HTML templates, env vars, logging config, feature flags). For secrets use [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/).

### Manifest: `manifest/configMap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: nginx-health-config

data:
  health.html: |
    <html>
    <body>
    <h1>Status: OK</h1>
    <p>Service: nginx-health</p>
    </body>
    </html>
```

The `data` key `health.html` becomes a file when the ConfigMap is mounted as a volume.

### Update Deployment to use ConfigMap

Add a **volume** (from the ConfigMap) and a **volumeMount** (into the container) so Nginx serves the ConfigMap content at `/usr/share/nginx/html/health.html`:

**Relevant addition to `manifest/deployment.yaml`:**

```yaml
spec:
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: health-volume
              mountPath: /usr/share/nginx/html/health.html
              subPath: health.html
      volumes:
        - name: health-volume
          configMap:
            name: nginx-health-config
```

- **`volumeMounts`** — Mounts the ConfigMap entry as a file at the given path. `subPath: health.html` uses only that key from the ConfigMap (so the file is named `health.html`).
- **`volumes`** — Defines a volume backed by the ConfigMap `nginx-health-config`.

This overrides the default Nginx index at that path so `/health.html` serves your custom page.

### Apply resources

From the project root, apply in order (ConfigMap first so the Deployment can mount it):

```bash
kubectl apply -f manifest/configMap.yaml
kubectl apply -f manifest/deployment.yaml
kubectl apply -f manifest/service.yaml
```

Or apply everything in the manifest directory:

```bash
kubectl apply -f manifest/
```

Expected (if creating for the first time):

```
configmap/nginx-health-config created
deployment.apps/nginx-health configured
service/nginx-health-service unchanged
```

### Verify ConfigMap

**List ConfigMaps:**

```bash
kubectl get configmap nginx-health-config
```

| NAME                 | DATA |
|----------------------|------|
| nginx-health-config  | 1    |

**Inspect the stored data:**

```bash
kubectl get configmap nginx-health-config -o yaml
```

### Test the health page

Port-forward the Service to localhost:

```bash
kubectl port-forward svc/nginx-health-service 8080:80
```

In another terminal, or in a browser:

```bash
curl -s http://localhost:8080/health.html
```

Or open in a browser: **http://localhost:8080/health.html**

Expected response body:

```html
Status: OK
Service: nginx-health
```

Stop the port-forward with `Ctrl+C`.

**Concepts in this step:** ConfigMap, `data` (file content), volume, volumeMount, subPath, decoupling config from image, updating config without rebuilding.

---

## Operations

### Self-healing

The ReplicaSet keeps the desired replica count. If a Pod is deleted, a new one is created automatically.

1. **List Pods and pick one to delete**

   ```bash
   kubectl get pods -l app=nginx-health
   ```

2. **Delete a Pod**

   ```bash
   kubectl delete pod <pod-name>
   # Example: kubectl delete pod nginx-health-6cc58544c4-bsw85
   ```

   Output:

   ```
   pod "nginx-health-6cc58544c4-bsw85" deleted
   ```

3. **Confirm a new Pod was created**

   ```bash
   kubectl get pods -l app=nginx-health
   ```

   A new Pod (e.g. `nginx-health-6cc58544c4-9fn8v`) appears while the old one is terminating. This is ReplicaSet self-healing.

---

### Manual scaling

Scale the Deployment to 5 replicas (useful for testing or load):

```bash
kubectl scale deployment nginx-health --replicas=5
```

Verify:

```bash
kubectl get pods -l app=nginx-health
```

Scale back to 3:

```bash
kubectl scale deployment nginx-health --replicas=3
```

---

### Rolling update

Update the container image with zero downtime:

```bash
kubectl set image deployment/nginx-health nginx=nginx:1.26-alpine
```

Watch the rollout:

```bash
kubectl rollout status deployment/nginx-health
```

Kubernetes replaces Pods gradually so the service stays available. To roll back the last rollout:

```bash
kubectl rollout undo deployment/nginx-health
```

---

## Cleanup

Remove the Service, Deployment, and ConfigMap (and all managed Pods):

```bash
kubectl delete -f manifest/service.yaml
kubectl delete -f manifest/deployment.yaml
kubectl delete -f manifest/configMap.yaml
```

Or delete all resources in the manifest directory:

```bash
kubectl delete -f manifest/
```

Or by name:

```bash
kubectl delete service nginx-health-service
kubectl delete deployment nginx-health
kubectl delete configmap nginx-health-config
```

---

## Concepts Summary

| Concept         | Explanation |
|----------------|-------------|
| **Deployment** | Manages application lifecycle and desired state |
| **ReplicaSet** | Keeps the desired number of Pods running |
| **Service** | Stable network endpoint (DNS + virtual IP) that load-balances to Pods matching the selector |
| **ClusterIP** | Default Service type; internal-only, reachable from within the cluster |
| **ConfigMap** | Stores non-sensitive configuration data; mounted as files or env vars |
| **Volume / volumeMount** | Injects ConfigMap (or other storage) into a container at a path |
| **Self-healing** | Failed or deleted Pods are replaced automatically |
| **Rolling updates** | New version deployed with zero downtime |
| **Scaling** | Change replica count to handle more or less load |

---

## Next steps

- Expose the Service externally: use [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) or [LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) for access from outside the cluster.
- Add [liveness and readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) to the container spec for better health checks.
- Use [Resource requests and limits](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) for CPU and memory.
- For sensitive data (passwords, tokens, certs), use [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) instead of ConfigMaps.
