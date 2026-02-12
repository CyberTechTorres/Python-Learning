# [Andres's Python Home Lab]() 

## 🧠 High-Level Overview of What This Project Is All About
This project modifies the SOC Agentic AI baseline to enable log queries using custom-written KQL (Kusto Query Language) within Log Analytics.<br>
<br>The current Agentic AI baseline has a limitation: it does not allow precise filtering of logs using a TimeGenerated range between two specific timestamps.<br> This capability is critical for effective threat hunting, especially after establishing a clear timeline for when an attack started and ended. I will show only the lines of code that were modified compared to the previous baseline. Everything else remains the same. <br>

## 🎯 What This Project Achieves
- Removes reliance on coarse, relative time parameters such as time_range_hours
- Introduces the precise, time-bounded KQL filter `TimeGenerated between (start_time .. end_time)` clause, enabling accurate investigation within known incident timelines.
- Clearly separates API-level query scope from KQL-level filtering logic
- Improves accuracy and reliability during incident response and threat hunting

### 1st. Steps: Modifying `TOOLS`
<be>This SOC Agentic AI analyst uses a JSON schema called `TOOLS` to define its callable functions. Within this schema, there is currently a parameter named time_range_hours.<be>

<be>We will remove this parameter entirely, as we no longer want the tool to rely on it.<be>

It is important to understand the difference between:

- The API-level time range used when querying Log Analytics, and
- The time range defined directly inside the KQL query itself

<be>These two time filters operate independently and serve different purposes.<be>

<be>The parameters defined inside the TOOLS schema should only represent the values that will be injected into the KQL query. We will later define a separate constant variable outside of this schema to control the API query time span.<be>

Instead of time_range_hours, we will add two new parameters to the TOOLS schema:
- `start_time` : "description": `"Start date/time into a full ISO 8601 UTC timestamp with microsecond precision, formatted like YYYY-MM-DDTHH:MM:SS.ssssssZ. (e.g., 2026-01-08T19:19:21.772188Z)."`
- `end_time` : "description": `"End date/time into a full ISO 8601 UTC timestamp with microsecond precision, formatted like YYYY-MM-DDTHH:MM:SS.ssssssZ. (e.g., 2026-01-08T19:19:21.772188Z)."`

These parameters will be used to dynamically construct the time filter inside the KQL query.

<img width="1496" height="954" alt="1st_Actual_1st" src="https://github.com/user-attachments/assets/d7b5d9fb-6b73-48f1-987c-d8c2b2ed1658" />

We also added these new fields to the `required` array, which `tool_choice="required"`in the API request forces the model to produce our `TOOLS` call. <br>
Together, these ensure that a function is always called and that it includes `start_time and end_time` later on line 27 in our `_main.py` file.
<br><br>


### 2nd. Steps: Modifying `_main.py` file
<img width="1192" height="260" alt="1st" src="https://github.com/user-attachments/assets/821cb9c7-80d6-4806-9262-625b640ecc2d" />


Once we have our `user_message` and the desired `model` from our previous lines of code in `_main.py`. 
- We run into the `years_ago` variable. `years_ago` variable will play a huge key role within our `UTILITIES.sanitize_query_context(unformatted_query_context)` on line 29. We will step into this function later to see how it it works.
- Line 27: `unformatted_query_context = EXECUTOR.get_log_query_from_agent(openai_client, user_message, model=model)` is ran next after `years_ago` variable.

This is what the `get_log_query_from_agent(openai_client, user_message, model=model)` function executes: 
<img width="1327" height="477" alt="2nd" src="https://github.com/user-attachments/assets/eb7e601d-5507-4070-b5ca-3ab224aa9c72" />
We see our `tool_choice="required"` array we mentioned earlier passed on to `openai_client.chat.completions.create()` which calls out to the API with our messages and query creation. 
<br><br>
 
