# CKA Exam – Workloads & Scheduling (15%) Deep Dive with Examples

---

## 1. Application Deployments: Rolling Updates and Rollbacks

### Deployments
- **Deployment** manages ReplicaSets and provides declarative updates for Pods.
- Supports rolling updates and rollbacks for zero-downtime deployments.

### Example: Create a Deployment
```sh
kubectl create deployment nginx-deploy --image=nginx:1.25
kubectl get deployments
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

### Rolling Update
- Default strategy is rolling update.
- Update image:
  ```sh
  kubectl set image deployment/nginx-deploy nginx=nginx:1.26
  ```
- Monitor rollout:
  ```sh
  kubectl rollout status deployment/nginx-deploy
  ```

### Rollback
- Roll back to previous version:
  ```sh
  kubectl rollout undo deployment/nginx-deploy
  ```

---

## 2. ConfigMaps and Secrets

### ConfigMaps
- Used to store non-confidential config data in key-value pairs.
- Can be consumed as environment variables, command-line args, or mounted as files.

**Create ConfigMap:**
```sh
kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2
```

**Mount ConfigMap as env vars:**
```yaml
envFrom:
- configMapRef:
    name: my-config
```

### Secrets
- Used to store sensitive data (passwords, tokens, etc.), base64-encoded.
- Can be used as env variables or mounted as files.

**Create Secret:**
```sh
kubectl create secret generic my-secret --from-literal=password=Pa$$w0rd
```

**Mount Secret in Pod:**
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: my-secret
      key: password
```

---

## 3. Workload Autoscaling

### Horizontal Pod Autoscaler (HPA)
- Scales the number of pods based on CPU/memory usage or custom metrics.

**Install metrics-server (required for HPA):**
```sh
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Create HPA:**
```sh
kubectl autoscale deployment nginx-deploy --cpu-percent=50 --min=2 --max=5
kubectl get hpa
```

**Example HPA YAML:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deploy
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

---

## 4. Self-Healing & Robust Deployments: Probes and ReplicaSets

### ReplicaSets
- Ensure a specified number of pod replicas are running at all times.

### Liveness and Readiness Probes

**Liveness Probe:** Restarts containers that become unresponsive.
**Readiness Probe:** Marks pod as ready to receive traffic.

**Example:**
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 3
  periodSeconds: 5
```

- If probe fails, kubelet restarts the container (liveness) or removes pod from service endpoints (readiness).

---

## 5. Pod Admission and Scheduling

### Resource Requests and Limits

**Specify in Pod/Deployment:**
```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

### Node Affinity

- Schedule pods on nodes that match node labels.

**Example:**
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
```

### Taints and Tolerations

- **Taints** prevent pods from being scheduled unless they have matching **tolerations**.

**Taint a node:**
```sh
kubectl taint nodes node1 key=value:NoSchedule
```

**Pod toleration:**
```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

### Pod Priority and Preemption

- Use `priorityClassName` to control pod priorities. Higher-priority pods can preempt lower-priority pods if resources are scarce.

**Example:**
```yaml
priorityClassName: high-priority
```

---

## Useful Commands

```sh
kubectl get deployments
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>
kubectl describe deployment <name>
kubectl get hpa
kubectl describe hpa <name>
kubectl get configmap,secret
kubectl describe pod <name>
kubectl top pod
kubectl taint nodes <node> key=value:NoSchedule
kubectl label nodes <node> disktype=ssd
```

---

## References

- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Pod Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)
