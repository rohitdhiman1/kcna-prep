# Lab 12: Containers

## Objective

Build a simple container image with `docker build`, tag it, use different image pull policies in Kubernetes pod specs, and inspect image layers.

## Prerequisites

- Docker installed and running.
- A running Kubernetes cluster (from [Lab 01](01-cluster-setup.md)).
- `kubectl` installed and configured.
- Read the concept file: [Container Runtimes](../concepts/container-runtimes.md)

---

## Step 1: Create a Simple Application

Create a working directory and a simple web application:

```bash
mkdir -p /tmp/myapp && cd /tmp/myapp
```

Create a simple shell-based web server script:

```bash
cat <<'EOF' > server.sh
#!/bin/sh
echo "Server starting on port 8080..."
while true; do
  echo -e "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\nHello from my custom container!\nHostname: $(hostname)\nDate: $(date)" \
    | nc -l -p 8080 -q 1
done
EOF
chmod +x server.sh
```

---

## Step 2: Write a Dockerfile

```bash
cat <<'EOF' > Dockerfile
# Use a minimal base image
FROM alpine:3.19

# Add metadata
LABEL maintainer="kcna-student"
LABEL version="1.0"

# Install netcat for our simple server
RUN apk add --no-cache netcat-openbsd

# Copy application code
COPY server.sh /app/server.sh

# Set the working directory
WORKDIR /app

# Expose the port
EXPOSE 8080

# Run the server
CMD ["./server.sh"]
EOF
```

### Understand each instruction

- **FROM** — the base image to build on.
- **LABEL** — metadata key-value pairs.
- **RUN** — executes a command during build (creates a new layer).
- **COPY** — copies files from the build context into the image.
- **WORKDIR** — sets the working directory for subsequent instructions.
- **EXPOSE** — documents which port the container listens on (does not actually publish).
- **CMD** — the default command to run when the container starts.

---

## Step 3: Build the Image

```bash
docker build -t myapp:1.0 /tmp/myapp
```

Expected output:

```
[+] Building 5.2s (9/9) FINISHED
 => [1/4] FROM alpine:3.19
 => [2/4] RUN apk add --no-cache netcat-openbsd
 => [3/4] COPY server.sh /app/server.sh
 => [4/4] WORKDIR /app
 => exporting to image
 => => naming to docker.io/library/myapp:1.0
```

### Verify the image

```bash
docker images myapp
```

Expected output:

```
REPOSITORY   TAG   IMAGE ID       CREATED          SIZE
myapp        1.0   abc123def456   10 seconds ago   8.5MB
```

---

## Step 4: Tag the Image

Tags are aliases for images. You can tag an image with multiple names.

### Add another tag

```bash
docker tag myapp:1.0 myapp:latest
```

### Tag for a registry (example)

```bash
docker tag myapp:1.0 myregistry.example.com/myapp:1.0
```

### Verify all tags

```bash
docker images myapp
```

Expected output:

```
REPOSITORY   TAG      IMAGE ID       CREATED         SIZE
myapp        1.0      abc123def456   30 seconds ago  8.5MB
myapp        latest   abc123def456   30 seconds ago  8.5MB
```

Note: both tags point to the same IMAGE ID.

---

## Step 5: Inspect Image Layers

### View image history

```bash
docker history myapp:1.0
```

Expected output:

```
IMAGE          CREATED          CREATED BY                                      SIZE
abc123         10 seconds ago   CMD ["./server.sh"]                             0B
def456         10 seconds ago   EXPOSE map[8080/tcp:{}]                         0B
ghi789         10 seconds ago   WORKDIR /app                                    0B
jkl012         11 seconds ago   COPY server.sh /app/server.sh                   200B
mno345         12 seconds ago   RUN apk add --no-cache netcat-openbsd           1.5MB
pqr678         2 weeks ago      /bin/sh -c #(nop) CMD ["/bin/sh"]               0B
stu901         2 weeks ago      /bin/sh -c #(nop) ADD file:... in /             7MB
```

Each instruction in the Dockerfile creates a layer. Layers are cached and shared between images.

### Inspect image metadata

```bash
docker inspect myapp:1.0
```

This shows the full image configuration including:
- Environment variables
- Labels
- Exposed ports
- Entrypoint and command
- Layer digests

### View just the labels

```bash
docker inspect myapp:1.0 --format='{{json .Config.Labels}}' | python3 -m json.tool
```

Expected output:

```json
{
    "maintainer": "kcna-student",
    "version": "1.0"
}
```

---

## Step 6: Test the Container Locally

```bash
docker run -d --name myapp-test -p 8080:8080 myapp:1.0
```

Test it:

```bash
curl http://localhost:8080
```

