# Lab 09: ConfigMaps and Secrets

## Objective

Create ConfigMaps and Secrets from literals and files, mount them as volumes and environment variables in pods, and observe what happens when a ConfigMap is updated.

## Prerequisites

- A running Kubernetes cluster (from [Lab 01](01-cluster-setup.md)).
- `kubectl` installed and configured.
- Read the concept file: [Kubernetes Architecture](../concepts/kubernetes-architecture.md)

---

## Step 1: Create a ConfigMap from Literals

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false \
  --from-literal=APP_PORT=8080
```

### Verify

```bash
kubectl get configmap app-config
kubectl describe configmap app-config
```

Expected output from describe:

```
Name:         app-config
Data
====
APP_DEBUG:
----
false
APP_ENV:
----
production
APP_PORT:
----
8080
```

### View as YAML

```bash
kubectl get configmap app-config -o yaml
```

---

## Step 2: Create a ConfigMap from a File

Create a config file:

```bash
cat <<'EOF' > /tmp/app.properties
database.host=db.example.com
database.port=5432
database.name=myapp
log.level=info
EOF
```

Create the ConfigMap:

```bash
kubectl create configmap file-config --from-file=/tmp/app.properties
```

### Verify

```bash
kubectl describe configmap file-config
```

The entire file content is stored as a single key (`app.properties`), with the file contents as the value.

---

## Step 3: Use a ConfigMap as Environment Variables

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cm-env-pod
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c', 'echo "ENV=$APP_ENV DEBUG=$APP_DEBUG PORT=$APP_PORT"; sleep 3600']
    envFrom:
    - configMapRef:
        name: app-config
  restartPolicy: Never
EOF
```

### Verify the environment variables

```bash
kubectl logs cm-env-pod
```

Expected output:

```
ENV=production DEBUG=false PORT=8080
```

### Check all env vars from the ConfigMap

```bash
kubectl exec cm-env-pod -- env | grep APP_
```

Expected output:

```
APP_DEBUG=false
APP_ENV=production
APP_PORT=8080
```

---

## Step 4: Use a ConfigMap as a Volume Mount

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cm-vol-pod
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c', 'cat /config/app.properties; sleep 3600']
    volumeMounts:
    - name: config-volume
      mountPath: /config
  volumes:
  - name: config-volume
    configMap:
      name: file-config
  restartPolicy: Never
EOF
```

### Verify

```bash
kubectl logs cm-vol-pod
```

Expected output:

```
database.host=db.example.com
database.port=5432
database.name=myapp
log.level=info
```

### List files in the mounted directory

```bash
kubectl exec cm-vol-pod -- ls /config/
```

Expected output:

```
app.properties
```

Each key in the ConfigMap becomes a file in the mounted directory.

---

## Step 5: Create a Secret from Literals

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS=s3cur3P@ss
```

### Verify

```bash
kubectl get secret db-secret
kubectl describe secret db-secret
```

Notice that `describe` shows the data keys but NOT the values (only the size in bytes).

### View secret values (base64 encoded)

```bash
kubectl get secret db-secret -o yaml
```

The values are base64-encoded. Decode one:

```bash
kubectl get secret db-secret -o jsonpath='{.data.DB_PASS}' | base64 -d
```

Expected output:

```
s3cur3P@ss
```

**Important:** Secrets are only base64-encoded, not encrypted. Anyone with API access can decode them. For real security, use encryption at rest and tools like Sealed Secrets or external secret managers.

---

## Step 6: Create a Secret from a File

```bash
cat <<'EOF' > /tmp/credentials.txt
username=admin
password=s3cur3P@ss
EOF
```

```bash
kubectl create secret generic file-secret --from-file=/tmp/credentials.txt
```

### Verify

```bash
kubectl describe secret file-secret
```

---

## Step 7: Use Secrets in a Pod

### As environment variables

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c', 'echo "User=$DB_USER"; sleep 3600']
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_USER
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_PASS
  restartPolicy: Never
EOF
```

```bash
kubectl logs secret-env-pod
```

Expected output:

```
User=admin
```

### As a volume mount

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol-pod
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c', 'ls /secrets; cat /secrets/DB_USER; echo; sleep 3600']
    volumeMounts:
    - name: secret-volume
      mountPath: /secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
  restartPolicy: Never
EOF
```

```bash
kubectl logs secret-vol-pod
```

Expected output:

```
DB_PASS
DB_USER
admin
```

Each key becomes a file in `/secrets/`. Mounted secret volumes are stored in a tmpfs (in-memory filesystem).

---

## Step 8: Update a ConfigMap and Observe Behavior

### Update the ConfigMap

```bash
kubectl edit configmap app-config
```

Change `APP_DEBUG` from `false` to `true`. Save and exit.

Alternatively, use patch:

```bash
kubectl patch configmap app-config --type merge -p '{"data":{"APP_DEBUG":"true"}}'
```

### Check the volume-mounted pod

If you had a volume-mounted ConfigMap pod, the file would update automatically (within 30-60 seconds by default). However, the `cm-env-pod` that uses environment variables will NOT see the change. Environment variables are set at pod startup and do not update.

### Demonstrate with a new volume-mounted pod

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cm-watch-pod
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c', 'while true; do echo "$(date): DEBUG=$(cat /config/APP_DEBUG)"; sleep 10; done']
    volumeMounts:
    - name: config-volume
      mountPath: /config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  restartPolicy: Never
EOF
```

Wait about 60 seconds, then check logs:

```bash
kubectl logs cm-watch-pod --tail=10
```

You should see the value change from `false` to `true` in the log output after the ConfigMap update propagates.

---

## Cleanup

```bash
kubectl delete pod cm-env-pod cm-vol-pod secret-env-pod secret-vol-pod cm-watch-pod
kubectl delete configmap app-config file-config
kubectl delete secret db-secret file-secret
rm -f /tmp/app.properties /tmp/credentials.txt
```

---

## Key Takeaways

1. **ConfigMaps** store non-sensitive configuration data as key-value pairs.
2. **Secrets** store sensitive data (passwords, tokens, keys). They are base64-encoded, not encrypted by default.
3. Both can be consumed as **environment variables** or **volume mounts**.
4. Environment variables are set at pod startup and do NOT update when the ConfigMap/Secret changes.
5. Volume-mounted ConfigMaps/Secrets are updated automatically (with a delay of up to 60 seconds).
6. Use `--from-literal` for individual key-value pairs and `--from-file` for files.
7. When mounted as a volume, each key becomes a filename and the value becomes the file content.
8. Secrets mounted as volumes use tmpfs (in-memory), so they are never written to disk on the node.

---

## Next Steps

- Proceed to [Lab 10: Workload Resources](10-workload-resources.md) to work with Jobs, CronJobs, and DaemonSets.
- Read about [Orchestration Fundamentals](../concepts/orchestration-fundamentals.md) concepts.
