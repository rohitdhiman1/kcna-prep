# Lab 14: Exploring Pod Networking

## Objective

Deploy pods, examine their IP addresses, test pod-to-pod communication, inspect the CNI plugin and kube-proxy configuration, and verify DNS resolution within the cluster.

## Prerequisites

- A running Kubernetes cluster (kind, minikube, or cloud).
- `kubectl` installed and configured.
- Read the concept file: [Networking](../concepts/networking.md)

---

## Step 1: Deploy Two Test Pods

Create two pods that we can use to test networking:

```bash
kubectl run pod-a --image=nginx:1.25 --labels="app=pod-a" --port=80
kubectl run pod-b --image=busybox:1.36 --labels="app=pod-b" --command -- sleep 3600
```

Wait for both pods to be running:

```bash
kubectl get pods -o wide
```

Expected output:

```
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
pod-a   1/1     Running   0          30s   10.244.0.5   kind-control-plane   <none>           <none>
pod-b   1/1     Running   0          28s   10.244.0.6   kind-control-plane   <none>           <none>
```

Note the `IP` column. Each pod gets its own unique IP address from the cluster's pod CIDR range.

---

## Step 2: Examine Pod IP Addresses

### Get pod IPs using jsonpath

```bash
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
```

### Inspect the pod networking details

```bash
kubectl describe pod pod-a | grep -A 5 "IP:"
```

### Check the pod CIDR range for the cluster

```bash
kubectl cluster-info dump | grep -m 1 "cluster-cidr"
```

Or check the node's pod CIDR allocation:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
```

Expected output:

```
kind-control-plane	10.244.0.0/24
```

---

## Step 3: Test Pod-to-Pod Communication

### Ping from pod-b to pod-a

```bash
kubectl exec pod-b -- wget -qO- --timeout=5 http://<pod-a-ip>
```

Replace `<pod-a-ip>` with the actual IP of pod-a from Step 1. You should see the default NGINX HTML response:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

### Use a one-liner to get pod-a's IP dynamically

```bash
POD_A_IP=$(kubectl get pod pod-a -o jsonpath='{.status.podIP}')
kubectl exec pod-b -- wget -qO- --timeout=5 http://$POD_A_IP
```

### Verify connectivity works in both directions

Run a command inside pod-a to reach pod-b (pod-b does not have a web server, but we can test basic IP connectivity):

```bash
POD_B_IP=$(kubectl get pod pod-b -o jsonpath='{.status.podIP}')
kubectl exec pod-a -- apt-get update -qq && kubectl exec pod-a -- apt-get install -y -qq iputils-ping > /dev/null 2>&1
kubectl exec pod-a -- ping -c 3 $POD_B_IP
```

You should see successful ping responses, confirming bidirectional pod-to-pod communication without NAT.

---

## Step 4: Check the CNI Plugin

The Container Network Interface (CNI) plugin is responsible for assigning IPs and configuring networking.

### List CNI configuration files

To inspect CNI config, access the node:

**For kind:**
```bash
docker exec kind-control-plane ls /etc/cni/net.d/
```

**For minikube:**
```bash
minikube ssh -- ls /etc/cni/net.d/
```

Expected output:

```
10-kindnet.conflist
```

### View the CNI configuration

**For kind:**
```bash
docker exec kind-control-plane cat /etc/cni/net.d/10-kindnet.conflist
```

This shows the JSON configuration including:
- The plugin type (e.g., `ptp`, `bridge`, `flannel`)
- IP address management (IPAM) settings
- Network ranges

### Identify common CNI plugins

| CNI Plugin | Used By |
|-----------|---------|
| kindnet | kind clusters |
| bridge | minikube (default) |
| flannel | Many on-prem clusters |
| calico | Production clusters (supports NetworkPolicy) |
| cilium | Advanced networking with eBPF |
| weave | Older clusters |

---

## Step 5: Inspect kube-proxy Mode

kube-proxy handles service networking. It runs on every node and can operate in different modes.

### Check kube-proxy pods

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

### View kube-proxy configuration

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
```

