<!-- Template Version: 2.0 | F369 Wave 9 (composite) | Composes: EKS_COST_OPTIMIZATION + EKS_KARPENTER_AUTOSCALING + EKS_OBSERVABILITY -->

# Template 06 — EKS Cost Optimization (Karpenter consolidation · VPA right-sizing · Spot strategy · Graviton · Kubecost · CUR + Athena · Savings Plans)

## Purpose

Take an existing EKS cluster's monthly bill down 40-70% in 1 weekend, with no application code changes and no impact to workload SLOs. Targets the 5 levers that account for ~80% of EKS cost reduction (request right-sizing, spot, Graviton, Karpenter consolidation, Savings Plans).

Output: per-team cost showback dashboards + automated right-sizing recommendations + spot-mix migration plan + SP/RI commit recommendation.

Generates production-deployable CDK + Athena queries + Kubecost install + CW dashboards.

---

## Role Definition

You are an expert FinOps engineer with deep expertise in:
- EKS cost drivers (compute > storage > LB > data transfer) and their levers
- Karpenter consolidation policies, disruption budgets, multi-NodePool spot strategies
- Vertical Pod Autoscaler (VPA) for request right-sizing in `recommend` mode
- AWS Compute Optimizer for ASG-backed managed node groups
- Kubecost installation + AWS Athena/CUR integration for accurate per-pod cost
- AWS Cost and Usage Report (CUR) with `SPLIT_COST_ALLOCATION_DATA` for pod-share
- AWS Cost Explorer Anomaly Detection
- Compute Savings Plans modelling vs RIs vs no-commitment
- Graviton (ARM64) migration assessment for Java / Python / Node / Go / Rust workloads

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
AWS_ACCOUNT_ID:              [REQUIRED]
ENV:                         [REQUIRED — prod typically]

# --- TARGET CLUSTER ---
CLUSTER_NAME:                [REQUIRED]
CURRENT_MONTHLY_SPEND_USD:   [REQUIRED — last 30d EC2+EBS for cluster nodes]
TARGET_REDUCTION_PCT:        [REQUIRED — 40 typical, 70 aggressive]

# --- WORKLOAD CONSTRAINTS ---
PROD_NAMESPACES:             [comma-separated]
SPOT_ELIGIBLE_NAMESPACES:    [subset of prod that can tolerate interruption]
GRAVITON_ELIGIBLE_APPS:      [comma-separated app names — verified ARM64 compat]
ALWAYS_ON_BASELINE_NODES:    [REQUIRED — count of nodes always running for ≥ 30 days]

# --- TOOLING ---
ENABLE_KUBECOST:             [true default]
KUBECOST_TIER:               [free | enterprise]
ENABLE_VPA:                  [true default — recommend mode only]
ENABLE_CUR:                  [true default — required for accurate per-pod cost]

# --- COMMIT STRATEGY ---
EVALUATE_SAVINGS_PLAN:       [true default]
SP_TERM_YEARS:               [1 default; 3 for established workloads]
SP_PAYMENT_OPTION:           [no-upfront default; partial-upfront for max discount]

# --- ALERTING ---
COST_ANOMALY_THRESHOLD_USD:  [100 default — alert on any anomaly ≥ $X]
FINOPS_EMAIL:                [REQUIRED — recipient for SP recommendations + anomalies]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `EKS_COST_OPTIMIZATION` | Karpenter consolidation knobs, VPA, Compute Optimizer, Kubecost, CUR, SP/RI |
| `EKS_KARPENTER_AUTOSCALING` | Multi-NodePool shape (spot+on-demand+graviton+gpu) |
| `EKS_OBSERVABILITY` | Cost dashboards in CW + AMG; pair with Kubecost |

---

## Architecture (cost-flow view)

