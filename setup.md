### 🧠 Objective

Create a 4-node KIND cluster (`1 control-plane + 3 workers`) where you can:

* Test **sidecars**
* Apply **resource quotas**
* Practice **RBAC, ConfigMaps, Secrets, PV/PVC** etc.

---

### ⚙️ Step-by-step Setup

#### 1️⃣ Create a KIND config file

Create file: `kind-4node-cluster.yaml`

```yaml
# kind-4node-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: devsecops-lab
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

Optional: expose ports (for NodePort testing)

```yaml
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 8080
        protocol: TCP
      - containerPort: 30443
        hostPort: 8443
        protocol: TCP
```

Final version (if you want NodePort exposed):

```yaml
# kind-4node-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: devsecops-lab
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 8080
        protocol: TCP
      - containerPort: 30443
        hostPort: 8443
        protocol: TCP
  - role: worker
  - role: worker
  - role: worker
```

---

#### 2️⃣ Create the cluster

```bash
kind create cluster --config kind-4node-cluster.yaml
```

Check nodes:

```bash
kubectl get nodes -o wide
```

You should see:

```
NAME                         STATUS   ROLES           AGE   VERSION
devsecops-lab-control-plane  Ready    control-plane   1m    v1.30.x
devsecops-lab-worker         Ready    <none>          1m    v1.30.x
devsecops-lab-worker2        Ready    <none>          1m    v1.30.x
devsecops-lab-worker3        Ready    <none>          1m    v1.30.x
```

---

#### 3️⃣ Verify Core Components

```bash
kubectl get pods -n kube-system
```

Ensure all are running (CoreDNS, kube-proxy, etc.).

---

#### 4️⃣ Test workloads

Deploy a simple Nginx to validate:

```bash
kubectl run nginx --image=nginx:alpine --port=80
kubectl expose pod nginx --port=80 --type=NodePort
kubectl get svc nginx
```

Access via `http://localhost:8080` if NodePort mapped.

---

#### 5️⃣ When done:

Delete everything cleanly:

```bash
kind delete cluster --name devsecops-lab
```