Expected output:

```
    mode: "iptables"
```

Common modes:
- **iptables** (default) — uses iptables rules for service routing
- **ipvs** — uses IPVS for better performance at scale
- **nftables** — newer option using nftables (Kubernetes v1.29+)

### View iptables rules created by kube-proxy (on the node)

**For kind:**
```bash
docker exec kind-control-plane iptables -t nat -L KUBE-SERVICES | head -20
```

This shows the NAT rules that route service ClusterIPs to pod endpoints.

---

## Step 6: Test DNS Resolution

Kubernetes runs CoreDNS to provide DNS-based service discovery.

### Resolve the kubernetes API service

```bash
kubectl exec pod-b -- nslookup kubernetes.default
```

Expected output:

```
Server:    10.96.0.10
Address:   10.96.0.10:53

Name:      kubernetes.default.svc.cluster.local
Address:   10.96.0.1
```

This shows:
- The DNS server IP (`10.96.0.10`) is the CoreDNS ClusterIP.
- `kubernetes.default` resolves to the API server's ClusterIP (`10.96.0.1`).

### Create a service and resolve it via DNS

```bash
kubectl expose pod pod-a --name=web-service --port=80
```

Now resolve it:

```bash
kubectl exec pod-b -- nslookup web-service.default
```

Expected output:

```
Server:    10.96.0.10
Address:   10.96.0.10:53

Name:      web-service.default.svc.cluster.local
Address:   10.96.22.135
```

### Access the service by DNS name

```bash
kubectl exec pod-b -- wget -qO- --timeout=5 http://web-service.default
```

You should see the NGINX welcome page, proving that DNS-based service discovery works.

### Understand DNS naming convention

The full DNS name for a service follows this pattern:

```
<service-name>.<namespace>.svc.cluster.local
```

You can use short names within the same namespace:
- `web-service` (same namespace)
- `web-service.default` (explicit namespace)
- `web-service.default.svc.cluster.local` (fully qualified)

---

## Step 7: Check CoreDNS

### View CoreDNS pods

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

### View CoreDNS configuration

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

Look for the `Corefile` key, which defines:
- The cluster domain (`cluster.local`)
- Upstream DNS servers (for external resolution)
- Caching and logging settings

### Test external DNS resolution from a pod

```bash
kubectl exec pod-b -- nslookup google.com
```

This should resolve, proving that CoreDNS forwards external queries to upstream DNS servers.

---

## Step 8: Examine Pod DNS Configuration

### Check the resolv.conf inside a pod

```bash
kubectl exec pod-b -- cat /etc/resolv.conf
```

Expected output:

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
ndots:5
```

- **nameserver** — points to the CoreDNS ClusterIP.
- **search** — allows short DNS names to be resolved by appending these suffixes.
- **ndots:5** — names with fewer than 5 dots are tried with search domains first.

---

## Cleanup

```bash
kubectl delete pod pod-a pod-b
kubectl delete service web-service
```

---

## Key Takeaways

1. Every pod gets a **unique IP address** from the cluster's pod CIDR range.
2. Pods can communicate directly with each other by IP **without NAT** (flat network model).
3. The **CNI plugin** is responsible for assigning pod IPs and configuring network interfaces.
4. **kube-proxy** implements service networking using iptables, IPVS, or nftables rules.
5. **CoreDNS** provides DNS-based service discovery within the cluster.
6. Services are reachable by DNS name: `<service>.<namespace>.svc.cluster.local`.
7. Pod DNS is configured via `/etc/resolv.conf`, which points to the CoreDNS ClusterIP.

---

## Next Steps

- Read about [Ingress and DNS](../concepts/ingress-dns.md) to learn how external traffic reaches your services.
- Try [Lab 15: Ingress](15-ingress.md) to configure external access with routing rules.
