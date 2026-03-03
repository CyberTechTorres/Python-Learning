# 🛡️ Agentic AI SOC Analyst — Improvements & Changelog

> A Python-based agentic AI that performs **automated threat hunting** against Microsoft Defender for Endpoint (MDE), Azure AD, and Azure resource logs — then takes **real containment actions** when high-confidence threats are found.

This repository documents every improvement I made to the baseline project, transforming it from a basic proof-of-concept into a more production-ready, resilient, and feature-rich tool.

---

## 📑 Table of Contents

- [Architecture Overview](#-architecture-overview)
- [Security & Credential Management](#-security--credential-management)
- [Time Range Handling](#-time-range-handling)
- [KQL Query Construction](#-kql-query-construction)
- [Row Limiting & Data Guardrails](#-row-limiting--data-guardrails)
- [API Migration (Chat Completions → Responses API)](#-api-migration-chat-completions--responses-api)
- [Error Handling & Resilience](#-error-handling--resilience)
- [Input Validation & User Confirmation](#-input-validation--user-confirmation)
- [Expanded Containment Actions](#-expanded-containment-actions)
- [NSG Firewall Containment](#-nsg-firewall-containment)
- [Summary Reporting](#-summary-reporting)
- [Prompt Engineering Enhancements](#-prompt-engineering-enhancements)
- [Guardrails Expansion](#-guardrails-expansion)
- [Model Configuration Updates](#-model-configuration-updates)
- [Code Quality & Maintainability](#-code-quality--maintainability)

---

## 🏗️ Architecture Overview

```
User Input → LLM Tool Call (query selection) → Guardrail Validation → KQL Query → Log Analytics
                                                                                       ↓
                              Containment Actions ← Analyst Review ← LLM Threat Hunt Analysis
                              (Isolate VM / Disable Account / Block IP at NSG)
```

**Files:**

| File | Purpose |
|------|---------|
| `_main.py` | Orchestration and control flow |
| `EXECUTOR.py` | API calls (OpenAI, MDE, Graph, Azure Management) |
| `PROMPT_MANAGEMENT.py` | System prompts, tool schemas, prompt builders |
| `GUARDRAILS.py` | Allowed tables/fields/models, validation logic |
| `MODEL_MANAGEMENT.py` | Token counting, cost estimation, model selection |
| `UTILITIES.py` | Display, sanitization, and reporting helpers |
| `_keys.py` | API keys and resource identifiers |

---

## 🔐 Security & Credential Management

**Before:** API keys and workspace IDs were hardcoded directly in `_keys.py`.

```python
# ❌ Before
OPENAI_API_KEY = ""
LOG_ANALYTICS_WORKSPACE_ID = "60c7f53e-249a-4077-b68e-55a4ae877d7c"
```

**After:** All secrets are loaded from environment variables with safe fallback defaults.

```python
# ✅ After
import os
OPENAI_API_KEY = os.environ.get("OPENAI_API_KEY", "")
LOG_ANALYTICS_WORKSPACE_ID = os.environ.get("LOG_ANALYTICS_WORKSPACE_ID", "")
NSG_RESOURCE_ID = os.environ.get("NSG_RESOURCE_ID", "")
```

**Why it matters:** Prevents accidental secret exposure in version control. Follows the twelve-factor app methodology for configuration management.

---

## ⏱️ Time Range Handling

**Before:** Used a simple `time_range_hours` integer, passed as a `timedelta` to the query. The LLM had to guess how many hours to look back.

```python
# ❌ Before — vague and imprecise
timespan=timedelta(hours=timerange_hours)
```

**After:** Introduced explicit `start_time` and `end_time` ISO 8601 timestamps with microsecond precision. The LLM now resolves relative time references (e.g., "last 48 hours") against the current UTC time injected into the system prompt.

```python
# ✅ After — precise time windows
| where TimeGenerated between (datetime({start_time}) .. datetime({end_time}))
```

**Why it matters:** Eliminates ambiguity, gives analysts exact control over the investigation window, and produces reproducible queries.

---

## 🔍 KQL Query Construction

**Before:** Queries relied on `timedelta` for time filtering and had no sort order.

**After:** Every query branch now includes:
- **Explicit time window** via `TimeGenerated between (datetime(...) .. datetime(...))`
- **Descending sort** (`| sort by TimeGenerated desc`) so the most recent events appear first
- **Consistent structure** across all table-specific query branches

---

## 📏 Row Limiting & Data Guardrails

**Before:** No limit on how many rows were sent to the LLM — a noisy environment could return thousands of rows, blowing past token limits or causing rate-limit errors.

**After:** Added a configurable `ROW_LIMIT` (default: 500) in `GUARDRAILS.py`. If the result set exceeds this limit, it is truncated with a clear warning.

```python
# ✅ After — controlled input size
ROW_LIMIT = 500

if original_count > GUARDRAILS.ROW_LIMIT:
    print(f"Result set contains {original_count} rows. Truncating to {GUARDRAILS.ROW_LIMIT}.")
    df = df.head(GUARDRAILS.ROW_LIMIT)
```

Also added `API_TIME_QUERY_LIMIT` (capped at 365 days) with a validation function to prevent unbounded time-range queries against Log Analytics.

---

## 🔄 API Migration (Chat Completions → Responses API)

**Before:** Used the older OpenAI Chat Completions endpoint (`chat.completions.create`), accessing results via `response.choices[0].message.content`.

**After:** Migrated to OpenAI's newer **Responses API** (`responses.create`), which uses `input=` instead of `messages=` and returns `output_text` instead of nested choices.

Added two helper functions to safely extract data from the new response format:

- **`_safe_output_text(response)`** — extracts text content with a fallback to scanning `response.output` blocks
- **`_extract_first_tool_call_args(response)`** — parses function/tool call arguments from the new response shape, handling both `function_call` and `tool_call` types

**Why it matters:** Keeps the project aligned with OpenAI's latest API direction and ensures forward compatibility.

---

## 🛡️ Error Handling & Resilience

### MDE Device Lookup

**Before:** Used `raise_for_status()` with no exception handling — any network error crashed the program.

```python
# ❌ Before — unhandled crash
resp.raise_for_status()
machines = resp.json().get("value", [])
if not machines:
    raise Exception(f"No machine found starting with {device_name}")
```

**After:** Wrapped in `try/except`, returns `None` on failure so the caller can handle it gracefully.

```python
# ✅ After — graceful degradation
except requests.exceptions.RequestException as e:
    print(f"Error connecting to MDE API: {e}")
    return None
```

### VM Isolation

**Before:** Used a buggy status code check (`if resp.status_code == 201 or 200` — always `True` because `200` is truthy in Python).

```python
# ❌ Before — always returns True due to operator precedence
if resp.status_code == 201 or 200:
    return True
```

**After:** Fixed to check against a tuple of valid status codes, with `raise_for_status()` for error cases.

```python
# ✅ After — correct status check
resp.raise_for_status()
if resp.status_code in (200, 201, 202, 204):
    return True
```

### Query Context / Tool Call

**Before:** No error handling around the tool call extraction — if the model didn't return a tool call, the program would crash with an index error.

**After:** Wrapped in `try/except` with a clear `RuntimeError` that includes context about the failure.

---

## ✅ Input Validation & User Confirmation

**Before:** Used `startswith("y")` for confirmation prompts, which accepted unexpected inputs like "yellow" or "yikes."

```python
# ❌ Before — accepts "yes", "yikes", "yellow"
confirm = input("Would you like to isolate this VM? (yes/no): ").strip().lower()
if confirm.startswith("y"):
```

**After:** Implemented strict `y/n` validation loops with re-prompting on invalid input. Added a **two-step confirmation pattern** for destructive actions — first a "do you want to?" then a "are you sure?" with a warning about impact.

```python
# ✅ After — strict y/n with double confirmation
while True:
    confirm = input("Would you like to isolate this VM? (y/n): ").strip().lower()
    if confirm == "y":
        while True:
            confirmed = input(
                "Warning: This may impact connected systems. Continue? (y/n): "
            ).strip().lower()
            if confirmed == "y":
                # execute action
                break
            elif confirmed == "n":
                print("[i] Isolation canceled.")
                break
            else:
                print("[!] Please enter only 'y' or 'n'.")
        break
    elif confirm == "n":
        print("[i] Isolation skipped.")
        break
    else:
        print("[!] Please enter only 'y' or 'n'.")
```

**Why it matters:** Prevents accidental containment actions in a production SOC environment where a mistyped response could isolate a critical server.

---

## 🚀 Expanded Containment Actions

### Before: Host Isolation Only
The baseline only supported isolating a VM via MDE when a high-confidence host-based threat was found. User-related and NSG-related threats had empty `pass` blocks.

### After: Three Containment Domains

| Domain | Actions Added | API Used |
|--------|--------------|----------|
| **Host** | VM isolation via MDE (improved) | Microsoft Defender for Endpoint |
| **User** | Session revocation + account disable | Microsoft Graph API |
| **NSG** | Dynamic deny-inbound rules per suspicious IP | Azure Management API |

#### 🧑 User Containment (`get_graph_token`, `revoke_user_sessions`, `disable_user_account`)
- **Session revocation** — invalidates all refresh tokens, forcing re-authentication everywhere
- **Account disable** — sets `accountEnabled: false` via Microsoft Graph, fully locking out the identity
- Both actions are behind double-confirmation prompts and only trigger on high-confidence findings

#### 🔥 NSG Containment (see next section)

---

## 🌐 NSG Firewall Containment

Built an entirely new containment pipeline for network-layer threats detected in `AzureNetworkAnalytics_CL` (NSG flow logs):

1. **`extract_suspicious_ips()`** — regex-extracts IPv4 addresses from the LLM's indicators of compromise
2. **`count_ip_in_flow_records()`** — counts how many flow rows mention the IP (threshold: ≥3 to avoid false positives)
3. **`build_flow_evidence()`** — builds a human-readable evidence summary (source IP, destination IPs/ports, affected VMs, flow count, time window)
4. **`add_nsg_deny_rule()`** — adds a `Deny Inbound` rule to the specified NSG via the Azure Management REST API

Key design decisions:
- **Deduplication** — tracks already-blocked IPs in a `set()` to avoid duplicate API calls within the same session
- **Threshold gating** — IPs appearing in fewer than 3 flow rows are skipped with a message
- **Granular error handling** — separate messages for 404 (NSG not found), 403 (insufficient permissions), and 409 (conflicting rule)
- **Lazy token acquisition** — the management token is only obtained when NSG containment is actually needed

---

## 📝 Summary Reporting

**Before:** Threat findings were written to a JSONL file automatically on every run — no user control.

**After:** Added an interactive prompt at the end of each session asking whether to save:
- **`_threats.jsonl`** — machine-readable findings (unchanged format)
- **`SOC_Report_<timestamp>.txt`** — a new human-readable incident summary report

The report includes:
- Query details (table, time range, fields, filters)
- All findings with MITRE ATT&CK mappings, IOCs, and recommendations
- A summary of all containment actions taken (VM isolated, sessions revoked, account disabled, IPs blocked)
- Model used and analysis duration

---

## 🧠 Prompt Engineering Enhancements

### Dynamic System Prompt for Tool Selection

**Before:** `SYSTEM_PROMPT_TOOL_SELECTION` was a static dictionary.

**After:** Replaced with `get_system_prompt_tool_selection()` — a function that injects the **current UTC time** into the prompt at runtime. This allows the LLM to resolve relative time references ("last 48 hours", "yesterday") against an accurate clock.

### Tool Schema Updates

- Added `start_time` and `end_time` parameters (ISO 8601 UTC with microsecond precision)
- Added `user_principal_name` parameter for identity-focused investigations
- Expanded field lists across all tables (e.g., `InitiatingProcessAccountName` for `DeviceProcessEvents`, `IsInitiatingProcessRemoteSession` for `DeviceRegistryEvents`)
- Added a comprehensive **parameter selection heuristics** section guiding the LLM on how to choose tables, fields, and filters
- Added an **investigation playbook** flow (scope → query → assess → pivot → report)
- Added **example parameter mappings** so the LLM learns from concrete scenarios

### Tool Usage Contract

Added explicit rules preventing the LLM from omitting parameters or returning partial schemas — unknown values must be set to `""`, `false`, or `[]`.

---

## 🚧 Guardrails Expansion

### Tables & Fields

Expanded allowed fields across multiple tables:

| Table | Fields Added |
|-------|-------------|
| `DeviceProcessEvents` | `InitiatingProcessAccountName` |
| `DeviceRegistryEvents` | `InitiatingProcessCommandLine`, `RegistryKey`, `RegistryValueName`, `IsInitiatingProcessRemoteSession` |
| `AlertInfo` | `TimeGenerated`, `AlertId`, `Title`, `Category`, `Severity`, `AttackTechniques` |
| `AlertEvidence` | `TimeGenerated`, `DeviceName`, `AccountName`, `Title`, `Categories`, `AttackTechniques`, `EntityType`, `EvidenceRole`, `FileName`, `ProcessCommandLine`, `FolderPath`, `AlertId`, `Severity` |

### New Guardrail Functions

- **`validate_API_allowed_timespan()`** — caps the API query time limit at 365 days
- **`ROW_LIMIT`** constant — prevents sending unbounded data to the LLM

---

## ⚙️ Model Configuration Updates

| Setting | Before | After |
|---------|--------|-------|
| Default Tier | `"4"` | `"1"` (more conservative default) |
| Assumed Output Tokens | `500` | `800` (more realistic estimate) |
| Model Config Format | Inline one-liners | Expanded multi-line dicts for readability |
| `gpt-4.1` Input Cost | `$1.00/M` | `$2.00/M` (updated pricing) |
| `gpt-5-mini` Max Input | `272,000` | `400,000` (updated limits) |
| `gpt-5` Max Input | `272,000` | `400,000` (updated limits) |

---

## 🧹 Code Quality & Maintainability

- **Removed `display_threats()` auto-writing to JSONL** — the old version called `append_threats_to_jsonl()` as a side effect of displaying results. Now saving is a separate, user-controlled action.
- **Added detailed inline comments** throughout `EXECUTOR.py` and `_main.py` explaining OAuth flows, bearer tokens, response parsing, and control flow decisions
- **Consistent response handling** — all API functions now return `None` or `False` on failure instead of raising unhandled exceptions
- **Lazy token acquisition** — Graph and Management tokens are only requested when their respective containment paths are triggered
- **Clean program exit** — added end-of-session messaging and a final goodbye print

---

## 🗺️ Roadmap / Future Improvements

- [ ] Multi-table pivot queries (query multiple tables in sequence)
- [ ] Automated MITRE ATT&CK enrichment from external threat intel feeds
- [ ] Slack/Teams webhook integration for real-time alerting
- [ ] Configurable confidence thresholds for containment actions
- [ ] Audit logging of all containment actions taken per session
- [ ] Support for Azure Sentinel / Microsoft Sentinel integration

---

## 🧰 Tech Stack

| Component | Technology |
|-----------|-----------|
| Language | Python 3.x |
| LLM | OpenAI GPT (Responses API) |
| Auth | Azure `DefaultAzureCredential` (OAuth2) |
| Log Source | Azure Log Analytics (KQL) |
| Endpoint Security | Microsoft Defender for Endpoint API |
| Identity | Microsoft Graph API |
| Network | Azure Management REST API (NSG rules) |
| Token Estimation | `tiktoken` |
| Data Processing | `pandas` |

---

*Built as a learning project to explore agentic AI in cybersecurity operations. Feedback and contributions welcome.*
