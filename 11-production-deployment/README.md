# 📘 Phần 11: Triển Khai K8s Trên Production

> Kiến thức thực tế để deploy và vận hành Kubernetes trong môi trường production

---

## 🎯 Mục Tiêu Phần Này

Sau khi hoàn thành Phần 11, bạn sẽ:

✅ **Biết setup** production cluster (managed vs self-hosted)  
✅ **Hiểu** deployment strategies và best practices  
✅ **Implement** CI/CD pipeline với K8s  
✅ **Secure** cluster theo security standards  
✅ **Setup** monitoring và logging đầy đủ  
✅ **Backup** và disaster recovery  
✅ **Optimize** cost và performance  
✅ **Troubleshoot** production issues  

---

## 📚 Nội Dung

### [11.1. Production Cluster Setup](./01-cluster-setup.md) ⭐⭐⭐⭐⭐
**Thời gian**: 90-120 phút

**Nội dung:**
- Managed Kubernetes (GKE, EKS, AKS) vs Self-Hosted
- Cluster sizing và node types
- Network architecture (VPC, subnets, security groups)
- High availability setup (multi-zone/region)
- Control plane HA
- Cluster autoscaling
- Node pools và taints/tolerations

**Key Topics:**
```
✓ Chọn cloud provider phù hợp
✓ Infrastructure as Code (Terraform)
✓ Network design (private vs public subnets)
✓ Multi-AZ deployment
✓ Bastion hosts và secure access
```

---

### [11.2. Deployment Strategies](./02-deployment-strategies.md) ⭐⭐⭐⭐⭐
**Thời gian**: 60-90 phút

**Nội dung:**
- Blue-Green Deployment
- Canary Deployment
- Rolling Deployment (advanced)
- A/B Testing
- Feature Flags
- Rollback strategies

**Key Topics:**
```
✓ Zero-downtime deployments
✓ Traffic shifting với Istio/NGINX Ingress
✓ Automated rollback
✓ Progressive delivery
✓ Testing in production
```

---

### [11.3. CI/CD Integration](./03-cicd-integration.md) ⭐⭐⭐⭐⭐
**Thời gian**: 90-120 phút

**Nội dung:**
- GitOps workflow (ArgoCD, Flux)
- CI pipelines (GitLab CI, GitHub Actions, Jenkins)
- Image building và registry
- Automated testing
- Security scanning (Trivy, Snyk)
- Deployment automation

**Key Topics:**
```
✓ Git as single source of truth
✓ Automated deployments
✓ Container scanning
✓ Secrets management trong CI/CD
✓ Environment promotion (dev → staging → prod)
```

---

### [11.4. Security Best Practices](./04-security-hardening.md) ⭐⭐⭐⭐⭐
**Thời gian**: 90-120 phút

**Nội dung:**
- RBAC (Role-Based Access Control)
- Pod Security Standards/Policies
- Network Policies
- Secrets management (External Secrets, Vault)
- Image security
- Runtime security (Falco)
- Compliance và auditing

**Key Topics:**
```
✓ Least privilege principle
✓ Zero-trust networking
✓ Encrypted secrets
✓ Security scanning
✓ Compliance frameworks (CIS, PCI-DSS)
```

---

### [11.5. Monitoring & Logging](./05-monitoring-logging.md) ⭐⭐⭐⭐⭐
**Thời gian**: 120-180 phút

**Nội dung:**
- Prometheus + Grafana stack
- Metrics collection và visualization
- Log aggregation (ELK, Loki)
- Distributed tracing (Jaeger)
- Alerting (AlertManager, PagerDuty)
- SLI/SLO/SLA implementation

**Key Topics:**
```
✓ Golden signals (latency, traffic, errors, saturation)
✓ Custom metrics
✓ Log retention policies
✓ Alert fatigue prevention
✓ On-call playbooks
```

---

### [11.6. Backup & Disaster Recovery](./06-backup-dr.md) ⭐⭐⭐⭐
**Thời gian**: 60-90 phút

**Nội dung:**
- Backup strategies (Velero)
- etcd backup/restore
- Database backup
- PersistentVolume snapshots
- Multi-region DR
- RTO/RPO planning

**Key Topics:**
```
✓ Automated backups
✓ Recovery procedures
✓ Testing DR plans
✓ Cross-region replication
✓ Business continuity
```

---

### [11.7. Cost Optimization](./07-cost-optimization.md) ⭐⭐⭐⭐
**Thời gian**: 60-90 phút

**Nội dung:**
- Resource right-sizing
- Spot/Preemptible instances
- Autoscaling strategies
- Cost monitoring tools (Kubecost)
- Namespace quotas
- Pod priorities
- Efficient storage usage

