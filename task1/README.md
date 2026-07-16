# Task 1: Deploy & Harden the Workload

## Overview
This document outlines the design decisions and evidence of execution for hardening the `ledger-api` microservice to meet production-grade and PCI DSS baseline requirements.

## Architecture

```mermaid
graph TD
    A[Kubernetes API Server] -->|Admission Webhook| B(Kyverno Guardrails)
    B -->|Validates| C{ledger-api Pod}
    C -->|Security Context| D[Non-Root User]
    C -->|Security Context| E[Read-Only RootFS]
    C -->|Security Context| F[Capabilities Dropped]
    C -->|Sealed Secrets| G[Secret Decryption]
```

---

## Proof of Execution (Screenshots)

### 1. Workload Overview (Deployments, Services, ConfigMaps, Ingress)
*Demonstrates the deployment of `ledger-api` alongside the neighbour service, with all corresponding Services, ConfigMaps, and Ingress resources running in the `payments` namespace.*

![Workload Overview](1.png)

### 2. Live Security Context (runAsNonRoot, Read-only FS, etc.)
*Proves the container is actively running with a restricted security context, including `runAsNonRoot: true`, a read-only root filesystem, dropped capabilities, and the `RuntimeDefault` seccomp profile.*

![Live Security Context](2.png)

### 3. Resource Limits & Probes
*Shows the pod configuration with CPU/Memory Requests and Limits properly defined, as well as the active Liveness and Readiness probes to ensure health monitoring.*

![Resource Limits & Probes](3.png)

### 4. RBAC & Service Accounts
*Demonstrates that the default ServiceAccount was dropped in favor of a dedicated, least-privilege ServiceAccount (`ledger-api-sa`), and shows the specific restricted `PolicyRule` assigned to it.*

![RBAC & Service Accounts](4.png)

### 5. Secret Management (GitOps Friendly)
*Proves that plaintext secrets were removed from Git and replaced with an encrypted Secret Management solution (Sealed Secrets).*

![Secret Management](5.png)

### 6. Admission Control Guardrails (Kyverno)
*Shows Kyverno admission webhooks actively intercepting and blocking insecure workloads (e.g., rejecting the `:latest` tag and root users).*

![Admission Control Guardrails](6.png)

---

## Bonus Implementations

### 7. Persona-based RBAC
*Shows the implementation of least-privilege Role-Based Access Control for distinct human personas (`developer` and `operator`), ensuring developers have read-only access while operators can manage deployments.*

![Persona-based RBAC](7.png)

### 8. Pod Security Standards (Restricted)
*Demonstrates the enforcement of Kubernetes Pod Security Standards (Restricted profile) at the namespace level via active namespace labels.*

![Pod Security Standards](8.png)
