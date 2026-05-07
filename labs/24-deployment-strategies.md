# Lab 24: Deployment Strategies

## Objective

Practice rolling updates, simulate blue-green deployments, and simulate canary deployments using standard Kubernetes resources.

## Prerequisites

- A running Kubernetes cluster (kind, minikube, or cloud-managed)
- `kubectl` installed and configured
- Basic familiarity with Deployments and Services

---

## Exercise 1: Rolling Update

### Step 1: Create a Deployment with v1

```bash
kubectl create deployment rolling-demo --image=nginx:1.24 --replicas=4
```

Verify the pods are running:

```bash
kubectl get pods -l app=rolling-demo -o wide
```

### Step 2: Expose the Deployment

```bash
kubectl expose deployment rolling-demo --port=80 --target-port=80 --name=rolling-demo-svc
```

### Step 3: Watch the rollout in real time

Open a second terminal and run:

```bash
kubectl get pods -l app=rolling-demo -w
```

### Step 4: Trigger a rolling update

In your first terminal, update the image:

```bash
kubectl set image deployment/rolling-demo nginx=nginx:1.25
```

Watch the second terminal — you will see new pods being created and old pods being terminated **gradually**.

### Step 5: Monitor the rollout status

```bash
kubectl rollout status deployment/rolling-demo
```

This will block until the rollout is complete and print progress.

### Step 6: Check rollout history

```bash
kubectl rollout history deployment/rolling-demo
```

### Step 7: Roll back to the previous version

```bash
kubectl rollout undo deployment/rolling-demo
```

Verify:

```bash
kubectl describe deployment rolling-demo | grep Image
```

You should see `nginx:1.24` again.

### Step 8: Experiment with maxSurge and maxUnavailable

Apply a Deployment with custom strategy settings:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-custom
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  selector:
    matchLabels:
      app: rolling-custom
  template:
    metadata:
      labels:
        app: rolling-custom
    spec:
      containers:
        - name: nginx
          image: nginx:1.24
EOF
```

Now update and watch:

```bash
kubectl set image deployment/rolling-custom nginx=nginx:1.25
kubectl rollout status deployment/rolling-custom
```

Notice that up to 2 extra pods can be created (maxSurge=2) and at most 1 pod can be unavailable at a time (maxUnavailable=1).

### Cleanup

```bash
kubectl delete deployment rolling-demo rolling-custom
kubectl delete svc rolling-demo-svc
```

---

## Exercise 2: Blue-Green Deployment

### Step 1: Deploy the "Blue" version (v1)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: bluegreen-demo
      version: blue
  template:
    metadata:
      labels:
        app: bluegreen-demo
        version: blue
    spec:
      containers:
        - name: nginx
          image: nginx:1.24
          ports:
            - containerPort: 80
EOF
```

### Step 2: Create a Service pointing to "Blue"

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: bluegreen-svc
spec:
  selector:
    app: bluegreen-demo
    version: blue
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
EOF
```

### Step 3: Verify traffic goes to Blue

```bash
kubectl get endpoints bluegreen-svc
```

You should see 3 endpoints (the 3 blue pods).

### Step 4: Deploy the "Green" version (v2)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: bluegreen-demo
      version: green
  template:
    metadata:
      labels:
        app: bluegreen-demo
        version: green
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
EOF
```

At this point, Green pods are running but receiving **no traffic** (the Service selector requires `version: blue`).

### Step 5: Verify both deployments are running

```bash
kubectl get pods -l app=bluegreen-demo --show-labels
```

You should see 6 pods: 3 blue, 3 green.

### Step 6: Switch traffic to Green (instant cutover)

```bash
kubectl patch svc bluegreen-svc -p '{"spec":{"selector":{"app":"bluegreen-demo","version":"green"}}}'
```

### Step 7: Verify traffic now goes to Green

```bash
kubectl get endpoints bluegreen-svc
kubectl describe svc bluegreen-svc | grep Selector
```

The endpoints should now point to the Green pods.

### Step 8: Rollback (switch back to Blue)

```bash
kubectl patch svc bluegreen-svc -p '{"spec":{"selector":{"app":"bluegreen-demo","version":"blue"}}}'
```

This is the power of blue-green: rollback is **instant** — just change the selector.

### Step 9: Clean up the old version

Once you are confident in the new version, delete the old deployment:

```bash
kubectl delete deployment app-blue
```

### Cleanup

```bash
kubectl delete deployment app-blue app-green 2>/dev/null
kubectl delete svc bluegreen-svc
```

---

## Exercise 3: Canary Deployment

### Step 1: Deploy the stable version (v1) with 9 replicas

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: canary-demo
      version: stable
  template:
    metadata:
      labels:
        app: canary-demo
        version: stable
    spec:
      containers:
        - name: nginx
          image: nginx:1.24
          ports:
            - containerPort: 80
EOF
```

### Step 2: Create a Service that selects by app label only

The Service uses only `app: canary-demo` — it will route to **any** pod with this label, regardless of version.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: canary-svc
spec:
  selector:
    app: canary-demo
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
EOF
```

### Step 3: Verify 9 endpoints (all stable)

```bash
kubectl get endpoints canary-svc
```

### Step 4: Deploy the canary version (v2) with 1 replica

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary-new
spec:
  replicas: 1
  selector:
    matchLabels:
      app: canary-demo
      version: canary
  template:
    metadata:
      labels:
        app: canary-demo
        version: canary
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
EOF
```

### Step 5: Verify 10 endpoints (9 stable + 1 canary)

```bash
kubectl get endpoints canary-svc
kubectl get pods -l app=canary-demo --show-labels
```

Now roughly **10% of traffic** goes to the canary (1 out of 10 pods).

### Step 6: Gradually shift traffic

If the canary is healthy, increase canary replicas and decrease stable replicas:

```bash
# Move to 30% canary
kubectl scale deployment canary-new --replicas=3
kubectl scale deployment canary-stable --replicas=7
```

Verify:

```bash
kubectl get pods -l app=canary-demo --show-labels
```

### Step 7: Complete the rollout

```bash
# 100% canary (now the new stable)
kubectl scale deployment canary-new --replicas=10
kubectl scale deployment canary-stable --replicas=0
```

### Step 8: Clean up the old deployment

```bash
kubectl delete deployment canary-stable
```

### Step 9: (Optional) Rollback the canary

If at any point the canary shows errors, simply scale it to 0:

```bash
kubectl scale deployment canary-new --replicas=0
kubectl scale deployment canary-stable --replicas=10
```

### Cleanup

```bash
kubectl delete deployment canary-stable canary-new 2>/dev/null
kubectl delete svc canary-svc
```

---

## Review Questions

1. What is the default deployment strategy in Kubernetes?
   - **Rolling Update**

2. During a blue-green deployment, how do you switch traffic?
   - **Change the Service selector** to point to the new version's labels

3. In a basic Kubernetes canary deployment, how is the traffic split controlled?
   - **By the ratio of replicas** between the stable and canary deployments

4. What command shows the progress of a rolling update?
   - `kubectl rollout status deployment/<name>`

5. What command rolls back a deployment to the previous revision?
   - `kubectl rollout undo deployment/<name>`
