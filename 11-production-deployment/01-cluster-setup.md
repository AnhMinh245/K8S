# 11.1. Production Cluster Setup

> Cách setup Kubernetes cluster production-ready từ đầu

---

## 🎯 Mục Tiêu

- ✅ Hiểu Managed vs Self-Hosted K8s
- ✅ Setup production cluster trên cloud
- ✅ Network architecture design
- ✅ High availability configuration
- ✅ Security từ infrastructure level

---

## 🏢 Managed vs Self-Hosted

### Managed Kubernetes (Recommended cho hầu hết cases)

**Các options:**

| Provider | Service | Pros | Cons |
|----------|---------|------|------|
| **Google Cloud** | GKE | Mature nhất, auto-upgrade, nhiều features | Vendor lock-in |
| **AWS** | EKS | Integration tốt với AWS services | Phức tạp hơn GKE |
| **Azure** | AKS | Windows containers support | Ít mature hơn |
| **DigitalOcean** | DOKS | Đơn giản, giá rẻ | Ít features advanced |

**Ưu điểm Managed:**
```
✅ Control Plane managed (không lo upgrade, patching)
✅ High availability built-in
✅ Auto-upgrades có thể enable
✅ Integration với cloud services (LB, storage)
✅ Managed add-ons (monitoring, logging)
✅ Less operational overhead
✅ SLA guaranteed
```

**Nhược điểm:**
```
❌ Cost cao hơn self-hosted
❌ Ít control hơn
❌ Vendor lock-in
❌ Một số customization bị limit
```

---

### Self-Hosted (Advanced users only)

**Khi nào nên self-host:**
- Có requirement đặc biệt về compliance
- Cần control hoàn toàn
- On-premise deployment
- Cost optimization cực đoan
- Team có expertise K8s operations

**Tools để setup:**
- **kubeadm** - Official K8s tool
- **kops** - AWS-focused
- **kubespray** - Ansible-based
- **Rancher** - Management platform

**Nhược điểm:**
```
❌ Phải manage Control Plane
❌ Upgrade phức tạp
❌ High availability cần setup manual
❌ Security patching là responsibility của bạn
❌ Operational overhead lớn
❌ Cần team có expertise
```

---

## 🚀 Production Cluster trên Google Cloud (GKE)

### Architecture Overview

```
┌───────────────────────────────────────────────────┐
│              PRODUCTION CLUSTER (GKE)             │
├───────────────────────────────────────────────────┤
│                                                   │
│  Region: us-central1 (Multi-zonal)               │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │  Control Plane (Managed by Google)          │ │
│  │  • Highly Available (3 zones)               │ │
│  │  • Auto-upgrades                            │ │
│  │  • Regional endpoint                        │ │
│  └─────────────────────────────────────────────┘ │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │  Node Pools                                 │ │
│  │                                             │ │
│  │  Pool 1: System (us-central1-a/b/c)        │ │
│  │  • 3 nodes (n2-standard-4)                 │ │
│  │  • Taints: CriticalAddonsOnly              │ │
│  │  • Monitoring, logging agents              │ │
│  │                                             │ │
│  │  Pool 2: Application (us-central1-a/b/c)   │ │
│  │  • 6-20 nodes (autoscaling)                │ │
│  │  • n2-standard-8                           │ │
│  │  • Workload pods                           │ │
│  │                                             │ │
│  │  Pool 3: Spot (us-central1-a/b/c)          │ │
│  │  • 0-10 nodes (autoscaling)                │ │
│  │  • Spot instances (cheap)                  │ │
│  │  • Non-critical workloads                  │ │
│  └─────────────────────────────────────────────┘ │
│                                                   │
└───────────────────────────────────────────────────┘
```

---

### Setup với Terraform

**Directory structure:**
```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── gke.tf
├── network.tf
└── modules/
    ├── gke-cluster/
    └── node-pools/
```

**gke.tf:**
```hcl
# Production GKE Cluster
resource "google_container_cluster" "primary" {
  name     = "production-cluster"
  location = var.region  # Regional cluster (HA)
  
  # Remove default node pool (we'll create custom pools)
  remove_default_node_pool = true
  initial_node_count       = 1
  
  # Network configuration
  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.private.name
  
  # Private cluster (nodes không có public IP)
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false  # API server có public endpoint (với whitelist)
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }
  
  # IP allocation for pods và services
  ip_allocation_policy {
    cluster_secondary_range_name  = "pod-range"
    services_secondary_range_name = "service-range"
  }
  
  # Master authorized networks (whitelist IPs)
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "0.0.0.0/0"  # Production: Replace với office/VPN IPs
      display_name = "All"
    }
  }
  
  # Workload Identity (recommended for security)
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
  
  # Add-ons
  addons_config {
    http_load_balancing {
      disabled = false  # Enable Ingress
    }
    horizontal_pod_autoscaling {
      disabled = false  # Enable HPA
    }
    network_policy_config {
      disabled = false  # Enable Network Policies
    }
  }
  
  # Network policy
  network_policy {
    enabled = true
  }
  
  # Maintenance window
  maintenance_policy {
    daily_maintenance_window {
      start_time = "03:00"  # 3 AM UTC
    }
  }
  
  # Cluster autoscaling
  cluster_autoscaling {
    enabled = true
    autoscaling_profile = "OPTIMIZE_UTILIZATION"
    
    resource_limits {
      resource_type = "cpu"
      minimum       = 10
      maximum       = 100
    }
    
    resource_limits {
      resource_type = "memory"
      minimum       = 40
      maximum       = 400
    }
  }
  
  # Release channel (auto-upgrades)
  release_channel {
    channel = "REGULAR"  # RAPID, REGULAR, hoặc STABLE
  }
  
  # Binary authorization (image security)
  binary_authorization {
    evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
  }
}
```

