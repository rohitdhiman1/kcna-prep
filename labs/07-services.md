# Lab 07: Services

## Objective

Create ClusterIP and NodePort services, expose a deployment, and test service discovery using DNS from within a pod.

## Prerequisites

- A running Kubernetes cluster (from [Lab 01](01-cluster-setup.md)).
- `kubectl` installed and configured.
- Read the concept file: [Services](../concepts/services.md)

---

## Step 1: Create a Deployment to Expose

```bash
kubectl create deployment web-svc --image=nginx:1.27 --replicas=3
```

Verify:

```bash
kubectl get pods -l app=web-svc -o wide
```

Note the pod IPs. Each pod has its own IP, but these IPs are ephemeral and change when pods are recreated. Services provide a stable endpoint.

---

## Step 2: Create a ClusterIP Service

ClusterIP is the default service type. It exposes the service on an internal cluster IP, accessible only from within the cluster.

```bash
kubectl expose deployment web-svc --name=web-clusterip --port=80 --target-port=80 --type=ClusterIP
```

### Verify the service

```bash
kubectl get service web-clusterip
```

Expected output:

```
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
web-clusterip    ClusterIP   10.96.45.123   <none>        80/TCP    5s
```

### View service details

```bash
kubectl describe service web-clusterip
```

Look for:
- **Selector** — `app=web-svc` (matches the deployment's pod labels).
- **Endpoints** — the IP addresses of all matching pods.

### Verify endpoints

```bash
kubectl get endpoints web-clusterip
```

Expected output:

```
NAME             ENDPOINTS                                      AGE
web-clusterip    10.244.1.5:80,10.244.2.3:80,10.244.2.4:80     30s
```

The endpoints list should match the pod IPs from Step 1.

---

## Step 3: Test the ClusterIP Service from Inside the Cluster

ClusterIP services are only reachable from within the cluster. Use a temporary pod to test:

```bash
kubectl run test-client --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://web-clusterip
```

Expected output: the default nginx HTML page.

The `--rm` flag deletes the test pod when it exits.

---

## Step 4: Create a NodePort Service

NodePort exposes the service on a static port on each node's IP, making it accessible from outside the cluster.

```bash
kubectl expose deployment web-svc --name=web-nodeport --port=80 --target-port=80 --type=NodePort
```

### Verify the service

```bash
kubectl get service web-nodeport
```

Expected output:

```
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
web-nodeport    NodePort   10.96.78.234    <none>        80:31234/TCP   5s
```

The `80:31234` means port 80 on the service maps to port 31234 on every node. The NodePort is randomly assigned from the range 30000-32767.

### Test the NodePort service

Get a node's internal IP:

```bash
kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}'
```

For kind clusters, you can test using docker exec:

```bash
NODE_PORT=$(kubectl get service web-nodeport -o jsonpath='{.spec.ports[0].nodePort}')
docker exec kcna-lab-worker curl -s http://localhost:$NODE_PORT
```

Expected output: the nginx welcome page HTML.

---

## Step 5: Create a Service from YAML

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-yaml-svc
spec:
  selector:
    app: web-svc
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 80
  type: ClusterIP
EOF
```

This creates a service on port 8080 that forwards to port 80 on the pods.

### Test it

```bash
kubectl run test-yaml --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://web-yaml-svc:8080
```

---

## Step 6: Service Discovery with DNS

Kubernetes creates DNS records for services automatically. Every service is reachable at:

```
<service-name>.<namespace>.svc.cluster.local
```

### Test DNS resolution from inside a pod

Start a debug pod with DNS tools:

```bash
kubectl run dns-test --rm -it --image=busybox:1.36 --restart=Never -- /bin/sh
```

Inside the pod, run:

```sh
# Short name (same namespace)
nslookup web-clusterip

# Fully qualified domain name
nslookup web-clusterip.default.svc.cluster.local
```

Expected output:

```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-clusterip
Address 1: 10.96.45.123 web-clusterip.default.svc.cluster.local
```

### Test DNS for the other services

```sh
nslookup web-nodeport
nslookup web-yaml-svc
```

Both should resolve to their respective ClusterIP addresses.

### Check /etc/resolv.conf

```sh
cat /etc/resolv.conf
```

Expected output:

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- **nameserver** — points to the CoreDNS service.
- **search** — allows short names like `web-clusterip` to be resolved by appending these suffixes.

Type `exit` to leave the pod.

---

## Step 7: Test with a Pod Using dig (Optional)

For more detailed DNS output, use a pod with `dig`:

```bash
kubectl run dig-test --rm -it --image=tutum/dnsutils --restart=Never -- dig web-clusterip.default.svc.cluster.local
```

Look at the **ANSWER SECTION** in the output for the resolved IP address.

---

## Step 8: Observe Service Load Balancing

Run multiple requests through the service to see it distribute traffic across pods:

```bash
kubectl run lb-test --rm -it --image=busybox:1.36 --restart=Never -- /bin/sh -c '
for i in $(seq 1 10); do
  wget -qO- http://web-clusterip | grep -o "Welcome to nginx"
  echo " - request $i"
done
'
```

All requests go through the single service IP, but kube-proxy distributes them across the backend pods.

---

## Cleanup

```bash
kubectl delete deployment web-svc
kubectl delete service web-clusterip web-nodeport web-yaml-svc
```

---

## Key Takeaways

1. **ClusterIP** (default) makes a service reachable only within the cluster via a stable virtual IP.
2. **NodePort** exposes the service on a static port on every node, making it accessible from outside.
3. Services use **label selectors** to discover which pods to route traffic to.
4. **Endpoints** are the actual pod IPs backing a service. They update automatically as pods come and go.
5. Kubernetes DNS creates records for every service: `<service>.<namespace>.svc.cluster.local`.
6. Pods can reach services in the same namespace using just the service name (e.g., `http://web-clusterip`).
7. The `search` domains in `/etc/resolv.conf` enable short-name resolution.
8. kube-proxy handles the actual load balancing using iptables or IPVS rules.

---

## Next Steps

- Proceed to [Lab 08: Namespaces and Labels](08-namespaces-labels.md) to organize resources.
- Read about [Networking](../concepts/networking.md) concepts.
