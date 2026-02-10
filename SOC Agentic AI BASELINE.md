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

### 1. First Section of the script

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

