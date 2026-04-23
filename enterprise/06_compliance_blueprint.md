<!-- Template Version: 1.0 | boto3: 1.35+ | aws-cdk-lib: 2.160+ | Model IDs: 2026-04-22 -->

# Template 06 — Compliance Blueprint (HIPAA / PCI-DSS / SOC2)

## Purpose
Generate a deployable compliance foundation stack: WORM audit bucket with S3 Object Lock, CloudTrail multi-region trail with file-integrity validation, AWS Config with ~15 managed rules, AWS Backup Vault Lock, Inspector v2 account enable, and a weekly evidence-collector Lambda. Per-standard retention defaults applied from `COMPLIANCE_STANDARD`. This is a thin router — the WORM/lock semantics, Config rule set, Backup Vault Lock math (grace/min/max), and Inspector enable choreography are codified in the `COMPLIANCE_HIPAA_PCIDSS` partial.

---

## Role Definition

You are an expert AWS compliance/governance engineer with deep expertise in:
- HIPAA, PCI-DSS v4.0, SOC2 Type II control mappings to AWS primitives
- S3 Object Lock (GOVERNANCE vs COMPLIANCE mode — irreversible) and legal holds
- CloudTrail organization + multi-region trails, log file integrity validation, CloudTrail Lake
- AWS Config conformance packs + managed rules (encryption, MFA, public access, CloudTrail-enabled, VPC flow logs)
- AWS Backup Vault Lock (min/max/grace retention, compliance-mode irreversibility)
- Amazon Inspector v2 account enablement for EC2 / ECR / Lambda
- Architectural patterns from the `COMPLIANCE_HIPAA_PCIDSS` partial in F369_CICD_Template

Generate complete, production-deployable code. No placeholders.

---

## Context & Inputs

```
PROJECT_NAME:             [REQUIRED]
AWS_REGION:               [REQUIRED]
AWS_ACCOUNT_ID:           [REQUIRED]
ENV:                      [REQUIRED - dev | stage | prod]
TARGET_LANGUAGE:          [REQUIRED - python | typescript]

COMPLIANCE_STANDARD:      [REQUIRED - HIPAA | PCI_DSS | SOC2 | ALL]
EVIDENCE_RETENTION_DAYS:  [OPTIONAL - default matches standard: HIPAA 2190 (6y), PCI 365 (1y), SOC2 1095 (3y), ALL 2190]
AUDIT_BUCKET_NAME:        [OPTIONAL - default {PROJECT_NAME}-compliance-audit-{ENV}]
OBJECT_LOCK_MODE:         [OPTIONAL - governance (default) | compliance (irreversible — use for prod)]
BACKUP_VAULT_LOCK:        [OPTIONAL - default enabled; grace 3d, min 7d, max 7y]
ALERT_SNS_TOPIC_SSM:      [OPTIONAL - SSM param name holding SNS ARN for compliance alerts]
CEDAR_POLICY_S3_PATH:     [OPTIONAL - s3://.../cedar.policies for agent-control tie-in]
ENABLE_INSPECTOR:         [OPTIONAL - EC2 | ECR | LAMBDA | ALL — default ALL]
```

---

## Task

Generate all code for this compliance foundation. MUST conform to the architectural patterns codified in the partial below — treat the partial as non-negotiable:

  **Load this partial as context before generating code:**
  https://github.com/surendersara1/F369_CICD_Template/blob/main/prompt_templates/partials/COMPLIANCE_HIPAA_PCIDSS.md

