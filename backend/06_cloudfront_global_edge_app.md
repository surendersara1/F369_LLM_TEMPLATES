<!-- Template Version: 2.0 | F369 Wave 17 (composite) | Composes: CDN_CLOUDFRONT_FOUNDATION + CDN_EDGE_COMPUTE + CDN_MULTI_ORIGIN_FAILOVER + LAYER_FRONTEND -->

# Template 06 — CloudFront Global Edge App (S3 + ALB origin · OAC · WAF · edge compute · multi-region failover · 2-3 day deploy)

## Purpose

Stand up a **production-grade CloudFront edge layer** in 2-3 days. Output: CloudFront distribution serving S3 (static SPA) + ALB (API) with OAC, WAF, security headers, edge compute (Functions for SPA routing + redirects), KeyValueStore for redirect tables, and Origin Group for multi-region failover.

This is the **canonical "global edge layer" engagement** — every customer-facing app needs CDN. Quick win for existing customers; recurring upsell to add edge compute features.

---

## Role Definition

You are an expert AWS edge architect with deep expertise in:
- CloudFront distribution + Origin Access Control (OAC, replaces OAI)
- Cache behaviors (path-pattern based) + Cache/Origin Request/Response Headers Policies
- WAF v2 (CLOUDFRONT scope, must be in us-east-1)
- CloudFront Functions vs Lambda@Edge decision tree
- KeyValueStore (Mar 2024 GA) for fast key-value reads
- Origin Groups for multi-region failover
- ACM cert in us-east-1 + Route 53 alias
- Real-time logs + standard logs
- Shield Standard / Advanced

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED — primary app region]
ENV:                         [REQUIRED — dev | stage | prod]

# --- DOMAIN ---
HOSTED_ZONE_ID:              [REQUIRED]
HOSTED_ZONE_NAME:            [REQUIRED]
DOMAIN_NAMES:                [REQUIRED — comma-separated; e.g. app.example.com,www.app.example.com]

# --- ORIGINS ---
S3_STATIC_BUCKET:            [REQUIRED for static site / SPA]
ALB_DNS_NAME:                [optional — if API origin]
ALB_HOSTED_ZONE_ID:          [optional]
LAMBDA_FUNCTION_URL:         [optional — alternative API origin]

# --- MULTI-REGION (failover) ---
ENABLE_FAILOVER:             [false default — true for prod]
FALLBACK_ALB_DNS:            [REQUIRED if failover]
FALLBACK_ALB_REGION:         [REQUIRED if failover]

# --- EDGE COMPUTE ---
ENABLE_SPA_ROUTING:          [true default — Function rewrites paths to /index.html]
ENABLE_REDIRECTS:            [false default — true uses KVS for redirect table]
ENABLE_GEO_ROUTING:          [false default — true requires Lambda@Edge]
ENABLE_AB_TESTING:           [false default — true uses cookie-based variant]
ENABLE_AUTH_AT_EDGE:         [false default — true uses Lambda@Edge for JWT validation]

# --- WAF ---
ENABLE_WAF:                  [true required for prod]
WAF_RATE_LIMIT_PER_IP:       [2000 default]
WAF_GEO_BLOCK:               [comma-separated country codes; empty for none]
ENABLE_BOT_CONTROL:          [true for prod]

# --- SECURITY ---
KMS_KEY_ARN:                 [REQUIRED]
PRICE_CLASS:                 [PRICE_CLASS_100 default — also: 200 | All]
ENABLE_SHIELD_ADVANCED:      [false default — $3000/mo]

# --- LOGS ---
ENABLE_STANDARD_LOGS:        [true default]
ENABLE_REAL_TIME_LOGS:       [false default — Kinesis-based; expensive at scale]
LOG_RETENTION_DAYS:          [90 default]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `CDN_CLOUDFRONT_FOUNDATION` | Distribution + OAC + behaviors + WAF + ACM (us-east-1) |
| `CDN_EDGE_COMPUTE` | Functions + Lambda@Edge + KVS + canonical patterns |
| `CDN_MULTI_ORIGIN_FAILOVER` | Origin Groups + Lambda@Edge geo routing |
| `LAYER_FRONTEND` | S3 + bundling + deployment of SPA |
| `LAYER_OBSERVABILITY` | CW alarms + dashboards |
| `LAYER_SECURITY` | KMS + IAM patterns |

---

## Architecture

