# Nginx Health Monitoring

##  Step 1 : Deploy with High Availability (kind: Deployment)

Running a single Pod is not production safe.

Production workloads use Deployments, which automatically manage ReplicaSets and Pods.

Benefits:

1. High availability

2. Self-healing

3. Rolling updates

4. Easy scaling


### deployment.yaml
``` yaml
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

### Deploy the Deployment

```
kubectl apply -f deployment.yaml
```

### Verify Deployment

Check deployment

```
kubectl get deployments
```

Expected output

```
output:

NAME           READY   UP-TO-DATE   AVAILABLE
nginx-health   3/3     3            3
```

Check pods

```
kubectl get pods
```

Example (In my case 3 pods available)

```
output:

NAME                            READY   STATUS    RESTARTS   AGE
nginx-health-6cc58544c4-bsw85   1/1     Running   0          11m
nginx-health-6cc58544c4-kghvv   1/1     Running   0          12m
nginx-health-6cc58544c4-r6rxk   1/1     Running   0          11m
```

### Self Healing
By using replicas, you will achieve the self healing means if any of pods will go down then new pods created automatically.

You can test self healing by below steps:
1. Get Pods name

```
kubectl get pods
```

```
output:

NAME                            READY   STATUS    RESTARTS   AGE
nginx-health-6cc58544c4-bsw85   1/1     Running   0          11m
nginx-health-6cc58544c4-kghvv   1/1     Running   0          12m
nginx-health-6cc58544c4-r6rxk   1/1     Running   0          11m
```

2. Delete the pod

```
kubectl delete pod nginx-health-6cc58544c4-bsw85
```

```
output:

pod "nginx-health-6cc58544c4-bsw85" deleted from default namespace
```

3. Check pods again

```
kubectl get pods
```

```
output:

NAME                            READY   STATUS    RESTARTS   AGE
nginx-health-6cc58544c4-9fn8v   1/1     Running   0          53s
nginx-health-6cc58544c4-kghvv   1/1     Running   0          19m
nginx-health-6cc58544c4-r6rxk   1/1     Running   0          18m
```
You will see a new pod automatically created.

`nginx-health-6cc58544c4-9fn8v` pod created automatically.

This is ReplicaSet self-healing.

### Manual Scaling

If you want manual scaling for testing purpose only. You can use below command

```
kubectl scale deployment nginx-health --replicas=5
```

```
output:

NAME                            READY   STATUS    RESTARTS   AGE
nginx-health-6cc58544c4-9fn8v   1/1     Running   0          5m43s
nginx-health-6cc58544c4-cjfnl   1/1     Running   0          3s
nginx-health-6cc58544c4-kghvv   1/1     Running   0          23m
nginx-health-6cc58544c4-l82tz   1/1     Running   0          3s
nginx-health-6cc58544c4-r6rxk   1/1     Running   0          23m
```

### Rolling Update
If you want to rolling out the updates, So kubernetes will do without downtime.

In below example we will update container image

```
kubectl set image deployment/nginx-health nginx=nginx:1.26-alpine
```

Watch rollout

```
kubectl rollout status deployment/nginx-health
```

Kubernetes updates pods without downtime.


### Learnings in this step

| Concept         | Explanation                        |
| --------------- | ---------------------------------- |
| Deployment      | Manages application lifecycle      |
| ReplicaSet      | Maintains desired number of pods   |
| Self-healing    | Automatically replaces failed pods |
| Rolling Updates | Zero downtime deployments          |
| Scaling         | Increase/decrease pods             |
