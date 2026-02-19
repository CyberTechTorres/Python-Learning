# [Andres's Python Home Lab]() 

## 🧠 High-Level Overview of What This Project Is All About

During development, I encountered a breaking architectural change caused by the deprecation of the `openai_client.chat.completions.create()` method in favor of the newer `Responses API`. The original implementation relied on the Chat Completions response structure (`choices[0].message.content` and `message.tool_calls`), which is no longer aligned with the updated SDK design.

This created multiple compatibility issues, including:

- Type errors when passing unsupported parameters such as `response_format`

- Missing `choices` objects in responses

- Inability to properly parse tool/function calls

- Inconsistent JSON extraction from model outputs

The legacy architecture assumed a single assistant message response format. However, the updated Responses API returns a structured output composed of multiple content blocks, including text segments and tool call objects. As a result, the previous parsing logic became unreliable and caused runtime failures.

Through analysis, I identified that the issue was not merely a syntax update, but a fundamental shift in how the OpenAI API handles:

- Input formatting (`messages` → `input`)

- Output structure (`choices[]` → `output[]`)

- Tool invocation handling

- Text extraction methodology

To resolve this, I refactored the executor layer to fully migrate from the deprecated Chat Completions architecture to the modern Responses API while preserving existing SOC automation workflows and tool-calling logic.

This refactor ensures forward compatibility with the OpenAI SDK, improves response robustness, and aligns the system with current API best practices for agentic and tool-driven applications.

## 🎯 What This Project Achieves

- Migrates deprecated Chat Completions architecture to the modern Responses API
- Eliminates runtime errors caused by outdated response parsing logic
- Enables reliable tool/function calling using the new structured output format
- Improves JSON extraction resilience across different model output structures
- Future-proofs the SOC automation pipeline against SDK deprecations
- Maintains backward logical behavior while upgrading core API infrastructure
- Enhances stability for agentic threat hunting workflows

## 🔧 Key Changes Implemented

- Replaced all instances of:
`openai_client.chat.completions.create()`
with:
`openai_client.responses.create()`
- Updated request payload format from `messages=` to `input=`
- Removed deprecated `response_format={"type": "json_object"}` parameter causing SDK type errors
- Implemented `_safe_output_text()` helper to robustly extract model text from structured Responses API outputs
- Implemented `_extract_first_tool_call_args()` to correctly parse tool/function calls from `response.output` blocks
- Improved tool invocation structure logic to align with the new tool call schema (no longer using `message.tool_calls`)
- Updated `get_query_context()` to support structured tool selection via the Responses API
- Hardened error handling for malformed outputs and missing tool calls
- Preserved existing SOC guardrails, KQL generation logic, and Defender for Endpoint integration without modification
<br><br>


## 🧩 Step 1: Migrating from Chat Completions to Responses API

Legacy Chat Completions API architecture:
<img width="929" height="645" alt="1st" src="https://github.com/user-attachments/assets/7cef9db4-c103-4f1d-b8ce-6b0438e227ab" />

Previously, the system used the legacy Chat Completions endpoint:
```
openai_client.chat.completions.create(
    model=model,
    messages=messages
)
```
This architecture returned responses in the form:
```
response.choices[0].message.content
```
and tool calls under:
```
response.choices[0].message.tool_calls
```
After the API update, this structure became obsolete. The new Responses API returns a structured output object containing multiple content blocks instead of a single message string.


The refactor replaced:
```
messages=messages
```
with:
```
input=messages
```
<img width="906" height="418" alt="2nd" src="https://github.com/user-attachments/assets/f7e02d43-1797-4e00-83d2-9504710798b2" />

This transitioned response parsing to the new `response.output` format.

This change was necessary to maintain compatibility with the latest OpenAI SDK and prevent runtime parsing failures.
<br><br>


## 🧩 Step 2: Implementing Robust Output Extraction

Unlike the legacy API, the Responses API may return text across multiple content segments rather than a single unified string. To address this, a dedicated helper function `_safe_output_text()` was introduced.

This function:
- Prioritizes `response.output_text` when available
- Falls back to iterating through `response.output[]` content blocks
- Safely concatenates all `output_text` segments
- Prevents crashes if the model returns structured or multi-part outputs
This significantly improves resilience in production environments where output formats may vary between model versions.
<br><br>


## 🧩 Step 3: Refactoring Tool Calling Architecture
The legacy implementation extracted tool calls using:
```
response.choices[0].message.tool_calls
```
However, the Responses API emits tool calls as structured items within `response.output`.

To accommodate this architectural shift, a new parser `_extract_first_tool_call_args()` was implemented to:
- Traverse structured output items
- Detect `function_call` and `tool_call` types
- Extract and safely deserialize JSON arguments
- Handle nested content block variations across SDK versions

This ensures deterministic tool execution in an agentic SOC environment where function calls are required for log analytics queries.
<br><br>


## 🧩 Step 4: Removing Deprecated Parameters and Fixing SDK Errors
During migration, the previous use of:
<img width="879" height="439" alt="3rd" src="https://github.com/user-attachments/assets/a5a54a50-e625-4479-ad56-641630048712" />

```
response_format={"type": "json_object"}
```
caused a `TypeError` because this parameter is not supported in the Responses API method signature within the current SDK.

The solution involved:
- Removing the deprecated parameter
- Enforcing JSON-only outputs via prompt design
- Parsing responses explicitly using `json.loads()` after safe text extraction

Updated enforced JSON-only outputs in system prompt:
<img width="1307" height="531" alt="4th" src="https://github.com/user-attachments/assets/4697bc5d-0aeb-4fea-9fc8-bf166948f562" />

Updated `responses` API in `hunt()` definition:
<img width="1100" height="412" alt="5th" src="https://github.com/user-attachments/assets/ec57a151-ce35-462e-8289-967bc0c398ea" />


This approach maintains structured output reliability without relying on deprecated API arguments.
<br><br>


## 🧩 Step 5: Updating Query Context Tool Selection Flow

The `get_query_context()` function was updated to align with the new API by:
- Switching from `messages=` to `input=`
- Explicitly defining `tool_choice` using the structured function schema
- Replacing legacy tool call extraction with the new parser
This guarantees that the model consistently returns the required function arguments for KQL query construction while operating within the updated agentic framework.







## ✅ Project Conclusion
This migration was not a simple syntax update, but a full architectural refactor to accommodate the transition from the deprecated Chat Completions API to the modern Responses API.

By redesigning response parsing, tool call extraction, and request formatting, the system now operates reliably under the new SDK while preserving all existing SOC automation capabilities, including:
- Threat hunting logic
- Guardrail enforcement
- Log Analytics querying
- Defender for Endpoint integrations
This enhancement demonstrates practical experience with API lifecycle management, agentic AI architecture, structured tool calling, and production-grade error handling.

Most importantly, the refactor future-proofs the application against further SDK deprecations while maintaining deterministic, high-reliability behavior required in real-world Security Operations Center (SOC) automation workflows.
