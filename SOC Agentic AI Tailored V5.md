# [Andres's Python Home Lab]()  

## 🧠 High-Level Overview of What This Project Is All About
When querying Microsoft Log Analytics during threat hunting, result sets can become extremely large—especially when querying high-volume tables like DeviceProcessEvents or DeviceNetworkEvents. Large result sets introduce several practical issues in a SOC automation pipeline:

- slower execution time and increased API latency
- excessive memory usage when converting results into a DataFrame
- oversized outputs that can exceed LLM context limits
- noisy investigations where analysts can’t easily interpret the most relevant data

To solve this, I implemented a row limit guardrail that enforces a maximum number of results returned from any query. If the result set exceeds the defined limit, the system automatically truncates the output down to a safe maximum while still preserving useful investigation data.

This ensures the system remains performant, cost-aware, and compatible with downstream AI analysis workflows.


## 🎯 What This Project Achieves
- Prevents overly large Log Analytics result sets from overwhelming the application
- Enforces a consistent maximum output size for all queries
- Improves performance by limiting DataFrame size and output serialization
- Reduces the risk of LLM context overflow when sending log results to the model
- Keeps threat hunt results readable and actionable for SOC analysts


## 🔧 Key Changes Implemented
- Added a central guardrail constant in GUARDRAILS.py: ROW_LIMIT = 500
- Implemented logic in EXECUTOR.query_log_analytics() to:
   - measure the number of returned rows
   - truncate the DataFrame if it exceeds the maximum
   - report the truncation to the user for transparency
- Ensured output formatting remains consistent by always returning:
   - a CSV string (records)
   - and an accurate row count (count)
<br><br>


## 🧩 Step 1: Defining the Maximum Allowed Result Size in `GUARDRAILS.py`
To avoid hardcoding limits throughout the codebase, I centralized the policy in a single module:
<img width="615" height="105" alt="1" src="https://github.com/user-attachments/assets/cbefc861-f71f-4582-806d-9b3a051eb57a" />

By defining ROW_LIMIT in GUARDRAILS.py, the limit becomes:
- easy to tune
- reusable across the project
- enforceable in a consistent and maintainable way
<br><br>


## 🧩 Step 2: Measuring Query Result Size in `EXECUTOR.query_log_analytics()`
After retrieving results from Log Analytics and converting them into a DataFrame, the system captures the original size:
<img width="1363" height="764" alt="2" src="https://github.com/user-attachments/assets/75f868c6-c4a8-4e38-b7c4-00a77e278c3d" />

This establishes the “true” size of the response before any enforcement occurs.
<br><br>


## 🧩 Step 3: Truncating Results When the Row Limit Is Exceeded
If the returned result set is larger than the allowed maximum, the system truncates the DataFrame using `.head()`:
<img width="1376" height="790" alt="3" src="https://github.com/user-attachments/assets/16f2660f-9678-4c1c-899e-83f402a18370" />
<br><br>


## 🧩 Step 4: Preserving Output Consistency and Returning a Clean Payload
Even after truncation, I allowed the system produces a consistent return structure:
<img width="1359" height="764" alt="4th" src="https://github.com/user-attachments/assets/bfbf1b23-d91a-4461-8c2d-f7d8b4176683" />

## ✅ Project Conclusion
By adding a row limit guardrail, the system now prevents excessive Log Analytics result sets from degrading performance or overwhelming downstream AI workflows.

This enhancement enforces operational reliability in real-world SOC automation by:
- bounding output size
- maintaining consistent response formatting
- and improving analyst usability through cleaner, more actionable results

The result is a more production-ready threat hunting pipeline that balances investigative flexibility with performance and scalability constraints.