```
   ┌────────────────────────────────────────────────────────────────┐
   │  EKS cluster                                                   │
   │                                                                │
   │  Karpenter NodePools (cost-optimised)                         │
   │   ┌──────────────────────────────────────────────────────────┐ │
   │   │ baseline-od  (3× m6i.large, on-demand, RI/SP-covered)    │ │
   │   │ general-spot (5+ instance types, AMD64, spot)             │ │
   │   │ graviton-spot (m7g/c7g, ARM64, spot, -25% vs amd64)      │ │
   │   │ gpu-od (g5/g6, on-demand, only when needed)              │ │
   │   └──────────────────────────────────────────────────────────┘ │
   │                                                                │
   │  Workload pods (with VPA recommendations applied)             │
   │                                                                │
   │  Kubecost (analyzer + CUR integration via Athena)             │
   │   - Real-time waste detection                                  │
   │   - Showback per namespace, deployment, label                  │
   │   - Right-sizing recommendations                               │
   └─────────────────┬──────────────────────────────────────────────┘
                     │
       ┌─────────────┴─────────────┐
       ▼                           ▼
   ┌──────────────┐         ┌─────────────────┐
   │ AWS CUR      │         │ Compute         │
   │ (hourly      │         │ Optimizer       │
   │  Parquet to  │         │ (per-ASG        │
   │  S3 + Athena)│         │  recs)          │
   └──────┬───────┘         └─────────────────┘
          │
          ▼
   ┌──────────────────────────────────────────┐
   │ Athena queries — per-team showback       │
   │   - Cost / namespace / day               │
   │   - Cost / pod / hour                    │
   │   - Spot $ vs on-demand $                │
   │   - Idle capacity $ wasted               │
   └──────┬───────────────────────────────────┘
          │
          ▼
   ┌──────────────────────────────────────────┐
   │ AMG / QuickSight dashboards              │
   │  + Cost Anomaly Detection alerts         │
   │  + Monthly SP commit recommendation      │
   └──────────────────────────────────────────┘
```

---

## Day-by-day execution (1 weekend, 1 dev)

### Saturday morning — Baseline + observability
- Pull current month bill: EC2 + EBS + LB + Data Transfer + EKS control plane
- Identify cluster's nodes (filter `tag:karpenter.sh/cluster` or `tag:eks:cluster-name`)
- Enable CUR with `SPLIT_COST_ALLOCATION_DATA`, Parquet to S3, Athena integration
- Activate cost allocation tags: `eks:cluster-name`, `eks:namespace`, `eks:deployment`, `team`, `cost-center`
- Install Kubecost (Helm) — point at AMP (or bundled Prom) + CUR via Athena
- Enable Compute Optimizer at account level (waits 14d for first ML recs — start clock now)
- Create Cost Explorer Anomaly Monitor + subscription → SNS → email
- **Deliverable:** Kubecost dashboard URL emailed to stakeholders; current waste % visible (typically 30-50% in first audit)

### Saturday afternoon — Right-sizing (immediate wins)
- Install VPA in `recommend` mode (no auto-restart)
- For each Deployment in `PROD_NAMESPACES`:
  - Read current `requests` from manifest
  - Apply `VerticalPodAutoscaler { updateMode: "Off" }`
  - Wait 24h for first recommendations (start now, finalise Sunday)
- For Deployments where Kubecost shows > 30% over-provisioned (CPU OR memory):
  - Reduce requests in Helm values to Kubecost target (NOT VPA target — Kubecost incorporates CUR data)
  - Roll deployment in stage first, verify SLO unchanged, promote to prod
- **Deliverable:** Right-sized requests applied to top-10 cost-driving Deployments. Expected savings: 15-25% of compute bill.

### Saturday evening — Karpenter NodePool migration
- Audit current NodePools (or Cluster Autoscaler ASGs if not yet on Karpenter)
- For Karpenter clusters:
  - Update existing NodePool: `consolidationPolicy: WhenEmptyOrUnderutilized`, `consolidateAfter: 30s`, `expireAfter: 720h`
  - Add `general-spot` NodePool with 5+ instance types, AMD64, spot capacity
  - Add `graviton-spot` NodePool with `m7g/c7g/r7g`, ARM64
  - Add `taints` on baseline-od pool to keep workload off it (force spot pickup)
- For Cluster Autoscaler clusters: install Karpenter alongside; migrate workloads pool-by-pool over coming sprints.
- For `SPOT_ELIGIBLE_NAMESPACES`: add `nodeSelector: karpenter.sh/capacity-type: spot` OR remove all selectors so Karpenter chooses cheapest.
- **Deliverable:** ≥ 50% of cluster nodes are spot by Sunday morning. Expected savings: 30-40% of compute bill.

### Sunday morning — Graviton migration
- For each app in `GRAVITON_ELIGIBLE_APPS`:
  - Build multi-arch image: `docker buildx build --platform linux/amd64,linux/arm64 -t app:1.0.0 --push`
  - Update Deployment `nodeSelector` to allow ARM64: `kubernetes.io/arch: [amd64, arm64]`
  - Apps land on whichever pool is cheaper (Karpenter picks)
- Verify perf: compare p95 latency week-over-week from CW Application Signals
- **Deliverable:** Graviton-eligible apps running on ARM64 spot. Expected additional savings: 10-20% on those workloads.

