

# 🧱 1) CLUSTER SETUP (FIXED VALUES)

## 🖥 Nodes

```text
Worker1 → 192.168.30.101
Worker2 → 192.168.30.102
```

---

## 📦 Pods (from your nginx deployment)

```text
nginx-pod-1 → 10.244.1.10 (on Worker1)
nginx-pod-2 → 10.244.2.10 (on Worker2)
```

---

## 🌐 Service (NodePort)

```text
nginx-service
ClusterIP → 10.96.0.10
NodePort  → 30007
type      → NodePort
externalTrafficPolicy → Cluster (default)
```

---

## 🌍 Client (Browser)

```text
Client IP → 10.0.0.50
```

---

# 🚀 2) USER REQUEST (START POINT)

User opens browser:

```text
http://192.168.30.102:30007
```

👉 Request goes to **Worker2 NodePort**

---

# 🔥 3) FULL PACKET FLOW (REAL NETWORK VIEW)

---

## 🟦 STEP 1 — Packet enters node

```text
SRC: 10.0.0.50
DST: 192.168.30.102:30007
```

👉 Packet reaches **Worker2 network interface**

---

## 🟦 STEP 2 — kube-proxy sees NodePort rule

Inside node (iptables / IPVS), rule exists:

```text
30007 → nginx-service (10.96.0.10:80)
```

👉 Now **DNAT starts**

---

# 🔁 DNAT PHASE (Destination NAT)

## 🟦 STEP 3 — NodePort → Service IP

Packet becomes:

```text
SRC: 10.0.0.50
DST: 10.96.0.10:80   (ClusterIP)
```

👉 Destination changed (NodePort → Service)

---

## 🟦 STEP 4 — Service → Pod (load balancing)

kube-proxy selects one pod:

```text
Option A → nginx-pod-1 (10.244.1.10)
Option B → nginx-pod-2 (10.244.2.10)
```

Let’s assume:

```text
Selected → nginx-pod-1 (Worker1)
```

---

## 🟦 STEP 5 — DNAT continues (FINAL destination)

Packet becomes:

```text
SRC: 10.0.0.50
DST: 10.244.1.10:80
```

👉 Now destination is **actual pod**

---

# 🔁 SNAT PHASE (Source NAT)

## 🟦 STEP 6 — SNAT happens BEFORE leaving node

Because:

```text
externalTrafficPolicy = Cluster
```

Node modifies source:

```text
SRC: 10.0.0.50 ❌
→ becomes
SRC: 192.168.30.102 ✔
```

---

## 🟦 FINAL packet sent to pod

```text
SRC: 192.168.30.102
DST: 10.244.1.10
```

👉 This packet now travels:

```text
Worker2 → Worker1 (via CNI network)
```

---

# 📦 4) PACKET ARRIVES AT POD

Pod sees:

```text
SRC: 192.168.30.102   (Node IP)
DST: 10.244.1.10
```

👉 ❗ Pod DOES NOT know real client IP

---

# 🔁 5) RESPONSE FLOW (VERY IMPORTANT)

## 🟦 STEP 7 — Pod replies

```text
SRC: 10.244.1.10
DST: 192.168.30.102
```

👉 Because SNAT changed source earlier

---

## 🟦 STEP 8 — Node receives response

Worker2 gets it and reverses NAT (conntrack):

### Reverse SNAT:

```text
DST: 192.168.30.102 → 10.0.0.50
```

### Reverse DNAT:

```text
SRC: 10.244.1.10 → 192.168.30.102
```

---

## 🟦 FINAL response to client

```text
SRC: 192.168.30.102
DST: 10.0.0.50
```

---

# 🧠 FULL FLOW (ONE LINE)

```text
Client
→ NodeIP:NodePort
→ DNAT → ServiceIP
→ DNAT → PodIP
→ SNAT → NodeIP
→ Pod
→ Response → Node
→ Reverse NAT
→ Client
```

---

# 🔥 IMPORTANT OBSERVATIONS

## ✔ DNAT happens TWO TIMES

```text
1) NodePort → ClusterIP
2) ClusterIP → PodIP
```

---

## ✔ SNAT happens ONE TIME

```text
Client IP → Node IP
```

---

## ✔ Happens ALWAYS in Cluster mode

Even if:

```text
Worker2 → nginx-pod-2 (same node)
```

👉 SNAT still happens

---

# 🧠 WHY SNAT IS NEEDED

Without SNAT:

```text
Pod → Client directly (may bypass Node)
→ routing breaks
→ connection fails
```

With SNAT:

```text
Pod → Node → Client
→ stable return path ✔
```

---

# ⚖️ MINI COMPARISON (for memory)

```text
Cluster mode:
DNAT ✔
SNAT ✔
Any pod ✔

Local mode:
DNAT ✔
SNAT ❌
Only local pod ✔
```

---

# 🧠 FINAL MENTAL MODEL

Think like this:

```text
NodePort = "Node becomes proxy"

DNAT → sends traffic to pod
SNAT → forces pod to reply back to node
```

---


