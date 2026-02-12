# [Andres's Python Home Lab]() 

## 🧠 High-Level Overview of What This Project Is All About
This project modifies the SOC Agentic AI baseline to enable log queries using custom-written KQL (Kusto Query Language) within Log Analytics.<br>
<br>The current Agentic AI baseline has a limitation: it does not allow precise filtering of logs using a TimeGenerated range between two specific timestamps. This capability is critical for effective threat hunting, especially after establishing a clear timeline for when an attack started and ended. I will show only the lines of code that were modified compared to the previous baseline. Everything else remains the same. <br>
<br>This project enhances the baseline by adding support for time-bounded KQL queries, allowing more accurate investigations and better alignment with real-world SOC workflows.<br>

## ⚙️ End-to-End Workflow
1️⃣ 
<br>2️⃣ Telling ChatGPT what we're trying to log for between two times frames Line 16 - 23
<br>3️⃣ Adjusting the schema to add in start_time and end_time. Before execution of script
<br>4️⃣ ChatGPT creating and filling in those parameters for query. Line 25 - 31
<br>5️⃣ It's then going to be validated via guardrails. 
<br>5️⃣ Then finally it will be executed in query_log_analytics. But we are assessing keys from our structured received query dictionary to be passed on into the function. 


## 🎯 What This Project Achieves
- 
- 
- 


### 1st. Steps 
<be>This SOC Agentic AI analyst uses a JSON schema called `TOOLS` to define its callable functions. Within this schema, there is currently a parameter named time_range_hours.<be>

<be>We will remove this parameter entirely, as we no longer want the tool to rely on it.<be>

It is important to understand the difference between:

- The API-level time range used when querying Log Analytics, and
- The time range defined directly inside the KQL query itself

<be>These two time filters operate independently and serve different purposes.<be>

<be>The parameters defined inside the TOOLS schema should only represent the values that will be injected into the KQL query. We will later define a separate constant variable outside of this schema to control the API query time span.<be>

Instead of time_range_hours, we will add two new parameters to the TOOLS schema:
- `start_time` : "description": "Start date/time into a full ISO 8601 UTC timestamp with microsecond precision, formatted like YYYY-MM-DDTHH:MM:SS.ssssssZ. (e.g., 2026-01-08T19:19:21.772188Z)."
- `end_time` : "description": "End date/time into a full ISO 8601 UTC timestamp with microsecond precision, formatted like YYYY-MM-DDTHH:MM:SS.ssssssZ. (e.g., 2026-01-08T19:19:21.772188Z)."

These parameters will be used to dynamically construct the time filter inside the KQL query.

<img width="1496" height="954" alt="1st_Actual_1st" src="https://github.com/user-attachments/assets/d7b5d9fb-6b73-48f1-987c-d8c2b2ed1658" />

We also added these new fields to the `required` array, which `tool_choice="required"`in the API request forces the model to produce our `TOOLS` call. <br>
Together, these ensure that a function is always called and that it includes `start_time and end_time` later on line 27 in our `_main.py` file.

<br><br>



### 2nd. Steps: What I modify in `_main.py` file
<img width="1192" height="260" alt="1st" src="https://github.com/user-attachments/assets/821cb9c7-80d6-4806-9262-625b640ecc2d" />


Once we have our `user_message` and the desired `model` from our previous lines of code in `_main.py`. 
- We run into the `years_ago` variable. `years_ago` variable will play a huge key role within our `UTILITIES.sanitize_query_context(unformatted_query_context)` on line 29. We will step into this function later to see how it it works.
- Line 27: `unformatted_query_context = EXECUTOR.get_log_query_from_agent(openai_client, user_message, model=model)` is ran next after `years_ago` variable.

This is what the `get_log_query_from_agent(openai_client, user_message, model=model)` function executes: 
<img width="1327" height="477" alt="2nd" src="https://github.com/user-attachments/assets/eb7e601d-5507-4070-b5ca-3ab224aa9c72" />
 We see our `tool_choice="required"` array we mentioned earlier passed on to `openai_client.chat.completions.create()` which calls out to the API with our messages and query creation. 

 












