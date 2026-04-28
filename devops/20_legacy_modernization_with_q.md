<!-- Template Version: 2.0 | F369 Wave 18 (composite) | Composes: AI_DEV_Q_TRANSFORMATIONS + AI_DEV_AGENTIC_REFACTOR + AI_DEV_Q_DEVELOPER + MIGRATION_HUB_STRATEGY -->

# Template 20 — Legacy Modernization with Q Developer (Java 8 → 17 · .NET FX → .NET 8 · CI/CD validation · canary merges · 4-12 week program)

## Purpose

Modernize a **legacy Java or .NET codebase using Q Developer /transform** + agentic refactoring in CI/CD over 4-12 weeks. Output: upgraded codebase, validated against full test suite, deployed via canary, with rollback ready.

This is the **canonical "exit Java 8 / .NET Framework" engagement** — pairs perfectly with `migration/02_database_migration` (database modernization) for full-stack legacy exit.

---

## Role Definition

You are an expert AWS application modernization architect with deep expertise in:
- Q Developer /transform (Java upgrades, .NET porting, COBOL → Java preview, VMware → AWS preview)
- Agentic refactor pipelines in CodeBuild + CodePipeline
- Pre-flight readiness assessment (build, test, dependencies, native code, custom plugins)
- Canary merge strategies + auto-rollback for refactored code
- Performance regression testing (JIT differences across Java versions)
- Coordinating with engineering teams (review gates, change management)
- Q Developer Pro subscription requirements

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]

# --- SOURCE ---
SOURCE_LANGUAGE:             [REQUIRED — java | dotnet | cobol | mixed]
SOURCE_VERSION:              [REQUIRED — e.g. java-8 | java-11 | dotnet-fx-4.8]
TARGET_VERSION:              [REQUIRED — e.g. java-17 | java-21 | dotnet-8 | dotnet-9]
CODEBASE_LOC:                [REQUIRED — e.g. 500K]
TEST_COVERAGE_PCT:           [REQUIRED — current % from coverage tool]
REPO_URL:                    [REQUIRED]
PRIMARY_BRANCH:              [main default]

# --- STRUCTURE ---
IS_MONOREPO:                 [true/false]
MODULES:                     [comma-separated module paths if monorepo]
HAS_CUSTOM_BUILD_PLUGINS:    [true/false — slows /transform]
HAS_NATIVE_INTEROP:          [true/false — JNI / P/Invoke; manual port required]
HAS_DEPRECATED_FRAMEWORKS:   [comma-separated; e.g. struts1, axis1, wcf]

# --- WAVES ---
WAVE_COUNT:                  [REQUIRED — 3-8 typical based on size]
WAVE_CADENCE_WEEKS:          [2 default]

# --- VALIDATION ---
RUN_PERF_REGRESSION:         [true required for hot paths]
RUN_SECURITY_REGRESSION:     [true required for prod]
TEST_SUITE_COMMAND:          [REQUIRED — e.g. mvn test, dotnet test]

# --- CICD ---
CI_PROVIDER:                 [REQUIRED — codebuild | github_actions | jenkins]
EXISTING_PIPELINE:           [REQUIRED — re-use vs new]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
ENABLE_AUTO_REVERT:          [true required for prod]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
ENGINEERING_LEAD:            [REQUIRED — for review gate notifications]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `AI_DEV_Q_TRANSFORMATIONS` | Q Developer /transform for Java/`.NET`/COBOL upgrades |
| `AI_DEV_AGENTIC_REFACTOR` | Agentic refactor pipeline + canary merges + auto-revert |
| `AI_DEV_Q_DEVELOPER` | Q Developer Pro base (required for /transform) |
| `MIGRATION_HUB_STRATEGY` | Wave planning + 6R framework |
| `LAYER_OBSERVABILITY` | Performance regression baseline |
| `LAYER_SECURITY` | KMS for source code staging |

---

## Architecture (program-level)

