<!-- Template Version: 2.0 | F369 Wave 9 (composite) | Composes: EKS_SECURITY + EKS_POD_IDENTITY + EKS_NETWORKING + EKS_OBSERVABILITY + EKS_CLUSTER_FOUNDATION -->

# Template 08 — EKS Security Hardening (PSS · NetworkPolicies · ECR Inspector · GuardDuty Runtime · Kyverno · image signing)

## Purpose

Bring an existing or new EKS cluster up to **regulated-workload posture** (PCI / HIPAA / SOC2 / FedRAMP-Moderate). Layered defense: control plane hardening → image supply chain → network segmentation → pod-level constraints → runtime threat detection → admission control.

Output: a hardened EKS posture that passes a typical 3rd-party assessment without rework.

Generates production-deployable CDK + admission policies + IR runbook.

---

## Role Definition

You are an expert AWS / Kubernetes security engineer with deep expertise in:
- Pod Security Standards (PSS) profiles + admission webhook tuning
- VPC CNI Network Policy Agent + default-deny + egress allow-list patterns
- ECR + Inspector enhanced scanning + CI/CD blocking gates
- GuardDuty for EKS — Audit Logs + Runtime Monitoring + finding triage
- Kyverno admission control (preferred over OPA Gatekeeper for new clusters)
- Notary v2 / cosign image signing with KMS keys
- IMDSv2 enforcement + node OS hardening + SSM Session Manager (no SSH)
- Compliance frameworks: PCI 4.0, HIPAA Security Rule, SOC2 CC6/CC7, FedRAMP

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
AWS_ACCOUNT_ID:              [REQUIRED]
ENV:                         [REQUIRED — prod | regulated-prod]

# --- TARGET CLUSTER (existing) ---
CLUSTER_NAME:                [REQUIRED]
CLUSTER_OIDC_ISSUER_URL:     [REQUIRED — for IRSA fallback if needed]

# --- COMPLIANCE FRAMEWORK ---
COMPLIANCE_FRAMEWORK:        [REQUIRED — pci | hipaa | soc2 | fedramp-moderate | combined]
DATA_CLASSIFICATION:         [REQUIRED — public | internal | confidential | restricted]

# --- IMAGE SUPPLY CHAIN ---
ECR_REPO_NAMES:              [comma-separated repos to harden]
ENABLE_IMAGE_SIGNING:        [true for prod/regulated]
COSIGN_KMS_KEY_ARN:          [REQUIRED if signing]

# --- NAMESPACES ---
PROD_NAMESPACES:             [comma-separated — apply restricted PSS + default-deny netpol]
SYSTEM_NAMESPACE_OVERRIDES:  [comma-separated — kube-system, karpenter, etc. need 'privileged' PSS]

# --- THREAT DETECTION ---
GUARDDUTY_DETECTOR_ID:       [REQUIRED — existing or to-create]
SECURITY_FINDINGS_BUCKET:    [REQUIRED — S3 for GuardDuty findings export]
INCIDENT_SNS_TOPIC_ARN:      [REQUIRED — wired to PagerDuty/Slack]

# --- ADMISSION CONTROL ---
ENABLE_KYVERNO:              [true default]
KYVERNO_MODE:                [audit (start) | enforce (after validation)]

# --- COMPLIANCE EVIDENCE ---
EVIDENCE_BUCKET:             [REQUIRED — S3 for assessor artifacts]
KMS_KEY_ARN:                 [REQUIRED — log + EBS + ECR encryption]
LOG_RETENTION_DAYS:          [365 default for regulated]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `EKS_SECURITY` | PSS + NetworkPolicies + ECR/Inspector + GuardDuty + Kyverno + IMDSv2 + image signing |
| `EKS_POD_IDENTITY` | Replace IRSA with Pod Identity for tighter session-tag ABAC |
| `EKS_NETWORKING` | VPC CNI Network Policy Agent must be enabled |
| `EKS_OBSERVABILITY` | Audit log shipping for SIEM ingestion |
| `EKS_CLUSTER_FOUNDATION` | KMS + private endpoint + control plane logs (re-verify) |
| `LAYER_SECURITY` | KMS key rotation + IAM least-privilege baseline |

---

## Architecture (defense-in-depth layers)

