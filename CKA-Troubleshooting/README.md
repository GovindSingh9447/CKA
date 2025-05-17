# CKA Exam – Troubleshooting (30%) Deep Dive with Examples

---

## 1. Troubleshoot Clusters and Nodes

### Common Node/Cluster Issues

| Symptom                 | Possible Causes                            | Example Troubleshooting Steps         |
|-------------------------|--------------------------------------------|--------------------------------------|
| Node NotReady           | Kubelet down, network, resource pressure   | Check kubelet logs, node events      |
| Node lost/not joining   | Token/cert issues, network, kubelet config | Check `/var/log/kubelet.log`         |
| Frequent node restarts  | Resource starvation, OOM                   | Check system logs, `kubectl top`     |
| Scheduling failures     | Taints, NoSchedule, insufficient resources | Describe node/pod, check taints      |

### Example: Node NotReady

1. **Check node status**
   ```sh
   kubectl get nodes
   ```
   Output:
   ```
   NAME        STATUS     ROLES    AGE   VERSION
   worker-1    NotReady   <none>   10d   v1.29.0
   ```

2. **Describe node**
   ```sh
   kubectl describe node worker-1
   ```
   Look for `Conditions`:
   ```
   Conditions:
     Type             Status  LastHeartbeatTime
     Ready            False   ...
     MemoryPressure   False   ...
     DiskPressure     True    ...
   ```

3. **Check kubelet logs**
   ```sh
   journalctl -u kubelet -l
   ```
   Look for errors about disk, network, or kubelet startup.

4. **Check for disk usage**
   ```sh
   df -h
   ```

### Example: Node not joining cluster

- On the node:
  ```sh
  sudo kubeadm join ... # rerun the join command
  journalctl -xeu kubelet
  ```
- Common issues:
  - Expired token (regenerate with `kubeadm token create`)
  - Firewall/network (check `iptables`, security groups)
  - Wrong API server endpoint (check `/etc/kubernetes/kubelet.conf`)

---

## 2. Troubleshoot Cluster Components

### Example: API Server Down

1. **Check API server pod/container**
   ```sh
   docker ps | grep kube-apiserver
   # OR for static pods
   ls /etc/kubernetes/manifests/
   ```

2. **Check logs**
   ```sh
   journalctl -u kubelet
   docker logs <apiserver-container>
   ```

3. **Common errors**
   - Certificate expired/misconfigured
   - Port 6443 blocked
   - etcd unavailable

4. **Check etcd health**
   ```sh
   ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt endpoint health
   ```

---

## 3. Monitor Cluster and Application Resource Usage

### Example: Pod OOMKilled

1. **Check pod status**
   ```sh
   kubectl get pods
   kubectl describe pod <pod>
   ```
   Look for:
   ```
   State:          Terminated
   Reason:         OOMKilled
   ```

2. **Check resource usage**
   ```sh
   kubectl top pod <pod>
   ```

3. **Check resource requests/limits**
   ```sh
   kubectl get pod <pod> -o yaml | grep resources -A5
   ```
   - Update deployment with higher memory limit if necessary.

### Example: Monitoring Node Resources

```sh
kubectl top nodes
```
Output:
```
NAME        CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
worker-1    120m         6%     2500Mi          80%
```
High memory usage? Consider scheduling fewer pods or increasing node size.

---

## 4. Manage and Evaluate Container Output Streams

### Example: Inspect Container Logs

1. **View logs of a running pod**
   ```sh
   kubectl logs nginx-6db489d4b7-5k4rq
   ```

2. **View logs for a specific container**
   ```sh
   kubectl logs mypod -c mycontainer
   ```

3. **View previous logs after a crash**
   ```sh
   kubectl logs mypod --previous
   ```

4. **Stream logs in real-time**
   ```sh
   kubectl logs -f mypod
   ```

### Example: Debugging a CrashLoopBackOff

```sh
kubectl get pods
kubectl describe pod <crashing-pod>
kubectl logs <crashing-pod>
```
Look for errors related to initialization, config, or missing files.

---

## 5. Troubleshoot Services and Networking

### Service Not Reachable Example

1. **Check service and endpoints**
   ```sh
   kubectl get svc mysvc
   kubectl describe svc mysvc
   kubectl get endpoints mysvc
   ```
   If endpoints are empty, check pod labels and selector.

2. **Check DNS resolution**
   ```sh
   kubectl exec -it testpod -- nslookup mysvc
   ```

3. **Test reachability**
   ```sh
   kubectl exec -it testpod -- curl mysvc:80
   ```

4. **Check NetworkPolicy**
   ```sh
   kubectl get networkpolicy
   kubectl describe networkpolicy <policy>
   ```
   Ensure there isn’t a policy blocking traffic.

### Example: NodePort Not Accessible Externally

1. **Get NodePort**
   ```sh
   kubectl get svc mysvc
   ```
   Output:
   ```
   NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
   mysvc   NodePort   10.96.181.195   <none>        80:31234/TCP     10m
   ```

2. **Test from outside the cluster**
   ```sh
   curl <node-ip>:31234
   ```

3. **Check firewall rules on the node**

---

## General Troubleshooting Tips

- Use `kubectl get events --sort-by='.lastTimestamp'` for a timeline of recent events.
- Use `kubectl get all -n <namespace>` to see all resources in a namespace.
- Confirm resource existence and status before further debugging.
- For RBAC issues, check:
  ```sh
  kubectl auth can-i <verb> <resource> --as=<user>
  ```
- For stuck pods, check scheduling events and node taints:
  ```sh
  kubectl describe pod <pod>
  kubectl describe node <node>
  ```

---

## Useful One-Liners

- List all pods not running:
  ```sh
  kubectl get pods -A --field-selector=status.phase!=Running
  ```
- Find CrashLoopBackOff pods:
  ```sh
  kubectl get pods --all-namespaces | grep CrashLoopBackOff
  ```
- Print all events in the cluster:
  ```sh
  kubectl get events --all-namespaces --sort-by='.lastTimestamp'
  ```
- Check pod resource usage (requires metrics-server):
  ```sh
  kubectl top pod -A
  ```

---

## References

- [Kubernetes Troubleshooting Official Docs](https://kubernetes.io/docs/tasks/debug/)
- [Debugging Services](https://kubernetes.io/docs/tasks/debug/debug-service/)
- [Node Issues](https://kubernetes.io/docs/tasks/debug/debug-cluster/debug-cluster/)
- [Resource Monitoring](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/)
