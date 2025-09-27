# 0) Quick checklist (before starting)

* Windows: Docker Desktop installed and using **WSL2 backend** (Docker Desktop → Settings → Resources → WSL Integration ON for your distro).
* WSL2 distro (Ubuntu) is installed and default (run `wsl -l -v`).
* Docker Desktop is running.
* You have ~8 GB RAM (we’ll be conservative with resource usage).
  If any of these are missing, tell me and I’ll show install steps — otherwise continue.

---

# 1) Install small tooling in WSL2

Run these in WSL2 (Ubuntu):

```bash
# update and essentials
sudo apt update && sudo apt install -y curl ca-certificates gnupg lsb-release

# kubectl (client)
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client --short

# kind
curl -Lo ./kind "https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64"
chmod +x ./kind && sudo mv ./kind /usr/local/bin/
kind version

# (optional) helm (useful later)
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version --short
```

(If any command fails, paste the error and I’ll troubleshoot.)

---

# 2) Set Docker resources (recommended)

Open Docker Desktop → Settings → Resources:

* CPUs: **2**
* Memory: **4 GB** (or 5 GB if you can spare)
  This prevents your laptop from becoming painfully slow. Kind will run inside Docker containers, which use these resources.

---

# 3) Create a Kind config file (multi-node + useful mappings)

Create `kind-config.yaml` in WSL2 (edit with `nano` or `code`):

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      # map host 8080 -> control-plane:80 (example for testing)
      - containerPort: 80
        hostPort: 8080
        protocol: TCP
      - containerPort: 443
        hostPort: 8443
        protocol: TCP
  - role: worker
  - role: worker
```

Notes:

* `extraPortMappings` lets you hit NodePort/Ingress on `localhost:8080` etc.
* 3 nodes (1 control-plane + 2 workers) mimic small prod clusters.

---

# 4) Create the cluster

Run (still in WSL2):

```bash
kind create cluster --name dev-lab --config kind-config.yaml
```

Expected checks:

```bash
kubectl cluster-info
kubectl get nodes -o wide
```

You should see three nodes and the kubeconfig context `kind-dev-lab` will be set automatically.

---

# 5) Quick smoke test — deploy nginx

Run:

```bash
kubectl create deployment web --image=nginx:stable --replicas=2
kubectl expose deployment web --port=80 --type=NodePort --name=web-svc
kubectl get pods -o wide
kubectl get svc web-svc
```

Find the NodePort (30000-32767). If you want to hit it from Windows, either:

* Use `kubectl port-forward`:

```bash
kubectl port-forward svc/web-svc 8080:80
# then open http://localhost:8080 in Windows browser
```

* OR, if you set `extraPortMappings` for 80 -> 8080, create a Service with `nodePort: 80` (or use `hostPort` mapping shown earlier). For most cases `port-forward` is easiest.

Test:

```bash
curl -sS http://localhost:8080 | head -n 5
```

---

# 6) Using local images (build in WSL2 + load into Kind)

If you build a local image and want to use it in the Kind cluster:

```bash
# build with Docker (uses Docker Desktop daemon)
docker build -t myapp:1.0 .

# load into kind cluster
kind load docker-image myapp:1.0 --name dev-lab

# then deploy referencing myapp:1.0
kubectl create deployment myapp --image=myapp:1.0
```

`kind load` copies the image into all kind node container runtimes so pods can pull it instantly.

---

# 7) Helpful YAMLs (Deployment + NodePort service example)

Save this as `web-deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:stable
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Apply:

```bash
kubectl apply -f web-deploy.yaml
# access via:
kubectl port-forward svc/web-svc 8080:80   # easiest
# or if you mapped host/containers, curl http://localhost:30080
```

---

# 8) Scripts: quick start / rebuild / stop

Create `start-cluster.sh`:

```bash
#!/usr/bin/env bash
set -e
kind create cluster --name dev-lab --config kind-config.yaml
kubectl cluster-info
```

Create `delete-cluster.sh`:

```bash
#!/usr/bin/env bash
set -e
kind delete cluster --name dev-lab
```

Make them executable: `chmod +x start-cluster.sh delete-cluster.sh`.

**Why recreate?** Kind clusters are ephemeral and fast to recreate. Recreate is the most reliable way to get a clean state.