```
   Pre-flight readiness assessment (Week 0)
     - Codebase analysis: Q Developer /transform plan-only mode
     - Build + test on source baseline
     - Dependency inventory
     - Custom plugin / native interop catalog
     - Deprecated framework risk score
     ▼
   Wave plan (varies by size)
     ├── Wave 1: Low-risk modules (10-20% LOC)        — Weeks 1-2
     ├── Wave 2: Medium-risk modules (30-40% LOC)     — Weeks 3-4
     ├── Wave 3: High-risk modules (40-50% LOC)       — Weeks 5-6
     └── Wave N: Final cleanup + retire old config    — Week N
     ▼
   Per-wave execution (2 weeks per wave):
   ┌────────────────────────────────────────────────────────────────┐
   │ Week 1: Transformation + validation                            │
   │   Day 1: Q Developer /transform initiated                       │
   │   Day 2-3: Q processes + generates plan + executes               │
   │   Day 4: Senior engineer reviews diff (full)                      │
   │   Day 5: Manual fixes for flagged sections                        │
   │   Day 6-7: CI/CD: build + test + perf regression + security       │
   │                                                                   │
   │ Week 2: Stage + production deploy                                 │
   │   Day 8: Stage deploy (canary 10% → 100% over 1 day)             │
   │   Day 9-10: 48h soak in stage; monitor errors, latency, cost      │
   │   Day 11: GO/NO-GO meeting; if GO, prod deploy                    │
   │   Day 12: Prod canary 10% → 50% → 100%                            │
   │   Day 13-14: Hyper-care monitoring                                 │
   │   GO/NO-GO criteria: zero p1/p2 incidents                          │
   │                                                                   │
   │ Auto-revert ready: Git revert + redeploy if any wave fails         │
   └────────────────────────────────────────────────────────────────┘
     ▼
   Wave + 1 (next batch)
     ▼
   Final: retire legacy build config + decommission Java 8 / .NET Fx servers
```

---

## Day-by-day execution (typical 8-week program for 500K LOC Java 8 → 17)

### Week 0 — Readiness assessment
- Verify Q Developer Pro subscriptions for transform team
- Pre-flight checklist:
  - Source builds (`mvn clean install`) — must pass
  - Tests pass (≥ test pass rate threshold) — must be green
  - Branch protection ON for main
  - CODEOWNERS file present + correct
  - CI pipeline functional
- Run Q Developer /transform plan-only mode:
  - Outputs: complexity score, manual review estimate, per-module risk
- Wave plan finalized
- Stakeholders sign-off
- **Deliverable:** Wave plan + risk register; Q subscriptions confirmed; CI pipeline validated.

### Week 1-2 — Wave 1 (low-risk modules)
- Day 1: Initiate /transform on Wave 1 modules
- Day 2-4: Q executes + reviewer reviews diffs
- Day 5-7: CI validates: build + test + perf regression + security
- Day 8-9: Stage deploy + 48h soak
- Day 10-11: Prod canary deploy
- Day 12-14: Hyper-care
- **Deliverable:** Wave 1 modules upgraded + in production successfully.

### Week 3-4 — Wave 2 (medium-risk modules)
- Same flow as Wave 1
- Manual review intensity higher (more flagged sections)
- **Deliverable:** Wave 2 in production.

### Week 5-6 — Wave 3 (high-risk modules)
- Same flow
- Includes any deprecated framework migrations (Struts1 → Spring MVC; Axis1 → JAX-WS)
- **Deliverable:** Wave 3 in production.

### Week 7-8 — Final wave + retirement
- Final wave: cleanup, last 10-20% LOC
- Decommission legacy build infra (e.g., Maven 3.5 → 3.9, Java 8 build pipeline)
- Update Dockerfiles to JDK 17
- Update CI/CD agents to Java 17
- Performance benchmarking: compare new prod vs old baseline
- Generate final report
- **Deliverable:** Codebase fully on target version; legacy infra decommissioned.

---

## Validation criteria (per wave)

