# ConfigMaps and Secrets

**ConfigMaps store non-sensitive configuration data as key-value pairs, while Secrets store sensitive data (passwords, tokens, keys) using base64 encoding. Both decouple configuration from container images.**

---

## Table of Contents

- [ConfigMaps](#configmaps)
- [ConfigMap YAML Spec](#configmap-yaml-spec)
- [Consuming ConfigMaps](#consuming-configmaps)
- [Secrets](#secrets)
- [Secret YAML Spec](#secret-yaml-spec)
- [Consuming Secrets](#consuming-secrets)
- [How Pods Consume ConfigMaps and Secrets Diagram](#how-pods-consume-configmaps-and-secrets-diagram)
- [Volume Mounts vs Environment Variables](#volume-mounts-vs-environment-variables)
- [Immutable ConfigMaps and Secrets](#immutable-configmaps-and-secrets)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## ConfigMaps

A ConfigMap lets you store configuration data separately from your application code. Common use cases:

- Application configuration files (e.g., `nginx.conf`, `app.properties`)
- Environment variables (e.g., `DATABASE_HOST=db.example.com`)
- Command-line arguments
- Feature flags and settings

ConfigMaps are for **non-sensitive** data only. For sensitive data, use Secrets.

### Creating ConfigMaps

```bash
# From literal key-value pairs
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=db.example.com \
  --from-literal=LOG_LEVEL=info

# From a file
kubectl create configmap nginx-config --from-file=nginx.conf

# From an env file
kubectl create configmap app-env --from-env-file=app.env
```

---

## ConfigMap YAML Spec

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Simple key-value pairs
  DATABASE_HOST: "db.example.com"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"

  # Multi-line file content
  app.properties: |
    server.port=8080
    server.context-path=/api
    logging.level.root=INFO

  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://backend:8080;
      }
    }
```

### Key Points

- `data` holds UTF-8 string key-value pairs
- `binaryData` holds base64-encoded binary data (e.g., images, certs)
- ConfigMap size is limited to **1 MiB**
- ConfigMap must exist before the Pod that references it (unless marked `optional`)

---

## Consuming ConfigMaps

### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: my-app:v2
    env:
    # Single key from ConfigMap
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    # All keys from ConfigMap as env vars
    envFrom:
    - configMapRef:
        name: app-config
```

### As Volume Mount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config
```

When mounted as a volume, each key becomes a file in the directory, and the value becomes the file content.

---

## Secrets

Secrets are similar to ConfigMaps but are intended for **sensitive data**:

- Passwords and tokens
- TLS certificates and keys
- Docker registry credentials
- API keys and SSH keys

### Important: Base64 Encoding is NOT Encryption

```
  Secret values in etcd:
  +-----------------------------------------------+
  |  password: cGFzc3dvcmQxMjM=                   |
  |            ^^^^^^^^^^^^^^^^^^^^                |
  |            This is base64-encoded, NOT encrypted|
  |            echo "cGFzc3dvcmQxMjM=" | base64 -d |
  |            => password123                      |
  +-----------------------------------------------+
```

Secrets are base64-encoded by default. This is NOT a security measure -- anyone with access can decode them. For actual encryption:
- Enable **encryption at rest** in etcd
- Use external secret management (Vault, AWS Secrets Manager, etc.)
- Use RBAC to restrict access to Secrets

### Secret Types

| Type                                | Description                           |
|-------------------------------------|---------------------------------------|
| `Opaque`                           | Arbitrary user-defined data (default) |
| `kubernetes.io/tls`               | TLS certificate and key               |
| `kubernetes.io/dockerconfigjson`  | Docker registry credentials           |
| `kubernetes.io/basic-auth`        | Basic authentication credentials      |
| `kubernetes.io/ssh-auth`          | SSH private key                       |
| `kubernetes.io/service-account-token` | Service account token             |

---

## Secret YAML Spec

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque
data:
  # Values must be base64-encoded
  username: YWRtaW4=          # echo -n "admin" | base64
  password: cGFzc3dvcmQxMjM=  # echo -n "password123" | base64

---
# Alternative: use stringData for plain text (auto-encoded)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin              # Kubernetes base64-encodes this for you
  password: password123
```

### TLS Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

```bash
# Or create via kubectl
kubectl create secret tls tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

---

## Consuming Secrets

### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: my-app:v2
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

### As Volume Mount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: my-app:v2
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      defaultMode: 0400           # File permissions (read-only by owner)
```

When mounted as a volume, each key becomes a file:

```
  /etc/secrets/
    |-- username    (contains: admin)
    |-- password    (contains: password123)
```

---

## How Pods Consume ConfigMaps and Secrets Diagram

```
+------------------------------------------------------------------+
|                        KUBERNETES CLUSTER                         |
|                                                                   |
|   +------------------+          +------------------+              |
|   |    ConfigMap      |          |     Secret        |             |
|   |    "app-config"   |          |   "db-credentials"|            |
|   |                   |          |                    |            |
|   | DATABASE_HOST:    |          | username: YWRtaW4= |           |
|   |   db.example.com  |          | password: cGFzc3.. |           |
|   | LOG_LEVEL: info   |          |   (base64-encoded) |           |
|   +--------+---------+          +---------+----------+            |
|            |                              |                       |
|            |   Referenced by Pod spec     |                       |
|            +----------+   +--------------+                        |
|                       |   |                                       |
|                       v   v                                       |
|   +--------------------------------------------------+           |
|   |  POD                                              |           |
|   |                                                   |           |
|   |  +---------------------------------------------+ |           |
|   |  |  Container: app                              | |           |
|   |  |                                              | |           |
|   |  |  ENV VARS (from ConfigMap):                  | |           |
|   |  |    DATABASE_HOST=db.example.com              | |           |
|   |  |    LOG_LEVEL=info                            | |           |
|   |  |                                              | |           |
|   |  |  ENV VARS (from Secret):                     | |           |
|   |  |    DB_USERNAME=admin       (auto-decoded)    | |           |
|   |  |    DB_PASSWORD=password123 (auto-decoded)    | |           |
|   |  |                                              | |           |
|   |  |  MOUNTED FILES (from ConfigMap volume):      | |           |
|   |  |    /etc/config/app.properties                | |           |
|   |  |                                              | |           |
|   |  |  MOUNTED FILES (from Secret volume):         | |           |
|   |  |    /etc/secrets/username                     | |           |
|   |  |    /etc/secrets/password                     | |           |
|   |  +---------------------------------------------+ |           |
|   +--------------------------------------------------+           |
+------------------------------------------------------------------+
```

---

## Volume Mounts vs Environment Variables

| Aspect                    | Environment Variables               | Volume Mounts                     |
|---------------------------|-------------------------------------|-----------------------------------|
| **How consumed**          | `env` or `envFrom` in Pod spec     | Mounted as files in a directory   |
| **Update behavior**       | NOT updated after Pod starts       | Updated automatically (with delay)|
| **Use case**              | Simple key-value settings          | Config files (nginx.conf, etc.)   |
| **Visibility**            | Visible via `kubectl exec env`     | Visible as files in mountPath     |
| **Size suitability**      | Short values                       | Large config files                |
| **Subpath support**       | N/A                                | Yes, mount specific keys as files |

```
  Volume Mount Update Behavior:
  +-----------------------------------------------+
  |  ConfigMap updated (kubectl apply)             |
  |       |                                        |
  |       v                                        |
  |  kubelet detects change                        |
  |       |                                        |
  |       v                                        |
  |  Files in mounted volume updated               |
  |  (may take up to kubelet sync period, ~1 min)  |
  |                                                |
  |  NOTE: App must watch for file changes         |
  |        or be restarted to pick up new values   |
  +-----------------------------------------------+

  Env Var Behavior:
  +-----------------------------------------------+
  |  ConfigMap updated (kubectl apply)             |
  |       |                                        |
  |       X  Environment variables do NOT update   |
  |          Pod must be restarted to see changes  |
  +-----------------------------------------------+
```

---

## Immutable ConfigMaps and Secrets

Starting with Kubernetes 1.21 (stable), you can mark ConfigMaps and Secrets as **immutable**:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "db.example.com"
immutable: true                    # Cannot be changed after creation
```

### Benefits of Immutability

- **Performance**: Kubernetes does not need to watch for changes, reducing load on the API server
- **Safety**: Prevents accidental modifications that could break running applications
- **Scale**: Significant performance improvement in clusters with many ConfigMaps/Secrets

### Rules

- Once set to `immutable: true`, it **cannot** be changed back to mutable
- To update the config, you must **delete and recreate** the ConfigMap/Secret
- Pods referencing the old ConfigMap/Secret must be updated to reference the new one

---

## What to Remember for the Exam

1. **ConfigMaps** are for non-sensitive configuration data. **Secrets** are for sensitive data (passwords, tokens, keys).

2. **Secrets are base64-encoded, NOT encrypted**. Base64 is encoding, not security. Anyone who can read the Secret can decode it. Use encryption at rest and RBAC for real security.

3. **Two ways to consume** ConfigMaps and Secrets in Pods: as **environment variables** (`env`/`envFrom`) or as **volume mounts** (files in a directory).

4. **Environment variables are NOT updated** when ConfigMap/Secret changes -- the Pod must be restarted. **Volume mounts ARE updated** automatically (with a short delay).

5. **stringData** in Secrets lets you write values in plain text -- Kubernetes encodes them to base64 automatically. The `data` field requires values to already be base64-encoded.

6. **ConfigMaps and Secrets are namespaced** -- a Pod can only reference ConfigMaps and Secrets in the same namespace.

7. **Size limit**: Both ConfigMaps and Secrets are limited to **1 MiB**.

8. **Immutable ConfigMaps/Secrets** (immutable: true) cannot be modified after creation. They improve cluster performance and prevent accidental changes.

9. **Secret types** include Opaque (default), kubernetes.io/tls, kubernetes.io/dockerconfigjson, and others. Know that Opaque is the default.

10. **ConfigMap/Secret must exist** before the Pod starts (unless the reference is marked as `optional`). Otherwise, the Pod will not start.

11. **Volume-mounted Secrets** should use `readOnly: true` and restrictive file permissions (`defaultMode: 0400`) for security.