---

# 9) Stop / start without deleting (less reliable)

If you really want to stop and later try to resume (not guaranteed):

```bash
# stop
docker ps --filter "name=dev-lab" --format '{{.Names}}' | xargs -r docker stop

# start
docker ps -a --filter "name=dev-lab" --format '{{.Names}}' | xargs -r docker start
```

If something breaks on restart, best is `kind delete` + `kind create` again.

---

# 10) Useful commands & debugging

* Get nodes, pods, services:

  ```bash
  kubectl get nodes
  kubectl get pods -A
  kubectl get svc -A
  ```
* Pod logs:

  ```bash
  kubectl logs -l app=web
  kubectl logs <pod-name> -c <container-name>
  ```
* Describe objects:

  ```bash
  kubectl describe pod <pod>
  kubectl describe node <node>
  ```
* See node container logs (Docker side):

  ```bash
  docker logs dev-lab-control-plane
  ```
* If nodes become `NotReady`: restart Docker Desktop; check container status `docker ps`.

---

# 11) Resource / performance tips for 8GB laptop

* Keep images small (nginx, busybox). Avoid Elasticsearch / heavy DBs locally.
* Use 2 workers only and keep sample app replicas small.
* Lower container concurrency and scale down when not practicing.
* If laptop gets slow, delete cluster and recreate later — Kind is fast.

---

# 12) Short practice roadmap (what to practice once cluster is ready)

* Day 1: Pods, Deployments, ReplicaSets, Services (ClusterIP, NodePort), port-forward.
* Day 2: ConfigMaps, Secrets, Namespaces, Resource requests/limits.
* Day 3: Rolling updates, rollbacks, probes (readiness/liveness).
* Day 4: Affinity/anti-affinity, taints & tolerations, cordon/drain.
* Day 5: Storage basics — PersistentVolume + hostPath (use `extraMounts` if you want host persistence).
  (We’ll expand any day into commands & manifests.)

---

# 13) If you want host-mounted volumes (persist data)

Add `extraMounts` to the control-plane/worker node section of `kind-config.yaml`:

```yaml
  - role: worker
    extraMounts:
      - hostPath: /home/youruser/kind-pv
        containerPath: /kind-pv
```

Then create a HostPath PV in Kubernetes pointing to `/kind-pv`. This lets you persist test data between cluster recreations **only** if you keep the host directory and re-create the cluster with the same mounts.

---

Good thinking 👍 because with **Kind**, shutdown/restart isn’t as “clean” as with a VM or K3s.

Here’s how it works:

---

## 🛑 Shutting Down Kind

* A **Kind cluster is made of Docker containers** (1 for control-plane, N for workers).

* To stop the cluster **gracefully**:

  ```bash
  kind delete cluster --name my-cluster
  ```

  👉 This wipes the cluster completely.

* If you **just stop Docker** (`sudo service docker stop` or quitting Docker Desktop), the Kind node-containers stop too. But:

  * When you restart Docker, the containers may not always reconnect properly.
  * Often you’ll end up recreating the cluster.

---

## 🔄 Restarting

* If you shut down your laptop or Docker → you’ll likely need to recreate the cluster:

  ```bash
  kind create cluster --name my-cluster --config kind-config.yaml
  ```

  👉 This is why we keep a **YAML config file** for reproducible setup.

* If you want persistence (not deleting every time):

  * Just **pause/resume Docker Desktop** instead of deleting.
  * But be ready — sometimes clusters break → then `delete` + `create` is the cleanest way.

---

## 💡 Practical Workflow

1. Day 1:

   ```bash
   kind create cluster --name dev --config kind-config.yaml
   ```

   Work with Deployments, StatefulSets, etc.

2. End of day:

   * If you don’t care about keeping workloads →

     ```bash
     kind delete cluster --name dev
     ```
   * If you want to keep workloads → just **close laptop / stop Docker Desktop** (may or may not survive cleanly).

3. Next day:

   * Usually faster to **recreate from config**.
   * Apply your YAMLs again (`kubectl apply -f ...`).

---

⚖️ TL;DR:

* **Kind = ephemeral clusters** → delete & recreate is the norm.
* If you want a **persistent lab you can start/stop daily**, **K3s** or **Minikube** is better.

---

