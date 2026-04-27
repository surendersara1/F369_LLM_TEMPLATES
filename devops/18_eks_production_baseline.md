<!-- Template Version: 2.0 | F369 Wave 9 (composite) | Composes: EKS_CLUSTER_FOUNDATION + EKS_KARPENTER_AUTOSCALING + EKS_POD_IDENTITY + EKS_NETWORKING + EKS_OBSERVABILITY + EKS_STORAGE + EKS_GITOPS -->

# Template 18 — EKS Production Baseline (cluster · Karpenter · Pod Identity · LBC · Container Insights · ArgoCD)

## Purpose

Stand up a production-grade EKS cluster from zero in 2-3 days. Output is a multi-tenant cluster ready to host a portfolio of microservices, ML inference, batch jobs, or any combination — without re-deriving cluster architecture for every engagement.

This is the **canonical EKS engagement** — the foundation every other EKS workload (security hardening, cost opt, ML on EKS) builds on. Single-cluster mono-stack OR multi-cluster GitOps; both shapes covered.

Generates production-deployable CDK + manifests + ArgoCD bootstrap.

---

## Role Definition

You are an expert AWS / Kubernetes platform engineer with deep expertise in:
- Amazon EKS 1.32+ (control plane, managed node groups, Fargate, access entries)
- Karpenter v1.0+ (NodePools, EC2NodeClass, NodeClaims, consolidation, spot)
- AWS networking on EKS (VPC CNI prefix delegation, AWS Load Balancer Controller, ALB IP-mode, ExternalDNS, Gateway API)
- EKS Pod Identity Associations + IRSA fallback
- CloudWatch Container Insights with enhanced observability + ADOT + AMP/AMG
- EBS / EFS / FSx CSI drivers + StorageClasses + VolumeSnapshots
- ArgoCD HA + App-of-Apps + ApplicationSet + External Secrets Operator
- AWS Identity Center OIDC for cluster + ArgoCD SSO

Generate complete, production-deployable code. No TODOs, no placeholder comments.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
AWS_ACCOUNT_ID:              [REQUIRED]
ENV:                         [REQUIRED — dev | stage | prod]

# --- CLUSTER ---
CLUSTER_NAME:                [e.g. f369-prod-cluster]
KUBERNETES_VERSION:          [1.32 default]
VPC_ID:                      [REQUIRED — existing or to-create]
PRIVATE_SUBNET_IDS:          [REQUIRED — comma-separated, ≥ 3 AZs]
PUBLIC_SUBNET_IDS:           [REQUIRED if internet-facing ALB]
SECONDARY_POD_CIDR:          [OPTIONAL — for IP exhaustion mitigation; e.g. 100.64.0.0/16]

# --- ACCESS ---
CLUSTER_ADMIN_PRINCIPAL_ARN: [IAM role ARN of cluster admin (Identity Center role)]
DEVELOPER_GROUP_ID:          [Identity Center group for developers]

# --- NODE STRATEGY ---
BASELINE_NG_INSTANCE_TYPE:   [m6i.large default]
BASELINE_NG_DESIRED:         [3 default — Karpenter controller, ArgoCD, system pods]
KARPENTER_VERSION:           [1.0.0 default]
ENABLE_GRAVITON_POOL:        [true/false]
ENABLE_GPU_POOL:             [true/false]

# --- WORKLOAD SHAPE ---
APP_NAMESPACES:              [comma-separated list, e.g. checkout,catalog,users]
ENABLE_GITOPS:               [true default — installs ArgoCD]
GITOPS_REPO_URL:             [REQUIRED if ENABLE_GITOPS]
GITOPS_REPO_PATH:            [clusters/{ENV}]
ENABLE_ESO:                  [true default — External Secrets]

# --- INGRESS + DNS ---
HOSTED_ZONE_NAME:            [REQUIRED — e.g. example.com]
HOSTED_ZONE_ID:              [REQUIRED]
ACM_CERT_ARN_REGIONAL:       [REQUIRED — for ALB; us-east-1 if CloudFront]
WAF_ACL_ARN:                 [OPTIONAL]