### Sunday afternoon — Savings Plans + cost dashboards
- Pull Cost Explorer SP recommendations: `aws ce get-savings-plans-purchase-recommendation --savings-plans-type COMPUTE_SP --term-in-years <SP_TERM_YEARS> --payment-option <SP_PAYMENT>`
- Validate against `ALWAYS_ON_BASELINE_NODES` × hourly rate (don't over-commit — only commit ≤ 60% of historical baseline)
- Email FinOps stakeholders the SP recommendation + projected ROI; do NOT auto-purchase
- Build CW dashboards (or import Kubecost dashboards into AMG):
  - Total cluster $/day (rolling 30d, 90d)
  - $/namespace stacked
  - Spot vs on-demand $ split
  - Top 10 most-expensive deployments
  - Idle capacity (requested but unused) $ wasted
- Set Cost Anomaly Detection threshold = `COST_ANOMALY_THRESHOLD_USD`
- **Deliverable:** End-of-weekend report — current monthly run-rate vs new projected run-rate, with breakdown by lever. Typical: 40-60% reduction.

---

## Validation criteria

- [ ] **CUR enabled** with `SPLIT_COST_ALLOCATION_DATA`; Athena table created; sample query returns rows
- [ ] **Cost allocation tags activated** in billing console: `eks:cluster-name`, `eks:namespace`, `team` minimum
- [ ] **Kubecost UI accessible**; "Allocations" report shows non-zero $ per namespace
- [ ] **VPA recommendations available** for top-10 Deployments (`kubectl describe vpa -n <ns>` shows Target values)
- [ ] **Karpenter consolidation policy** = `WhenEmptyOrUnderutilized` on at least one NodePool
- [ ] **Spot percentage ≥ 50%** of cluster nodes (verified via `aws ec2 describe-instances + InstanceLifecycle == spot`)
- [ ] **Graviton workload running** if `GRAVITON_ELIGIBLE_APPS` non-empty (`kubectl get pod -o jsonpath='{.spec.nodeName}'` → node arch == arm64)
- [ ] **Cost Anomaly Detection** monitor + subscription in place; test trigger (force a temporary spike) → email received within 24h
- [ ] **SP recommendation report** generated and emailed; NO auto-purchase
- [ ] **Cost reduction validated**: compare last 7d $/day vs prior 30d $/day; report % reduction (target: ≥ 40%)

---

## Common gotchas (claude must address proactively)

- **Right-sizing requests too aggressively can OOM-kill pods under load.** Use VPA target (P95 over 7+ days), not minimum. Test in stage first.
- **Karpenter `consolidateAfter` < 30s causes thrashing** — pods evicted before they finish. Default to 30-60s.
- **Spot interruption ignores PDB** — only Karpenter-managed graceful drain respects it. Apps must `SIGTERM` correctly.
- **Graviton compatibility hidden gotchas:** Java apps with native libs (e.g., `org.rocksdb.RocksDB`, `JNA`, Snappy native), Python wheels with C extensions (numpy, scipy work; legacy ones may not). TEST.
- **CUR + SPLIT_COST_ALLOCATION_DATA takes 24-48h to start populating.** Set up Day 1, query on Day 2+.
- **Compute Optimizer needs 14 days of data** before it gives recommendations for new ASGs/instances.
- **Kubecost free tier limits:** 15-day retention, single cluster. For multi-cluster, eval Enterprise (~$0.012/CPU-hr).
- **Don't commit to RIs for spot-replaceable workloads** — wastes the discount when spot would have been cheaper.
- **Don't commit to SPs > 60% of historical baseline.** Excess commitment cannot be returned.
- **Karpenter NodePool weight matters** — heavier weight = preferred. Set spot pool weight=50, on-demand pool weight=10 → Karpenter prefers spot.
- **PDB `maxUnavailable: 0`** prevents Karpenter from consolidating any node hosting that pod. Use `maxUnavailable: 1` minimum.

---

## Output artifacts

1. CDK stack: `FinOpsStack` (CUR enable, Cost Anomaly Monitor, Kubecost Helm, VPA Helm, IAM for Athena CUR access)
2. Karpenter NodePool YAMLs (3-4 pools: baseline-od, general-spot, graviton-spot, gpu-od)
3. VPA YAML templates per Deployment (recommend mode)
4. Athena saved queries (per-namespace cost / day, spot vs OD split, idle waste, top deployments)
5. CW / AMG dashboard JSON — cluster cost overview + per-team showback
6. **Multi-arch image build script** (`build-multiarch.sh`) for Graviton migration
7. **Savings Plan analysis script** (`sp-recommend.py`) — pulls CE recommendations + sanity-checks against baseline
8. **Right-sizing CSV report generator** (`right-size-report.py`) — joins Kubecost + VPA recommendations, outputs CSV for engineering teams
9. Pytest suite — validates each criterion
10. **End-of-weekend PDF report** (`generate-savings-report.py`) — current vs projected spend, lever breakdown, action items

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite template. Composes EKS_COST_OPTIMIZATION + Karpenter + Observability. 1-weekend execution. Wave 9. |
