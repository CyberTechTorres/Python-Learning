# [Andres's Python Home Lab]()  

## 🧠 High-Level Overview of What This Project Is All About
During threat hunting workflows, the Log Analytics query timeframe is one of the most important variables affecting performance, cost, and reliability. Without a strict upper bound, it’s easy for a user (or even the model) to accidentally request an excessively large time range (e.g., “last 2 years”), which can lead to:

- slow queries and timeouts
- high data scan volume and unexpected cost
- noisy / unhelpful results
- inconsistent behavior across environments

To prevent this, I introduced a hard guardrail that ensures the API query timespan never exceeds 365 days. This guardrail is enforced at runtime before the query is executed.

This design keeps the system safe, predictable, and production-oriented. Especially for SOC automation where queries must remain bounded and reliable.


## 🎯 What This Project Achieves
- Prevents overly large Log Analytics queries from running (greater than 365 days)
- Reduces the risk of slow queries, timeouts, and excessive data scanning
- Establishes consistent query behavior regardless of user input
- Adds a reusable “policy-style” limit that can be adjusted centrally
- Aligns the automation with real-world SOC performance and cost constraints

## 🔧 Key Changes Implemented
- Added an API time guardrail constant:
`API_TIME_QUERY_LIMIT = timedelta(days=365)`
- Implemented a guardrail function that caps oversized query windows:
`validate_API_allowed_timespan(API_QUERY_LIMIT)`
- Enforced the validated timespan in `_main.py` before query execution
- Enforced the limit using:
`GUARDRAILS.validate_API_allowed_timespan(API_TIME_QUERY_LIMIT)`
- Added a defensive re-validation inside `EXECUTOR.query_log_analytics()` to ensure safety even if the function is called from other entry points
- Passed the validated timespan into the Log Analytics SDK:
`timespan=API_TIME_QUERY_LIMIT`
<br><br>


## 🧩 Step 1: Adding the Time Limit Constant in `GUARDRAILS.py`
To ensure the guardrail is consistent and reusable, I defined the maximum allowed query window in one place:
<img width="604" height="97" alt="3rd" src="https://github.com/user-attachments/assets/ace56be4-3959-4b79-9415-fb51d9371bea" />

This approach keeps the system configurable and prevents “magic numbers” from being scattered throughout the codebase. If the organization ever wants to reduce this (e.g., 30 days) or increase it (e.g., 730 days), it can be updated in a single location.
<br><br>


## 🧩 Step 2: Implementing the Timespan Guardrail Validator
I added a validator function to enforce a hard upper limit. If an input timespan exceeds 365 days, it is capped to the allowed maximum:
<img width="1059" height="189" alt="3rd" src="https://github.com/user-attachments/assets/9d44416d-b281-4c00-a268-5b25e5e6d666" />
This keeps the behavior explicit and prevents oversized time windows from silently impacting query reliability.
<br><br>


## 🧩 Step 3: Enforcing the Guardrail in `_main.py` Before Query Execution
To apply policy at the orchestration layer, _main.py retrieves the configured limit and validates it before passing it into the executor:
<img width="783" height="401" alt="2nd" src="https://github.com/user-attachments/assets/de0cbfe5-cae6-4f6f-b4ab-ca097d5ba07d" />
<br><br>


## 🧩 Step 4: Defense-in-Depth Enforcement Inside `EXECUTOR.query_log_analytics()`
Even though `_main.py` enforces the rule, I added a second validation inside the executor function:
<img width="1365" height="915" alt="4th" src="https://github.com/user-attachments/assets/0c3d327e-d4a9-477d-9bf7-0200826a489e" />
This is intentional defensive engineering, if the executor is ever called from another script, test harness, or future entry point, the guardrail still prevents oversized queries.

Finally, the validated timespan is applied directly to the Log Analytics query call as shown above:
```
response = log_analytics_client.query_workspace(
    workspace_id=workspace_id,
    query=user_query,
    timespan=API_TIME_QUERY_LIMIT
)
```
<br><br>

## ✅ Project Conclusion
By introducing a centralized 365-day timespan guardrail and enforcing it at both the orchestration and execution layers, the threat hunting pipeline now prevents oversized Log Analytics queries that can cause timeouts, excessive scanning, and unpredictable performance.

This enhancement reflects real SOC engineering priorities, balancing investigative flexibility with production constraints like cost control, reliability, and operational consistency.























