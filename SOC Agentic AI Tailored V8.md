# 🌐 High-Level Overview of What This NSG Enhancement Introduces

This update expands the Agentic AI SOC Analyst beyond endpoint and identity containment to support **network-layer containment via Azure Network Security Groups (NSGs)**.

Previously, the system could:

* Perform AI-driven threat hunting
* Isolate compromised virtual machines (MDE)
* Revoke user sessions
* Disable compromised identities

However, malicious traffic often originates from **external IP addresses targeting multiple resources within a shared network boundary**.

Because this environment shares **one centralized NSG**, and I have permissions to modify inbound rules, this enhancement enables:

* Flow-log-backed malicious IP validation
* Blast-radius-aware NSG deny rule creation
* Automated Azure Management API integration
* Double-confirm containment workflow
* Per-IP idempotency safeguards

This transitions the platform into a **network-aware containment engine**, not just host and identity response automation.

---

# 🎯 What This NSG Enhancement Achieves

* Extends containment to Azure Network Security Groups
* Correlates AI findings with AzureNetworkAnalytics flow logs
* Validates malicious IP activity before enforcement
* Prevents duplicate deny rule creation
* Enforces operator confirmation for network-level actions
* Uses Azure Management REST API for rule deployment
* Applies production-grade error handling (404 / 403 / 409)

---

# 🔧 Key Configuration Changes

## 1️⃣ NSG Resource ID Configuration

A new configuration placeholder was introduced in `_keys.py`:

```python
NSG_RESOURCE_ID = ""
# e.g. "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Network/networkSecurityGroups/<nsg>"
```

Source: 

This allows the system to dynamically target the correct NSG when adding deny rules.

If this value is empty, the system safely skips containment.

---

# 🧩 Step 1: Azure Management Token Support

A new function was added to acquire a management-scoped token:

```python
def get_management_token():
    credential = DefaultAzureCredential()
    token = credential.get_token("https://management.azure.com/.default")
    return token
```
<img width="835" height="157" alt="1" src="https://github.com/user-attachments/assets/e5fcb8f3-223b-4a94-b09d-44df789dbedd" />

Source: 

This enables secure authentication against the Azure Management REST API for NSG rule creation.

The token is acquired lazily — only when an NSG action is required.

---

# 🧩 Step 2: Suspicious IP Extraction from AI Findings

Instead of blindly blocking IPs, the system now extracts IPv4 addresses from AI-provided indicators of compromise:

```python
def extract_suspicious_ips(indicators_of_compromise):
```
<img width="744" height="226" alt="2" src="https://github.com/user-attachments/assets/d5304c3a-eaa6-4986-8801-bb6b2a148d6e" />

Source: 

This uses regex to:

* Parse IoC content
* Deduplicate IPs
* Return a clean list of candidate source IPs

This ensures containment decisions are data-driven and not prompt-fragile.

---

# 🧩 Step 3: Flow Log Correlation Before Enforcement

Before adding a deny rule, the system validates IP activity against flow logs returned from Log Analytics.

### Count IP Occurrences

```python
def count_ip_in_flow_records(records_csv, ip):
```
<img width="778" height="291" alt="3" src="https://github.com/user-attachments/assets/38a27ef4-940b-4e82-812d-96ff19cf3cec" />

Source: 

If an IP appears in fewer than 3 flow rows, containment is skipped.

This threshold prevents overreaction to low-signal events.

---

### Build Flow Evidence Summary

```python
def build_flow_evidence(records_csv, ip, start_time, end_time):
```
<img width="1250" height="600" alt="3 1" src="https://github.com/user-attachments/assets/8f7191c2-fea2-4c1b-9efb-cc49e8ba46f3" />

Source: 

This extracts:

* Source IP
* Unique destination IPs
* Unique destination ports
* Affected VMs
* Flow count
* Time window

This creates operator-visible evidence prior to enforcement.

---

# 🧩 Step 4: NSG Deny Rule Automation

The system now supports programmatic creation of Deny Inbound rules:

```python
def add_nsg_deny_rule(token, nsg_resource_id, source_ip, rule_name, priority=100):
```
<img width="778" height="987" alt="4" src="https://github.com/user-attachments/assets/90b37641-73ad-4b9b-8036-ad455fe91f6e" />

Source: 

The rule:

* Blocks all inbound traffic from the source IP
* Uses priority-based ordering
* Is uniquely named per IP
* Applies direction: Inbound
* Access: Deny

### Hardened Error Handling

Explicit handling for:

| Status Code | Meaning                           |
| ----------- | --------------------------------- |
| 404         | NSG not found                     |
| 403         | Insufficient permissions          |
| 409         | Rule name or priority conflict    |
| 4xx / 5xx   | Captured via `raise_for_status()` |

This ensures production-grade reliability and safe failure.

---

# 🧩 Step 5: Main Containment Branch Logic

Previously, the NSG branch was empty:

```python
elif query_is_about_network_security_group:
    pass
```

Now replaced with full containment logic:

Source: 

### New Behavior

When:

* `query_is_about_network_security_group`
* AND threat confidence is high
<img width="1171" height="982" alt="5" src="https://github.com/user-attachments/assets/5c75bdc9-af82-48e6-bc34-c966c4357199" />

The system:

1. Extracts suspicious IPs from AI findings
2. Correlates them against flow logs
3. Applies ≥ 3 flow threshold
4. Displays structured evidence summary
5. Validates NSG_RESOURCE_ID
6. Lazily acquires management token
7. Prompts for confirmation
8. Adds deny rule via Azure Management API
9. Tracks blocked IP in session

---

# 🧩 Step 6: Per-IP Idempotency Safeguard

To prevent duplicate rule creation:

```python
nsg_blocked_ips = set()
```
<img width="810" height="181" alt="6" src="https://github.com/user-attachments/assets/457eeb96-ab4c-47f6-a284-a62bc9d8dfac" />

Source: 

Before adding a rule:

```python
if suspicious_ip in nsg_blocked_ips:
    continue
```

This ensures:

* No repeated rule creation
* No duplicate deny entries
* Clean session containment state
* Reduced API calls

---

# 🛡️ Security Engineering Principles Applied (NSG Scope)

* Evidence-backed containment decisions
* Flow-log correlation before enforcement
* Config validation prior to action
* Least-assumption automation
* Human-in-the-loop confirmation gating
* Azure RBAC-aware permission validation
* Idempotent rule tracking
* Production-grade REST error handling
* Network-layer blast radius control

---

# ✅ Project Conclusion — NSG Enhancement

This update evolves the system from:

> Endpoint and identity containment automation

Into:

> A multi-layer containment engine capable of enforcing network perimeter controls at the NSG level.

With this change, the Agentic AI SOC Analyst can now:

* Detect malicious flows
* Validate them with telemetry
* Present structured evidence
* Securely add NSG deny rules
* Prevent duplicate enforcement
* Operate safely within a shared NSG architecture

Because the environment uses **one centralized NSG**, this feature enables rapid containment of external threat sources affecting multiple internal resources.

This significantly strengthens the platform’s defensive posture by integrating:

**AI-driven detection + flow validation + programmable network enforcement**

— within a controlled, operator-confirmed SOC workflow.

