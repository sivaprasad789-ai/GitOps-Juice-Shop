# ![Juice Shop Logo](https://raw.githubusercontent.com/juice-shop/juice-shop/master/frontend/src/assets/public/images/JuiceShop_Logo_100px.png) PipelineFortress — Secure CI/CD with GenAI-Assisted Triage

## Project Overview

**PipelineFortress** wraps the deliberately vulnerable **OWASP Juice Shop** in a production-grade
CI/CD pipeline with five security stages — SAST, SCA, secrets scanning, IaC scanning, and DAST —
and a **GenAI triage assistant** that clusters findings, prioritizes them, and drafts remediation
Pull Requests, all behind a strict redaction layer that prevents secrets or source code from ever
reaching an external LLM.

> **Real-World Context:** An e-commerce team shipping 30+ times per day needs security reviews
> that happen *before* merge, not at sprint-end. This project prototypes the reference pipeline
> a platform-security team would build — catching 80% of vulnerabilities before they reach main.

> ⚠️ **Juice Shop is deliberately vulnerable by design.**
> All scanning, exploitation, and testing is confined to the local lab environment.
> Never deploy to a public-facing environment. No real-world targets.

---

## Table of Contents

- [Architecture](#architecture)
- [Week 1 — Baseline Pipeline](#week-1--baseline-pipeline)
- [Week 2 — Shift-Left Coverage](#week-2--shift-left-coverage)
- [Week 3 — GenAI Triage](#week-3--genai-triage)
- [Pipeline Stages & Security Gates](#pipeline-stages--security-gates)
- [Repository Structure](#repository-structure)
- [Setup & Installation](#setup--installation)
- [Running Scans Locally](#running-scans-locally)
- [Baseline Vulnerabilities](#baseline-vulnerabilities)
- [Metrics Dashboard](#metrics-dashboard)
- [GenAI Triage — Redaction Layer](#genai-triage--redaction-layer)
- [Evaluation Rubric Progress](#evaluation-rubric-progress)
- [Ethics & Responsible AI Use](#ethics--responsible-ai-use)
- [References](#references)

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          DEVELOPER WORKSTATION                               │
│             git push → github.com/sivaprasad789-ai/juice-shop                │
└─────────────────────────────────┬────────────────────────────────────────────┘
                                  │ Push to main / PR to main
                                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│               GITHUB ACTIONS — PipelineFortress CI/CD                       │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ WEEK 1 — BASELINE SECURITY STAGES                                   │    │
│  │                                                                     │    │
│  │  Stage 1 ──▶ 🔑 Gitleaks      Secrets scan (full git history)      │    │
│  │                  │             Gate: 0 secrets — HARD FAIL          │    │
│  │             ┌────┴─────┐                                            │    │
│  │  Stage 2 ───▶ Semgrep  │  Stage 3 ──▶ Trivy FS (SCA)              │    │
│  │  (SAST)    │ OWASP Top │  Dependency CVE scan                      │    │
│  │            │ 10 + JWT  │  Gate: CRITICAL CVEs > 5 — FAIL           │    │
│  │            └────┬──────┘       │                                    │    │
│  │                 └──────┬───────┘                                    │    │
│  │  Stage 4 ──▶ 🐳 Docker Build + Save .tar                           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ WEEK 2 — SHIFT-LEFT COVERAGE                                        │    │
│  │                                                                     │    │
│  │  Stage 5 ──▶ 🛡️ Trivy Image    Container CVE scan                  │    │
│  │                                 Gate: 0 CRITICAL OS CVEs            │    │
│  │  Stage 6 ──▶ 📋 Checkov        IaC scan (k8s/ manifests)           │    │
│  │                                 Gate: CRITICAL IaC misconfigs       │    │
│  │  Stage 7 ──▶ 📦 Syft           SBOM generation (JSON + SPDX)       │    │
│  │  Stage 8 ──▶ 🕷️  OWASP ZAP     DAST baseline scan (deployed app)   │    │
│  │                                 Gate: HIGH alerts                   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ WEEK 3 — GENAI TRIAGE                                               │    │
│  │                                                                     │    │
│  │  Stage 9 ──▶ 🤖 Redaction Layer   Strip secrets / source / PII     │    │
│  │  Stage 10 ─▶ 🧠 LLM Triage        Cluster + prioritize + fix       │    │
│  │  Stage 11 ─▶ 🔀 Auto PR           Draft remediation PRs (top 3)    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  Stage 12 ─▶ ☸️  Push → Docker Hub + Deploy → kind + Smoke Test            │
│              └──▶ GitOps: commit updated deployment.yaml → repo             │
└─────────────────────────────────┬────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    kind Cluster (Kubernetes-in-Docker)                       │
│                                                                              │
│  Deployment:    juiceshop-deployment (replicas: 1)                           │
│  Service:       NodePort 30000 → container 3000                              │
│  NetworkPolicy: default-deny + internal + DNS egress only                   │
│  SecurityCtx:   runAsNonRoot, readOnlyRootFilesystem, drop ALL caps          │
│  Optional:      Falco runtime threat detection (stretch goal)                │
└──────────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                     🧃 http://localhost:3000
```

---

## Week 1 — Baseline Pipeline

**Deliverable:** Working pipeline + baseline vulnerability report
**Effort:** ~12 hours

### What was built

**1. Forked OWASP Juice Shop** into `github.com/sivaprasad789-ai/juice-shop` and added the
DevSecOps pipeline files on top of the upstream source without modifying any Juice Shop code.

**2. Baseline deployment to kind cluster** — a 2-node kind cluster
(1 control-plane + 1 worker) with host port 3000 mapped via `extraPortMappings`.
The Deployment uses a hardened `securityContext` (non-root, drop ALL capabilities).

**3. Baseline vulnerability documentation** — See [`docs/baseline-vulnerabilities.md`](docs/baseline-vulnerabilities.md).
60+ intentional vulnerabilities mapped to OWASP Top 10 (2021) with CWE, CVSS score, exploit
payload, and which pipeline stage catches each one.

**4. Three security stages added:**

| Stage | Tool | Rulesets | Gate |
|---|---|---|---|
| Secrets | Gitleaks | Default + custom allowlist | 0 secrets — hard fail |
| SAST | Semgrep | `p/javascript`, `p/nodejs`, `p/owasp-top-ten`, `p/jwt`, `p/xss`, `p/sql-injection` | 0 ERROR findings — hard fail |
| SCA | Trivy FS | `package.json`, `node_modules` | CRITICAL CVEs > 5 — fail |

> The SCA threshold is 5 (not 0) because Juice Shop intentionally ships `node-serialize` (RCE)
> and `vm2` (sandbox escape) as challenge components. In a production pipeline this must be 0.

**Gitleaks allowlist** in [`.gitleaks.toml`](.gitleaks.toml) covers Juice Shop's intentional fake
JWTs and test credentials in `test/`, `ftp/`, and `data/staticData.ts`.

---

## Week 2 — Shift-Left Coverage

**Deliverable:** Full multi-stage pipeline + SBOM + metrics dashboard
**Effort:** ~14 hours

### What was built

**1. IaC Scanning (Checkov)** — scans all files in `k8s/` against CIS Kubernetes benchmarks and
Kubernetes security best practices. Gate fails on `CRITICAL` and `HIGH` severity misconfigurations.

```bash
checkov -d k8s/ --framework kubernetes --soft-fail-on MEDIUM
```

Key checks enforced:
- `CKV_K8S_6` — Do not admit root containers
- `CKV_K8S_8` — Liveness probe must be configured
- `CKV_K8S_14` — Image tag must not be `latest`
- `CKV_K8S_28` — Do not admit containers with added capabilities
- `CKV_K8S_30` — Apply security context to pods and containers
- `CKV_K8S_37` — Minimize the admission of containers with added capability

**2. Container Scanning (Trivy image)** — scans the built `.tar` artifact (downloaded via
GitHub artifact storage — same pattern as Week 1 to avoid cross-job file loss).
Gate: **0 CRITICAL CVEs in OS/runtime layers**.

**3. SBOM Generation (Syft)** — generates a Software Bill of Materials in both JSON and SPDX
formats for every build. Uploaded as a pipeline artifact and committed to `docs/sbom/`.

```bash
syft docker.io/<REPO>/juiceshop:<TAG> -o json > docs/sbom/sbom.json
syft docker.io/<REPO>/juiceshop:<TAG> -o spdx-json > docs/sbom/sbom.spdx.json
```

**4. DAST — OWASP ZAP Baseline Scan** — runs against the deployed staging Juice Shop instance
after the kind deployment smoke test passes.

```bash
docker run --rm --network host \
  ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
  -t http://localhost:3000 \
  -r zap-report.html \
  -J zap-report.json \
  -I           # Do not fail on warnings — gate on HIGH only
```

Gate: Fail if ZAP reports any `HIGH` risk alerts. `MEDIUM` and `LOW` are collected and
reported in the metrics dashboard but do not block the pipeline.

**5. Metrics Dashboard** — see [Metrics Dashboard](#metrics-dashboard) section.

---

## Week 3 — GenAI Triage

**Deliverable:** LLM triage tool + redaction layer + 3 auto-generated PRs + final report
**Effort:** ~12 hours

### Redaction Layer (Critical — runs BEFORE any LLM call)

The redaction layer (`scripts/redact.py`) strips the following from all scanner JSON outputs
**before** any content is sent to the LLM API:

| Data Type | Redaction Method |
|---|---|
| Secrets / credentials | Regex patterns matching API keys, tokens, passwords, JWTs |
| Source code blocks | Remove any `code`, `snippet`, `context` fields from scanner JSON |
| File paths with PII | Normalize to relative paths, strip usernames / absolute paths |
| CVE descriptions with PoC | Keep CVE ID and severity only — strip full description if it contains exploit code |
| Proprietary identifiers | Strip internal hostnames, IPs, org-specific names |

```python
# scripts/redact.py — example redaction pipeline
import re, json

SECRET_PATTERNS = [
    r'(?i)(password|passwd|secret|token|api[_-]?key)\s*[:=]\s*\S+',
    r'eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+',   # JWT
    r'(?i)(aws_access_key_id|aws_secret)\s*=\s*\S+',
]

def redact(text):
    for pattern in SECRET_PATTERNS:
        text = re.sub(pattern, '[REDACTED]', text)
    return text

def redact_scanner_json(findings: dict) -> dict:
    # Remove source code context fields entirely
    for finding in findings.get('results', []):
        finding.pop('extra', None)       # Semgrep source context
        finding.pop('snippet', None)     # Trivy code snippets
        finding.pop('code', None)
        # Redact any string values
        for k, v in finding.items():
            if isinstance(v, str):
                finding[k] = redact(v)
    return findings
```

**Verification:** The redaction layer is tested with a synthetic secrets injection test before
every pipeline run — a known fake credential is injected into the input JSON and verified absent
from the LLM prompt.

### LLM Triage Tool

Inputs: Consolidated scanner JSON (Semgrep + Trivy + Gitleaks + Checkov + ZAP outputs merged).
Model: `claude-sonnet-4-6` via Anthropic API (or OpenAI `gpt-4o` — configurable via `LLM_PROVIDER`).

Outputs per run:
- **Deduplicated findings** — cross-tool duplicates clustered (e.g. same SQLi caught by both Semgrep and ZAP)
- **Prioritized list** — ranked by exploitability × impact, not just severity label
- **3–5 line fix** — concise remediation suggestion per finding
- **False positive flags** — LLM confidence score on each finding

Prompt library in `scripts/prompts/`:

```
prompts/
├── triage.txt          ← Main triage prompt (finding cluster + prioritize)
├── fix_suggest.txt     ← Fix suggestion prompt (3-5 line code fix)
├── pr_draft.txt        ← PR description template prompt
└── fp_analysis.txt     ← False positive / false negative analysis prompt
```

### Auto-Generated Remediation PRs

For the **top 3 findings** by risk score, the pipeline:

1. Checks out a new branch: `fix/llm-triage-<finding-id>`
2. Applies the LLM-suggested fix as a patch
3. Opens a Pull Request with the LLM-generated description
4. Labels it `llm-generated` + `requires-human-review`
5. **A human must review and approve** — the pipeline never auto-merges

Example PRs generated:
- `fix/sqli-login-endpoint` — parameterized query replacing string concatenation in `routes/login.ts`
- `fix/jwt-alg-none` — enforcing `algorithms: ['RS256']` in JWT middleware
- `fix/xxe-xml-parser` — disabling external entity resolution in XML parser config

**Honest FP/FN Analysis** — see `docs/final-report.md`:

| Tool | True Positives | False Positives | False Negatives | Notes |
|---|---|---|---|---|
| Semgrep | 18 | 4 | 6 | FPs mainly in test fixtures |
| Trivy SCA | 34 | 2 | 1 | Very accurate on known CVEs |
| Gitleaks | 3 | 8 | 0 | FPs on Juice Shop challenge JWTs |
| Checkov | 11 | 1 | 2 | FN on custom admission controllers |
| ZAP | 12 | 5 | 9 | DAST misses auth-required pages |
| LLM Triage | 22 | 3 | 4 | No hallucinated CVEs detected |

---

## Pipeline Stages & Security Gates

```
Stage  Tool          Type       Gate Condition                     On Fail
─────  ────────────  ─────────  ────────────────────────────────  ──────────────
1      Gitleaks      Secrets    Any real secret detected           ❌ Hard fail
2      Semgrep       SAST       Any ERROR-level finding            ❌ Hard fail
3      Trivy FS      SCA        CRITICAL CVE count > 5             ❌ Hard fail
4      Docker Build  Build      Build failure                      ❌ Hard fail
5      Trivy Image   Container  Any CRITICAL OS/runtime CVE        ❌ Hard fail
6      Checkov       IaC        Any CRITICAL/HIGH misconfiguration ❌ Hard fail
7      Syft          SBOM       Generation failure                 ⚠️ Warn only
8      OWASP ZAP     DAST       Any HIGH risk alert                ❌ Hard fail
9      Redact+LLM    GenAI      Redaction verification fails       ❌ Hard fail
10     Smoke test    Deploy     HTTP status ≠ 200                  ❌ Hard fail
```

---

## Repository Structure

```
juice-shop/                                  ← Fork of OWASP Juice Shop
│
├── .github/
│   └── workflows/
│       └── devsecops-pipeline.yml           ← Full 12-stage pipeline (Weeks 1–3)
│
├── k8s/
│   ├── kind-config.yaml                     ← kind cluster (control-plane + worker)
│   ├── deployment.yaml                      ← K8s Deployment + hardened securityContext
│   ├── service.yaml                         ← NodePort 30000 → 3000
│   └── networkpolicy.yaml                   ← Default-deny + internal + DNS egress
│
├── scripts/
│   ├── redact.py                            ← Redaction layer (Week 3)
│   ├── triage.py                            ← LLM triage orchestrator (Week 3)
│   ├── pr_generator.py                      ← Auto PR creation (Week 3)
│   ├── merge_findings.py                    ← Merge multi-tool scanner JSON
│   └── prompts/
│       ├── triage.txt
│       ├── fix_suggest.txt
│       ├── pr_draft.txt
│       └── fp_analysis.txt
│
├── docs/
│   ├── baseline-vulnerabilities.md          ← Week 1: OWASP Top 10 baseline catalogue
│   ├── sbom/
│   │   ├── sbom.json                        ← Syft SBOM (JSON format)
│   │   └── sbom.spdx.json                  ← Syft SBOM (SPDX format)
│   ├── metrics-dashboard.md                 ← Week 2: vulnerability metrics
│   ├── zap-report.html                      ← DAST report (generated)
│   └── final-report.md                      ← Week 3: FP/FN analysis + maturity assessment
│
├── policy/
│   └── checkov-custom.yaml                  ← Custom Checkov policy-as-code rules
│
├── .gitleaks.toml                           ← Gitleaks allowlist (Juice Shop test data)
│
├── frontend/                                ← Angular frontend (upstream Juice Shop)
├── routes/                                  ← Express routes (upstream Juice Shop)
├── data/                                    ← Challenge data (upstream Juice Shop)
├── Dockerfile                               ← Upstream Juice Shop Dockerfile
└── package.json                             ← Node.js dependencies
```

---

## Setup & Installation

### Prerequisites

| Tool | Version | Install |
|---|---|---|
| Git | 2.40+ | [git-scm.com](https://git-scm.com) |
| Docker Desktop | 24.0+ | [docker.com](https://docker.com) |
| kubectl | 1.28+ | [kubernetes.io](https://kubernetes.io/docs/tasks/tools/) |
| kind | 0.23+ | [kind.sigs.k8s.io](https://kind.sigs.k8s.io/) |
| Node.js | 22.x or 24.x | [nodejs.org](https://nodejs.org) |
| Python | 3.11+ | [python.org](https://python.org) |
| Trivy | 0.50+ | [trivy.dev](https://trivy.dev) |
| Syft | 1.0+ | [github.com/anchore/syft](https://github.com/anchore/syft) |

### 1. Fork & Clone

```bash
# Fork https://github.com/juice-shop/juice-shop on GitHub, then:
git clone https://github.com/sivaprasad789-ai/juice-shop.git --depth 1
cd juice-shop
```

### 2. Run Juice Shop Locally

```bash
# Via Docker (fastest):
docker run --rm -p 127.0.0.1:3000:3000 bkimminich/juice-shop

# Via source (Node.js 22.x or 24.x required):
npm install && npm start

# Browse to http://localhost:3000
```

### 3. Deploy to kind Cluster

```bash
# Create cluster
kind create cluster --config k8s/kind-config.yaml

# Build and load image
docker build -t juiceshop:local .
kind load docker-image juiceshop:local --name juiceshop-cluster

# Deploy all manifests
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/networkpolicy.yaml

# Wait and verify
kubectl rollout status deployment/juiceshop-deployment
open http://localhost:3000
```

### 4. Configure GitHub Secrets

Go to your fork → **Settings → Secrets and variables → Actions → New repository secret**

| Secret | Description |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `GIT_EMAIL` | Git commit email for pipeline |
| `GIT_USERNAME` | Git commit username for pipeline |
| `LLM_API_KEY` | Anthropic / OpenAI API key (Week 3 only) |

Enable write permissions: **Settings → Actions → General → Workflow permissions → Read and write permissions → Save**

### 5. Trigger the Pipeline

```bash
git add .github/ k8s/ docs/ scripts/ policy/ .gitleaks.toml
git commit -m "feat: add PipelineFortress DevSecOps pipeline"
git push
```

---

## Running Scans Locally

### Secrets — Gitleaks

```bash
docker run --rm -v $(pwd):/repo \
  zricethezav/gitleaks detect \
  --source /repo --config /repo/.gitleaks.toml -v
```

### SAST — Semgrep

```bash
docker run --rm -v $(pwd):/src semgrep/semgrep semgrep \
  --config "p/owasp-top-ten" --config "p/javascript" \
  --config "p/nodejs" --config "p/jwt" /src
```

### SCA — Trivy Filesystem

```bash
trivy fs . --severity CRITICAL,HIGH --format json -o trivy-fs.json
```

### IaC — Checkov

```bash
checkov -d k8s/ --framework kubernetes \
  --check CKV_K8S_6,CKV_K8S_8,CKV_K8S_14,CKV_K8S_28,CKV_K8S_30,CKV_K8S_37
```

### SBOM — Syft

```bash
docker build -t juiceshop:local .
syft juiceshop:local -o json > docs/sbom/sbom.json
syft juiceshop:local -o spdx-json > docs/sbom/sbom.spdx.json
```

### Container Image — Trivy

```bash
docker save juiceshop:local -o juiceshop.tar
trivy image --input juiceshop.tar --severity CRITICAL,HIGH
```

### DAST — OWASP ZAP

```bash
# Requires app running on localhost:3000
docker run --rm --network host \
  ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
  -t http://localhost:3000 \
  -r docs/zap-report.html \
  -J docs/zap-report.json -I
```

### LLM Triage (Week 3)

```bash
# Merge all scanner outputs
python3 scripts/merge_findings.py \
  --semgrep semgrep-results.sarif \
  --trivy trivy-fs.json \
  --gitleaks results.sarif \
  --checkov checkov-results.json \
  --zap docs/zap-report.json \
  --output merged-findings.json

# Run redaction layer
python3 scripts/redact.py \
  --input merged-findings.json \
  --output redacted-findings.json

# Run LLM triage
python3 scripts/triage.py \
  --input redacted-findings.json \
  --model claude-sonnet-4-6 \
  --output triage-report.json
```

---

## Baseline Vulnerabilities

See [`docs/baseline-vulnerabilities.md`](docs/baseline-vulnerabilities.md) for the full Week 1
baseline catalogue mapped to OWASP Top 10 (2021):

| OWASP Category | Count | Key Findings | Pipeline Stage That Catches It |
|---|---|---|---|
| A01 — Broken Access Control | 12 | Admin bypass, IDOR, JWT `alg:none` | Semgrep (SAST) |
| A02 — Cryptographic Failures | 8 | MD5 passwords, JWT in localStorage | Semgrep + ZAP |
| A03 — Injection | 15 | SQLi, XSS, SSTI, NoSQLi | Semgrep + ZAP |
| A05 — Security Misconfiguration | 9 | Stack traces, dir listing, no CSP | ZAP (DAST) |
| A06 — Vulnerable Components | 30+ | `node-serialize` RCE, `vm2` escape | Trivy SCA |
| A07 — Auth Failures | 10 | Weak passwords, hardcoded JWT secret | Semgrep + Gitleaks |
| A10 — SSRF | 2 | Image URL fetch → cloud metadata | ZAP (DAST) |

---

## Metrics Dashboard

See [`docs/metrics-dashboard.md`](docs/metrics-dashboard.md) for the full Week 2 dashboard.
Key metrics tracked per pipeline run:

| Metric | Description | Target |
|---|---|---|
| **Vulnerabilities caught pre-merge** | Findings blocked before reaching main | ≥ 80% |
| **True positive rate per tool** | TP / (TP + FN) per scanner | ≥ 85% |
| **False positive rate per tool** | FP / (FP + TN) per scanner | ≤ 15% |
| **Pipeline gate failure rate** | Builds blocked by security gates | Track trend |
| **Mean time to feedback** | Time from push to security results | < 10 min |
| **SBOM component count** | Total packages tracked per build | Track growth |
| **DAST HIGH alerts** | ZAP HIGH-risk findings per run | 0 in hardened build |
| **LLM triage accuracy** | % findings correctly triaged by LLM | ≥ 80% |

---

## GenAI Triage — Redaction Layer

The redaction layer is the **highest-leverage learning component** of this project.
It answers the question: *what data should and should not flow to an external LLM?*

**Never sent to the LLM:**
- Actual secret values (passwords, tokens, API keys, private keys)
- Raw source code blocks or file snippets
- Absolute file paths containing usernames or internal hostnames
- Internal IP addresses or org-specific identifiers
- PoC exploit code from CVE descriptions

**Sent to the LLM (after redaction):**
- Finding type and category (e.g. `sql-injection`, `A03`)
- Severity and CVSS score
- Affected file path (relative, normalized)
- Line number range
- Semgrep rule ID or CVE ID
- Recommended fix category (e.g. `parameterize-query`)

**Redaction verification test** — run before every LLM call:

```bash
python3 scripts/redact.py --self-test
# Injects 5 synthetic secrets into test JSON
# Verifies all 5 are absent from output
# Fails pipeline if any secret passes through
```

---

## Evaluation Rubric Progress

| Criterion | Weight | Status | Evidence |
|---|---|---|---|
| Pipeline coverage (5 stages) | 20% | ✅ All 5 stages + DAST | `.github/workflows/devsecops-pipeline.yml` |
| Container + IaC hardening | 15% | ✅ Trivy image + Checkov + SBOM | `k8s/`, `docs/sbom/` |
| LLM triage soundness | 20% | ✅ FP/FN analysis, 0 hallucinated CVEs | `docs/final-report.md` |
| Redaction layer | 10% | ✅ Verified via self-test | `scripts/redact.py` |
| Sample PRs | 10% | ✅ 2 PRs (individual mode) | GitHub PR history |
| Metrics + dashboard | 10% | ✅ 8-metric dashboard | `docs/metrics-dashboard.md` |
| Report + oral defense | 15% | ✅ 10–12 pages | `docs/final-report.md` |

---

## Ethics & Responsible AI Use

- All scanning, exploitation, and testing is confined to the local kind lab environment.
  **No real-world targets were used at any stage.**
- GenAI usage disclosure: `claude-sonnet-4-6` (Anthropic) used for finding triage, fix
  suggestion generation, and PR description drafting. All LLM outputs were reviewed by a
  human before any PR was submitted.
- No real credentials, customer data, secrets, or copyrighted content were pasted into any LLM.
- Juice Shop challenge credentials and test JWTs are allowlisted in Gitleaks and excluded from
  all LLM prompts via the redaction layer.
- Responsible AI Use and Lab Conduct acknowledgement signed prior to Week 2.

---

## Node.js Version Compatibility

| node.js | Supported | Tested | Docker images |
|---|---|---|---|
| 26.x | ✅ | ✅ | |
| 24.x | ✅ | ✅ | `latest` (amd64, arm64) |
| 22.x | ✅ | ✅ | |
| < 22.x | ❌ | ❌ | |

---

## References

- EC-Council Essentials Series — DevSecOps module outlines (via program LMS)
- [MITRE ATT&CK](https://attack.mitre.org)
- [NIST Cybersecurity Framework 2.0](https://www.nist.gov/cyberframework)
- [OWASP Top 10](https://owasp.org/Top10/) | [OWASP Juice Shop](https://owasp-juice.shop)
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [CERT-In Advisories](https://www.cert-in.org.in)
- [Semgrep Registry](https://semgrep.dev/r)
- [Trivy Documentation](https://trivy.dev)
- [Gitleaks](https://github.com/gitleaks/gitleaks)
- [Checkov](https://www.checkov.io/)
- [OWASP ZAP](https://www.zaproxy.org/)
- [Syft SBOM](https://github.com/anchore/syft)
- [kind — Kubernetes in Docker](https://kind.sigs.k8s.io/)
- [ArgoCD](https://argo-cd.readthedocs.io/)

---

## Licensing

[![license](https://img.shields.io/github/license/juice-shop/juice-shop.svg)](LICENSE)

OWASP Juice Shop is free software under the [MIT License](LICENSE).
Copyright © Bjoern Kimminich & the OWASP Juice Shop contributors 2014-2026.

PipelineFortress DevSecOps pipeline additions © 2026 Siva Prasad Kusethi.

![Juice Shop Logo](https://raw.githubusercontent.com/juice-shop/juice-shop/master/frontend/src/assets/public/images/JuiceShop_Logo_400px.png)
