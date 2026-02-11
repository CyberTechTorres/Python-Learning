# [Andres's Python Problem Set Home Lab]() 

## Platforms and Languages Leveraged
- Windows 11
- VSCode
- Python
- Log(N) Pacific Cyber Range

##  High-Level Overview of What This Project Is All About
This is a baseline AI agent SOC analyst (I will continue to add what's needed to make it more accurately automated)
We are particularly communicating to OpenAI API. 
We'll have the API then connect to Azure Log Analytics. 
We will send in a KQL query to retrieve logs from Azure Log Analytics. 
The logs retrieved will be quickly analyzed by OpenAI, which will look for IOCs or suspicious activity. 
OpenAI will then send its findings over to us in a formatted, structured way that's easy to see. 

##  High-Level Technical Overview


### 1st. Section of the script

<img width="1136" height="326" alt="1st_section" src="https://github.com/user-attachments/assets/f0e9a333-1636-4a78-84a5-45bd846ac2b7" />

- `time` is to simply import a stopwatch. 
- `coloroma` is what we'll use to color our text. 
- `OpenAI` of course, is to communicate with OpenAI API
- `DefaultAzureCredential` is what we'll use to log in with our credentials. 
- `LogQueryClient` is to query our logs in Log Analytics. 

- `UTILITIES` file is what we'll use to "sanitize" aka convert strings into structured syntax to be displayed and readable easily.
- `_keys` of course, is our API key to communicate to OpenAI. 
- `MODEL_MANAGEMENT` is where we will find the cost estimate and make sure that the motto is within reasonable usage boundaries. Theres imported "tikToken" for token estimates and GUARDRAILS for boundaries
- `PROMPT_MANAGEMENT` this is where we define formatting instructions for our prompts, whether it be from system or user. This is where we also include our JSON structured function as "TOOLS"
- `EXECUTOR` this is where we send and retrieve query results 
- `GUARDRAILS` this is where we define boundaries for the tables allowed in our queries and allowed ChatGPT models to use with OpenAI API 

### 2nd. Section of the script

<img width="1146" height="157" alt="2nd" src="https://github.com/user-attachments/assets/d5f3828a-c53e-47c1-beee-ad143e65f661" />


- `DefaultAzureCredential` applied into `LogQueryClient` as credential login for queries.
- `_keys` applied into `OpenAI` for API connectivity.
- `MODEL_MANAGEMENT.DEAFULT_MODEL` to specify which model you would like to use for the ChatGPT API.<br>
  Here's what the `DEAFAULT_MODEL` looks like within MODEL_MANAGEMENT:
  
<img width="1257" height="482" alt="4th" src="https://github.com/user-attachments/assets/684d35ac-b78c-4bb5-ba88-966767ce9ab5" />
 More in-depth details about this later in the script. 
 
### 3rd. Section of the script

<img width="1150" height="244" alt="3rd" src="https://github.com/user-attachments/assets/c03056a6-6a1d-4e1c-84f1-369c8649f235" />


`PROMPT_MANAGEMENT.get_user_message()` is where we will prompt the user to get a message to send over to the API.<br>
 This is what `get_user_message()` function does:
 
<img width="1259" height="659" alt="6th" src="https://github.com/user-attachments/assets/cf968d0e-9ea4-462c-a089-3ef753c58109" />

- The user will be presented with a light blue text saying "Agenetic Sock Analyst at your service! What would you like to do?"
- Whatever the user inputs will be stored as `user_input`. 
- If user happens to not put anything into the input, we will go ahead and reference the `prompt` variable for our input instead. In this case, there's nothing in the prompt variable.
- Now the `user_message` consists of a dictionary with two elements in it, which is important for the API to acknowledge. 
- The API will understand that the role will be the user, and the question in which the user is asking will be listed under content. 
- This gives context for structuring and how to interpret the data. 

Now back into main.py (3rd Section)...<br> the next line of code is:<br>  

`EXECUTOR.get_log_query_from_agent(openai_client, user_message, model=model)` <br>

This is where we begin to gather our system_message and our user_message to then send it to OpenAPI. <br>
Heres what the `get_log_query_from_agent()` does: 

<img width="1261" height="642" alt="7th" src="https://github.com/user-attachments/assets/9d1bbd75-0eb2-4f77-b497-19dd4309a5aa" />

- `SYSTEM_PROMPT_TOOL_SELECTION` gives context of how the system we'll interpret the `user_message` data and how it will reply. System includes core objectives and parameter selection when given parameter properties. 
- Now our `system_message` variable consist of our system context. 
- `openai_client.chat.completions.create()` is the function we'll use to send over our message. Our required argument "message" includes the system and the user.
- `PROMPT_MANAGEMENT.TOOLS` this is the bread and butter for what the OpenAI will use as structure to create a function for what we want it to do. It's in within here where we include our parameters, which translate to conditions when OpenAI creates the function. One of our parameters is labeled as "required". These will be the required parameters that OpenAI will have to use in the function it creates from this JSON `TOOLS` file. 
- So we use `tools=PROMPT_MANAGEMENT.TOOLS` to find our TOOL string JSON structure, OpenAI uses this JSON string structure to create our desired function. Now we specify the required parameters to use within that function called `required` which `openai_client.chat.completions.create()` refrences as `tool_choice`
- We then use `json.loads` to convert the received OpenAI string results and convert it to JSON format.
  
Back to our main.py section (3rd  Section)... <br>The result we get from `EXECUTOR.get_log_query_from_agent(openai_client, user_message, model=model)` is `unformatted_query_context` which is this big JSON list with dictionaries within. We now need to access those keys for their values to further format our results because we are not too sure yet what exactly OpenAI has given us. This is where we go ahead and create some if conditions to see if certain keys do exist, and then we could extract those values from those keys into a new variable in which we will call `query_context`

It's within `UTILITIES.sanitize_query_context(unformatted_query_context)` where we will go ahead and fetch these keys and values to then have a more sanitized, formatted query. 
This is what the function looks like:

<img width="1155" height="711" alt="8th" src="https://github.com/user-attachments/assets/70b2fdb0-b936-40ec-8b36-fa6e8ee111ea" />

 All the listed keys here in these if conditions were parameters that were required in our `TOOLS` JSON structure for OpenAI to create a function.
 

<br>Back to our main.py section (3rd Section)... <br>
- `UTILITIES.display_query_context(query_context)` will display to the user a draft of what our query looks like 
- `UTILITIES.display_query_context_rationale(query_context)` we will give us the rationale for explaining why we are using the table and parameters provided. 
- `GUARDRAILS.validate_tables_and_fields(query_context["table_name"], query_context["fields"])` This is where GUARDRAILS checks to make sure that our `table_name` and fields are in fact in the allowed list heres what the function looks like:
<img width="1139" height="736" alt="9th" src="https://github.com/user-attachments/assets/61f56c66-78b9-47fc-bb87-c2e1407e6144" />

The keyword arguments from `validate_tables_and_fields(query_context["table_name"], query_context["fields"])` in main.py correlate positionally to `def validate_tables_and_fields(table, fields)` in GUARDRAILS.py <br>
In other words:
- Key `table_name` via `query_context["table_name"]`  = "table" argument for `validate_tables_and_fields()` function
- Key `fields` via `query_context["fields"]` = "field" argument for `validate_tables_and_fields()` function

### 4th. Section of the script
<img width="1140" height="380" alt="10th(4th_section_script)" src="https://github.com/user-attachments/assets/60b07ee9-d23d-4f3a-9b11-191fd84a33d0" /> 

`EXECUTOR.query_log_analytics()` it's here where we're gathering all of our keys for their values to correspond to the parameters of what `query_log_analytics()` needs which then creates our query and send it over to log analytics <br>

This is how `query_log_analytics()` interprets and processes the keys received:
<img width="1260" height="853" alt="11th" src="https://github.com/user-attachments/assets/3aec97a7-a1a8-42be-917c-6a7a2716a5f8" />

This section in particular within EXECUTOR under that same `query_log_analytics()` is what sends it over to log analytics via:
```
response = log_analytics_client.query_workspace(
        workspace_id=workspace_id,
        query=user_query,
        timespan=timedelta(hours=timerange_hours)

```
- What OpenAI receives here is actually a string query in which we had created by pulling all the keys and values in this same `query_log_analytics()` function, which is now all under `user_query`.
- `timespan` is the time span in which we allow the API to look back on. 

- The rest of the following in this script within `query_log_analytics()` goes ahead and indexes to the variable `response` for its tables (aka logs) we received from our query, which then we convert into a CSV format with the help of Pandas pd. It's then and there where we created a dictionary again and no longer a string. <br>
- Our result tables is organized into a variable called "records". There's also an included variable "count" of the number of records.


<br>Back to our main.py section (4rd Section)...<br>
- `number_of_records = law_query_results['count']` Is accessing a value by key called "count" from our results and putting it under this new variable. <br>
- The next following lines of code we'll go ahead and display the `number_of_records`. And then if `number_of_records` is zero, we exit the program. 

### 5th. Section of the script
we now proceed to the next lines of code in our main.py. 
Keep in mind we now have:
- our user_message
- our query (dictionary with its elements)
- our results (Logs form our query)

It's within `PROMPT_MANAGEMENT.build_threat_hunt_prompt()` here where we create our prompt for OpenAI to threat hunt for us using our log results. 
<img width="1141" height="205" alt="13th(5th_section_script)" src="https://github.com/user-attachments/assets/df1a5929-8acc-481d-8077-71dd1faecf6c" />

`def build_threat_hunt_prompt()` includes formatting instructions, the `user_prompt`, and of course our `log_data` which correlates with our results aka `records` within `law_query_results` variable
<img width="1149" height="359" alt="14th" src="https://github.com/user-attachments/assets/2c68e84e-c9f4-4179-b4e9-e4d32737997f" />

Back to our main.py section (5th Section)... <br>
- Next, the `PROMPT_MANAGEMENT.SYSTEM_PROMPT_THREAT_HUNT` which is our system prompt for context and we assign it to `threat_hunt_system_message` variable.
- Now, as a result, we have our `threat_hunt_user_message` which consists of our records, our query, and our user message. And now we also have the `threat_hunt_system_message` of how to interpret our `threat_hunt_user_message`
- `threat_hunt_messages` will hold both variables in which we can index via 0 or 1 to gather whichever one we'd like.

### 6th. Section of the script
<img width="1141" height="177" alt="15th(6th_section)" src="https://github.com/user-attachments/assets/0fa6f759-5eb6-45ea-bae1-b3c195591159" />

- This next part of our main.py script, we are going to go ahead and get the token count for a rough estimate of how much it will cost for AI to analyze our logs and reply with our custom system and user messages. 
- We are considering our `threat_hunt_messages` variable as a key word argument along with `model` keyword argument for our `count_tokens()` function. As you may remember, `model` is our constant variable `DEAFULT_MODEL` we had in the beginning of our model_management.py file.
  
 <br>This is what `count_tokens()` looks like:<br>
<img width="1226" height="323" alt="18th" src="https://github.com/user-attachments/assets/cee81908-1210-4f8e-84d2-8840e40c5488" />

- Pretty simple and straight to the point. We are going to specify the tokenizer process for our `model` via `encoding_for_model` for which is a function provided by the tiktoken library. Sometimes with newer models, we need to specifically request certain tokenizer models such as `cl100k_base`
- We will then extract the elements within the dictionary within our `messages` variables specifically "role" & "content" values. This data will now be under the `text` variable. This new variable is what we'll go ahead and tokenize to get an estimate.
- So now we have our tokenizer as `enc` and what we would like to tokenize, which is within the `text` variable. Our results are returned under `number_of_tokens` variable back in our main.py file.
  
 <br>Back into our main.py section (6th section).... <br>
 
 Our next line of code is to run `MODEL_MANAGEMENT.choose_model(model, number_of_tokens)` this is what the function looks like:
<img width="1149" height="915" alt="16th" src="https://github.com/user-attachments/assets/c858b420-15a1-4c85-b73d-4466633b1ebd" />

- The arguments within this function that perhaps needs clarification is `tier`, within OpenAI depending on which subscription you opt in for and how much you utilize determines your tier. Every tier has boundaries and limitations, such as token per minute. The details of tier limitation is what we had added within our `GUARDRAILS` file under allowed models. Each model within there has all the boundaries and limitations.
- Another argument within this function that perhaps needs clarification is `assumed_output_tokens` this is a limitation of the amount of tokens we would like to receive from OpenAI for what it had discovered from our threat hut. In this case, we have set it at 500 simply so we make sure we don't spend beyond our budget of 500 output tokens.
  
<br>Let's look at our allowed models and its correlating limitations in which `choose_model(model, number_of_tokens)` references:<br>
<img width="1250" height="233" alt="17th" src="https://github.com/user-attachments/assets/1203f70a-42cf-4fbe-9100-6c99546397cc" />

<br>Back to our main.py section (6th Section)... <br>
- We will go ahead and validate the model in which we've been given from the `choose_model` function Via `GUARDRAILS.py`.

### 7th. Section of the script (FINAL SECTION)
<img width="1441" height="408" alt="19th(7th_section)" src="https://github.com/user-attachments/assets/19e5169f-aba3-4914-ae5c-5f0e62de5592" />

- Starting a timer for how long our actual execution takes for OpenAI to analyze the logs and give us the threat hunt results as `start_time`

<br> Next, `EXECUTOR.hunt()` we'll be doing all the heavy lifting here in this section. Heres what it does: <br>
<img width="1175" height="875" alt="20th" src="https://github.com/user-attachments/assets/eb71d208-7190-4207-976a-e1cc6fd1d3ba" />

- Highlight of within `EXECUTOR.hunt()` is what `openai_client.chat.completions.create()` does. We had gone ahead and organized some variables to contain our system_message and our user_message and a `results` variable with an empty list which will then be filled out from the function. After all, this function can receive structured data for it to be processed.
- We just need to remember whatever results we receive from OpenAI will be a string. When we receive strings, it's not something we can actually process structurally AS IS, so we have to make sure to convert it via JSON.loads to actually have it in JSON format.

Back to our main.py section (7th Section)... <br>

- We go ahead and add a line of code, in case there is no hunt results we exit the program.
- Next, we go ahead and end our `start_time`.
- Next, we ask the user if they would like to see the results we got back as a number of as potential threats. 
- If user opts to see results we go ahead and within `display_threats()` which displays our potential threats prints our `findings` within `hunt_results`. We almost hope OpenAI did create a `findings` field name because we asked ChatGPT to return our results in JSON format, so we expect to have a `findings` section but of course, remember that we actually converted our results into JSON using json.loads, so `findings` can be something we can actually access its value by key into within a dictionary/list. 