Steps:
1. Create a local CMK for compliance encryption — owned by this stack, not imported (non-negotiable #5). Enable key rotation. Alias `alias/{project}-compliance-{env}`.
2. Create the WORM audit bucket with Object Lock enabled at creation time (Object Lock CANNOT be enabled after creation), default retention per `OBJECT_LOCK_MODE` + standard-derived years, versioning on, block-public-access on, KMS via local CMK.
3. Create a CloudTrail multi-region trail with `includeGlobalServiceEvents=true`, `isMultiRegionTrail=true`, `enableLogFileValidation=true`, shipping to both the audit bucket and a CloudWatch log group (365-day retention).
4. Create ~15 AWS Config managed rules covering: `ENCRYPTED_VOLUMES`, `S3_BUCKET_PUBLIC_READ_PROHIBITED`, `S3_BUCKET_PUBLIC_WRITE_PROHIBITED`, `S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED`, `ROOT_ACCOUNT_MFA_ENABLED`, `IAM_USER_MFA_ENABLED`, `CLOUD_TRAIL_ENABLED`, `CLOUD_TRAIL_LOG_FILE_VALIDATION_ENABLED`, `CLOUD_TRAIL_ENCRYPTION_ENABLED`, `VPC_FLOW_LOGS_ENABLED`, `RDS_STORAGE_ENCRYPTED`, `EBS_SNAPSHOT_PUBLIC_RESTORABLE_CHECK`, `INCOMING_SSH_DISABLED`, `RESTRICTED_INCOMING_TRAFFIC`, `SECRETSMANAGER_ROTATION_ENABLED_CHECK`.
5. Create an AWS Backup Vault with Vault Lock: `minRetentionDays=7`, `maxRetentionDays=2555` (7y), `changeableForDays=3` (grace). Add a daily backup plan covering tagged resources.
6. Enable Inspector v2 for the account via `AwsCustomResource` invoking `inspector2:Enable` with the scope(s) from `ENABLE_INSPECTOR`.
7. Create a weekly evidence-collector Lambda (EventBridge `rate(7 days)`) that snapshots Config evaluation results + CloudTrail digest + Inspector findings → audit bucket under `evidence/{YYYY}/{MM}/{DD}/`.
8. Publish audit bucket name, backup vault ARN, and compliance CMK ARN via SSM for consumer stacks (string params only — non-negotiable #5).

### 5 non-negotiables (from `LAYER_BACKEND_LAMBDA §4.1` — all apply here)

1. `Path(__file__).resolve().parents[N]` (Python) or `path.join(__dirname, ...)` (TS) asset paths. Never CWD-relative.
2. Cross-stack grants are **identity-side only** — never `bucket.grant_read_write(cross_stack_role)`.
3. Cross-stack EventBridge → Lambda uses `events.CfnRule` with static-ARN target.
4. Bucket + CloudFront OAC live in the **same stack**.
5. Never `encryption_key=ext_key` — pass KMS ARN as a string via SSM.

---

## Output Format

1. `cdk/stacks/compliance_foundation_stack.py` (or `.ts`) — CMK, audit bucket, CloudTrail, Config recorder + rules, Backup Vault Lock, Inspector enable
2. `cdk/stacks/compliance_evidence_stack.py` — evidence-collector Lambda + schedule (split so the foundation stack stays lock-stable)
3. `lambda/evidence_collector/index.py` — collector handler (Config + CloudTrail digest + Inspector findings)
4. `tests/sop/test_compliance_foundation.py` — pytest offline CDK synth harness asserting Object Lock enabled, log file validation on, Vault Lock parameters correct, correct rule count per standard
5. `README.md` — deploy order, how to verify Object Lock retention, `OBJECT_LOCK_MODE=compliance` warning

---

## Requirements

- If `OBJECT_LOCK_MODE=compliance`, emit a loud README warning: COMPLIANCE mode cannot be shortened or removed even by root — use only in prod after legal sign-off.
- If `COMPLIANCE_STANDARD=HIPAA`, default retention 6 years (HIPAA §164.316(b)(2)).
- If `COMPLIANCE_STANDARD=PCI_DSS`, default retention 1 year (PCI-DSS 10.5.1), but audit trail retained 1 year with 3 months immediately available.
- If `COMPLIANCE_STANDARD=SOC2`, default retention 3 years.
- CloudTrail S3 bucket policy must include the CloudTrail service-principal ACL grant (`ACL: bucket-owner-full-control`, `Condition: s3:x-amz-acl`). The partial shows the canonical shape.
- Publish `{PROJECT}/{ENV}/compliance/audit_bucket_name`, `.../backup_vault_arn`, `.../kms_key_arn` via SSM.

---

## Integration Points

- Inputs from: none (this is a foundation stack — deploy first)
- Outputs to: every downstream stack handling regulated data (via SSM-shared audit bucket + CMK ARN)
- Related kits: `kits/hr-interview-analyzer.md` (PII/PHI in candidate recordings), `kits/acoustic-fault-diagnostic-agent.md` (industrial audit trail), `kits/rag-chatbot-per-client.md` (when deployed for healthcare/finance clients — must attach this blueprint first)