```
   ┌──────────────────────────────────────────────────────────────┐
   │ Layer 1: Cluster control plane                              │
   │   - Private endpoint only                                    │
   │   - All 5 control plane logs → CW Logs (KMS, 365d retention) │
   │   - Secrets envelope encryption (CMK)                         │
   │   - EKS access entries only (no aws-auth ConfigMap)           │
   └──────────────────┬───────────────────────────────────────────┘
                      │
   ┌──────────────────▼───────────────────────────────────────────┐
   │ Layer 2: Image supply chain                                  │
   │   - ECR: KMS encrypt + IMMUTABLE tags + scan-on-push          │
   │   - Inspector enhanced scan (continuous CVE)                  │
   │   - CI gate: block deploy if HIGH/CRITICAL                    │
   │   - cosign signature with KMS key                             │
   │   - Kyverno verify-image-signature on admission               │
   └──────────────────┬───────────────────────────────────────────┘
                      │
   ┌──────────────────▼───────────────────────────────────────────┐
   │ Layer 3: Node                                                │
   │   - IMDSv2 only (iptables blocks v1)                          │
   │   - SSH disabled (SSM Session Manager only)                   │
   │   - EBS encrypted with CMK                                     │
   │   - Bottlerocket or AL2023 (read-only root + immutable infra) │
   └──────────────────┬───────────────────────────────────────────┘
                      │
   ┌──────────────────▼───────────────────────────────────────────┐
   │ Layer 4: Pod                                                 │
   │   - Pod Security Standards: restricted (enforce)              │
   │   - readOnlyRootFilesystem, drop ALL capabilities             │
   │   - non-root, seccomp RuntimeDefault                          │
   │   - Pod Identity (least-privilege IAM, session tags)          │
   └──────────────────┬───────────────────────────────────────────┘
                      │
   ┌──────────────────▼───────────────────────────────────────────┐
   │ Layer 5: Network                                             │
   │   - Default-deny NetworkPolicy per namespace                  │
   │   - Explicit allow-list (DNS, RDS, S3 endpoints, ALB)         │
   │   - VPC endpoints for all AWS APIs (no public egress)         │
   └──────────────────┬───────────────────────────────────────────┘
                      │
   ┌──────────────────▼───────────────────────────────────────────┐
   │ Layer 6: Admission control                                   │
   │   - Kyverno cluster policies (enforce mode)                   │
   │     - require-image-signature                                 │
   │     - disallow-latest-tag                                     │
   │     - require-resources (CPU/mem requests + limits)           │
   │     - disallow-privilege-escalation                           │
   │     - require-pod-identity-association (no static creds)      │
   └──────────────────┬───────────────────────────────────────────┘
                      │
   ┌──────────────────▼───────────────────────────────────────────┐
   │ Layer 7: Runtime detection + response                        │
   │   - GuardDuty EKS Audit Logs (control plane anomalies)        │
   │   - GuardDuty Runtime Monitoring (DaemonSet, syscall watch)   │
   │   - Findings → S3 + SNS → PagerDuty + SIEM                    │
   │   - IR runbook: container compromise → cordon node, isolate    │
   │     pod via NetworkPolicy quarantine, snapshot for forensics   │
   └──────────────────────────────────────────────────────────────┘
```

---

## Day-by-day execution (3-day hardening sprint)

### Day 1 — Inventory + image supply chain
- Inventory existing pods: Helm charts in use, base images, exposed ports, current SecurityContext
- For each `ECR_REPO_NAMES`: enable KMS + IMMUTABLE + scan-on-push (`EKS_SECURITY` §5)
- Enable Inspector enhanced scanning at account level
- CI/CD CodeBuild buildspec gate: query Inspector findings post-push, block on CRITICAL/HIGH
- (If `ENABLE_IMAGE_SIGNING`) Configure cosign + KMS; sign existing prod images; document key rotation
- **Deliverable:** ECR scan dashboard shows zero CRITICAL across `ECR_REPO_NAMES`; CI gate tested by force-pushing a vulnerable image (build fails).

### Day 2 — Pod + Network hardening + admission control
- For each `PROD_NAMESPACE`:
  - Apply PSS labels (`enforce: restricted` for prod; `audit: restricted` first day for warning-only)
  - Apply default-deny NetworkPolicy
  - Apply per-app explicit-allow NetworkPolicy (ingress from LBC SG, egress to RDS + DNS + AWS APIs)
- For each `SYSTEM_NAMESPACE_OVERRIDES`: PSS = `privileged` (Karpenter/ADOT/Fluent Bit need hostNetwork)
- Migrate IRSA → Pod Identity for in-account workloads (`EKS_POD_IDENTITY` §5)
- Install Kyverno (`EKS_SECURITY` §7) in `audit` mode
- Apply 5 Kyverno cluster policies; review findings for 24h
- Promote Kyverno to `enforce` mode (per-namespace rollout, watch for breakage)
- **Deliverable:** Day 2 evening: PSS restricted enforced; Kyverno enforce; all NetworkPolicies in place; sample compromised-pod test fails admission.

### Day 3 — Runtime detection + IR runbook + evidence pack
- Enable GuardDuty EKS Audit Logs + Runtime Monitoring (auto-installs agent DaemonSet)
- Configure GuardDuty findings → S3 (KMS encrypted) + SNS → PagerDuty
- Tune `EKS_SUSPICIOUS_*` finding thresholds; create suppress filters for known-benign patterns (e.g., security scanner traffic)
- Author **Incident Response runbook**:
  - Detection: GuardDuty finding triage matrix (severity → action)
  - Containment: `kubectl cordon`, `kubectl drain`, NetworkPolicy quarantine pattern
  - Forensics: `nsenter` snapshot, `kubectl logs --previous`, EBS volume snapshot for offline analysis
  - Recovery: pod replacement, node replacement, image rebuild, post-incident review template
