# Threat Model — Secure ML Sentiment Analysis API (DevSecMLOps)

This document outlines the threat modeling exercise for the Secure ML Sentiment Analysis API project, built by [thriniiiiiiiiiiii](https://github.com/thriniiiiiiiiiiii).

Methodology: STRIDE framework, aligned with OWASP Top 10 (2021) and cloud/Kubernetes-specific risks.

The goal is to document potential threats, risks, and mitigations across the application, infrastructure, CI/CD pipeline, and runtime layers.

---

## System Context

- **Application**: FastAPI ML API using VADER sentiment analysis
- **Authentication**: JWT bearer tokens (short-lived, 30 min expiry)
- **Containerisation**: Docker (slim, non-root user, read-only FS)
- **Infrastructure**: AWS EKS (private endpoints only, KMS for secrets, Terraform IaC)
- **CI/CD**: GitHub Actions (DevSecMLOps pipeline with least-privilege job tokens)
- **Supply Chain Security**: Cosign keyless signing via GitHub OIDC
- **Runtime Security**: Falco (syscall monitoring)
- **Observability**: Prometheus + Grafana
- **Docker Image**: `thriniiiiiiiiiiii/secure-ml-sentiment-api:latest`

---

## STRIDE Threat Analysis

### 1. Spoofing Identity

- **Threat**: API endpoints accessed without authentication.
- **Mitigation**: JWT-based auth on `/predict` and `/token` endpoints. Tokens signed with `JWT_SECRET` stored in a Kubernetes Secret (not hardcoded).

- **Threat**: CI/CD jobs impersonate trusted identities.
- **Mitigation**: GitHub OIDC federation for AWS — short-lived IAM roles, scoped to repo (`thriniiiiiiiiiiii/Secure-CI-CD-Pipeline-for-an-ML-Sentiment-Analysis-API-DevSecMLOps-`), not forks.

---

### 2. Tampering

- **Threat**: Malicious images pushed to Docker Hub.
- **Mitigation**: Cosign keyless signing (GitHub OIDC); Argo Rollouts verifies signatures before deployment. Unsigned images are rejected.

- **Threat**: Terraform code modified to weaken security posture.
- **Mitigation**: Checkov + Trivy scan Terraform IaC on every PR; PR dependency review gate blocks HIGH/CRITICAL CVEs.

- **Threat**: Source code tampered between commit and container build.
- **Mitigation**: GitHub commit SHA labels baked into container image (`org.opencontainers.image.revision`); SBOM generated and attached.

---

### 3. Repudiation

- **Threat**: No audit trail for cluster or pipeline activity.
- **Mitigation**:
  - AWS CloudTrail + CloudWatch logging enabled for all EKS control plane actions.
  - GitHub Actions workflow logs retained.
  - Cosign transparency log (Rekor) provides cryptographic proof of image signing.

---

### 4. Information Disclosure

- **Threat**: API leaks sensitive data (tokens, secrets, internal errors).
- **Mitigation**:
  - Rate limiting via `slowapi` (60/min global, 10/min for token endpoint).
  - No sensitive data in API responses — only label, score, and username.
  - Security headers enforced: `X-Content-Type-Options`, `X-Frame-Options`, `CSP`, `Cache-Control: no-store`.

- **Threat**: Long-lived AWS IAM keys leaked from GitHub Actions secrets.
- **Mitigation**:
  - Static keys eliminated entirely.
  - GitHub Actions assumes AWS roles via OIDC federation (tokens expire after each run).
  - Forked repos cannot access OIDC roles (audience restricted).

- **Threat**: Container image exposes secrets at build time.
- **Mitigation**: Multi-stage build, no secrets in `Dockerfile` or image layers. SBOM generated post-push to confirm no unexpected packages.

---

### 5. Denial of Service

- **Threat**: Attackers flood the API with requests.
- **Mitigation**:
  - Rate limiting: 60/min global, 30/min for `/predict`, 10/min for `/token`.
  - HPA (Horizontal Pod Autoscaler) scales pods under load.

- **Threat**: Resource exhaustion in CI/CD (dependency bombs, slow builds).
- **Mitigation**: `timeout-minutes` set on Terraform plan job; Trivy fast-exits on HIGH/CRITICAL findings.

---

### 6. Elevation of Privilege

- **Threat**: Compromised pod escalates privileges inside the cluster.
- **Mitigation**:
  - Pods run as non-root user (`appuser`, UID 10001).
  - `readOnlyRootFilesystem: true` (Dockerfile sets `PYTHONDONTWRITEBYTECODE=1`).
  - Namespace-level `pod-security.kubernetes.io/enforce: restricted`.
  - NetworkPolicy restricts ingress to NGINX ingress controller only.
  - Falco detects privilege escalation attempts in real-time.

---

## Mapping to OWASP Top 10 (2021)

| OWASP Top 10 | Risk in this project | Mitigations Applied |
|---|---|---|
| A01: Broken Access Control | Unauthorised API access | JWT auth on all sensitive endpoints, NetworkPolicy |
| A02: Cryptographic Failures | Weak key handling, exposed secrets | KMS for EKS secrets, no hardcoded keys, GitHub OIDC |
| A03: Injection | Input misuse in ML model or query params | Pydantic schema validation on all inputs |
| A04: Insecure Design | Insecure defaults in containers/infra | Non-root pods, read-only FS, private EKS endpoints |
| A05: Security Misconfiguration | Public EKS API endpoint, open SGs | `endpoint_public_access = false`; default SG locked down |
| A06: Vulnerable & Outdated Components | Vulnerable Python libs or OS packages | Dependabot + Trivy scans on every push/PR |
| A07: Identification & Auth Failures | JWT token stolen or replayed | Short 30-min expiry, HTTPS enforced via Ingress |
| A08: Software/Data Integrity Failures | Malicious image deployed | Cosign signing + Argo Rollouts signature verification |
| A09: Security Logging & Monitoring Failures | Blind to attacks | CloudWatch, Prometheus, Grafana, Falco |
| A10: SSRF | Cluster metadata endpoint abuse | No public metadata access; IAM roles scoped to service |

---

## Overall Mitigations Summary

- Static AWS keys eliminated — GitHub OIDC federation used throughout
- Containers signed with Cosign (keyless, transparent log via Rekor)
- Argo Rollouts only deploys verified, signed images
- EKS cluster is private-only — no public API endpoint
- Kubernetes Secrets encrypted with KMS
- Runtime detection with Falco (syscall monitoring)
- Full observability via Prometheus/Grafana
- SBOM generated on every publish (supply chain visibility)
- Defense in depth at code, CI/CD, infra, and runtime layers

---

*Threat model maintained by [thriniiiiiiiiiiii](https://github.com/thriniiiiiiiiiiii)*
