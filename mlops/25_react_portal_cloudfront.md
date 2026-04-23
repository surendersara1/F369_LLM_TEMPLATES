<!-- Template Version: 1.0 | boto3: 1.35+ | aws-cdk-lib: 2.160+ | Model IDs: 2026-04-22 -->

# Template 25 — React Portal (S3 + CloudFront + OAC)

## Purpose
Generate a production-deployable React SPA fronted by CloudFront with Origin Access Control (OAC) over a private, KMS-encrypted S3 bucket. Optional Cognito auth, optional ACM/Route 53 custom domain, optional WAF, optional API Gateway proxy-behaviour. This is a thin router — all architectural decisions (bucket/distribution co-location, OAC wiring, us-east-1 cert handling, deployment role scoping) are codified in the `LAYER_FRONTEND` partial.

---

## Role Definition

You are an expert AWS frontend-delivery engineer with deep expertise in:
- CloudFront distributions, Origin Access Control (OAC, not legacy OAI), cache policies, response headers policies
- S3 static hosting with bucket-policy-driven OAC access (never public, never website-endpoint)
- ACM certificate provisioning in `us-east-1` via `DnsValidatedCertificate` / cross-region references
- Route 53 alias records, WAFv2 WebACLs scoped to `CLOUDFRONT`
- Cognito User Pool auth flows and Lambda@Edge authorizers
- `aws_s3_deployment.BucketDeployment` with build-step bundling
- Architectural patterns from the `LAYER_FRONTEND` partial in F369_CICD_Template

Generate complete, production-deployable code. No placeholders.

---

## Context & Inputs

```
PROJECT_NAME:             [REQUIRED]
AWS_REGION:               [REQUIRED]
AWS_ACCOUNT_ID:           [REQUIRED]
ENV:                      [REQUIRED - dev | stage | prod]
TARGET_LANGUAGE:          [REQUIRED - python | typescript]

PORTAL_NAME:              [REQUIRED - e.g. diagnostic-dashboard]
BUILD_COMMAND:            [OPTIONAL - default "npm run build"]
BUILD_OUTPUT_DIR:         [OPTIONAL - default "dist"]
CUSTOM_DOMAIN:            [OPTIONAL - e.g. portal.client.com — triggers ACM + Route 53]
CUSTOM_DOMAIN_CERT_ARN:   [OPTIONAL - if cert already exists in us-east-1]
AUTH_MODE:                [OPTIONAL - none | cognito | lambda_authorizer — default none]
COGNITO_USER_POOL_ID_SSM: [OPTIONAL - SSM param name holding pool ID if AUTH_MODE=cognito]
ENABLE_WAF:               [OPTIONAL - default true in prod, false elsewhere]
CACHE_POLICY:             [OPTIONAL - static_assets | api_passthrough | mixed — default mixed]
API_ENDPOINT_SSM:         [OPTIONAL - SSM param holding API GW invoke URL for /api/* behaviour]
```

---

## Task

Generate all code for this portal. MUST conform to the architectural patterns codified in the partial below — treat the partial as non-negotiable:

  **Load this partial as context before generating code:**
  https://github.com/surendersara1/F369_CICD_Template/blob/main/prompt_templates/partials/LAYER_FRONTEND.md

Steps:
1. Build the React bundle via `BUILD_COMMAND` into `BUILD_OUTPUT_DIR` (use CDK `Source.asset(..., bundling={...})` so the build runs inside the synth).
2. Create the S3 origin bucket — private, KMS-encrypted (local CMK owned by this stack), block-public-access on, versioned, access-logs on. It MUST live in the **same stack** as the CloudFront distribution (non-negotiable #4).
3. Create the CloudFront distribution with an OAC attached to the origin bucket. Set default root object to `index.html`, 403/404 → `/index.html` for SPA routing.
4. If `CUSTOM_DOMAIN` is set and `CUSTOM_DOMAIN_CERT_ARN` is not, provision an ACM cert in `us-east-1` via `DnsValidatedCertificate` + add a Route 53 A-alias.
5. If `ENABLE_WAF=true`, create a WAFv2 WebACL (scope `CLOUDFRONT`) with AWS-managed CommonRuleSet + KnownBadInputs + RateLimit, attach via `webAclId`.
6. If `AUTH_MODE=cognito`, resolve `COGNITO_USER_POOL_ID_SSM` and wire a Lambda@Edge viewer-request authorizer (deployed from `us-east-1`).
7. If `API_ENDPOINT_SSM` is set, add a second origin (HTTP origin, custom) with path pattern `/api/*` and the `api_passthrough` cache policy (forward Authorization, disable caching).
8. Deploy the built assets via `aws_s3_deployment.BucketDeployment` with `distribution` + `distributionPaths=["/*"]` for invalidation.

### 5 non-negotiables (from `LAYER_BACKEND_LAMBDA §4.1` — all apply here)

1. `Path(__file__).resolve().parents[N]` (Python) or `path.join(__dirname, ...)` (TS) asset paths. Never CWD-relative.
2. Cross-stack grants are **identity-side only** — never `bucket.grant_read_write(cross_stack_role)`.
3. Cross-stack EventBridge → Lambda uses `events.CfnRule` with static-ARN target.
4. Bucket + CloudFront OAC live in the **same stack**.
5. Never `encryption_key=ext_key` — pass KMS ARN as a string via SSM.

---

## Output Format

1. `cdk/stacks/{portal_name}_portal_stack.py` (or `.ts`) — full stack with bucket, OAC, distribution, optional WAF/Cognito/API origin
2. `cdk/stacks/{portal_name}_cert_stack.py` — us-east-1 cert stack (only if `CUSTOM_DOMAIN` + no existing cert)
3. `lambda_edge/auth/index.js` — viewer-request authorizer (only if `AUTH_MODE=cognito`)
4. `frontend/` — minimal React scaffold referencing `API_ENDPOINT_SSM` value at build time via `VITE_API_ENDPOINT` env
5. `tests/sop/test_{portal_name}_portal.py` — pytest offline CDK synth harness asserting OAC present, bucket private, WAF attached where expected
6. `README.md` — deploy order (cert stack first in us-east-1), smoke test via curl

---

## Requirements

- OAC (NOT legacy OAI). Bucket policy grants `s3:GetObject` only to the distribution's service principal with `aws:SourceArn` condition equal to the distribution ARN.
- `viewerProtocolPolicy=REDIRECT_TO_HTTPS` on every behaviour; minimum TLS 1.2_2021.
- Response headers policy with HSTS (31536000 preload), `X-Content-Type-Options: nosniff`, CSP appropriate for React.
- All SSM lookups use `StringParameter.value_from_lookup` at synth time — never `value_for_string_parameter` for cross-region cert ARNs.
- Publish distribution domain + distribution ID via SSM for consumer stacks.

---

## Integration Points

- Inputs from: `enterprise/06` (compliance KMS + audit bucket if portal handles regulated data), `devops/17` (API orchestration endpoint via SSM)
- Outputs to: consumer kits via SSM params (distribution domain, distribution ID)
- Related kits: `kits/hr-interview-analyzer.md`, `kits/rag-chatbot-per-client.md`, `kits/deep-research-agent.md`, `kits/acoustic-fault-diagnostic-agent.md` — all four consume this template for their end-user portal