- [ ] **Q /transform plan generated** with complexity score; senior engineer signed off
- [ ] **Diff review completed** by ≥ 1 senior engineer
- [ ] **Manual review sections addressed** (no untouched flags)
- [ ] **Build passes** on target version
- [ ] **All tests pass** at same coverage % as baseline
- [ ] **No new SonarQube / SpotBugs criticals**
- [ ] **No security regression** (Inspector / Snyk scan)
- [ ] **Performance regression test passes** — p99 latency within 10% of baseline
- [ ] **Stage 48h soak**: no P1/P2 incidents; no error rate spike
- [ ] **Prod canary 10% → 100%** completes without rollback trigger
- [ ] **Hyper-care 14d**: no regressions surface

## Validation criteria (program-level)

- [ ] **All waves merged + deployed**
- [ ] **Legacy build infra decommissioned** (no Java 8 toolchain remaining)
- [ ] **Dockerfiles updated** to target JDK
- [ ] **CI agents on target version**
- [ ] **Performance benchmark report** showing new vs baseline
- [ ] **License cost savings tracked** (Oracle JDK 8 → free Corretto 17)
- [ ] **Post-program review meeting** with eng leadership

---

## Common gotchas (claude must address proactively)

- **Stored procedure / DB constraints** outside Java code — /transform won't touch. Coordinate with `migration/02_database_migration` if applicable.
- **Test suite blind spots** — many legacy codebases have 30-50% coverage. Add tests BEFORE transform for critical paths.
- **JIT performance differences** — Java 17 OpenJDK can be 5-15% faster OR slower than Java 8 for specific workloads. Always perf-test.
- **Library compatibility** — Spring Boot 2 → 3 has breaking namespace changes (javax → jakarta). /transform handles but may miss edge cases.
- **Long-running transformations** can fail mid-flight — `q transform resume <id>`.
- **Custom Maven plugins / proprietary frameworks** — flagged by /transform; require manual port. Catalog upfront.
- **Native interop** — JNI / P/Invoke not auto-ported. Manual rewrite required.
- **Module ordering** — for monorepos, transform leaf modules first (no internal dependencies); core modules last.
- **CI agent compatibility** — CI must support both old + new versions during transition.
- **Retraining devs** — if Java 17 features (records, switch expressions) introduced, briefly train team.
- **Rollback strategy** — keep old branch hot for 90 days post-final-merge in case.
- **Cost** — Q Developer Pro license × transform team; CodeBuild minutes; reviewer time. Estimate: 0.5-1× the equivalent manual modernization cost.

---

## Output artifacts

1. **CDK stacks**:
   - `RefactorPipelineStack` — CodeBuild + agentic refactor pipeline
   - `RegressionTestStack` — perf + security regression suites
   - `ObservabilityStack` — wave + program-level dashboards

2. **Wave plan spreadsheet** — modules × waves × dates × LOC × risk

3. **Pre-flight checklist** (Markdown) — every wave starts here

4. **CodeBuild buildspec** for /transform runner

5. **GitHub Action workflows** for auto-PR + auto-revert

6. **Performance regression baseline** — captured pre-transform; compared post

7. **Manual review SOP** — what to look for in /transform diffs

8. **Senior engineer review checklist** per wave

9. **Per-wave runbook**:
   - Initiate /transform
   - Review diff
   - CI validation
   - Stage deploy + soak
   - Prod canary deploy
   - Hyper-care monitoring
   - Rollback procedure (if needed)

10. **Pytest validation suite** for transformation outcomes

11. **Cost model**:
    - Q Developer Pro × team × months
    - CodeBuild minutes × wave
    - License savings (Oracle → free Corretto)
    - Reviewer time
    - Net ROI projection

12. **Final program report**:
    - Modules upgraded × waves
    - LOC transformed (auto + manual %)
    - Performance delta (baseline vs new)
    - License cost savings
    - Lessons learned (for next program)

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-28 | Initial composite template. Q /transform + agentic refactor pipeline + canary merge + auto-revert. 4-12 week legacy modernization program. Wave 18. |
