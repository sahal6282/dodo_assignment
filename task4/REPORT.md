# Task 4: Reconnaissance & Penetration Testing

## Part A: OSINT Reconnaissance (dodopayments.tech)

**Executive Summary:**
Passive reconnaissance of the external attack surface for `dodopayments.tech` reveals several exposed assets, providing potential entry points into the infrastructure.

**Findings:**
1. **Subdomain Enumeration:** Using tools like `crt.sh`, `subfinder`, and `amass`, we identified several subdomains:
   - `api.dodopayments.tech` (Likely the core API gateway)
   - `staging.dodopayments.tech` (Potential unhardened test environment)
   - `admin.dodopayments.tech` (Internal administrative portal, highly sensitive)

2. **Technology Fingerprinting:** HTTP banners and `httpx` scans show:
   - `api.dodopayments.tech` runs Envoy proxy (indicating a service mesh or ingress controller).
   - `staging.dodopayments.tech` leaks a debug page revealing Flask/Werkzeug (Python).

3. **TLS Posture:** A `testssl.sh` scan on the exposed endpoints indicates valid TLS certificates, but some legacy endpoints (e.g. `legacy.dodopayments.tech`) still support TLS 1.1, posing a downgrade attack risk.

**Risk Observations:**
The presence of a `staging` subdomain publicly accessible is a high risk. Staging environments often have weaker authentication, debug modes enabled (like Werkzeug debugger), and can be leveraged to pivot into production if network segmentation is poor.

---

## Part B: Penetration Test (Authorized Target: `ledger-api`)

**Target:** `ledger-api` microservice (vulnerable app running locally).

### Finding 1: Insecure Deserialization (Remote Code Execution)
* **Severity:** CRITICAL (CVSS v3.1: 9.8 - `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`)
* **Endpoint:** `POST /import`
* **Impact:** An attacker can execute arbitrary commands on the container. By exploiting this, they can steal the Kubernetes ServiceAccount token, pivot into the cluster, and compromise the entire Node or namespace.
* **Reproduction Steps / PoC:**
  The endpoint uses the unsafe `yaml.load()` function from PyYAML 5.1.
  ```yaml
  # Payload
  !!python/object/apply:subprocess.Popen
  args: [["whoami"]]
  ```
  ```bash
  curl -X POST http://ledger-api:8080/import -H "Content-Type: application/x-yaml" -d '!!python/object/apply:subprocess.Popen [["whoami"]]'
  # Returns: {"loaded":"<subprocess.Popen object at 0x7f...>"}
  ```
* **Remediation:** Replace `yaml.load(request.data)` with `yaml.safe_load(request.data)`. Update PyYAML to version 6.0+.

### Finding 2: Server-Side Request Forgery (SSRF)
* **Severity:** HIGH (CVSS v3.1: 8.6 - `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:N/A:N`)
* **Endpoint:** `GET /fetch?url=`
* **Impact:** The application fetches any URL provided by the user. An attacker can use this to scan internal network ports, access the cloud provider metadata service (e.g., `169.254.169.254` on AWS) to steal IAM credentials, or bypass firewalls to access internal-only APIs.
* **Reproduction Steps / PoC:**
  ```bash
  curl -s "http://ledger-api:8080/fetch?url=http://169.254.169.254/latest/meta-data/"
  ```
* **Remediation:** Validate the `url` parameter against a strict allowlist of domains. Use a network library configuration that blocks access to private IP ranges (RFC 1918) and the cloud metadata IP.

### Finding 3: Plaintext Cardholder Data Exposure (PCI-DSS Violation)
* **Severity:** HIGH (CVSS v3.1: 7.5 - `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N`)
* **Endpoint:** `GET /transactions`
* **Impact:** The endpoint leaks full 16-digit Primary Account Numbers (PANs). This is a direct violation of PCI-DSS Requirement 3.4.
* **Reproduction Steps / PoC:**
  ```bash
  curl -s http://ledger-api:8080/transactions
  ```
  *Response excerpt:*
  ```json
  {"amount":4200,"currency":"USD","id":"txn_1001","pan":"4242424242424242","status":"captured"}
  ```
* **Remediation:** Truncate or mask the PAN (e.g., return only the last 4 digits: `************4242`) or securely tokenize it before returning the response.

### Bonus: Attack Chain & Defensive Mapping
**Attack Chain:** An attacker first performs OSINT (Part A) and discovers the unhardened `staging` or internal endpoint. They exploit the SSRF (Finding 2) to probe the internal network and find the `ledger-api`. They then use the Insecure Deserialization (Finding 1) to gain a shell on the `ledger-api` pod. Finally, they access `/transactions` locally to steal the plaintext PANs.

**Defensive Controls (Tasks 1-3) that would stop this:**
1. **NetworkPolicy (Task 3):** Our default-deny egress policy would prevent the SSRF from reaching arbitrary external or internal IPs like `169.254.169.254`.
2. **Istio AuthorizationPolicy (Task 3):** STRICT mTLS and RBAC would prevent an external attacker from directly hitting the `/import` endpoint because they lack the `reporting-sa` SPIFFE ID.
3. **Kyverno Pod Security (Task 1):** The `restricted` profile and `runAsNonRoot` prevents the RCE payload from escalating to root privileges or altering system files if exploited.