### 3rd. Step: Modifying `UTILITIES.sanitize_query_context()`
Back into our `_main.py`
We now move on to line 29: `query_context = UTILITIES.sanitize_query_context(unformatted_query_context)`<br>
This is where we gather all the context from our keys within `unformatted_query_context` and begin to organize it with if-conditions.<br>

This is what  `UTILITIES.sanitize_query_context(unformatted_query_context)` function looks like with removed `time_range_hours` key and replaced with `star_time` and `end_time keys`, This ensures that if certain keys are missing from the query context, they are initialized with empty values. It also sanitizes the input by removing characters that could disrupt or manipulate the KQL query, using the `sanitize_literal()` function:<br>
<img width="983" height="687" alt="3rd" src="https://github.com/user-attachments/assets/9a1fdc8a-fa40-4cac-b9f6-ea3ab2656bc0" />
<br><br>
 
### 4th. Steps: Modifying `UTILITIES.display_query_context()`
Back into our `_main.py`
We now move on to line 31: `UTILITIES.display_query_context(query_context)`<br>
This function displays the finalized log search parameters gathered from the query context. It prints them before execution so the user can review the values that will be used in the KQL query. <br>

So here within `display_query_context(query_context)` we removed `time_range_hours` and included `start_time` and `end_time` variables: <br>
<img width="950" height="433" alt="4th" src="https://github.com/user-attachments/assets/91192700-bf9b-4a77-9056-68ce1b421d8f" /> <br>
Here's an example of what `display_query_context` function would display with our new `start_time` and `end_time` variables as 24 hours ago:
<img width="720" height="94" alt="5th" src="https://github.com/user-attachments/assets/69e305a5-bd13-4d48-a0a5-9457e3353d59" />
<br><br>
 
### 5th. Steps: Modifying `EXECUTOR.query_log_analytics()`
Back into our `_main.py`<br>
Line 33: `UTILITIES.display_query_context_rationale(query_context)` stays unmodified.<br>
Line 35: `GUARDRAILS.validate_tables_and_fields(query_context["table_name"], query_context["fields"])` stays unmodified. <br>
Line 37: `law_query_results = EXECUTOR.query_log_analytics()` is the final section of the script where we modify. It's here where we construct our final query to be sent over to log analytics with our created KQL query from previous lines of code. <br>
<img width="1007" height="276" alt="7th" src="https://github.com/user-attachments/assets/6122d34a-409d-483c-b3b5-15e3e4ca2b62" />
What `query_log_analytics()` function processes:<br>
<img width="1152" height="867" alt="8th" src="https://github.com/user-attachments/assets/6cbae49f-4b37-4837-b6e9-fd25cc4d9fcb" /> <br>
We pass all the keys and their values from `query_context` into `query_log_analytics()`. The function uses these values to construct the `user_query` KQL string, which is then sent to the Log Analytics API via the `query` parameter.
 ```
 response = log_analytics_client.query_workspace(
        workspace_id=workspace_id,
        query=user_query,
        timespan=timedelta(days=365* years_ago)
```
This is the bread and butter for querying our KQL within log_analytics. <br>
The `query` argument contains the actual KQL query that will be executed.<br>
The `timespan` argument defines how far back in time the API is allowed to search for logs.<br>

Since `timedelta` does not support years or months directly, we use the `days` parameter instead. We set it to 365 and multiply it by the `years_ago` variable to approximate a yearly time range.<br>
For example, when `years_ag`o is set to 1, the API searches one year back. This approach provides flexibility, allowing us to easily adjust the lookback window — such as using 0.5 for half a year or 1.5 for one and a half years.<br>

## ✅ Project Conclusion
This project successfully extends the SOC Agentic AI baseline by introducing precise, time-bounded KQL querying capabilities. By separating API-level query scope from KQL-level time filtering and enforcing explicit start_time and end_time parameters, the agent now supports investigations that align with real-world SOC timelines.



















