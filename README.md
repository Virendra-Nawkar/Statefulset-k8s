# 🗄️ Kubernetes StatefulSet — Hands-on with MySQL

A practical, step-by-step guide to understanding and running Kubernetes StatefulSets using MySQL on a bare-metal / kubeadm cluster.

---

## 📖 What is a StatefulSet?

A **StatefulSet** is a Kubernetes workload object designed for **stateful applications** — apps that need:

- A **stable, unique network identity** per pod (e.g. `mysql-0`, `mysql-1`)
- **Persistent storage that survives pod restarts** (each pod keeps its own PVC)
- **Ordered, graceful deployment and scaling** (pods created/deleted sequentially)

### StatefulSet vs Deployment

| Feature | StatefulSet | Deployment |
|---|---|---|
| Pod names | Stable (`mysql-0`, `mysql-1`) | Random (`mysql-abc12`) |
| Storage | Own PVC per pod | Shared volume |
| Pod creation order | Sequential (0 → 1 → 2) | Parallel / random |
| Pod deletion order | Reverse (2 → 1 → 0) | Random |
| DNS per pod | ✅ Yes (via headless service) | ❌ No |
| PVC on pod delete | PVC survives | N/A |
| Use case | Databases, queues, Zookeeper | Stateless APIs, web apps |

---

## 🏗️ Architecture

```
Client / App
     │
     ▼
Headless Service (clusterIP: None)
mysql-svc.mysql-demo.svc.cluster.local
     │
     ├──────────────┬──────────────┐
     ▼              ▼              ▼
  mysql-0        mysql-1        mysql-2
(Primary)       (Replica)      (Replica)
     │              │              │
     ▼              ▼              ▼
PVC: data-     PVC: data-     PVC: data-
  mysql-0        mysql-1        mysql-2
     │              │              │
     ▼              ▼              ▼
PersistentVolume  PersistentVolume  PersistentVolume
```

Each pod gets its own **dedicated PersistentVolumeClaim** automatically named `mysql-data-mysql-0`, `mysql-data-mysql-1`, etc.

Pod DNS format:
```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
# Example:
mysql-0.mysql-svc.mysql-demo.svc.cluster.local
```

---

## ✅ Prerequisites

### 1. A running Kubernetes cluster (kubeadm)

This guide is tested on a **kubeadm bare-metal / VM cluster**. You need:

- Kubernetes `>= 1.24`
- `kubectl` configured and pointing to your cluster
- At least **1 worker node** with **2GB+ free memory**

### 2. Install local-path Provisioner (Required for bare-metal)

kubeadm does **not** include a storage provisioner by default. Install `local-path-provisioner` from Rancher:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

Wait for the provisioner pod to be Running:

```bash
kubectl get pods -n local-path-storage
# NAME                                      READY   STATUS
# local-path-provisioner-764fd965ff-xxxxx   1/1     Running ✓
```

### 3. Set local-path as the default StorageClass

```bash
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Verify it is marked as default:

```bash
kubectl get storageclass
# NAME                   PROVISIONER             VOLUMEBINDINGMODE
# local-path (default)   rancher.io/local-path   WaitForFirstConsumer ✓
```

> **Why is this needed?**  
> The `volumeClaimTemplates` in a StatefulSet automatically provisions PVCs. Without a storage provisioner, PVCs stay in `Pending` forever and pods never start.

---

## 📁 File Structure

```
mysql-statefulset-demo/
├── namespace.yaml           # Isolated namespace
├── mysql-secret.yaml        # Root password + DB name (never hardcode!)
├── mysql-headless-svc.yaml  # Headless service for stable pod DNS
└── mysql-statefulset.yaml   # The StatefulSet with volumeClaimTemplates
```

---

## 🚀 How to Run

### Step 1 — Clone or create the directory

```bash
mkdir mysql-statefulset-demo && cd mysql-statefulset-demo
```

### Step 2 — Create all manifest files

**`namespace.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mysql-demo
```

**`mysql-secret.yaml`**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: mysql-demo
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: "MyStr0ngPass!"
  MYSQL_DATABASE: "appdb"
```