**Key Topics:**
```
✓ Cost visibility
✓ Resource optimization
✓ Savings plans
✓ Chargeback/showback
✓ Idle resource detection
```

---

### [11.8. Troubleshooting Production Issues](./08-troubleshooting.md) ⭐⭐⭐⭐⭐
**Thời gian**: 90-120 phút

**Nội dung:**
- Common production issues
- Debugging workflows
- Performance troubleshooting
- Network issues
- Storage issues
- Application crashes
- Resource exhaustion
- Tools và commands

**Key Topics:**
```
✓ Systematic debugging approach
✓ kubectl debug techniques
✓ Log analysis
✓ Metrics interpretation
✓ War room procedures
```

---

### [11.9. Real-World Case Studies](./09-case-studies.md) ⭐⭐⭐⭐
**Thời gian**: 60-90 phút

**Nội dung:**
- E-commerce platform deployment
- Microservices migration
- Multi-tenant SaaS
- Data pipeline on K8s
- Gaming backend
- Lessons learned
- War stories

**Key Topics:**
```
✓ Architecture decisions
✓ Scaling challenges
✓ Incident responses
✓ Migration strategies
✓ Success metrics
```

---

## 🎯 Learning Approach

### Thứ Tự Học Recommend

**1. Fundamental Production Skills (Học Trước):**
```
├─ 11.1. Cluster Setup (must know)
├─ 11.4. Security (critical)
├─ 11.5. Monitoring & Logging (critical)
└─ 11.8. Troubleshooting (essential)
```

**2. Advanced Topics (Học Tiếp):**
```
├─ 11.2. Deployment Strategies
├─ 11.3. CI/CD Integration
├─ 11.6. Backup & DR
└─ 11.7. Cost Optimization
```

**3. Real-World Learning:**
```
└─ 11.9. Case Studies (học cuối để consolidate)
```

---

## 💻 Hands-On Projects

### Project 1: Setup Production Cluster
```
Goal: Tạo production-ready K8s cluster

Tasks:
├─ Setup GKE/EKS cluster với Terraform
├─ Configure VPC, subnets, security groups
├─ Multi-AZ deployment
├─ Setup cluster autoscaling
└─ Configure IAM/RBAC

Deliverable: Infrastructure as Code repository
```

### Project 2: Complete CI/CD Pipeline
```
Goal: Automated deployment pipeline

Tasks:
├─ Setup GitLab CI/GitHub Actions
├─ Build và push Docker images
├─ Security scanning
├─ Deploy với ArgoCD/Flux
└─ Automated rollback

Deliverable: Working CI/CD pipeline
```

### Project 3: Monitoring Stack
```
Goal: Comprehensive observability

Tasks:
├─ Deploy Prometheus + Grafana
├─ Setup log aggregation (Loki)
├─ Configure alerting
├─ Create dashboards
└─ Setup distributed tracing

Deliverable: Full monitoring stack
```

### Project 4: Security Hardening
```
Goal: Secure cluster theo best practices

Tasks:
├─ Implement RBAC
├─ Setup Pod Security Standards
├─ Network Policies
├─ External secrets với Vault
└─ Security scanning

Deliverable: Hardened cluster
```

---

## 📊 Production Readiness Checklist

### Infrastructure
```
□ Multi-AZ/region deployment
□ Cluster autoscaling configured
□ Node pools optimized
□ Network policies in place
□ LoadBalancer/Ingress configured
□ DNS và certificates setup
```

### Security
```
□ RBAC configured
□ Pod Security Standards enforced
□ Network Policies active
□ Secrets encrypted
□ Image scanning automated
□ Audit logging enabled
```

### Monitoring & Logging
```
□ Metrics collection (Prometheus)
□ Dashboards (Grafana)
□ Log aggregation (ELK/Loki)
□ Distributed tracing (Jaeger)
□ Alerting configured
□ On-call rotation setup
```

### Deployment
```
□ CI/CD pipeline functional
□ GitOps workflow
□ Automated testing
□ Canary/blue-green capability
□ Rollback procedures tested
```

### Disaster Recovery
```
□ Backup automation (Velero)
□ etcd backup schedule
□ DR plan documented
□ Recovery tested
□ RTO/RPO defined
```

### Cost Management
```
□ Resource quotas set
□ Cost monitoring tools
□ Autoscaling optimized
□ Spot instances utilized
□ Idle resource alerts
```

---

## 🏆 Real-World Scenarios

### Scenario 1: High-Traffic Event (Black Friday)
```
Preparation:
├─ Capacity planning
├─ Pre-scale infrastructure
├─ Optimize autoscaling
├─ War room setup
└─ Rollback procedures ready

During Event:
├─ Monitor metrics closely
├─ Quick incident response
├─ Dynamic scaling
└─ Performance optimization

Post-Event:
├─ Cost analysis
├─ Performance review
└─ Lessons learned
```