```
   Global users (US, EU, APAC)
        │
        │ DNS: app.example.com → CloudFront IP
        ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │ CloudFront Distribution (Price Class 100/200/All)                │
   │   ACM cert (us-east-1)                                            │
   │   WAF v2 attached:                                                 │
   │     - AWSManagedRulesCommonRuleSet                                 │
   │     - AWSManagedRulesBotControlRuleSet (if enabled)                │
   │     - Rate limit per IP (5000 / 5 min default)                     │
   │     - Geo block (optional)                                          │
   │   TLS 1.2+ minimum, HTTP/3 enabled                                  │
   │   IPv6 enabled                                                       │
   │   Standard logs → S3 + (optional) real-time logs → Kinesis          │
   │                                                                     │
   │   Cache behaviors (path-pattern):                                  │
   │     /                  → S3 (HTML, 5 min TTL)                       │
   │     /static/*          → S3 (assets, 1y TTL)                        │
   │     /api/*             → Origin Group (ALB primary + fallback)     │
   │                                                                     │
   │   Edge Compute:                                                    │
   │     viewer-request:                                                  │
   │       - URL rewrite Function (SPA paths → /index.html)              │
   │       - Redirects Function (KVS lookup, 301)                         │
   │       - JWT auth Lambda@Edge (if enabled)                            │
   │     origin-request:                                                  │
   │       - Geo routing Lambda@Edge (if multi-region active)             │
   │     viewer-response:                                                  │
   │       - Dynamic header Function (request-id, A/B variant)             │
   │                                                                     │
   │   KeyValueStore: 100s of redirect entries; updated via SDK         │
   │                                                                     │
   │   Custom error pages:                                              │
   │     403/404 → /index.html (SPA fallback)                             │
   │     5xx → /error/500.html (branded)                                  │
   └─────────────────────────────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   ┌──────────┐    ┌──────────┐    ┌──────────────────┐
   │ S3       │    │ ALB      │    │ Fallback ALB      │
   │ (static  │    │ (primary │    │ (us-west-2;       │
   │ SPA)     │    │  region) │    │  warm standby)    │
   │ OAC-prot │    │ Custom   │    │                    │
   │          │    │ header   │    │                    │
   │          │    │ verify   │    │                    │
   └──────────┘    └──────────┘    └──────────────────┘
                        │
                        ▼
                   ECS / Lambda / EKS
                        │
                        ▼
                   Aurora / DDB / S3
```

---

## Day-by-day execution (2-3 day deploy, 1 dev)

### Day 1 — Foundation: distribution + OAC + WAF + first behavior
- KMS multi-region CMK (if not already)
- ACM cert in us-east-1 (DNS-validated)
- WAF v2 ACL in us-east-1 with managed rules + rate limit (+ optional bot control + geo block)
- S3 static bucket (KMS, BlockPublicAccess.BLOCK_ALL, versioned)
- Cache policies (HTML 5min, assets 1y) + Origin Request + Response Headers Policy (security headers + CSP)
- CloudFront distribution with OAC for S3
- Default behavior → S3 with HTML cache policy
- `/static/*` behavior → S3 with optimized cache (1y)
- Custom error pages (403/404 → /index.html for SPA)
- Route 53 alias to distribution
- **Deliverable:** End of Day 1: `https://app.example.com/` returns 200 with security headers + WAF active.

### Day 2 — API origin + edge compute (Functions + KVS)
- (If `ALB_DNS_NAME`) add `/api/*` behavior → ALB origin with custom header verification
- (If `ENABLE_FAILOVER`) Origin Group: primary ALB + fallback ALB
- **CloudFront Functions:**
  - URL rewrite (SPA path → /index.html or /<path>/index.html)
  - Dynamic header injection (request-id, A/B variant from cookie)
- (If `ENABLE_REDIRECTS`) KeyValueStore + Function for redirect table
- (If `ENABLE_GEO_ROUTING`) Lambda@Edge for per-country origin routing
- (If `ENABLE_AUTH_AT_EDGE`) Lambda@Edge JWT validation on `/api/*` viewer-request
- **Deliverable:** End of Day 2: SPA routing works for all paths; redirects fire from KVS; (optional) auth blocks unauthenticated.

### Day 3 — Hardening + observability + tests
- Standard logs to S3 with 90-day retention + (optional) real-time logs to Kinesis with 5% sampling
- (If `ENABLE_SHIELD_ADVANCED`) subscribe + protect distribution
- CloudWatch dashboards: requests/s, cache hit ratio, error rate by status code, WAF blocked count
- 6 alarms wired to SNS:
  - 5xx rate > 1%
  - 4xx rate > 5%
  - Cache hit ratio < 80%
  - WAF blocked rate spike (DDoS indicator)
  - Origin Group failover triggered
  - Real-time log delivery failures
- Pytest suite covering all validation criteria
- Failover test: drain primary ALB; verify fallback serves; restore
- **Deliverable:** End of Day 3: all tests pass; first failover drill successful; runbook for ops.

---

## Validation criteria

