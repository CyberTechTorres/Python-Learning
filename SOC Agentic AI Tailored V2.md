# [Andres's Python Home Lab]() 

## 🧠 High-Level Overview of What This Project Is All About
During development, I encountered a limitation where the API could not accurately resolve relative time inputs (e.g., “four hours ago” or “seven days ago”) because the model does not have inherent awareness of the current UTC time.

Initially, the system prompt was designed to expect explicit start_time and end_time values from the user. However, in real-world threat hunting scenarios, users frequently provide relative time ranges instead of exact timestamps. Relative expressions require a reference point (the current time) in order to be converted into precise ISO 8601 UTC timestamps.

If a user provided explicit timestamps, the system functioned correctly. However, when relative time was used, the results were inaccurate because the model attempted to infer the current time on its own. This led to inconsistent and incorrect time resolution.

Through this process, I identified a key architectural constraint: Large Language Models do not have access to real-time system clocks. Therefore, the current UTC time must be explicitly fetched at runtime and injected into the prompt as contextual data.

To address this, I refactored the prompt management logic to dynamically retrieve the current UTC time during execution and inject it into the system prompt. This allows the model to use a reliable reference point when resolving relative time expressions into exact timestamps.


## 🎯 What This Project Achieves
- Enables accurate handling of relative time inputs (e.g., “last 4 hours”, “last 48 hours”, “yesterday”)
- Eliminates incorrect time assumptions made by the model
- Improves timestamp precision by using a real UTC reference instead of inferred time
- Enhances threat hunting query accuracy in time-bounded log investigations
- Aligns model behavior with real-world SOC analyst workflows

## 🔧 Key Changes Implemented
- In `PROMPT_MANAGEMENT.py`, converted SYSTEM_PROMPT_TOOL_SELECTION from a static dictionary into a dynamic function:
`get_system_prompt_tool_selection()`
- Added logic to fetch the current UTC time using:
`datetime.now(timezone.utc)`
- Injected the fetched UTC time directly into the system prompt under the `Time Resolution Rules` section
- Escaped curly braces in the prompt template to prevent f-string formatting errors
- Updated `EXECUTOR.py` to call
`PROMPT_MANAGEMENT.get_system_prompt_tool_selection()` instead of referencing the deprecated static constant
- Standardized how relative and absolute time inputs are interpreted and converted into ISO 8601 UTC format with microsecond precision
  
```
TIME RESOLUTION RULES
    Current UTC Time: {current_utc_time}
    If the user specifies a relative time range (e.g., “last few hours”, “last 48 hours”, “yesterday”), interpret the range using the current UTC time provided above (dont round up or down. Provide the exact current UTC time).
    So do these steps in order... Resolve the current UTC time then Convert both timestamps to ISO 8601 UTC format with microsecond precision then format as: YYYY-MM-DDTHH:MM:SS.ssssssZ (e.g., 2026-02-08T19:19:21.772188Z) then Populate the computed values into the start_time and end_time arrays.       
    If the user does not provide an ISO 8601 UTC timestamp or a relative time range, assume user is asking for last 4 days (96 hours).
    If the user simply provides a start time and end time timestamps (e.g. "2026-02-08T19:19:21.772188Z" or "2026-02-08") then populate the given values into the start_time and end_time arrays.
  
```


## 🧩 Step 1: Modifying `PROMPT_MANAGEMENT.py`
<img width="1441" height="600" alt="1st" src="https://github.com/user-attachments/assets/2e465e96-ada5-4024-8055-5312ef9c75a4" />

Previously, `SYSTEM_PROMPT_TOOL_SELECTION` was implemented as a static dictionary.
This was refactored into a function that dynamically retrieves the current UTC time at runtime.

Since LLMs do not have access to real-time clocks, the system must explicitly fetch the current UTC timestamp and inject it into the prompt. This ensures that any relative time expressions are resolved against an accurate and deterministic reference point.

This design decision improves consistency, reproducibility, and correctness in time-based query generation.




## 🧩 Step 2: Modifying EXECUTOR.py
EXECUTOR.py was originally referencing the old static dictionary.
This was updated to call the new dynamic function:
<img width="1066" height="640" alt="2nd" src="https://github.com/user-attachments/assets/d26882b2-b7ed-4549-9f99-bbf5e225aa4a" />
This change ensures that every query execution uses a freshly generated prompt containing the current UTC time, rather than a stale, hardcoded prompt.



## ✅ Project Conclusion
By replacing a static prompt configuration with a dynamic, time-aware function, the system can now accurately process both absolute and relative time inputs.

If a user provides explicit timestamps, the system preserves and formats them correctly.
If a user provides relative time expressions, the application now fetches the current UTC time at runtime and uses it as a reference to generate precise ISO 8601 timestamps.

This enhancement significantly improves the reliability of time-scoped threat hunting queries and demonstrates practical awareness of LLM limitations, prompt engineering, and real-world SOC automation design.
