# Task 2: Secure CI/CD Pipeline & Supply Chain

## Overview
This document details the transition from manual deployments to a fully automated, secure software supply chain. We implemented a GitHub Actions pipeline that enforces security gates before building, signing, and deploying the image via GitOps (ArgoCD).

---

## 1. Security Gates & Fail Policies

To ensure security is enforced by the pipeline and not by good intentions, we integrated multiple scanning tools. Our fail policy ensures that no vulnerable code makes it to production while preventing developer friction for unfixable issues.

| Tool | Focus Area | Fail Policy & Handling |
|------|------------|-------------------------|
| **Gitleaks** | Secrets (Passwords, Tokens) | **Hard-block**. The pipeline will immediately fail and exit if any plaintext secrets are detected in the commit. |
| **Semgrep** | SAST (Code vulnerabilities) | **Hard-block**. Configured to block on any high-severity structural flaws or injection vectors in the Python code. |
| **Trivy** | CVEs in OS/Dependencies | **Conditional Hard-block**. Blocks on `HIGH` or `CRITICAL` vulnerabilities. However, we use the `--ignore-unfixed=true` flag to **warn** (allow the build) if a CVE exists but no patch has been released by the vendor yet. |

*Bonus: The results of Semgrep and Trivy are uploaded as SARIF files, surfacing natively in the GitHub Security tab.*

---

## 2. Image Signing & SLSA Provenance

To protect against tampering in the registry, we implemented **Cosign Keyless Signing** (OIDC via GitHub Actions).
1. The pipeline authenticates securely with Sigstore.
2. The image is signed and pushed to GHCR.
3. An SLSA-style provenance attestation is generated, proving the image was built by our specific GitHub Actions workflow.

### Verification of Signature
To prove the image was signed by this workflow, you can run the following command in your terminal:
```bash
cosign verify --certificate-oidc-issuer https://token.actions.githubusercontent.com --certificate-identity https://github.com/<YOUR-USERNAME>/ledger-api-assignment/.github/workflows/secure-pipeline.yml@refs/heads/main ghcr.io/<YOUR-USERNAME>/ledger-api-assignment-ledger-api:latest
```

**[INSERT IMAGE 1 HERE: Screenshot of successful `cosign verify` output]**

---

## 3. GitOps & Drift Detection (ArgoCD)

Deployment is no longer handled via manual `kubectl apply`. Instead, the CI pipeline updates the image tag in the Git repository, and **ArgoCD** syncs those changes to the cluster.

ArgoCD is configured with `selfHeal: true`. This ensures that the cluster state exactly matches the Git state (Single Source of Truth).

### Proof of Drift Detection & Self-Heal
If an attacker or operator attempts to manually alter the deployment in the cluster (e.g., bypassing the CI/CD pipeline to change the image or replicas), ArgoCD immediately detects the drift and overwrites the manual change.

**[INSERT IMAGE 2 HERE: Screenshot of ArgoCD UI showing a "Healthy/Synced" state]**

**[INSERT IMAGE 3 HERE: Screenshot showing a terminal where you manually edit a pod/deployment, and ArgoCD instantly reverting it back]**

---

## 4. Pipeline Execution Proof

**[INSERT IMAGE 4 HERE: Screenshot of the GitHub Actions run showing all successful steps (Gitleaks, Semgrep, Trivy, Cosign, Push)]**

**[INSERT IMAGE 5 HERE: Screenshot of the GitHub Security Tab showing the SARIF upload results (Bonus)]**
