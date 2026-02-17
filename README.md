# Secure Software Supply Chain Pipeline

A complete, self-contained GitHub Actions pipeline that demonstrates all 10 software supply chain security characteristics — no external infrastructure required.

Everything runs inside the GitHub Actions runner: building, scanning, signing, deploying to Kubernetes (via KinD), admission control (Kyverno), and DAST testing.

## Pipeline Stages

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECURE SUPPLY CHAIN PIPELINE                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐                                                   │
│  │ 1. VERIFY    │  Git GPG/SSH signature check on commits           │
│  │    COMMITS   │  → Ensures code changes are from known authors    │
│  └──────┬───────┘                                                   │
│         │                                                           │
│  ┌──────▼───────┐  ┌──────────────┐                                 │
│  │ 2a. SEMGREP  │  │ 2b. CODEQL   │  SAST: Static analysis         │
│  │    (SAST)    │  │    (SAST)    │  → Catches vulnerabilities      │
│  └──────┬───────┘  └──────┬───────┘    in source code               │
│         └────────┬────────┘                                         │
│                  │                                                   │
│  ┌───────────────▼──────────────────┐                               │
│  │ 3. BUILD CONTAINER IMAGE         │  Multi-stage Docker build     │
│  │    (Multi-Stage + Minimal Base)  │  → Minimal attack surface     │
│  └───────────────┬──────────────────┘                               │
│                  │                                                   │
│       ┌──────────┼──────────┐                                       │
│       │          │          │                                        │
│  ┌────▼────┐ ┌───▼───┐ ┌───▼─────┐                                 │
│  │ 4.TRIVY │ │ 5.SYFT│ │7. SLSA  │                                 │
│  │  (CVE)  │ │(SBOM) │ │PROVENANCE│                                │
│  └────┬────┘ └───┬───┘ └───┬─────┘                                 │
│       └──────────┼──────────┘                                       │
│                  │                                                   │
│  ┌───────────────▼──────────────────┐                               │
│  │ 6. SIGN IMAGE (Cosign Keyless)   │  OIDC-based signing          │
│  │    + Rekor Transparency Log      │  → Immutable audit trail      │
│  └───────────────┬──────────────────┘                               │
│                  │                                                   │
│  ┌───────────────▼──────────────────┐                               │
│  │ 8. DEPLOY TO KIND                │  Ephemeral K8s cluster        │
│  │ 9. KYVERNO ADMISSION CONTROL    │  → Signature verification     │
│  │ 10. OWASP ZAP (DAST)            │  → Runtime security testing   │
│  └──────────────────────────────────┘                               │
│                                                                     │
│  ┌──────────────────────────────────┐                               │
│  │ 📋 AUDIT SUMMARY                │  All artifacts collected      │
│  │    All artifacts → workflow run  │  → Immutable in GitHub        │
│  └──────────────────────────────────┘                               │
└─────────────────────────────────────────────────────────────────────┘
```

## Characteristic → Tool Mapping

| # | Characteristic | Tool | Cost |
|---|----------------|------|------|
| 1 | Commit signature validation | Git GPG/SSH + Gitsign | Free |
| 2 | SAST | Semgrep + CodeQL | Free |
| 3 | Minimal base image + multi-stage build | Docker BuildKit + Alpine/Distroless | Free |
| 4 | Container vulnerability scanning | Trivy | Free |
| 5 | SBOM generation | Syft (SPDX + CycloneDX) | Free |
| 6 | Container image signing | Cosign (keyless OIDC) | Free |
| 7 | SLSA provenance | in-toto attestation format | Free |
| 8 | Kubernetes deployment | KinD (ephemeral) | Free |
| 9 | Admission control (signature verify) | Kyverno | Free |
| 10 | DAST | OWASP ZAP | Free |
| — | Immutable audit trail | Rekor transparency log + GH artifacts | Free |

## Security Artifacts Generated

Each pipeline run produces and uploads these artifacts:

```
00-pipeline-audit-summary        → Markdown summary of all stages
01-commit-signature-report       → Commit author + signature verification
02-sast-semgrep-results          → Semgrep SARIF findings
03-build-metadata                → Image digest, base image, Dockerfile reference
04-trivy-vulnerability-report    → CVE scan results (table + SARIF)
05-sbom                          → SPDX JSON + CycloneDX JSON
06-cosign-signing-report         → Signature proof + Rekor entry
07-slsa-provenance               → in-toto provenance statement
08-09-deployment-evidence        → Cluster state, Kyverno policy reports
10-dast-zap-report               → OWASP ZAP HTML + JSON report
```

## Project Structure

```
.
├── .github/
│   └── workflows/
│       └── secure-pipeline.yml      # The complete pipeline
├── app/
│   ├── src/main/java/com/example/
│   │   └── HelloApplication.java    # Spring Boot hello world
│   ├── pom.xml                      # Maven build
│   └── Dockerfile                   # Multi-stage build
├── k8s/
│   ├── deployment.yaml              # K8s deployment + service
│   └── kyverno-policy.yaml          # Image signature policy
└── README.md
```

## Prerequisites

1. A GitHub repository with **GitHub Actions** enabled
2. **GHCR (GitHub Container Registry)** access — enabled by default
3. No secrets needed — Cosign uses keyless signing via GitHub's OIDC token

## Usage

1. Push this project to a GitHub repository
2. The pipeline triggers automatically on push to `main`
3. Check the **Actions** tab for pipeline execution
4. Download artifacts from the workflow run for your audit trail

## Adapting for Production

A few things to harden for real use:

- **Commit signing**: Set `verify-commits` to `exit 1` on unsigned commits and enforce via branch protection rules
- **Trivy**: Set `exit-code: "1"` to fail the pipeline on CRITICAL/HIGH CVEs
- **Kyverno**: Change `validationFailureAction` from `Audit` to `Enforce`
- **OWASP ZAP**: Use `zap-full-scan.py` instead of `zap-baseline.py` for deeper coverage
- **Base image**: Consider switching to Chainguard or Distroless for an even smaller attack surface
- **SBOM attach**: Uncomment the `cosign attach sbom` step to store the SBOM in the registry alongside the image
