# CKA Exam – Cluster Architecture, Installation & Configuration (25%) Deep Dive with Examples

---

## 1. Manage Role Based Access Control (RBAC)

### What is RBAC?
- RBAC regulates access to resources based on roles assigned to users, groups, or service accounts.
- Main resources: `Role`, `ClusterRole`, `RoleBinding`, `ClusterRoleBinding`.

**Role Example (Namespace-scoped):**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

**RoleBinding Example:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
- kind: User
  name: alice
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRole Example (Cluster-wide):**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

**ClusterRoleBinding Example:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-global
subjects:
- kind: User
  name: bob
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

**Test Permissions:**
```sh
kubectl auth can-i list pods --as alice -n dev
```

---

## 2. Prepare Underlying Infrastructure for Kubernetes Cluster

- **Supported OS:** Ubuntu, CentOS, RHEL, etc.
- **Requirements:**  
  - 2+ CPUs, 2GB+ RAM per node  
  - Unique hostname, MAC address, product_uuid for each node
  - Network connectivity between all machines
  - Disable swap: `swapoff -a` and remove from `/etc/fstab`
  - Set up required ports and firewall (e.g., 6443, 2379-2380, 10250, 10251, 10252, 10255, etc.)
  - Enable IP forwarding, set sysctl params:
    ```sh
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOF
    sudo sysctl --system
    ```
- **Install container runtime:** containerd (preferred), Docker, or CRI-O.
- **Install kubeadm, kubelet, kubectl:**  
  Follow [official docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

---

## 3. Create and Manage Kubernetes Clusters using kubeadm

### Initialize Cluster (Control Plane Node)
```sh
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
- Save the `kubeadm join ...` command for worker nodes.
- Set up kubeconfig for kubectl:
  ```sh
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```

### Join Worker Node
```sh
sudo kubeadm join <control-plane-endpoint>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Install Pod Network Add-On (e.g., Flannel, Calico)
```sh
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```

### Manage Cluster Lifecycle

- **Upgrade:**  
  ```sh
  sudo kubeadm upgrade plan
  sudo kubeadm upgrade apply v1.29.0
  ```
- **Add/remove nodes:** Use `kubeadm join` and `kubectl drain/delete node`.

---

## 4. Implement and Configure a Highly-Available (HA) Control Plane

- Requires at least 3 control plane nodes.
- Use external load balancer for the API server endpoint.
- All control plane nodes run etcd, kube-apiserver, kube-controller-manager, kube-scheduler.

**HA Init:**
```sh
sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:6443" --upload-certs
```
- Join additional control plane nodes:
  ```sh
  sudo kubeadm join LOAD_BALANCER_DNS:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <key>
  ```

---

## 5. Use Helm and Kustomize to Install Cluster Components

### Helm (Package Manager for Kubernetes)
- **Install Helm:**  
  ```sh
  curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
  ```
- **Add repo and install chart:**
  ```sh
  helm repo add bitnami https://charts.bitnami.com/bitnami
  helm install my-nginx bitnami/nginx
  ```
- **Upgrade, rollback, uninstall:**
  ```sh
  helm upgrade my-nginx bitnami/nginx --set replicaCount=2
  helm rollback my-nginx 1
  helm uninstall my-nginx
  ```

### Kustomize (Customize raw YAML files)
- **Directory structure:**
  ```
  my-app/
    deployment.yaml
    service.yaml
    kustomization.yaml
  ```
- **kustomization.yaml Example:**
  ```yaml
  resources:
    - deployment.yaml
    - service.yaml
  images:
    - name: nginx
      newTag: 1.26
  ```
- **Apply with kubectl:**
  ```sh
  kubectl apply -k my-app/
  ```

---

## 6. Understand Extension Interfaces (CNI, CSI, CRI, etc.)

- **CNI (Container Network Interface):** Network plugins (Calico, Flannel, Weave) for pod networking.
- **CSI (Container Storage Interface):** Storage plugins for dynamic volume provisioning (AWS EBS CSI, GCE PD CSI).
- **CRI (Container Runtime Interface):** Interface between kubelet and container runtimes (containerd, CRI-O).

**Check installed CNI:**
```sh
kubectl get pods -n kube-system
kubectl describe pod <cni-pod>
```

---

## 7. Understand CRDs, Install and Configure Operators

### Custom Resource Definitions (CRDs)
- Extend Kubernetes API with custom resources.

**CRD Example:**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: Crontab
    shortNames:
    - ct
```

### Operators

- **Operators** use CRDs and controllers to automate application management (e.g., install, update, backup).
- Install via manifests or with Helm.

**Example: Install Prometheus Operator via Helm**
```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus prometheus-community/kube-prometheus-stack
```

---

## Useful Commands

```sh
kubectl get roles,rolebindings,clusterroles,clusterrolebindings --all-namespaces
kubectl auth can-i create pods --as <user> -n <namespace>
kubectl get nodes
kubectl get pods -n kube-system
kubectl get crds
kubectl describe crd <name>
kubectl get all -n <namespace>
```

---

## References

- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [kubeadm Install](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Helm Docs](https://helm.sh/docs/)
- [Kustomize Docs](https://kubectl.docs.kubernetes.io/pages/app_management/introduction.html)
- [Kubernetes Operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [CNI](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
- [CSI](https://kubernetes.io/docs/concepts/storage/volumes/#csi)