**Node Pools:**

```hcl
# System node pool (cho critical add-ons)
resource "google_container_node_pool" "system" {
  name       = "system-pool"
  cluster    = google_container_cluster.primary.id
  node_count = 1  # Per zone (total 3 nodes)
  
  node_locations = [
    "${var.region}-a",
    "${var.region}-b",
    "${var.region}-c",
  ]
  
  autoscaling {
    min_node_count = 1  # Per zone
    max_node_count = 2
  }
  
  node_config {
    machine_type = "n2-standard-4"
    disk_size_gb = 50
    disk_type    = "pd-standard"
    
    # Service account với least privilege
    service_account = google_service_account.gke_nodes.email
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
    
    # Taints để chỉ system pods chạy ở đây
    taint {
      key    = "CriticalAddonsOnly"
      value  = "true"
      effect = "NO_SCHEDULE"
    }
    
    labels = {
      pool = "system"
      env  = "production"
    }
    
    # Security
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
    
    workload_metadata_config {
      mode = "GKE_METADATA"
    }
  }
  
  management {
    auto_repair  = true
    auto_upgrade = true
  }
}

# Application node pool
resource "google_container_node_pool" "applications" {
  name    = "app-pool"
  cluster = google_container_cluster.primary.id
  
  node_locations = [
    "${var.region}-a",
    "${var.region}-b",
    "${var.region}-c",
  ]
  
  autoscaling {
    min_node_count = 2  # Per zone (total 6 nodes minimum)
    max_node_count = 7  # Per zone (total 21 nodes maximum)
  }
  
  node_config {
    machine_type = "n2-standard-8"
    disk_size_gb = 100
    disk_type    = "pd-ssd"  # SSD cho better performance
    
    service_account = google_service_account.gke_nodes.email
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
    
    labels = {
      pool = "applications"
      env  = "production"
    }
    
    # Resource reservations
    reservation_affinity {
      consume_reservation_type = "NO_RESERVATION"
    }
    
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
    
    workload_metadata_config {
      mode = "GKE_METADATA"
    }
  }
  
  management {
    auto_repair  = true
    auto_upgrade = true
  }
}

# Spot instance pool (cho non-critical workloads)
resource "google_container_node_pool" "spot" {
  name    = "spot-pool"
  cluster = google_container_cluster.primary.id
  
  node_locations = [
    "${var.region}-a",
    "${var.region}-b",
    "${var.region}-c",
  ]
  
  autoscaling {
    min_node_count = 0
    max_node_count = 3  # Per zone
  }
  
  node_config {
    machine_type = "n2-standard-8"
    spot         = true  # Spot instances (80% cheaper!)
    disk_size_gb = 100
    
    service_account = google_service_account.gke_nodes.email
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
    
    taint {
      key    = "spot"
      value  = "true"
      effect = "NO_SCHEDULE"
    }
    
    labels = {
      pool     = "spot"
      env      = "production"
      workload = "non-critical"
    }
    
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
    
    workload_metadata_config {
      mode = "GKE_METADATA"
    }
  }
  
  management {
    auto_repair  = true
    auto_upgrade = true
  }
}
```

---

### Network Configuration

**VPC setup:**

```hcl
# VPC Network
resource "google_compute_network" "vpc" {
  name                    = "production-vpc"
  auto_create_subnetworks = false
}

# Private subnet cho GKE
resource "google_compute_subnetwork" "private" {
  name          = "gke-private-subnet"
  ip_cidr_range = "10.0.0.0/20"  # 4096 IPs cho nodes
  region        = var.region
  network       = google_compute_network.vpc.id
  
  # Secondary ranges cho pods và services
  secondary_ip_range {
    range_name    = "pod-range"
    ip_cidr_range = "10.4.0.0/14"  # 262144 IPs cho pods
  }
  
  secondary_ip_range {
    range_name    = "service-range"
    ip_cidr_range = "10.8.0.0/20"  # 4096 IPs cho services
  }
  
  private_ip_google_access = true  # Access Google APIs
}

# Cloud Router cho NAT
resource "google_compute_router" "router" {
  name    = "gke-router"
  region  = var.region
  network = google_compute_network.vpc.id
}

# Cloud NAT (cho nodes private access internet)
resource "google_compute_router_nat" "nat" {
  name                               = "gke-nat"
  router                             = google_compute_router.router.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}

# Firewall rules
resource "google_compute_firewall" "allow_internal" {
  name    = "gke-allow-internal"
  network = google_compute_network.vpc.name
  
  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }
  
  allow {
    protocol = "udp"
    ports    = ["0-65535"]
  }
  
  allow {
    protocol = "icmp"
  }
  
  source_ranges = ["10.0.0.0/8"]  # Internal traffic
}
```

