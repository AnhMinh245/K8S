# 5.1. Pod Networking

> Hiểu cách Pods giao tiếp với nhau trong K8s

---

## 🎯 Kubernetes Network Model

### Nguyên Tắc Cơ Bản

**1. Every Pod gets unique IP address**
```
Pod A: 10.1.1.5
Pod B: 10.1.1.6
Pod C: 10.1.2.10
```

**2. Pods can communicate without NAT**
```
Pod A (10.1.1.5) → Pod B (10.1.1.6)
  Direct communication, no translation
```

**3. Flat network space**
```
All Pods in same network (logically)
  Like all devices on same LAN
```

---

## 🌐 Network Architecture

```
┌─────────────────────────────────────────┐
│            Node 1                       │
│  ┌──────────┐  ┌──────────┐            │
│  │ Pod A    │  │ Pod B    │            │
│  │ 10.1.1.5 │  │ 10.1.1.6 │            │
│  └────┬─────┘  └────┬─────┘            │
│       └──────┬──────┘                   │
│              │ veth pairs               │
│         ┌────▼─────┐                    │
│         │  Bridge  │                    │
│         └────┬─────┘                    │
│              │                          │
└──────────────┼──────────────────────────┘
               │ CNI Plugin (Calico/Flannel)
               │
┌──────────────┼──────────────────────────┐
│         ┌────▼─────┐                    │
│         │  Bridge  │                    │
│         └────┬─────┘                    │
│       ┌──────┴──────┐                   │
│  ┌────▼─────┐  ┌───▼──────┐            │
│  │ Pod C    │  │ Pod D    │            │
│  │ 10.1.2.10│  │ 10.1.2.11│            │
│  └──────────┘  └──────────┘            │
│            Node 2                       │
└─────────────────────────────────────────┘
```

---

## 🔌 CNI (Container Network Interface)

**CNI plugins implement K8s networking**

### Popular CNI Plugins

**1. Calico**
- Feature-rich
- Network policies
- BGP routing
- Production-grade

**2. Flannel**
- Simple overlay network
- Easy setup
- Good for beginners

**3. Weave**
- Mesh network
- Automatic discovery
- Encryption support

**4. Cilium**
- eBPF-based
- Advanced features
- High performance

---

## 📡 Communication Patterns

### 1. Pod-to-Pod (Same Node)

```
┌─────────────────────┐
│       Node 1        │
│  ┌──────┐  ┌──────┐ │
│  │Pod A │→ │Pod B │ │
│  └──────┘  └──────┘ │
│      Via bridge      │
└─────────────────────┘

curl http://10.1.1.6:8080
  → Direct, fast (same host)
```

### 2. Pod-to-Pod (Different Nodes)

```
Node 1               Node 2
┌──────┐            ┌──────┐
│Pod A │ ─────────→ │Pod B │
└──────┘  Network   └──────┘
          (CNI)

curl http://10.1.2.10:8080
  → Routed via CNI plugin
```

### 3. Containers in Same Pod

```
┌──────────────────┐
│      Pod         │
│  ┌──────────┐    │
│  │Container1│    │
│  │localhost │    │
│  │  :8080   │    │
│  └──────────┘    │
│  ┌──────────┐    │
│  │Container2│    │
│  │localhost │    │
│  │  :9090   │    │
│  └──────────┘    │
└──────────────────┘

Container1: curl localhost:9090
  → Works! (shared network namespace)
```

---

## 🔍 DNS trong Kubernetes

**CoreDNS** provides DNS for cluster

### Pod DNS

```
Pod: my-pod
Namespace: default

Full DNS:
  my-pod.default.pod.cluster.local
  
But typically access via Service DNS (next section)
```

### Service DNS

```
Service: web-service
Namespace: default

DNS:
  web-service.default.svc.cluster.local
  web-service.default.svc
  web-service.default
  web-service  (if in same namespace)
```

---

## 💡 Key Takeaways

1. **Every Pod:** Unique IP
2. **No NAT:** Direct Pod-to-Pod communication
3. **CNI plugins:** Implement networking
4. **Same Pod:** Containers share localhost
5. **DNS:** CoreDNS resolves service/pod names
6. **Flat network:** All Pods logically on same network

---

[➡️ 5.2. Service](./02-services.md) | [🏠 Mục Lục](../README.md)

