# [Andres's Python Problem Set Home Lab]() 

## Platforms and Languages Leveraged
- Windows 11
- VSCode
- Python
- Log(N) Pacific Cyber Range

##  High Level Overview
This is a baseline AI agent SOC analyst (I will continue to add what's needed to make it more accurately automated)
We are particularly communicating to OpenAI API. 
We'll have the API then connect to Azure Log Analytics. 
We will send in a KQL query to retrieve logs from Azure Log Analytics. 
The logs retrieved will be quickly analyzed by OpenAI, which will look for IOCs or suspicious activity. 
OpenAI will then send its findings over to us in a formatted, structured way that's easy to see. 

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
 This is what get_user_message() function does:
 
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
  
Back to our main.py section (3rd  Section)... <br>The result we get from `EXECUTOR.get_log_query_from_agent(openai_client, user_message, model=model)` is `unformatted_query_context` which is this big JSON list with dictionaries within. We now need to index into those elements to further format our results because we are not too sure yet what exactly OpenAI has given us. This is where we go ahead and create some if conditions to see if certain keys do exist, and then we could extract those values from those keys into a new variable in which we will call `query_context`

It's within `UTILITIES.sanitize_query_context(unformatted_query_context)` where we will go ahead and fetch these keys and values to then have a more sanitized, formatted query. 
This is what the function looks like:

<img width="1155" height="711" alt="8th" src="https://github.com/user-attachments/assets/70b2fdb0-b936-40ec-8b36-fa6e8ee111ea" />
 All the listed keys here in these if conditions were parameters that were required in our `TOOLS` JSON structure for OpenAI to create a function. <br>
<br>

Back to our main.py section (3rd Section)... 
- `UTILITIES.display_query_context(query_context)` will display to the user a draft of what our query looks like 
- `UTILITIES.display_query_context_rationale(query_context)` we will give us the rationale for explaining why we are using the table and parameters provided. 
- `GUARDRAILS.validate_tables_and_fields(query_context["table_name"], query_context["fields"])` This is where GUARDRAILS checks to make sure that our table_name and fuels are in fact in the allowed list heres what the function looks like:
<img width="1139" height="736" alt="9th" src="https://github.com/user-attachments/assets/61f56c66-78b9-47fc-bb87-c2e1407e6144" />

The keyword arguments from `validate_tables_and_fields(query_context["table_name"], query_context["fields"])` in main.py correlate positionally to `def validate_tables_and_fields(table, fields)` in GUARDRAILS.py <br>
In other words:
- indexed table_name via `query_context["table_name"]`  = "table" argument for `validate_tables_and_fields()` function
- indexed fields via `query_context["fields"]` = "field" argument for `validate_tables_and_fields()` function

### 4th. Section of the script
<img width="1140" height="380" alt="10th(4th_section_script)" src="https://github.com/user-attachments/assets/60b07ee9-d23d-4f3a-9b11-191fd84a33d0" /> 

`EXECUTOR.query_log_analytics()` it's here where we're gathering all of our keys for their values to correspond to the parameters of what query_log_analytics() needs which then creates our query and send it over to log analytics <br>

This is how `query_log_analytics()` interprets and processes the parameters received:
<img width="1260" height="853" alt="11th" src="https://github.com/user-attachments/assets/3aec97a7-a1a8-42be-917c-6a7a2716a5f8" />

This section in particular within EXECUTOR under that same `query_log_analytics()` is what sends it over to log analytics via:
```
response = log_analytics_client.query_workspace(
        workspace_id=workspace_id,
        query=user_query,
        timespan=timedelta(hours=timerange_hours)

```
What OpenAI receives here is actually a string query in which we had created by pulling all the keys and values in this same query_log_analytics() function, which is now all under `user_query`. <br>
`timespan` is the time span in which we allow the API to look back on. <br>

 The rest of the following in this script within `query_log_analytics()` goes ahead and indexes to the variable `response` for its tables (aka logs) we received from our query, which then we convert into a CSV format with the help of Pandas pd. It's then and there where we created a dictionary again and no longer a string. <br>
Our result tables is organized into a variable called "records". There's also an included variable "count" of the number of records. <br>

Back to our main.py section (4rd Section)...
- `number_of_records = law_query_results['count']` Is indexing to the "count" from our results and putting it under this new variable. <br>
- The next following lines of code we'll go ahead and display the number_of_records. And then if number_of_records is zero, we exit the program. 

### 5th. Section of the script
we now proceed to the next lines of code in our main.py. 
Keep in mind we now have:
- our user_message
- our query (dictionary with its elements)
- our results (Logs form our query)

It's within `PROMPT_MANAGEMENT.build_threat_hunt_prompt()` here where we create our prompt for OpenAI to threat hunt for us using our log results. 
<img width="1141" height="205" alt="13th(5th_section_script)" src="https://github.com/user-attachments/assets/df1a5929-8acc-481d-8077-71dd1faecf6c" />

`def build_threat_hunt_prompt()` includes formatting instructions, the user_prompt, and of course our log_data which correlates with our results aka "records" within `law_query_results` variable
<img width="1149" height="359" alt="14th" src="https://github.com/user-attachments/assets/2c68e84e-c9f4-4179-b4e9-e4d32737997f" />

Back to our main.py section (5th Section)... <br>
- Next, the `PROMPT_MANAGEMENT.SYSTEM_PROMPT_THREAT_HUNT` which is our system prompt for context and we assign it to `threat_hunt_system_message` variable.
- Now, as a result, we have our `threat_hunt_user_message` which consists of our records, our query, and our user message. And now we also have the `threat_hunt_system_message` of how to interpret our `threat_hunt_user_message`
- `threat_hunt_messages` will hold both variables in which we can index via 0 or 1 to gather whichever one we'd like.