# --- OBSERVABILITY ---
ENABLE_AMP_AMG:              [false default — true for Prometheus-native teams]
SNS_ALARM_TOPIC_ARN:         [REQUIRED]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED — secrets envelope encryption + EBS + log groups]
LOG_RETENTION_DAYS:          [30 default; 365 for regulated]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `EKS_CLUSTER_FOUNDATION` | Control plane, managed node groups, OIDC, EKS access entries, KMS, all 5 logs |
| `EKS_KARPENTER_AUTOSCALING` | Karpenter v1.x, NodePools, spot+on-demand mix, consolidation, disruption budgets |
| `EKS_POD_IDENTITY` | Pod Identity Associations (preferred); IRSA fallback for cross-account |
| `EKS_NETWORKING` | VPC CNI prefix delegation, AWS LBC, ALB IP-mode + group.name, ExternalDNS |
| `EKS_OBSERVABILITY` | Container Insights enhanced + ADOT + Application Signals + baseline alarms |
| `EKS_STORAGE` | EBS gp3-encrypted default StorageClass + EFS access points + snapshots |
| `EKS_GITOPS` | ArgoCD HA + App-of-Apps + External Secrets Operator |
| `LAYER_NETWORKING` | VPC + private/public subnet shape + endpoints |
| `LAYER_SECURITY` | KMS key + Secrets Manager rotation foundations |

---

## Architecture

```
                            ┌─────────────────────────────┐
                            │  Route 53: *.example.com    │
                            └────────────┬────────────────┘
                                         │ ExternalDNS
                            ┌────────────▼────────────────┐
                            │  ALB (LBC, IP-mode, shared) │
                            │  TLS 1.3, WAF v2 attached   │
                            └────────────┬────────────────┘
                                         │ Pod IPs
                  ┌──────────────────────▼────────────────────────────────┐
                  │  EKS 1.32 cluster (private endpoint)                  │
                  │                                                       │
                  │  Baseline MNG (3× m6i.large)                          │
                  │    ├── Karpenter controller                            │
                  │    ├── ArgoCD (HA: 2 server, 2 controller)            │
                  │    ├── External Secrets Operator                       │
                  │    ├── AWS Load Balancer Controller (HA: 2)           │
                  │    ├── ADOT collector (DaemonSet)                      │
                  │    ├── CloudWatch Agent (DaemonSet)                    │
                  │    └── ExternalDNS                                     │
                  │                                                       │
                  │  Karpenter NodePools:                                 │
                  │    - general-spot (m/c/r 6+, AMD64, spot)             │
                  │    - general-od (m6i.large+, on-demand fallback)      │
                  │    - graviton-spot (m/c 7g, ARM64, spot) [opt]        │
                  │    - gpu-od (g5/g6, on-demand) [opt]                  │
                  │                                                       │
                  │  Workload namespaces (via App-of-Apps):               │
                  │    prod-checkout, prod-catalog, prod-users, ...       │
                  │                                                       │
                  │  Storage:                                             │
                  │    - SC: gp3-encrypted (default)                      │
                  │    - SC: efs-shared (RWX)                             │
                  └───────────┬───────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┬─────────────────┐
              ▼               ▼               ▼                 ▼
        ┌──────────┐    ┌──────────┐    ┌─────────┐    ┌──────────────┐
        │ CW Logs  │    │   AMP    │    │  X-Ray  │    │ Secrets Mgr  │
        │ + Insights│    │ + AMG    │    │ traces  │    │ + ESO sync   │
        └──────────┘    └──────────┘    └─────────┘    └──────────────┘
```

---

## Day-by-day execution (3-day POC, 1-2 devs)

### Day 1 — Foundation
- VPC review (3 AZs, private + public subnets correctly tagged for LBC discovery)
- EKS control plane (1.32, private endpoint, KMS envelope, 5 control plane logs)
- Baseline MNG (3× m6i.large, IMDSv2 mandatory, encrypted EBS, no SSH)
- EKS access entries: cluster admin role + developer group with view permissions
- 4 EKS add-ons: vpc-cni (with prefix delegation), coredns, kube-proxy, eks-pod-identity-agent
- OIDC provider for IRSA fallback
- VPC endpoints (S3, ECR, KMS, Secrets Manager, STS) → reduce NAT $ + improve security
- **Deliverable:** `kubectl get nodes` works from cluster-admin role; control plane logs flowing to CW.

### Day 2 — Platform layer (Karpenter + LBC + ESO + ArgoCD + Observability)
- **Karpenter** install (Helm v1.0.0) with Pod Identity, SQS interruption queue, 4 EventBridge rules
- 2-3 NodePools (`general-spot`, `general-od`, optional `graviton-spot`)
- **AWS Load Balancer Controller** install with Pod Identity; subnet tags verified
- **ExternalDNS** install; Route 53 hosted zone permissions
- **External Secrets Operator** install + ClusterSecretStore (aws-sm)
- **ArgoCD** HA install with ALB ingress, IAM Identity Center OIDC SSO
- Bootstrap **App-of-Apps root Application** pointing at `clusters/{ENV}/` in GitOps repo
- **Container Insights** add-on + Application Signals + 3 baseline alarms (CPU, failed pods, restart loop)
- **Deliverable:** ArgoCD UI accessible at `https://argocd.{HOSTED_ZONE}`; root app `Synced + Healthy`; first test deployment via Git commit visible in cluster.