- Generate **compliance evidence pack** in `EVIDENCE_BUCKET`:
  - Cluster config snapshot (`aws eks describe-cluster`)
  - All Kyverno policy files
  - PSS namespace labels report
  - NetworkPolicy inventory
  - ECR repo scan results (last 30 days)
  - GuardDuty findings summary
  - CloudTrail for cluster API actions (last 90 days)
- **Deliverable:** GuardDuty active + IR runbook in repo + evidence pack zip in S3 ready for assessor.

---

## Validation criteria

- [ ] **Cluster:** private endpoint only; `aws eks describe-cluster --name $C --query 'cluster.resourcesVpcConfig.endpointPublicAccess'` = `False`
- [ ] **Logs:** all 5 control plane logs enabled, retention ≥ 365d, KMS encrypted
- [ ] **Secrets:** envelope encryption with CMK (`aws eks describe-cluster --query cluster.encryptionConfig`)
- [ ] **PSS:** every prod namespace has `pod-security.kubernetes.io/enforce=restricted`
- [ ] **NetworkPolicy:** every prod namespace has `default-deny-all` netpol; `kubectl get netpol -A | grep default-deny | wc -l` ≥ count of prod namespaces
- [ ] **ECR:** all repos `IMMUTABLE`, KMS-encrypted, scan-on-push
- [ ] **Inspector:** zero CRITICAL findings across `ECR_REPO_NAMES`; HIGH justified or remediated
- [ ] **Image signing:** ≥ 1 Kyverno policy `verify-image-signature` enforce mode; pod admission rejects unsigned image (test)
- [ ] **GuardDuty:** detector active; `EKS_AUDIT_LOGS` + `EKS_RUNTIME_MONITORING` enabled; findings publishing to S3
- [ ] **Pod Identity:** Karpenter, LBC, ADOT, ESO using Pod Identity (not IRSA); `aws eks list-pod-identity-associations` shows expected entries
- [ ] **IMDSv2:** every node `httpTokens=required`; spot-test by SSH/SSM into a node and `curl -s 169.254.169.254/latest/meta-data/` returns 401
- [ ] **Admission:** Kyverno enforce mode; `kubectl describe pol` shows zero `Failed` policies in last 24h

---

## Common gotchas (claude must address proactively)

- **PSS `restricted` enforce breaks system DaemonSets** that need `hostNetwork` (CNI, Karpenter agent if any). Use `privileged` profile per `SYSTEM_NAMESPACE_OVERRIDES`.
- **VPC CNI Network Policy Agent must be installed before NetworkPolicies are applied** — without it, netpols are ignored silently.
- **Default-deny NetworkPolicy without DNS allow-rule kills the cluster** — every netpol set must include `egress to kube-dns:53/UDP` first.
- **Kyverno `verify-image-signature` requires the registry to be cosign-aware.** ECR fully supports this (2024+); custom registries may not.
- **GuardDuty Runtime Monitoring agent needs ~150MB image + 50MB RAM/node.** Plan node sizing.
- **`kubectl drain` on a node hosting StatefulSet pod with PVC requires `--ignore-daemonsets --delete-emptydir-data`** — and the PVC may not re-attach in another AZ. Test the IR runbook before relying on it.
- **CloudTrail for EKS API is `eks.amazonaws.com` events** — but `kubectl` actions go through the EKS API endpoint and require **K8s API audit logs** (different from CloudTrail). Both are needed.

---

## Output artifacts

1. CDK stack: `EksHardeningStack` (Inspector enable, GuardDuty features, Kyverno install, ECR repo updates)
2. Kyverno policies (5 cluster policies — verify-signature, disallow-latest, require-resources, disallow-privesc, require-pod-identity)
3. NetworkPolicy templates per namespace (default-deny + DNS + RDS + AWS-API allow)
4. PSS namespace labelling script (`apply-pss.sh`)
5. CI/CD buildspec snippet — Inspector blocking gate
6. Cosign signing CI step + key rotation runbook
7. **Incident Response runbook** (Markdown) — detection → containment → forensics → recovery
8. **Compliance evidence pack generator** (`generate-evidence.sh`) — pulls cluster config, Kyverno policies, PSS labels, ECR scan results, GuardDuty findings, CloudTrail summary into a zip
9. Pytest validation suite (one test per validation criterion)
10. CW dashboard JSON — security posture overview (Inspector severity counts, GuardDuty finding count, Kyverno policy violations, failed pod admissions)

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite template. Composes EKS_SECURITY + 4 partials. 7-layer defense-in-depth. Wave 9. |
