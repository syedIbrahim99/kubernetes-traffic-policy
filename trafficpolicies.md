

# 🧱 SINGLE CLUSTER SETUP (FIXED)

## 🖥 Nodes

```text id="n1"
Master: 192.168.30.100   (NOT in data path)
Worker1: 192.168.30.101
Worker2: 192.168.30.102
```

---

## 📦 Pods (Pod IPs assigned by CNI)

```text id="n2"
nginx-pod-1   → 10.244.1.10  (Worker1)

python-pod-1  → 10.244.1.20  (Worker1)
python-pod-2  → 10.244.2.20  (Worker2)
```

---

## 🌐 Services

```text id="n3"
nginx-svc:
  ClusterIP → 10.96.0.10
  NodePort  → 30080

python-svc:
  ClusterIP → 10.96.0.20
  NodePort  → 30090
  replicas → 2 pods (python-pod-1, python-pod-2)
```

---

# 🚀 NOW WE EXPLAIN: ACCESS python-svc

We test:

```text id="n4"
Browser → http://192.168.30.102:30090
```

(Directly hitting Worker2 NodePort)

---

# 🟦 1) NODEPORT FLOW (DEFAULT = externalTrafficPolicy: Cluster)

## 🔥 STEP-BY-STEP PACKET FLOW

### 1️⃣ Request enters cluster

```text id="n5"
Browser (10.0.0.50)
  ↓
192.168.30.102:30090 (Worker2 NodePort)
```

---

### 2️⃣ kube-proxy catches NodePort rule

```text id="n6"
30090 → python-svc (10.96.0.20)
```

👉 DNAT happens

---

### 3️⃣ Service load balancing

kube-proxy chooses ANY pod:

```text id="n7"
python-pod-1 OR python-pod-2 (random)
```

Let’s assume it picks:

```text id="n8"
python-pod-2 (10.244.2.20 on Worker2)
```

---

### 4️⃣ Cross-node routing (if needed)

If it picked Worker1 pod:

```text id="n9"
Worker2 → Worker1 via CNI
```

If Worker2 pod:

```text id="n10"
local delivery only
```

---

### 5️⃣ SNAT happens (IMPORTANT)

Before reaching pod:

```text id="n11"
SRC: 10.0.0.50 ❌
→ becomes
SRC: 192.168.30.102 (Node IP)
```

👉 Pod NEVER sees real client IP

---

### 6️⃣ Response back

```text id="n12"
Pod → Node → Browser
```

---

## 🧠 NODEPORT SUMMARY

```text id="n13"
Browser → NodePort → Service → ANY Pod (cluster-wide)
        → SNAT hides client IP
        → cross-node allowed
```

---

# 🟩 2) CLUSTERIP FLOW (internal only)

Now nginx inside cluster calls python-svc:

```text id="n14"
nginx-pod-1 → python-svc
```

---

## 🔥 STEP FLOW

### 1️⃣ DNS resolution

```text id="n15"
python-svc → 10.96.0.20
```

---

### 2️⃣ packet goes to Service IP

```text id="n16"
10.244.1.10 (nginx)
  ↓
10.96.0.20
```

---

### 3️⃣ kube-proxy selects pod

```text id="n17"
python-pod-1 OR python-pod-2
```

Assume:

```text id="n18"
python-pod-1 (10.244.1.20)
```

---

### 4️⃣ routing inside cluster

```text id="n19"
Worker1 → Worker1 (local)
```

or

```text id="n20"
Worker1 → Worker2 (if selected)
```

---

### 5️⃣ NO SNAT (important difference)

```text id="n21"
SRC: 10.244.1.10 (nginx pod IP) ✔ preserved
```

---

## 🧠 CLUSTERIP SUMMARY

```text id="n22"
Pod → Service IP → ANY Pod
    → no external entry
    → client IP preserved
```

---

# 🟨 3) NODEPORT + externalTrafficPolicy = LOCAL

Now SAME request:

```text id="n23"
Browser → 192.168.30.102:30090
```

---

## 🔥 STEP FLOW

### 1️⃣ Request enters Worker2

```text id="n24"
Node: Worker2
Port: 30090
```

---

### 2️⃣ kube-proxy restriction applies

```text id="n25"
ONLY local pods allowed
```

So kube-proxy checks:

```text id="n26"
Is python pod on Worker2?
→ YES (python-pod-2)
```

---

### 3️⃣ NO cross-node routing

👉 It will NOT go to Worker1 pod even if available

---

### 4️⃣ DNAT directly to local pod

```text id="n27"
10.96.0.20 → 10.244.2.20
```

---

### 5️⃣ NO SNAT (VERY IMPORTANT)

```text id="n28"
SRC: 10.0.0.50 ✔ real client IP visible
```

---

### 6️⃣ Response

```text id="n29"
Pod → Node → Browser
```

---

## ❌ What if Worker2 has NO python pod?

```text id="n30"
Request → DROP 🚫
(no fallback to Worker1)
```

---

## 🧠 LOCAL SUMMARY

```text id="n31"
Node → ONLY local pod
    → no cross-node traffic
    → no SNAT
    → may drop request
```

---

# 🔥 FINAL COMPARISON (ONE VIEW)

```text id="n32"
1) ClusterIP
nginx → Service → ANY Pod
(no external access)

2) NodePort (Cluster)
Browser → Node → Service → ANY Pod
(SNAT + cross-node allowed)

3) NodePort (Local)
Browser → Node → LOCAL Pod only
(NO SNAT + no cross-node + possible drop)
```

---

# 🧠 SUPER SIMPLE MEMORY

```text id="n33"
ClusterIP = internal traffic only
NodePort (Cluster) = flexible but hides client IP
NodePort (Local) = strict but preserves client IP
```

---