**`mysql-headless-svc.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
  namespace: mysql-demo
  labels:
    app: mysql
spec:
  clusterIP: None        # <-- Makes it "headless" — required for StatefulSet
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
      name: mysql
```

**`mysql-statefulset.yaml`**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: mysql-demo
spec:
  serviceName: "mysql-svc"     # Must match the headless service name above
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_DATABASE
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            exec:
              command: ["mysqladmin", "ping", "-h", "127.0.0.1"]
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3

  # KEY DIFFERENCE from Deployment:
  # Each pod automatically gets its own PVC named: mysql-data-mysql-0, mysql-data-mysql-1 ...
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-path    # Must match your StorageClass name
        resources:
          requests:
            storage: 1Gi
```

### Step 3 — Apply in order

```bash
kubectl apply -f namespace.yaml
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-headless-svc.yaml
kubectl apply -f mysql-statefulset.yaml
```

### Step 4 — Watch pods come up sequentially

```bash
kubectl get pods -n mysql-demo -w
```

Expected output — pods start **one by one**:
```
NAME      READY   STATUS              AGE
mysql-0   0/1     ContainerCreating   5s
mysql-0   1/1     Running             35s   ← mysql-1 only starts after this
mysql-1   0/1     ContainerCreating   36s
mysql-1   1/1     Running             65s   ← mysql-2 only starts after this
mysql-2   0/1     ContainerCreating   66s
mysql-2   1/1     Running             95s
```

### Step 5 — Verify PVCs are bound

```bash
kubectl get pvc -n mysql-demo
```

Expected output:
```
NAME                 STATUS   VOLUME         CAPACITY   ACCESS MODES
mysql-data-mysql-0   Bound    pvc-aaa-xxx    1Gi        RWO          ✓
mysql-data-mysql-1   Bound    pvc-bbb-yyy    1Gi        RWO          ✓
mysql-data-mysql-2   Bound    pvc-ccc-zzz    1Gi        RWO          ✓
```

---

## 🧪 Hands-on Experiments

### Experiment 1 — Stable pod identity (DNS)

Each pod is addressable by a unique, predictable DNS name:

```bash
kubectl exec -it mysql-0 -n mysql-demo -- bash -c "hostname && cat /etc/resolv.conf"
```

Connect from `mysql-0` to `mysql-1` by its DNS name:

```bash
kubectl exec -it mysql-0 -n mysql-demo -- \
  mysql -h mysql-1.mysql-svc.mysql-demo.svc.cluster.local -u root -pMyStr0ngPass! -e "SELECT 1;"
```

---

### Experiment 2 — Data persists after pod deletion ⭐ (Most Important)

This is the core value of StatefulSet.

```bash
# Step 1: Write data into mysql-0
kubectl exec -it mysql-0 -n mysql-demo -- \
  mysql -u root -pMyStr0ngPass! -e "
    USE appdb;
    CREATE TABLE students (id INT, name VARCHAR(50));
    INSERT INTO students VALUES (1,'Alice'),(2,'Bob'),(3,'Charlie');
    SELECT * FROM students;
  "

# Step 2: Delete the pod — simulates a crash
kubectl delete pod mysql-0 -n mysql-demo

# Step 3: Watch it come back with the SAME name
kubectl get pods -n mysql-demo -w

# Step 4: Data is still there — PVC was NOT deleted
kubectl exec -it mysql-0 -n mysql-demo -- \
  mysql -u root -pMyStr0ngPass! -e "USE appdb; SELECT * FROM students;"
```

Expected result after pod restart:
```
+----+---------+
| id | name    |
+----+---------+
|  1 | Alice   |
|  2 | Bob     |
|  3 | Charlie |
+----+---------+
```

> **Why?** When a pod is deleted, Kubernetes recreates it with the **same name** (`mysql-0`) and **reattaches the same PVC** (`mysql-data-mysql-0`). The data is on the volume, not inside the container.

---

### Experiment 3 — Ordered scale up and down

```bash
# Scale UP to 5 replicas — mysql-3 starts after mysql-2 is ready, then mysql-4
kubectl scale statefulset mysql -n mysql-demo --replicas=5
kubectl get pods -n mysql-demo -w

