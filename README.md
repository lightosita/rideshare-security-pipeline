# RideShare Pro — Week 11: Security-Gated CI/CD Pipeline

This repository extends the Week 10 RideShare CI/CD pipeline by embedding five automated
security gates into the build workflow. A container image is only promoted to AWS ECR after
every gate passes. No exceptions are permitted without documented justification.

---

## What changed from Week 10

Week 10 established a reusable CI pipeline with runtime validation (lint, build checks)
and a composite action that built and pushed Docker images to ECR.

Week 11 inserts a security enforcement layer **before** the Docker build and **before**
the ECR push. The composite action from Week 10 is unchanged — it is now simply unreachable
unless all upstream security gates pass.

---

## Architecture

Security logic lives in a single reusable workflow (`ci-reusable.yml`). Each service
has a thin caller workflow that passes only its service name, path, and runtime.
This means the security policy is defined once and enforced consistently across all services.


\`\`\`
                ┌─────────────┐
                │   codeql    │  Gate 1 — SAST
                └──────┬──────┘
                       │
                ┌──────▼──────┐
                │  gitleaks   │  Gate 2 — Secret scan
                └──────┬──────┘
                       │
                ┌──────▼──────┐
                │  trivy-fs   │  Gate 3 — Dependency scan
                └──────┬──────┘
                       │
                ┌──────▼──────┐
                │    build    │  docker build (composite action, build only)
                └──────┬──────┘
                       │
          ┌────────────┴────────────┐
          │                         │
   ┌──────▼──────┐           ┌──────▼──────┐
   │ trivy-image │           │  syft-sbom  │  Gates 4 & 5 (parallel)
   └──────┬──────┘           └──────┬──────┘
          │                         │
          └────────────┬────────────┘
                       │  both must pass
                ┌──────▼──────┐
                │  push-ecr   │  docker push (main branch only)
                └─────────────┘

\`\`\`
                ---

## Services secured

| Service | Runtime | CodeQL language |
|---|---|---|
| `rideshare-rider-service` | TypeScript/Node.js | javascript |
| `rideshare-trip-service` | Python/Flask | python |
| `rideshare-matching-service` | Go/Gin | go |

The driver service, email service, and frontend share the same gate pattern
through the reusable workflow — all services benefit from the security gates
automatically.

---

## Security Gate Policy

| Gate / Finding | Result | Exception |
|---|---|---|
| SAST finding (CodeQL) | **Fail pipeline** | None |
| Secret detected (Gitleaks) | **Fail pipeline** | None |
| Critical CVE | **Fail pipeline** | None |
| High CVE | **Fail pipeline** | Document: CVE ID · component · mitigation · remediation date |
| Medium / Low CVE | Report only | None required |
| SBOM missing | **Fail release** | None |
| Image scan failure | **Do not push to ECR** | None |

### Gate 1 — CodeQL (SAST)

Scans application source code for insecure patterns: SQL injection, path traversal,
insecure deserialization, and similar. Runs against the language of each service
(javascript, python, go). Any finding at any severity fails the pipeline immediately —
there are no exceptions. SARIF results are uploaded to the GitHub Security tab.

### Gate 2 — Gitleaks (Secret scanning)

Scans the full git commit history for accidentally committed credentials: API keys,
tokens, private keys, and connection strings. Requires `fetch-depth: 0` to scan
beyond the latest commit. A detected secret must be revoked immediately (treat it
as compromised) and the commit cleaned with `git filter-repo` before the pipeline
can pass again.

### Gate 3 — Trivy filesystem scan

Scans service directories, lockfiles, and dependency manifests for known CVEs before
the image is built. Covers `package-lock.json` (Node.js), `requirements.txt` (Python),
and `go.sum` (Go). Critical and High findings fail the pipeline. Medium and Low are
reported only and uploaded as SARIF artifacts to the Security tab.

### Gate 4 — Trivy image scan

Scans the built container image for OS-level and application-level CVEs — packages
installed in the base image, system libraries, and runtime dependencies. Critical or
High findings prevent the ECR push entirely. Medium and Low are reported only.

### Gate 5 — Syft SBOM

Generates a Software Bill of Materials in CycloneDX JSON format for each image.
The SBOM is uploaded as a GitHub Actions artifact named
`sbom-{service-name}-{commit-sha}` and retained for 90 days. A missing SBOM
fails the release stage — the push-ecr job will not run.

---

## Image tagging strategy

All images are tagged with the **full git commit SHA** as the primary tag,
matching the Week 10 convention:

\`\`\`
<ecr-registry>/rider-service:<commit-sha>
<ecr-registry>/rider-service:latest
\`\`\`

The SHA tag is immutable and ties the image to an exact commit. SBOM artifacts
are also named with the SHA so the bill of materials can always be matched back
to the exact image in ECR. The `latest` tag is updated as a convenience pointer
but is never the only tag applied.

---

## High CVE exception log

Add a row here whenever a High CVE is accepted under the gate policy.
Undocumented exceptions will not be accepted.

| CVE ID | Component | Service | Severity | Mitigation | Remediation date | Owner |
|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — |

---

## Secrets

No secrets or environment-specific values appear in any workflow file.
All sensitive values are stored as GitHub Actions repository secrets.

| Secret | Purpose |
|---|---|
| `AWS_ACCESS_KEY_ID` | AWS authentication |
| `AWS_SECRET_ACCESS_KEY` | AWS authentication |
| `AWS_REGION` | ECR region |
| `ECR_REGISTRY` | ECR registry URL |
| `EKS_CLUSTER_NAME` | Used by deployment.yml only |

---

## Running the pipeline

**On pull request:** All five security gates run plus runtime validation.
The ECR push is skipped — PRs validate only, they never push.

**On push to main:** Full pipeline runs. ECR push executes only if all
gates pass and the branch is `main`.

**Manual rerun:** Use `workflow_dispatch` on any service workflow for a
controlled rerun without a code change.

---

## SBOM artifacts

Available in the GitHub Actions run summary under **Artifacts** after each
successful main-branch run:

- Name: `sbom-{service-name}-{commit-sha}`
- Format: CycloneDX JSON (`.cdx.json`)
- Retained: 90 days
- Tied to: the exact image SHA tag pushed to ECR