### Scenario 2: Security Incident
```
Detection:
├─ Security alerts triggered
├─ Abnormal behavior detected
└─ Log analysis

Response:
├─ Isolate affected resources
├─ Investigate breach
├─ Apply patches
└─ Communication plan

Recovery:
├─ Restore from backup
├─ Security hardening
└─ Post-mortem
```

### Scenario 3: Database Migration
```
Planning:
├─ Migration strategy (blue-green)
├─ Data sync setup
├─ Rollback plan
└─ Testing in staging

Execution:
├─ DNS cutover
├─ Data validation
├─ Performance monitoring
└─ Gradual traffic shift

Post-Migration:
├─ Monitoring
├─ Cleanup old resources
└─ Documentation update
```

---

## 💡 Production Tips

### DO ✅
```
✓ Automate everything possible
✓ Document runbooks
✓ Test disaster recovery
✓ Monitor costs regularly
✓ Regular security audits
✓ Keep K8s version up-to-date
✓ Use Infrastructure as Code
✓ Implement proper logging
✓ Have rollback plans
✓ Practice chaos engineering
```

### DON'T ❌
```
✗ Manual deployments
✗ Root containers
✗ Secrets in plain text
✗ No resource limits
✗ Single point of failure
✗ No backups
✗ Ignore alerts
✗ Skip testing
✗ No documentation
✗ Over-provision resources
```

---

## 🎓 Skills Matrix

### Junior DevOps/SRE
```
Focus:
├─ 11.1. Cluster Setup (basic)
├─ 11.5. Monitoring basics
├─ 11.8. Basic troubleshooting
└─ Follow established procedures
```

### Mid-Level DevOps/SRE
```
Focus:
├─ All sections (good understanding)
├─ CI/CD implementation
├─ Security hardening
├─ Incident response
└─ Performance optimization
```

### Senior DevOps/SRE
```
Focus:
├─ Architecture design
├─ Complex troubleshooting
├─ Capacity planning
├─ Cost optimization
├─ Team mentoring
└─ Process improvement
```

---

## 🚀 Certification Relevance

### CKA (Certified Kubernetes Administrator)
```
Relevant sections:
├─ 11.1. Cluster Setup ⭐⭐⭐
├─ 11.4. Security (RBAC) ⭐⭐⭐
├─ 11.6. Backup & DR ⭐⭐⭐
└─ 11.8. Troubleshooting ⭐⭐⭐
```

### CKS (Certified Kubernetes Security Specialist)
```
Relevant sections:
├─ 11.4. Security Hardening ⭐⭐⭐⭐⭐
├─ 11.3. CI/CD (security scanning) ⭐⭐⭐
└─ 11.5. Monitoring (security events) ⭐⭐⭐
```

### CKAD (Certified Kubernetes Application Developer)
```
Relevant sections:
├─ 11.2. Deployment Strategies ⭐⭐⭐
├─ 11.3. CI/CD Integration ⭐⭐⭐
└─ 11.8. Troubleshooting ⭐⭐⭐
```

---

## 📚 Prerequisites

**Trước khi học Phần 11, bạn nên:**
- ✅ Hoàn thành Phần 1-8 (fundamentals)
- ✅ Có kinh nghiệm deploy apps trên K8s
- ✅ Hiểu kubectl commands cơ bản
- ✅ Biết YAML và command line
- ✅ Có access vào cloud provider (GCP/AWS/Azure)

---

## 🎯 Khi Nào "Production Ready"?

**Bạn đã production-ready khi:**

✅ Cluster có HA (multi-AZ)  
✅ Security hardened (RBAC, PSP, Network Policies)  
✅ Monitoring & alerting hoạt động  
✅ CI/CD automated  
✅ Backup & DR tested  
✅ Có runbooks và documentation  
✅ Team trained và on-call ready  
✅ Cost visibility và optimization  

---

## 🌟 Learning Outcome

**Sau khi hoàn thành Phần 11:**

🎓 **Kiến thức:**
- Hiểu production architecture patterns
- Biết security best practices
- Nắm deployment strategies

🛠️ **Kỹ năng:**
- Setup production clusters
- Implement CI/CD pipelines
- Troubleshoot production issues
- Optimize costs và performance

💼 **Career:**
- Ready cho DevOps/SRE roles
- Có thể manage production K8s
- Interview confidence cao

---

## 🚀 Bắt Đầu

**Recommended starting point:**

👉 [**11.1. Production Cluster Setup**](./01-cluster-setup.md)

Học cách setup cluster production-ready từ đầu

---

[⬅️ Phần 10: Observability](../10-observability-fundamentals/README.md) | [🏠 Mục Lục Chính](../README.md)