# Scale DOWN to 2 replicas — deleted in REVERSE: mysql-4 first, then mysql-3, then mysql-2
kubectl scale statefulset mysql -n mysql-demo --replicas=2
kubectl get pods -n mysql-demo -w

# PVCs of deleted pods still exist — data is preserved
kubectl get pvc -n mysql-demo
# mysql-data-mysql-3 and mysql-data-mysql-4 still show here
```

---

### Experiment 4 — Connect to a specific pod

```bash
# Connect to primary (index 0)
kubectl exec -it mysql-0 -n mysql-demo -- mysql -u root -pMyStr0ngPass!

# Connect to a specific replica
kubectl exec -it mysql-1 -n mysql-demo -- mysql -u root -pMyStr0ngPass!
```

---

## 🔑 Key Concepts

### Why a Headless Service?

A regular service load-balances across all pods and hides their individual identities. A headless service (`clusterIP: None`) does the opposite — it creates individual DNS `A` records for each pod, so you can address `mysql-0`, `mysql-1`, `mysql-2` separately. This is essential for databases where the primary and replicas have different roles.

### Why volumeClaimTemplates?

In a Deployment, all pods share the same volume. In a StatefulSet, `volumeClaimTemplates` tells Kubernetes to create a **new, separate PVC for each pod** automatically. The naming convention is `<template-name>-<pod-name>` — so `mysql-data-mysql-0`, `mysql-data-mysql-1`, etc.

### What happens to PVCs when you delete a StatefulSet?

```bash
kubectl delete statefulset mysql -n mysql-demo
kubectl get pvc -n mysql-demo   # PVCs still exist!
```

PVCs are **NOT** automatically deleted. This is intentional — it protects your data from accidental loss. You must delete them manually when you're sure you no longer need the data.

### local-path provisioner — where is data stored?

Data is stored on the **node's local disk** at:
```
/opt/local-path-provisioner/<namespace>_<pvc-name>_<pv-name>/
```

This is great for learning and development. For production, use a distributed storage solution like Ceph, Longhorn, or a cloud provider's CSI driver.

---

## 🧹 Cleanup

```bash
# Delete everything
kubectl delete namespace mysql-demo

# PVCs are NOT deleted with the namespace in some versions — check and delete manually
kubectl get pvc -n mysql-demo
kubectl delete pvc --all -n mysql-demo
```

---

## 📚 What to Learn Next

| Topic | Description |
|---|---|
| MySQL Replication | Use init containers to clone data from `mysql-0` to replicas on first boot |
| Rolling Updates | Change image version and watch pods update in reverse order |
| PodDisruptionBudget | Ensure minimum replicas stay up during node maintenance |
| Liveness vs Readiness Probes | Understand why readiness matters for ordered startup |
| Headless Service DNS | Deep dive into CoreDNS and how pod DNS records are created |
| Longhorn / Ceph | Production-grade distributed storage for bare-metal clusters |

---

## 🛠️ Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Pod stuck in `Pending` | PVC cannot bind — no storage provisioner | Install `local-path-provisioner` (see Prerequisites) |
| PVC stuck in `Pending` | StorageClass not found or not default | Run `kubectl get storageclass` and verify `local-path (default)` exists |
| Pod stuck in `ContainerCreating` | Image pull issue or resource limits too high | Run `kubectl describe pod mysql-0 -n mysql-demo` |
| `mysql-1` never starts | `mysql-0` not yet `1/1 Running` | StatefulSet waits by design — check `mysql-0` logs |
| Data lost after pod restart | Volume not mounted / wrong `mountPath` | Verify `mountPath: /var/lib/mysql` matches MySQL's data directory |

---

## 📝 License

MIT — free to use for learning and teaching.