- [ ] **Distribution status DEPLOYED**
- [ ] **HTTPS only**: HTTP redirects to HTTPS
- [ ] **TLS 1.3 supported** (verified via openssl client)
- [ ] **Security headers present**: HSTS + CSP + X-Frame-Options + X-Content-Type-Options
- [ ] **OAC configured** (not legacy OAI)
- [ ] **S3 bucket policy** restricts to `cloudfront.amazonaws.com` with `aws:SourceArn` condition
- [ ] **WAF active** + AWSManagedRulesCommonRuleSet + rate limit
- [ ] **WAF blocks SQL injection probe** (`?id=1'`) → 403
- [ ] **WAF rate limits**: 2000+ req/IP in 5 min → 429
- [ ] **SPA routing works**: `/about` → 200 (index.html content)
- [ ] **Static assets cache 1y**: `/static/main.css` returns `cache-control: max-age=31536000`
- [ ] **HTML cache 5 min**: `/index.html` returns `cache-control: max-age=300`
- [ ] **Cache hit ratio > 80%** after 24h warmup
- [ ] **Origin Group failover works** (test by draining primary)
- [ ] **KVS lookups fire**: redirect entry resolves within 60s of update
- [ ] **(If enabled) Lambda@Edge auth** blocks unauthenticated requests
- [ ] **Standard logs flowing** to S3
- [ ] **6 alarms** OK baseline
- [ ] **CW dashboards** populated with > 24h data

---

## Common gotchas (claude must address proactively)

- **ACM cert must be in us-east-1** for CloudFront — even if app is in eu-west-1. Common surprise.
- **WAF for CloudFront must be CLOUDFRONT scope** + in us-east-1.
- **OAC vs OAI** — OAI is legacy (no KMS, no non-S3). Always use OAC for new builds.
- **Distribution propagation = 5-15 min**. After config change, wait before testing.
- **Cache invalidation cost** — 1000/mo free; then $0.005/path. Use cache-busting URLs (`/static/v1.2.3/...`) instead.
- **CloudFront Functions vs Lambda@Edge** — Functions 6× cheaper. Use Lambda@Edge only for DB lookups, image processing, body manipulation.
- **CloudFront Functions hard limits**: 10 KB code, 2 MB memory, 10 ms exec, single file.
- **SPA routing via 404 → 200 + index.html** hides real 404s. Better: Function path-rewrite for known patterns + 404 only for unknown.
- **Real-time logs cost** — Kinesis ingestion + storage; 100% sampling at high traffic = $1000s/mo. Sample 5%.
- **KeyValueStore eventual consistency 60s** — propagation across edges. Don't use for time-sensitive auth tokens.
- **Origin Group + cached negative responses** — failed primary 5xx cached; subsequent requests skip primary check. After primary recovery, cached negatives still serve fallback until TTL expires.
- **Custom origin verification** (`X-CloudFront-Verify` shared secret) — protect direct ALB access. Rotate secret quarterly.
- **HTTP/3 (QUIC)** — opt-in via `http_version: HTTP2_AND_3`. ~5% mobile perf win.
- **Price class** affects edge location coverage + cost. PRICE_CLASS_100 = 50% cheaper than All; right for US/EU-mostly.

---

## Output artifacts

1. **CDK stacks** (deployed to us-east-1 for CloudFront resources):
   - `CdnFoundationStack` — S3 + ACM + WAF + distribution + OAC + cache policies
   - `CdnEdgeStack` — CloudFront Functions + KVS + (optional) Lambda@Edge
   - `CdnFailoverStack` (optional) — Origin Group with primary + fallback
   - `CdnObservabilityStack` — dashboards + alarms

2. **CloudFront Functions** (JavaScript):
   - `url-rewrite-spa.js` — SPA path fallback
   - `redirects-kvs.js` — KeyValueStore lookup for redirects
   - `dynamic-headers.js` — request-id + variant header

3. **Lambda@Edge** (if enabled):
   - `jwt-auth/` — viewer-request JWT validation
   - `origin-router/` — origin-request per-country routing
   - `image-resize/` — origin-response image manipulation

4. **KeyValueStore** content JSON (initial redirect table)

5. **WAF config** — managed rule sets + custom rules (rate limit, geo)

6. **Pytest suite** covering all validation criteria

7. **CloudWatch dashboard** — distribution metrics overview

8. **6 alarms YAML** — wired to SNS

9. **Failover runbook** — drain primary, verify fallback, restore, validate

10. **Cost projection**:
    - Distribution requests/mo × $0.0085/10K
    - Data transfer GB × $0.085/GB (US/EU)
    - Functions × $0.10/1M
    - Lambda@Edge × $0.60/1M (if enabled)
    - WAF rules + bot control + Shield Advanced (if enabled)

11. **README** — domain URL + deployment instructions + cache invalidation guide

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-27 | Initial composite template. CloudFront + OAC + WAF + Functions + KVS + Origin Groups. 2-3 day global edge deploy. Wave 17. |
