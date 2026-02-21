# 🧠 High-Level Overview of What This Enhancement Introduces

During continued development of my agentic SOC AI analyst pipeline, I expanded the system beyond host-level threat detection and containment to support **identity-based incident response actions** via Microsoft Graph API integration.

Previously, the system could:

* Query Log Analytics
* Perform AI-based threat hunting
* Isolate compromised virtual machines via Microsoft Defender for Endpoint (MDE)

However, modern security incidents are not limited to endpoints.
Compromised user identities are often the **initial access vector**, lateral movement enabler, or persistence mechanism.

To address this gap, I implemented:

* Microsoft Graph session revocation
* Microsoft Graph account disablement
* Tiered containment logic based on blast radius
* Defensive confirmation workflows for identity actions
* Session-aware execution safeguards

This enhancement transitions the system from endpoint-centric automation to a **multi-domain SOC containment framework**.

---

# 🎯 What This Project Achieves

* Extends containment capabilities from devices to user identities
* Integrates Microsoft Graph API for real-time identity response
* Introduces blast-radius-aware remediation sequencing
* Prevents duplicate containment actions within the same session
* Preserves human-in-the-loop decision authority
* Aligns identity containment with enterprise SOC best practices

---

# 🔧 Key Changes Implemented

* Added `get_graph_token()` for Microsoft Graph authentication 
* Implemented `revoke_user_sessions()` for token invalidation 
* Implemented `disable_user_account()` for identity containment 
* Added session-scoped state controls (`sessions_revoked`, `user_account_is_disabled`)
* Introduced two-stage containment flow (revoke → disable)
* Hardened API calls with `raise_for_status()`
* Preserved structured confirmation loops
* Expanded SOC containment logic to handle:

  * Host threats
  * Identity threats
  * (Reserved for NSG automation)

---

# 🧩 Step 1: Expanding from Endpoint Containment to Identity Containment

The original automation pipeline focused only on isolating compromised virtual machines.

Now, the system evaluates:

```python
query_is_about_individual_user = query_context["about_individual_user"]
```
<img width="1272" height="181" alt="1" src="https://github.com/user-attachments/assets/3b264f02-73e1-495b-bffc-4bfc9a7b9a51" />


If the threat hunt identifies a **high-confidence identity-related threat**, the agent transitions into a controlled identity containment workflow.

This is a critical architectural shift.

Instead of treating identity compromise as an investigation artifact, the system now supports **direct response orchestration**.

---

# 🧩 Step 2: Integrating Microsoft Graph for Identity Control

To support identity response, I introduced:

```python
def get_graph_token():
    credential = DefaultAzureCredential()
    token = credential.get_token("https://graph.microsoft.com/.default")
    return token
```
<img width="730" height="160" alt="2" src="https://github.com/user-attachments/assets/b548ece0-c37f-4b69-b164-50004c1bf00a" />

This allows secure OAuth-based authentication to Microsoft Graph using Azure-managed identity.

Two containment primitives were implemented:

### 1️⃣ Revoke Active Sessions

```python
POST https://graph.microsoft.com/v1.0/users/{upn}/revokeSignInSessions
```
<img width="1031" height="566" alt="3" src="https://github.com/user-attachments/assets/ec1a6323-9d42-414b-a189-772eeedf17bf" />

This invalidates refresh tokens and forces reauthentication.

Function:

```python
revoke_user_sessions(...)
```

This action reduces attacker persistence while minimizing immediate operational impact.

---

### 2️⃣ Disable User Account

```python
PATCH https://graph.microsoft.com/v1.0/users/{upn}
{
  "accountEnabled": false
}
```
<img width="998" height="637" alt="4" src="https://github.com/user-attachments/assets/a44c6069-a76a-4d88-997b-cb8c6dc419a8" />

Function:

```python
disable_user_account(...)
```

This is a high-impact containment action and prevents all future logins.

---

# 🧩 Step 3: Blast-Radius-Aware Containment Sequencing

Rather than immediately disabling accounts, I designed a staged containment model:

1. **Revoke Sessions** (lower impact, reversible)
2. **Disable Account** (higher impact, confirmed compromise)

This mirrors real-world SOC response escalation:

| Action          | Impact Level | Purpose                       |
| --------------- | ------------ | ----------------------------- |
| Revoke sessions | Moderate     | Remove active attacker tokens |
| Disable account | High         | Full containment              |

This sequencing reflects operational maturity and avoids unnecessary business disruption.

---

# 🧩 Step 4: Defensive Confirmation Workflow for Identity Actions

Both identity actions use structured nested confirmation loops:

```python
confirm = input("Revoke all active sign-in sessions? (y/n): ")
```

Then:

```python
Warning: This will invalidate all refresh tokens...
Do you wish to continue? (y/n):
```
<img width="1084" height="972" alt="5" src="https://github.com/user-attachments/assets/a0531a4f-f71b-47ba-bdde-c61997e7ede7" />

This ensures:

* Deliberate operator intent
* Reduced accidental disruption
* Explicit awareness of operational impact
* SOC-grade human-in-the-loop governance

The same pattern is reused for account disablement.

---

# 🧩 Step 5: Session-Aware Safeguards (Idempotency)

To prevent redundant or repeated API calls in a single hunt session:

```python
sessions_revoked = False
user_account_is_disabled = False
```
<img width="630" height="192" alt="6" src="https://github.com/user-attachments/assets/bc9371e3-05b8-4d76-bba9-0f8459b603ee" />

Before executing containment:

```python
if not sessions_revoked:
```
<img width="1098" height="434" alt="7" src="https://github.com/user-attachments/assets/cf3ae5cc-ab08-4fcc-8abc-5d1cc9831122" />

This ensures:

* No duplicate revocation attempts
* No repeated account disable calls
* Reduced unnecessary API consumption
* Cleaner incident response flow

This is an important production-grade improvement.

---

# 🧩 Step 6: Hardened API Error Handling

All Graph and MDE calls utilize:

```python
resp.raise_for_status()
```

This guarantees:

* Immediate failure on 4xx / 5xx errors
* Clear visibility into permission issues
* Safe exception handling without crashing the pipeline
* Accurate success-state detection

This aligns with REST API best practices.

---

# 🧩 Step 7: Multi-Domain Containment Architecture

The system now evaluates three operational scopes:

```python
about_individual_host
about_individual_user
```

This allows the SOC agent to dynamically determine:

* Endpoint containment (MDE isolation)
* Identity containment (Graph API actions)

This architecture is extensible and modular.

---

# 🛡️ Security Engineering Principles Applied

* Least-disruptive containment first
* Human-in-the-loop control
* Defensive confirmation gating
* Idempotent containment safeguards
* API error resilience
* SOC escalation modeling
* Multi-domain response capability
* OAuth-based secure authentication
* Session-aware state tracking

---

# ✅ Project Conclusion

This enhancement transforms the SOC Agentic AI analyst from:

> A log-querying AI with endpoint isolation capability

Into:

> A structured, multi-domain incident response orchestration engine capable of:
>
> * Host isolation
> * Identity session revocation
> * Account disablement
> * Safe containment sequencing
> * Operational blast-radius awareness

The system now mirrors real enterprise SOC workflows where:

* Identity compromise is treated with urgency
* Containment is deliberate and staged
* High-impact actions require explicit confirmation
* API interactions are hardened and production-safe

This update significantly elevates the platform from a defensive automation script to a **SOC-grade identity and endpoint containment framework** suitable for high-trust security environments.

