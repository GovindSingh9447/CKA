# CKA Exam – Services & Networking (20%) 
---

## 1. Pod-to-Pod Connectivity

### How Pod Networking Works

- **Kubernetes mandates a flat, routable network:** Every Pod gets a unique IP, and any Pod can communicate with any other Pod in the cluster—no NAT, no port mapping.
- **CNI (Container Network Interface):** Plugins like Flannel, Calico, Cilium, Weave Net implement this; each has different features (policy support, IPAM, encryption, etc.).
- **Pod IPs:** Assigned by the CNI. You can see a Pod's IP via `kubectl get pod -o wide`.

### Examples

```sh
kubectl run testbox --image=busybox --restart=Never -it -- sh
# From inside, try:
ip addr
ping <another-pod-ip>
# Or test DNS-based service discovery:
nslookup <service-name>
```

**If pods can't reach each other:**
- Check if CNI is installed: `kubectl get pods -n kube-system`
- Check CNI pod logs, Pod IP ranges, and node routing tables.

---

## 2. Network Policies: Restricting Pod Traffic

### Key Concepts

- **NetworkPolicy** objects specify how groups of pods are allowed to communicate with each other and with other network endpoints.
- **Without any NetworkPolicy:** All traffic is allowed (east-west open by default).
- **Once a policy selects a pod:** That pod will deny all traffic not explicitly allowed by policies.

### Types of Policies

- **Ingress:** Controls what is allowed to the selected pods.
- **Egress:** Controls what is allowed from the selected pods.

### Example: Deny All, Then Allow Specific

**Deny all ingress to pods with `role=db`:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-db
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
```

**Allow only frontend pods to access db pods:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-db
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
```

**Allow egress to external IPs (e.g., 8.8.8.8):**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-external
spec:
  podSelector:
    matchLabels:
      role: frontend
  egress:
    - to:
        - ipBlock:
            cidr: 8.8.8.8/32
```

**Testing:**
- Use `kubectl exec -it <pod> -- wget <target-ip>` or `curl` to verify.

---

## 3. Services and Endpoints

### Types of Services

| Type         | Use Case                            | Exposed Where         |
|--------------|-------------------------------------|-----------------------|
| ClusterIP    | Default, in-cluster comms           | Inside cluster        |
| NodePort     | Expose on every Node’s IP+Port      | External via Node IP  |
| LoadBalancer | Cloud load balancer (cloud only)    | External (cloud)      |
| ExternalName | Alias to external DNS name          | Cluster DNS           |

**Service YAML Example: NodePort**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
  type: NodePort
```

**Check Endpoints:**
```sh
kubectl get endpoints my-app
kubectl describe svc my-app
```
- If `endpoints` are empty, check Pod labels and selectors.

**Accessing a NodePort:**
- `curl http://<NodeIP>:30080`

### Traffic Flow

- **ClusterIP:** kube-proxy configures iptables/ipvs to route to Pods.
- **NodePort:** Exposes the Service on each node’s IP at the specified port.
- **LoadBalancer:** Integrates with cloud provider for external IP.

---

## 4. Gateway API and Ingress

### Gateway API

- **Gateway API**: Modern, extensible, and more expressive than legacy Ingress.
- **Resources:** GatewayClass, Gateway, HTTPRoute, TCPRoute, etc.
- **Controllers:** Must be installed (e.g., Istio, Envoy Gateway, GKE Gateway).

**Example: Gateway & HTTPRoute**

```yaml
# GatewayClass
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: my-gateway-class
spec:
  controllerName: example.net/gateway-controller

# Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: my-gateway-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80

# HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
    - name: my-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /app
      backendRefs:
        - name: my-app
          port: 80
```

---

### Ingress Controllers and Ingress Resources

- **Ingress**: Manages HTTP(S) routing to Services based on host/path.
- **Ingress Controller**: Watches for Ingress resources and configures proxy/load balancer (e.g., NGINX, Traefik, HAProxy).

**Install NGINX Ingress Controller (cloud example):**
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.5/deploy/static/provider/cloud/deploy.yaml
```

**Ingress Resource Example:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: www.example.com
      http:
        paths:
          - path: /test
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

**TLS Example:**
```yaml
spec:
  tls:
    - hosts:
        - www.example.com
      secretName: example-tls
```

**Check Ingress status:**
```sh
kubectl get ingress
kubectl describe ingress my-ingress
```

---

## 5. CoreDNS: Kubernetes DNS Service Discovery

- **CoreDNS** is deployed in `kube-system` namespace.
- **Service DNS names**: `<service>.<namespace>.svc.cluster.local`
- **Pod DNS names** (rare): `<pod-ip>.<namespace>.pod.cluster.local`

**How Service Discovery Works:**
- Applications reference other services by DNS name (e.g., `my-app.default.svc.cluster.local`).
- Pods use `/etc/resolv.conf` to resolve internal names via CoreDNS.

**Test DNS Resolution:**
```sh
kubectl run -it --rm --restart=Never --image=busybox testdns -- nslookup my-app
```

**Check CoreDNS health:**
```sh
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

**Troubleshooting:**
- If DNS lookups fail, check CoreDNS pods, ConfigMap, and node firewall rules (UDP/53).
- CoreDNS config at: `kubectl -n kube-system get configmap coredns -o yaml`

---

## 6. Troubleshooting and Deep Testing

**Check Service endpoints:**
```sh
kubectl get svc
kubectl describe svc <name>
kubectl get endpoints <name>
```

**Test connectivity from inside a pod:**
```sh
kubectl exec -it <pod> -- sh
curl http://<service>:<port>
nslookup <service>
```

**Test NetworkPolicy:**
- Create two different Pods (with and without correct labels).
- Try to access target Pod/Service from both. Only the allowed one should succeed.

**Check Ingress/Gateway logs:**
```sh
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
kubectl get gateway
kubectl describe gateway <name>
```

---

## 7. Useful Commands Summary

```sh
kubectl get svc
kubectl describe svc <name>
kubectl get endpoints <name>
kubectl get networkpolicy
kubectl describe networkpolicy <name>
kubectl get ingress
kubectl describe ingress <name>
kubectl get gateways.gateway.networking.k8s.io
kubectl get httproutes.gateway.networking.k8s.io
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl exec -it <pod> -- sh
kubectl run testbox --image=busybox --restart=Never -it -- sh
```

---

## Further Reading

- [Kubernetes Networking Concepts](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [CoreDNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [CNI Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