### Day 3 — Workload onboarding + validation
- Per `APP_NAMESPACES`: namespace with PSS labels + default-deny NetworkPolicy template
- Sample workload Helm chart in GitOps repo (Deployment + Service + Ingress + PDB + HPA + ExternalSecret)
- StorageClasses applied (`gp3-encrypted` default, `efs-shared`)
- VolumeSnapshotClass for backup
- End-to-end test:
  - Push commit → ArgoCD syncs → ALB target group registers pods → DNS resolves → HTTP 200 on `/healthz`
  - Trigger pod scale → Karpenter provisions new node within 60s
  - Verify CloudWatch logs show pod stdout in `/aws/containerinsights/{cluster}/application`
  - Verify Application Signals shows the new service in CloudWatch console
- **Deliverable:** running app + observable metrics + GitOps reconciliation working end-to-end.

---

## Validation criteria

- [ ] `kubectl get nodes -o wide` shows ≥ 3 baseline nodes across ≥ 3 AZs
- [ ] `kubectl get pods -A` shows all platform pods Running + Ready (Karpenter, LBC, ArgoCD, ADOT, ExternalDNS, ESO)
- [ ] Karpenter provisions new node in < 60s when 1000 unschedulable pods created
- [ ] ALB Ingress reachable; HTTPS only (HTTP redirects); TLS 1.3 SSL policy
- [ ] ExternalDNS creates Route 53 record within 1 min of Ingress create
- [ ] ExternalSecret syncs from AWS Secrets Manager → K8s Secret within 1 min
- [ ] ArgoCD root Application Synced + Healthy, all child apps Synced + Healthy
- [ ] CW Container Insights shows `node_cpu_utilization` metric
- [ ] CW alarms ARM_OK = `OK` for CPU / failed pods / restart loop baselines
- [ ] EBS volumes encrypted with CMK (verified via `aws ec2 describe-volumes`)
- [ ] Pod Identity verified: `kubectl exec` into pod; `aws sts get-caller-identity` shows assumed role with session tags
- [ ] Cluster API endpoint is private (cannot reach from public internet without VPC bastion / SSM)

---

## Common gotchas (claude must address proactively)

- **Subnet tagging is invisible-failure mode** for LBC. Without `kubernetes.io/role/elb=1` (public) or `internal-elb=1` (private) + `kubernetes.io/cluster/{name}=shared`, ALB provisioning hangs with no clear error. Verify with `aws ec2 describe-subnets`.
- **ArgoCD sync wave + finalizer dance** — root app must include itself with `resources-finalizer.argocd.argoproj.io` and `selfHeal: true`; otherwise first prune deletes ArgoCD itself.
- **VPC CNI prefix delegation can't be unset.** Once enabled on a node, replacing the node is the only way to revert. Decide upfront.
- **Karpenter SQS interruption queue must be in same region as cluster.** Cross-region = silent spot interruption misses.
- **ExternalSecret refresh polls AWS** — every 1h default. For tighter rotation, set `refreshInterval: 5m` BUT app must reconnect on every secret change.
- **PSS `restricted` rejects most off-the-shelf Helm charts** — start with `audit` profile in dev, validate, then `enforce` in prod (per `EKS_SECURITY` partial).

---

## Output artifacts

1. CDK Python app — stacks: `NetworkStack` → `EksClusterStack` → `KarpenterStack` → `LbcStack` → `ExternalDnsStack` → `EsoStack` → `ArgoCdStack` → `ObservabilityStack`
2. Karpenter NodePool + EC2NodeClass YAML (3 pools)
3. ArgoCD root Application YAML + sample child app
4. Sample workload Helm chart with Deployment + Service + Ingress + PDB + HPA + ExternalSecret + VPA(recommend)
5. ClusterSecretStore + 1 ExternalSecret per app
6. StorageClass YAML (`gp3-encrypted`, `efs-shared`) + VolumeSnapshotClass
7. CloudWatch dashboard JSON — cluster overview (CPU, memory, pod count, ALB latency, restart rate)
8. Pytest validation suite (10 tests covering each validation criterion)
9. Runbook: cluster upgrade procedure, node group rotation, ArgoCD recovery from corrupt state

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite template. Composes 7 EKS partials. Wave 9. |