---

## 🔐 Security Configuration

### IAM và Service Accounts

```hcl
# GKE Node service account (least privilege)
resource "google_service_account" "gke_nodes" {
  account_id   = "gke-nodes"
  display_name = "GKE Nodes Service Account"
}

# Minimum required roles
resource "google_project_iam_member" "gke_nodes_roles" {
  for_each = toset([
    "roles/logging.logWriter",
    "roles/monitoring.metricWriter",
    "roles/monitoring.viewer",
    "roles/stackdriver.resourceMetadata.writer",
  ])
  
  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}

# Workload Identity (cho pods access GCP services)
resource "google_service_account" "workload_identity" {
  account_id   = "workload-identity"
  display_name = "Workload Identity Service Account"
}

# Grant roles to workload identity
resource "google_service_account_iam_member" "workload_identity_user" {
  service_account_id = google_service_account.workload_identity.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[NAMESPACE/KSA_NAME]"
}
```

---

## 📊 Post-Setup Configuration

### 1. Connect to Cluster

```bash
# Get cluster credentials
gcloud container clusters get-credentials production-cluster \
  --region us-central1 \
  --project YOUR_PROJECT_ID

# Verify connection
kubectl cluster-info
kubectl get nodes
```

### 2. Install Essential Add-ons

**cert-manager (cho TLS certificates):**
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

**Nginx Ingress Controller:**
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

**Prometheus + Grafana:**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

---

### 3. Configure Node Pools

**Taint system pool:**
```bash
kubectl taint nodes -l pool=system CriticalAddonsOnly=true:NoSchedule
```

**Tolerations cho system pods:**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: logging-agent
  namespace: kube-system
spec:
  template:
    spec:
      tolerations:
      - key: CriticalAddonsOnly
        operator: Equal
        value: "true"
        effect: NoSchedule
      nodeSelector:
        pool: system
```

---

## 🎯 Production Cluster Checklist

### Infrastructure
```
□ Multi-zone deployment (3+ zones)
□ Private nodes (no public IPs)
□ Cloud NAT configured
□ Network policies enabled
□ Regional cluster (not zonal)
□ Cluster autoscaling enabled
```

### Security
```
□ Private cluster enabled
□ Master authorized networks configured
□ Workload Identity enabled
□ Binary authorization enabled
□ Shielded nodes enabled
□ Service accounts với least privilege
```

### Reliability
```
□ Multiple node pools (system, app, spot)
□ Node auto-repair enabled
□ Node auto-upgrade enabled
□ Maintenance window configured
□ Pod disruption budgets set
```

### Monitoring
```
□ Cloud Monitoring integrated
□ Cloud Logging integrated
□ Prometheus deployed
□ Grafana dashboards
□ Alert rules configured
```

### Networking
```
□ Load balancer type: LoadBalancer hoặc Ingress
□ DNS configured
□ SSL/TLS certificates
□ Firewall rules reviewed
```

---

## 💰 Cost Optimization Tips

**1. Use Spot Instances:**
```
✓ 60-90% cheaper
✓ Good cho non-critical workloads
✓ Combine với PodDisruptionBudgets
```

**2. Right-size Node Types:**
```
✓ Monitor actual usage
✓ Don't over-provision
✓ Use autoscaling
```

**3. Use Cluster Autoscaling:**
```
✓ Scale down khi idle
✓ Set appropriate min/max
✓ Monitor scaling behavior
```

**4. Reserve Capacity (nếu usage predictable):**
```
✓ Committed use discounts (30-70% off)
✓ For stable workloads
```

---

## 🎓 Key Takeaways

**1. Managed Kubernetes:**
- Preferred cho production
- Less operational overhead
- Built-in HA và upgrades

**2. Multi-Zone Deployment:**
- High availability
- Resilient to zone failures
- Regional clusters recommended

**3. Security from Start:**
- Private nodes
- Workload Identity
- Least privilege IAM

**4. Cost Optimization:**
- Mix of standard và spot instances
- Cluster autoscaling
- Right-sized node types

**5. Infrastructure as Code:**
- Terraform for reproducibility
- Version controlled
- Easy to replicate

---

[⬅️ Phần 11: Overview](./README.md) | [➡️ 11.2. Deployment Strategies](./02-deployment-strategies.md) | [🏠 Mục Lục](../README.md)