Expected output:

```
Hello from my custom container!
Hostname: abc123def456
Date: Sat Mar 14 12:00:00 UTC 2026
```

Stop and remove:

```bash
docker stop myapp-test && docker rm myapp-test
```

---

## Step 7: Load the Image into kind

kind clusters cannot pull from your local Docker daemon directly. You need to load the image:

```bash
kind load docker-image myapp:1.0 --name kcna-lab
```

Expected output:

```
Image: "myapp:1.0" with ID "sha256:abc123..." not yet present on node "kcna-lab-control-plane", loading...
Image: "myapp:1.0" with ID "sha256:abc123..." not yet present on node "kcna-lab-worker", loading...
Image: "myapp:1.0" with ID "sha256:abc123..." not yet present on node "kcna-lab-worker2", loading...
```

---

## Step 8: Image Pull Policies in Kubernetes

Kubernetes supports three image pull policies:

- **Always** — always pull the image from the registry, even if it exists locally.
- **IfNotPresent** — pull only if the image is not already on the node.
- **Never** — never pull; only use images already on the node.

### Pod with imagePullPolicy: Never

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-never
spec:
  containers:
  - name: app
    image: myapp:1.0
    imagePullPolicy: Never
    ports:
    - containerPort: 8080
EOF
```

```bash
kubectl get pod app-never
```

This works because we loaded the image into kind in the previous step.

### Pod with imagePullPolicy: IfNotPresent

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-ifnotpresent
spec:
  containers:
  - name: app
    image: myapp:1.0
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 8080
EOF
```

```bash
kubectl get pod app-ifnotpresent
```

This also works because the image is already on the nodes.

### Pod with imagePullPolicy: Always (will fail for local-only images)

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-always
spec:
  containers:
  - name: app
    image: myapp:1.0
    imagePullPolicy: Always
    ports:
    - containerPort: 8080
EOF
```

```bash
kubectl get pod app-always
```

Expected output:

```
NAME         READY   STATUS             RESTARTS   AGE
app-always   0/1     ImagePullBackOff   0          15s
```

This fails because `Always` tries to pull from a registry, but `myapp:1.0` does not exist in any registry.

### Check the error details

```bash
kubectl describe pod app-always | grep -A 5 "Events:"
```

You will see `Failed to pull image` and `ImagePullBackOff` events.

---

## Step 9: Default Pull Policy Behavior

The default imagePullPolicy depends on the tag:

- If the tag is `:latest` or no tag is specified, the default is **Always**.
- If a specific tag is used (e.g., `:1.0`), the default is **IfNotPresent**.

### Demonstrate with :latest tag

```bash
kind load docker-image myapp:latest --name kcna-lab
```

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-latest
spec:
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080
EOF
```

```bash
kubectl describe pod app-latest | grep "Image:" -A 1
```

Since the tag is `latest`, Kubernetes defaults to `imagePullPolicy: Always`, which will try to pull from a registry. This is why you should avoid using `:latest` in production and always specify explicit tags.

---

## Step 10: Verify the Running Application

```bash
kubectl port-forward pod/app-never 8080:8080
```

In another terminal:

```bash
curl http://localhost:8080
```

Expected output:

```
Hello from my custom container!
Hostname: app-never
Date: Sat Mar 14 12:05:00 UTC 2026
```

Press `Ctrl+C` to stop port-forwarding.

---

## Cleanup

```bash
kubectl delete pod app-never app-ifnotpresent app-always app-latest --ignore-not-found
rm -rf /tmp/myapp
```

---

## Key Takeaways

1. A **Dockerfile** defines how to build a container image layer by layer.
2. Each instruction (FROM, RUN, COPY) creates a new layer. Layers are cached for faster rebuilds.
3. `docker build -t name:tag .` builds the image. `docker tag` adds additional tags.
4. `docker history` shows the layers and their sizes. `docker inspect` shows full metadata.
5. For kind clusters, use `kind load docker-image` to make local images available to the cluster.
6. **imagePullPolicy** controls when Kubernetes pulls images:
   - `Always` — always pull (default for `:latest`).
   - `IfNotPresent` — pull only if not cached (default for specific tags).
   - `Never` — never pull, use local only.
7. Always use specific image tags in production. Avoid `:latest` because it defaults to `Always` pull and makes rollbacks harder.
8. `ImagePullBackOff` means Kubernetes cannot pull the image. Check events with `kubectl describe pod` for details.

---

## Next Steps

- Continue with [Lab 20: Autoscaling](20-autoscaling.md) for more advanced Kubernetes topics.
- Read about [Cloud Native Principles](../concepts/cloud-native-principles.md) to understand how containers fit into the bigger picture.
