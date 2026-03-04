---
name: xai-sdk
description:
  Use this to develop Python code using X.ai SDK (Grok LLM). It provides details on the proper use of modern X.ai API.
---

# xAI API Integration Guide (Grok-4)

This guide provides technical specifications and implementation patterns for the xAI API, specifically optimized for Grok-4 and Grok-3 models.

## 1. Model Matrix & Pricing

| Model | Context | Rate Limits | Input/Output Price (per 1M tokens) |
| :--- | :--- | :--- | :--- |
| **grok-4-1-fast-reasoning** | 2,000,000 | 4M TPM / 480 RPM | $0.20 / $0.50 |
| **grok-4-1-fast-non-reasoning** | 2,000,000 | 4M TPM / 480 RPM | $0.20 / $0.50 |
| **grok-code-fast-1** | 256,000 | 2M TPM / 480 RPM | $0.20 / $1.50 |

### Model Behavior Notes
* **Grok-4 Reasoning:** Strictly a reasoning model. Does not support `presence_penalty`, `frequency_penalty`, `stop`, or `reasoning_effort` parameters. Including these results in an error.
* **Knowledge Cut-off:** November 2024 for both Grok-3 and Grok-4. Use Search Tools for real-time data.

---

## 2. Server-Side Tools & Agentic Workflows

When using tools, the model operates in an agentic mode, autonomously deciding how many times to invoke a tool.

### Tool Pricing
| Tool | Cost | Description |
| :--- | :--- | :--- |
| **Web Search** | $5 / 1k calls | Search and browse the live web. |
| **X Search** | $5 / 1k calls | Access X posts, profiles, and threads. |
| **Code Execution** | $5 / 1k calls | Sandbox Python environment for data/math. |
| **File Attachments** | $10 / 1k calls | Search/reason over uploaded documents. |
| **Collections** | $2.50 / 1k calls | Persistent RAG storage. |

### Implementation: Agentic Web & X Search
Use the `tools` parameter to enable autonomous research capabilities.

```python
import os
from xai_sdk import Client
from xai_sdk.tools import web_search, x_search

client = Client(api_key=os.getenv("XAI_API_KEY"))

chat = client.chat.create(
    model="grok-4-1-fast",
    tools=[
        web_search(allowed_domains=["wikipedia.org", "arxiv.org"]),
        x_search(enable_video_understanding=True)
    ],
    include=["inline_citations"] # Embeds [[1]](url) in response
)

chat.append({"role": "user", "content": "What are the latest performance benchmarks for HFT kernels?"})
response = chat.sample()

print(f"Response: {response.content}")
print(f"Citations: {response.citations}")

```

---

## 3. Real-Time Search API (Legacy Migration)

The `Live Search API` is being deprecated (Jan 12, 2026). Use the `search_parameters` field within the Chat Completion endpoint for the new "Agentic Search" behavior.

### Live Search Configuration

* **Cost:** $25 per 1,000 sources used ($0.025/source).
* **Modes:** `auto` (default), `on`, `off`.

```python
from xai_sdk.search import SearchParameters, web_source

# Restricting search to a specific region and date range
params = SearchParameters(
    mode="auto",
    from_date="2025-01-01",
    to_date="2025-12-31",
    sources=[web_source(country="US", excluded_websites=["clickbait.com"])]
)

chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    search_parameters=params
)

```

---

## 4. Document Processing (Files API)

The Files API enables agentic reasoning over local documents.

* **Limit:** 48 MB per file.
* **Logic:** Attaching a file implicitly activates the `document_search` tool.

```python
# 1. Upload
with open("strategy.pdf", "rb") as f:
    uploaded_file = client.files.upload(f.read(), filename="strategy.pdf")

# 2. Query
chat = client.chat.create(model="grok-4-fast")
chat.append({
    "role": "user", 
    "content": "Summarize the risk section", 
    "file_ids": [uploaded_file.id]
})

response = chat.sample()
# 3. Cleanup
client.files.delete(uploaded_file.id)

```

---

## 5. Specialized APIs

### Voice Agent API

Real-time conversational interface billed at a flat rate.

* **Price:** $0.05 / minute ($3.00 / hour).
* **Concurrency:** 100 sessions per team.
* **Tools:** Supports function calling, web search, and X search (tool invocation fees apply additionally).

### Batch API

For high-volume, non-latency-sensitive processing.

* **Price:** 50% discount on all token types.
* **Latency:** Typically completed within 24 hours.
* **Constraint:** Text/Language models only; no image/video generation.

---

## 6. Prompt Caching

Caching is **automatically enabled** for all requests. Reusing identical prompt prefixes or large documents reduces cost.

* Check `response.usage.prompt_tokens_details.cached_tokens` to verify savings.

---

## 7. Technical Guardrails

* **Max Image Size:** 20 MiB (JPG/PNG).
* **Violation Fee:** $0.05 per request for violations caught before generation.
* **Role Order:** No restriction. `system`, `user`, and `assistant` can be mixed in any sequence.
* **Timeout:** For reasoning models, it is recommended to set `timeout=3600` in the client configuration to handle extended thinking periods.


---

## Generate Text

The Responses API is the preferred way of interacting with our models via API. It allows optional **stateful interactions** with our models, where **previous input prompts, reasoning content and model responses are saved and stored on xAI's servers**. You can continue the interaction by appending new prompt messages instead of resending the full conversation. This behavior is on by default. If you would like to store your request/response locally, please see [Disable storing previous request/response on server](https://docs.x.ai/developers/model-capabilities/text/#disable-storing-previous-requestresponse-on-server).

Although you don't need to enter the conversation history in the request body, you will still be billed for the entire conversation history when using Responses API. The cost might be reduced as part of the conversation history is [automatically cached](https://docs.x.ai/developers/models#cached-prompt-tokens).

**The responses will be stored for 30 days, after which they will be removed. This means you can use the response ID to retrieve or continue a conversation within 30 days of sending the request.**If you want to continue a conversation after 30 days, please store your responses history and the encrypted thinking content locally, and pass them in a new request body.

For Python, we also offer our [xAI SDK](https://github.com/xai-org/xai-sdk-python) which covers all of our features and uses gRPC for optimal performance. It's fine to mix both. The xAI SDK allows you to interact with all our products such as Collections, Voice API, API key management, and more, while the Responses API is more suited for chatbots and usage in RESTful APIs.

---

## Creating a new model response

The first step in using Responses API is analogous to using the legacy Chat Completions API. You will create a new response with prompts. By default, your request/response history is stored on our server.

`instructions` parameter is currently not supported. The API will return an error if it is specified.

```python
import os
from xai_sdk import Client
from xai_sdk.chat import user, system

client = Client(
    api_key=os.getenv("XAI_API_KEY"),
    management_api_key=os.getenv("XAI_MANAGEMENT_API_KEY"),
    timeout=3600,
)

chat = client.chat.create(model="grok-4-1-fast-reasoning")
chat.append(system("You are Grok, a chatbot inspired by the Hitchhiker's Guide to the Galaxy."))
chat.append(user("What is the meaning of life, the universe, and everything?"))

response = chat.sample()
print(response)

# The response ID that can be used to continue the conversation later
print(response.id)
```

The `developer` role is supported as an alias for `system`. Only a **single** system/developer message should be used, and it should always be the **first message** in your conversation.

### Disable storing previous request/response on server

If you do not want to store your previous request/response on the server, you can set `store: false` on the request.

```python
import os
from xai_sdk import Client
from xai_sdk.chat import user, system

client = Client(
    api_key=os.getenv("XAI_API_KEY"),
    management_api_key=os.getenv("XAI_MANAGEMENT_API_KEY"),
    timeout=3600,
)

chat = client.chat.create(model="grok-4-1-fast-reasoning", store_messages=False)
chat.append(system("You are Grok, a chatbot inspired by the Hitchhiker's Guide to the Galaxy."))
chat.append(user("What is the meaning of life, the universe, and everything?"))
response = chat.sample()
print(response)
```

### Returning encrypted thinking content

If you want to return the encrypted thinking traces, you need to specify `use_encrypted_content=True` in xAI SDK or gRPC request message, or `include: ["reasoning.encrypted_content"]` in the request body.

Modify the steps to create a chat client (xAI SDK) or change the request body as following:

```python
chat = client.chat.create(model="grok-4-1-fast-reasoning",

        use_encrypted_content=True)
```

See [Adding encrypted thinking content](https://docs.x.ai/developers/model-capabilities/text/#adding-encrypted-thinking-content) on how to use the returned encrypted thinking content when making a new request.

---

## Chaining the conversation

We now have the `id` of the first response. With Chat Completions API, we typically send a stateless new request with all the previous messages.

With Responses API, we can send the `id` of the previous response, and the new messages to append to it.

```python
import os
from xai_sdk import Client
from xai_sdk.chat import user, system

client = Client(
    api_key=os.getenv("XAI_API_KEY"),
    management_api_key=os.getenv("XAI_MANAGEMENT_API_KEY"),
    timeout=3600,
)

chat = client.chat.create(model="grok-4-1-fast-reasoning", store_messages=True)
chat.append(system("You are Grok, a chatbot inspired by the Hitchhiker's Guide to the Galaxy."))
chat.append(user("What is the meaning of life, the universe, and everything?"))

response = chat.sample()
print(response)

# The response ID that can be used to continue the conversation later
print(response.id)

# New steps
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    previous_response_id=response.id,
    store_messages=True,
)

chat.append(user("What is the meaning of 42?"))
second_response = chat.sample()

print(second_response)

# The response ID that can be used to continue the conversation later
print(second_response.id)
```

### Adding encrypted thinking content

After returning the encrypted thinking content, you can also add it to a new response's input:

```python
import os
from xai_sdk import Client
from xai_sdk.chat import user, system

client = Client(
    api_key=os.getenv("XAI_API_KEY"),
    management_api_key=os.getenv("XAI_MANAGEMENT_API_KEY"),
    timeout=3600,
)

chat = client.chat.create(model="grok-4-1-fast-reasoning", store_messages=True, use_encrypted_content=True)
chat.append(system("You are Grok, a chatbot inspired by the Hitchhiker's Guide to the Galaxy."))
chat.append(user("What is the meaning of life, the universe, and everything?"))

response = chat.sample()
print(response)

# The response ID that can be used to continue the conversation later
print(response.id)

# New steps
chat.append(response)  ## Append the response and the SDK will automatically add the outputs from response to message history
chat.append(user("What is the meaning of 42?"))

second_response = chat.sample()
print(second_response)

# The response ID that can be used to continue the conversation later
print(second_response.id)
```

---

## Retrieving a previous model response

If you have a previous response's ID, you can retrieve the content of the response.

```python
import os
from xai_sdk import Client
from xai_sdk.chat import user, system

client = Client(
    api_key=os.getenv("XAI_API_KEY"),
    management_api_key=os.getenv("XAI_MANAGEMENT_API_KEY"),
    timeout=3600,
)

response = client.chat.get_stored_completion("<The previous response's id>")
print(response)
```

---

## Delete a model response

If you no longer want to store the previous model response, you can delete it.

```python
import os
from xai_sdk import Client
from xai_sdk.chat import user, system
client = Client(
    api_key=os.getenv("XAI_API_KEY"),
    management_api_key=os.getenv("XAI_MANAGEMENT_API_KEY"),
    timeout=3600,
)
response = client.chat.delete_stored_completion("<The previous response's id>")
print(response)
```



## Reasoning

`presencePenalty`, `frequencyPenalty` and `stop` parameters are not supported by reasoning models. Adding them in the request would result in an error.

---

## Key Features

- **Think Before Responding**: Thinks through problems step-by-step before delivering an answer.
- **Math & Quantitative Strength**: Excels at numerical challenges and logic puzzles.
- **Reasoning Trace**: Usage metrics expose `reasoning_tokens`. Some models can also return encrypted reasoning via `include: ["reasoning.encrypted_content"]` (see below).

In Chat Completions, only `grok-3-mini` returns `message.reasoning_content`.

  

`grok-3`, `grok-4` and `grok-4-fast-reasoning` do not return `reasoning_content`. If supported, you can request [encrypted reasoning content](https://docs.x.ai/developers/model-capabilities/text/#encrypted-reasoning-content) via `include: ["reasoning.encrypted_content"]` in the Responses API instead.

---

### Encrypted Reasoning Content

For `grok-4`, the reasoning content is encrypted by us and can be returned if you pass `include: ["reasoning.encrypted_content"]` to the Responses API. You can send the encrypted content back to provide more context to a previous conversation. See [Adding encrypted thinking content](https://docs.x.ai/developers/model-capabilities/text/generate-text#adding-encrypted-thinking-content) for more details on how to use the content.

---


## Notes on Consumption

When you use a reasoning model, the reasoning tokens are also added to your final consumption amount. The reasoning token consumption will likely increase when you use a higher `reasoning_effort` setting.


## Structured Outputs

Structured Outputs is a feature that lets the API return responses in a specific, organized format, like JSON or other schemas you define. Instead of getting free-form text, you receive data that's consistent and easy to parse.

Ideal for tasks like document parsing, entity extraction, or report generation, it lets you define schemas using tools like [Pydantic](https://pydantic.dev/) or [Zod](https://zod.dev/) to enforce data types, constraints, and structure.

When using structured outputs, the LLM's response is **guaranteed** to match your input schema.

---

## Supported models

Structured outputs is supported by all language models.

---

## Supported schemas

For structured output, the following types are supported for structured output:

- string
	- `minLength` and `maxLength` properties are not supported
- number
	- integer
	- float
- object
- array
	- `minItems` and `maxItem` properties are not supported
	- `maxContains` and `minContains` properties are not supported
- boolean
- enum
- anyOf

`allOf` is not supported at the moment.

---

## Example: Invoice Parsing

A common use case for Structured Outputs is parsing raw documents. For example, invoices contain structured data like vendor details, amounts, and dates, but extracting this data from raw text can be error-prone. Structured Outputs ensure the extracted data matches a predefined schema.

Let's say you want to extract the following data from an invoice:

- Vendor name and address
- Invoice number and date
- Line items (description, quantity, price)
- Total amount and currency

Use the structured outputs feature of the the SDK to parse the invoice.

```python
import os
from datetime import date
from enum import Enum
from pydantic import BaseModel, Field
from xai_sdk import Client
from xai_sdk.chat import system, user

# Pydantic Schemas
class Currency(str, Enum):
    USD = "USD"
    EUR = "EUR"
    GBP = "GBP"

class LineItem(BaseModel):
    description: str = Field(description="Description of the item or service")
    quantity: int = Field(description="Number of units", ge=1)
    unit_price: float = Field(description="Price per unit", ge=0)

class Address(BaseModel):
    street: str = Field(description="Street address")
    city: str = Field(description="City")
    postal_code: str = Field(description="Postal/ZIP code")
    country: str = Field(description="Country")

class Invoice(BaseModel):
    vendor_name: str = Field(description="Name of the vendor")
    vendor_address: Address = Field(description="Vendor's address")
    invoice_number: str = Field(description="Unique invoice identifier")
    invoice_date: date = Field(description="Date the invoice was issued")
    line_items: list[LineItem] = Field(description="List of purchased items/services")
    total_amount: float = Field(description="Total amount due", ge=0)
    currency: Currency = Field(description="Currency of the invoice")

client = Client(api_key=os.getenv("XAI_API_KEY"))
chat = client.chat.create(model="grok-4-1-fast-reasoning")
chat.append(system("Given a raw invoice, carefully analyze the text and extract the invoice data into JSON format."))
chat.append(
user("""
Vendor: Acme Corp, 123 Main St, Springfield, IL 62704
Invoice Number: INV-2025-001
Date: 2025-02-10
Items: - Widget A, 5 units, $10.00 each - Widget B, 2 units, $15.00 each
Total: $80.00 USD
""")
)

# The parse method returns a tuple of the full response object as well as the parsed pydantic object.
response, invoice = chat.parse(Invoice)
assert isinstance(invoice, Invoice)
# Can access fields of the parsed invoice object directly

print(invoice.vendor_name)
print(invoice.invoice_number)
print(invoice.invoice_date)
print(invoice.line_items)
print(invoice.total_amount)
print(invoice.currency)

# Can also access fields from the raw response object such as the content.
# In this case, the content is the JSON schema representation of the parsed invoice object
print(response.content)
```

---

### Step 4: Type-safe Output

The output will **always** be type-safe and respect the input schema.

```json
{
  "vendor_name": "Acme Corp",
  "vendor_address": {
    "street": "123 Main St",
    "city": "Springfield",
    "postal_code": "62704",
    "country": "IL"
  },

  "invoice_number": "INV-2025-001",
  "invoice_date": "2025-02-10",
  "line_items": [
    { "description": "Widget A", "quantity": 5, "unit_price": 10.0 },
    { "description": "Widget B", "quantity": 2, "unit_price": 15.0 }
  ],
  "total_amount": 80.0,
  "currency": "USD"
}
```

---

## Structured Outputs with Tools

Structured outputs with tools is only available for the Grok 4 family of models (e.g., `grok-4-1-fast`, `grok-4-fast`, `grok-4-1-fast-non-reasoning`, `grok-4-fast-non-reasoning`).

You can combine structured outputs with tool calling to get type-safe responses from tool-augmented queries. This works with both:

- **[Agentic tool calling](https://docs.x.ai/developers/tools/overview)**: Server-side tools like web search, X search, and code execution that the model orchestrates autonomously.
- **[Function calling](https://docs.x.ai/developers/tools/function-calling)**: User-supplied tools where you define custom functions and handle tool execution yourself.

This combination enables workflows where the model can use tools to gather information and return results in a predictable, strongly-typed format.

### Example: Agentic Tools with Structured Output

This example uses web search to find the latest research on a topic and extracts structured data into a schema:

```python
from pydantic import BaseModel, Field

class ProofInfo(BaseModel):
    name: str = Field(description="Name of the proof or paper")
    authors: str = Field(description="Authors of the proof")
    year: str = Field(description="Year published")
    summary: str = Field(description="Brief summary of the approach")
```

```python
import os

from pydantic import BaseModel, Field
from xai_sdk import Client
from xai_sdk.chat import user
from xai_sdk.tools import web_search

# ProofInfo schema defined above
client = Client(api_key=os.getenv("XAI_API_KEY"))
chat = client.chat.create(
    model="grok-4-1-fast",
    tools=[web_search()],
)

chat.append(user("Find the latest machine-checked proof of the four color theorem."))
response, proof = chat.parse(ProofInfo)
print(f"Name: {proof.name}")
print(f"Authors: {proof.authors}")
print(f"Year: {proof.year}")
print(f"Summary: {proof.summary}")
```

### Example: Client-side Tools with Structured Output

This example uses a client-side function tool to compute Collatz sequence steps and returns the result in a structured format:

```python
from pydantic import BaseModel, Field
class CollatzResult(BaseModel):
    starting_number: int = Field(description="The input number")
    steps: int = Field(description="Number of steps to reach 1")
```

```python
import os
import json
from pydantic import BaseModel, Field
from xai_sdk import Client
from xai_sdk.chat import tool, tool_result, user

# CollatzResult schema defined above
def collatz_steps(n: int) -> int:
    """Returns the number of steps for n to reach 1 in the Collatz sequence."""
    steps = 0
    while n != 1:
        n = n // 2 if n % 2 == 0 else 3 * n + 1
        steps += 1
    return steps

collatz_tool = tool(
    name="collatz_steps",
    description="Compute the number of steps for a number to reach 1 in the Collatz sequence",
    parameters={
        "type": "object",
        "properties": {
            "n": {"type": "integer", "description": "The starting number"},
        },
        "required": ["n"],
    },
)

client = Client(api_key=os.getenv("XAI_API_KEY"))
chat = client.chat.create(
    model="grok-4-1-fast-non-reasoning",
    tools=[collatz_tool],
)

chat.append(user("Use the collatz_steps tool to find how many steps it takes for 20250709 to reach 1."))
# Handle tool calls until we get a final response

while True:
    response = chat.sample()

    if not response.tool_calls:
        break

    chat.append(response)
    for tc in response.tool_calls:
        args = json.loads(tc.function.arguments)
        result = collatz_steps(args["n"])
        chat.append(tool_result(str(result)))

# Parse the final response into structured output
response, result = chat.parse(CollatzResult)
print(f"Starting number: {result.starting_number}")
print(f"Steps to reach 1: {result.steps}")
```

---

## Alternative: Using response\_format with sample() or stream()

When using the xAI Python SDK, there's an alternative way to retrieve structured outputs. Instead of using the `parse()` method, you can pass your Pydantic model directly to the `response_format` parameter when creating a chat, and then use `sample()` or `stream()` to get the response.

### How It Works

When you pass a Pydantic model to `response_format`, the SDK automatically:

1. Converts your Pydantic model to a JSON schema
2. Constrains the model's output to conform to that schema
3. Returns the response as a JSON string, that is conforming to the Pydantic model, in `response.content`

You then manually parse the JSON string into your Pydantic model instance.

### Key Differences

| Approach | Method | Returns | Parsing |
| --- | --- | --- | --- |
| **Using `parse()`** | `chat.parse(Model)` | Tuple of `(Response, Model)` | Automatic - SDK parses for you |
| **Using `response_format`** | `chat.sample()` or `chat.stream()` | `Response` with JSON string | Manual - You parse `response.content` |

### When to Use Each Approach

- **Use `parse()`** when you want the simplest, most convenient experience with automatic parsing
- **Use `response_format` + `sample()` or `stream()`** when you:
	- Want more control over the parsing process
	- Need to handle the raw JSON string before parsing
	- Want to use streaming with structured outputs
	- Are integrating with existing code that expects to work with `sample()` or `stream()`

### Example Using response\_format

Python

```python
import os
from datetime import date
from enum import Enum
from pydantic import BaseModel, Field
from xai_sdk import Client
from xai_sdk.chat import system, user

# Pydantic Schemas
class Currency(str, Enum):
    USD = "USD"
    EUR = "EUR"
    GBP = "GBP"

class LineItem(BaseModel):
    description: str = Field(description="Description of the item or service")
    quantity: int = Field(description="Number of units", ge=1)
    unit_price: float = Field(description="Price per unit", ge=0)

class Address(BaseModel):
    street: str = Field(description="Street address")
    city: str = Field(description="City")
    postal_code: str = Field(description="Postal/ZIP code")
    country: str = Field(description="Country")

class Invoice(BaseModel):
    vendor_name: str = Field(description="Name of the vendor")
    vendor_address: Address = Field(description="Vendor's address")
    invoice_number: str = Field(description="Unique invoice identifier")
    invoice_date: date = Field(description="Date the invoice was issued")
    line_items: list[LineItem] = Field(description="List of purchased items/services")
    total_amount: float = Field(description="Total amount due", ge=0)
    currency: Currency = Field(description="Currency of the invoice")

client = Client(api_key=os.getenv("XAI_API_KEY"))

# Pass the Pydantic model to response_format instead of using parse()
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    response_format=Invoice,  # Pass the Pydantic model here
)

chat.append(system("Given a raw invoice, carefully analyze the text and extract the invoice data into JSON format."))
chat.append(
    user("""
Vendor: Acme Corp, 123 Main St, Springfield, IL 62704
Invoice Number: INV-2025-001
Date: 2025-02-10
Items: - Widget A, 5 units, $10.00 each - Widget B, 2 units, $15.00 each
Total: $80.00 USD
""")
)

# Use sample() instead of parse() - returns Response object
response = chat.sample()

# The response.content is a valid JSON string conforming to your schema
print(response.content)

# Output: {"vendor_name": "Acme Corp", "vendor_address": {...}, ...}
# Manually parse the JSON string into your Pydantic model
invoice = Invoice.model_validate_json(response.content)
assert isinstance(invoice, Invoice)

# Access fields of the parsed invoice object
print(invoice.vendor_name)
print(invoice.invoice_number)
print(invoice.total_amount)
```

### Streaming with Structured Outputs

You can also use `stream()` with `response_format` to get streaming structured output. The chunks will progressively build up the JSON string:


```python
import os
from pydantic import BaseModel, Field
from xai_sdk import Client
from xai_sdk.chat import system, user

class Summary(BaseModel):
    title: str = Field(description="A brief title")
    key_points: list[str] = Field(description="Main points from the text")
    sentiment: str = Field(description="Overall sentiment: positive, negative, or neutral")

client = Client(api_key=os.getenv("XAI_API_KEY"))
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    response_format=Summary,  # Pass the Pydantic model here
)

chat.append(system("Analyze the following text and provide a structured summary."))
chat.append(user("The new product launch exceeded expectations with record sales..."))

# Stream the response - chunks contain partial JSON

for response, chunk in chat.stream():
    print(chunk.content, end="", flush=True)

# Parse the complete JSON string into your model
summary = Summary.model_validate_json(response.content)
print(f"Title: {summary.title}")
print(f"Sentiment: {summary.sentiment}")
```

---

# Web Search tool

The Web Search tool enables Grok to search the web in real-time and browse web pages to find information. This powerful tool allows the model to search the internet, access web pages, and extract relevant information to answer queries with up-to-date content.

---

## SDK Support

This tool is also supported in all Responses API compatible SDKs.

---

## Basic Usage

```python
import os
from xai_sdk import Client
from xai_sdk.chat import user
from xai_sdk.tools import web_search
client = Client(api_key=os.getenv("XAI_API_KEY"))

chat = client.chat.create(
    model="grok-4-1-fast-reasoning",  # reasoning model
    tools=[web_search()],
    include=["verbose_streaming"],
)

chat.append(user("What is xAI?"))

is_thinking = True
for response, chunk in chat.stream():
    for tool_call in chunk.tool_calls:
        print(f"\nCalling tool: {tool_call.function.name} with arguments: {tool_call.function.arguments}")
    if response.usage.reasoning_tokens and is_thinking:
        print(f"\rThinking... ({response.usage.reasoning_tokens} tokens)", end="", flush=True)
    if chunk.content and is_thinking:
        print("\n\nFinal Response:")
        is_thinking = False
    if chunk.content and not is_thinking:
        print(chunk.content, end="", flush=True)
print("\n\nCitations:")
print(response.citations)
```

---

| Parameter | Description |
| --- | --- |
| `allowed_domains` | Only search within specific domains (max 5) |
| `excluded_domains` | Exclude specific domains from search (max 5) |
| `enable_image_understanding` | Enable analysis of images found during browsing |

Use `allowed_domains` to make the web search **only** perform the search and web browsing on web pages that fall within the specified domains.

`allowed_domains` cannot be set together with `excluded_domains` in the same request.

### Exclude Specific Domains

Use `excluded_domains` to prevent the model from including the specified domains in any web search tool invocations.

### Enable Image Understanding

Setting `enable_image_understanding` to true equips the agent with access to the `view_image` tool, allowing it to analyze images encountered during the search process.

When enabled, you will see `SERVER_SIDE_TOOL_VIEW_IMAGE` in `response.server_side_tool_usage` along with the number of times it was called.

Enabling this parameter for Web Search will also enable the image understanding for X Search tool if it's also included in the request.

---

## Citations

For details on how to retrieve and use citations from search results, see the [Citations](https://docs.x.ai/developers/tools/citations) page.

# X Search

The X Search tool enables Grok to perform keyword search, semantic search, user search, and thread fetch on X (formerly Twitter). This powerful tool allows the model to access real-time social media content, analyze posts, and gather insights from X's vast data.

---

## SDK Support

This tool is also supported in all Responses API compatible SDKs.

---

## Basic Usage

```python
import os
from xai_sdk import Client
from xai_sdk.chat import user
from xai_sdk.tools import x_search

client = Client(api_key=os.getenv("XAI_API_KEY"))

chat = client.chat.create(
    model="grok-4-1-fast-reasoning",  # reasoning model
    tools=[x_search()],
    include=["verbose_streaming"],
)

chat.append(user("What are people saying about xAI on X?"))
is_thinking = True

for response, chunk in chat.stream():

    for tool_call in chunk.tool_calls:
        print(f"\nCalling tool: {tool_call.function.name} with arguments: {tool_call.function.arguments}")

    if response.usage.reasoning_tokens and is_thinking:
        print(f"\rThinking... ({response.usage.reasoning_tokens} tokens)", end="", flush=True)

    if chunk.content and is_thinking:
        print("\n\nFinal Response:")
        is_thinking = False

    if chunk.content and not is_thinking:
        print(chunk.content, end="", flush=True)

print("\n\nCitations:")
print(response.citations)
```

---

| Parameter | Description |
| --- | --- |
| `allowed_x_handles` | Only consider posts from specific X handles (max 10) |
| `excluded_x_handles` | Exclude posts from specific X handles (max 10) |
| `from_date` | Start date for search range (ISO8601 format) |
| `to_date` | End date for search range (ISO8601 format) |
| `enable_image_understanding` | Enable analysis of images in posts |
| `enable_video_understanding` | Enable analysis of videos in posts |

### Only Consider Posts from Specific Handles

Use `allowed_x_handles` to consider X posts only from a given list of X handles. The maximum number of handles you can include is 10.

`allowed_x_handles` cannot be set together with `excluded_x_handles` in the same request.

### Exclude Posts from Specific Handles

Use `excluded_x_handles` to prevent the model from including X posts from the specified handles in any X search tool invocations. The maximum number of handles you can exclude is 10.

### Date Range

You can restrict the date range of search data used by specifying `from_date` and `to_date`. This limits the data to the period from `from_date` to `to_date`, including both dates.

Both fields need to be in ISO8601 format, e.g., "YYYY-MM-DD". If you're using the xAI Python SDK, the `from_date` and `to_date` fields can be passed as `datetime.datetime` objects.

### Enable Image Understanding

Setting `enable_image_understanding` to true allows the agent to analyze images in X posts encountered during the search process.

### Enable Video Understanding

Setting `enable_video_understanding` to true allows the agent to analyze videos in X posts. This is only available for X Search (not Web Search).


## Tool Usage Details

This page covers the technical details of how tool calls are tracked, billed, and how to understand token usage in agentic requests.

---

## Real-time Server-side Tool Calls

When streaming agentic requests, you can observe **every tool call decision** the model makes in real-time via the `tool_calls` attribute on the `chunk` object:

Python

```
for tool_call in chunk.tool_calls:

    print(f"\nCalling tool: {tool_call.function.name} with arguments: {tool_call.function.arguments}")
```

**Note**: Only the tool call invocations are shown — **server-side tool call outputs are not returned** in the API response. The agent uses these outputs internally to formulate its final response.

---

## Server-side Tool Calls vs Tool Usage

The API provides two related but distinct metrics for server-side tool executions:

### tool\_calls - All Attempted Calls

Python

```
response.tool_calls
```

Returns a list of all **attempted** tool calls made during the agentic process. Each entry contains:

- `id`: Unique identifier for the tool call
- `function.name`: The name of the specific server-side tool called
- `function.arguments`: The parameters passed to the server-side tool

This includes **every tool call attempt**, even if some fail.

### server\_side\_tool\_usage - Successful Calls (Billable)

Python

```
response.server_side_tool_usage
```

Returns a map of successfully executed tools and their invocation counts. This represents only the tool calls that returned meaningful responses and **determines your billing**.

Output

```output
{'SERVER_SIDE_TOOL_X_SEARCH': 3, 'SERVER_SIDE_TOOL_WEB_SEARCH': 2}
```

---

## Tool Call Function Names vs Usage Categories

The function names in `tool_calls` represent the precise name of the tool invoked, while the entries in `server_side_tool_usage` provide a high-level categorization that aligns with the original tool passed in the `tools` array.

| Usage Category | Function Name(s) |
| --- | --- |
| `SERVER_SIDE_TOOL_WEB_SEARCH` | `web_search`, `web_search_with_snippets`, `browse_page` |
| `SERVER_SIDE_TOOL_X_SEARCH` | `x_user_search`, `x_keyword_search`, `x_semantic_search`, `x_thread_fetch` |
| `SERVER_SIDE_TOOL_CODE_EXECUTION` | `code_execution` |
| `SERVER_SIDE_TOOL_VIEW_X_VIDEO` | `view_x_video` |
| `SERVER_SIDE_TOOL_VIEW_IMAGE` | `view_image` |
| `SERVER_SIDE_TOOL_COLLECTIONS_SEARCH` | `collections_search` |
| `SERVER_SIDE_TOOL_MCP` | `{server_label}.{tool_name}` if `server_label` provided, otherwise `{tool_name}` |

---

## When Tool Calls and Usage Differ

In most cases, `tool_calls` and `server_side_tool_usage` will show the same tools. However, they can differ when:

- **Failed tool executions**: The model attempts to browse a non-existent webpage, fetch a deleted X post, or encounters other execution errors
- **Invalid parameters**: Tool calls with malformed arguments that can't be processed
- **Network or service issues**: Temporary failures in the tool execution pipeline

The agentic system handles these failures gracefully, updating its trajectory and continuing with alternative approaches when needed.

**Billing Note**: Only successful tool executions (`server_side_tool_usage`) are billed. Failed attempts are not charged.

---

## Understanding Token Usage

Agentic requests have unique token usage patterns compared to standard chat completions:

### completion\_tokens

Represents **only the final text output** of the model. This is typically much smaller than you might expect, as the agent performs all its intermediate reasoning and tool orchestration internally.

### prompt\_tokens

Represents the **cumulative input tokens** across all inference requests made during the agentic process. Each request includes the full conversation history up to that point, which grows as the agent progresses.

While this can result in higher `prompt_tokens` counts, agentic requests benefit significantly from **prompt caching**. The majority of the prompt remains unchanged between steps, allowing for efficient caching.

### reasoning\_tokens

Represents the tokens used for the model's internal reasoning process. This includes planning tool calls, analyzing results, and formulating responses, but excludes the final output tokens.

### cached\_prompt\_text\_tokens

Indicates how many prompt tokens were served from cache rather than recomputed. Higher values indicate better cache utilization and lower costs.

### prompt\_image\_tokens

Represents tokens from visual content that the agent processes. These are counted separately from text tokens. If no images or videos are processed, this value will be zero.

---

## Limiting Tool Call Turns

The `max_turns` parameter allows you to control the maximum number of assistant/tool-call turns the agent can perform during a single request.

### Understanding Turns vs Tool Calls

**Important**: `max_turns` does **not** directly limit the number of individual tool calls. Instead, it limits the number of *assistant turns* in the agentic loop. During a single turn, the model may invoke multiple tools in parallel.

A "turn" represents one iteration of the agentic reasoning loop:

1. The model analyzes the current context
2. The model decides to call one or more tools (potentially in parallel)
3. Tools execute and return results
4. The model processes the results

```python
import os
from xai_sdk import Client
from xai_sdk.chat import user
from xai_sdk.tools import web_search, x_search
client = Client(api_key=os.getenv("XAI_API_KEY"))

chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[
        web_search(),
        x_search(),
    ],
    max_turns=3,  # Limit to 3 assistant/tool-call turns
)

chat.append(user("What is the latest news from xAI?"))
response = chat.sample()
print(response.content)
```

### When to Use max\_turns

### Default Behavior

If `max_turns` is not specified, the server applies a global default cap. When the agent reaches the limit, it will stop making additional tool calls and generate a final response based on information gathered so far.

---

## Identifying Tool Call Types

To determine whether a returned tool call is a client-side tool that needs local execution:

### Using xAI SDK

Use the `get_tool_call_type` function:

```python
from xai_sdk.tools import get_tool_call_type

for tool_call in response.tool_calls:
    print(get_tool_call_type(tool_call))
```

| Tool call types | Description |
| --- | --- |
| `"client_side_tool"` | Client-side tool call - requires local execution |
| `"web_search_tool"` | Web-search tool - handled by xAI server |
| `"x_search_tool"` | X-search tool - handled by xAI server |
| `"code_execution_tool"` | Code-execution tool - handled by xAI server |
| `"collections_search_tool"` | Collections-search tool - handled by xAI server |
| `"mcp_tool"` | MCP tool - handled by xAI server |

### Using Responses API

Check the `type` field of output entries (`response.output[].type`):

| Types | Description |
| --- | --- |
| `"function_call"` | Client-side tool - requires local execution |
| `"web_search_call"` | Web-search tool - handled by xAI server |
| `"x_search_call"` | X-search tool - handled by xAI server |
| `"code_interpreter_call"` | Code-execution tool - handled by xAI server |
| `"file_search_call"` | Collections-search tool - handled by xAI server |
| `"mcp_call"` | MCP tool - handled by xAI server |



## Function Calling

Define custom tools that the model can invoke during a conversation. The model requests the call, you execute it locally, and return the result. This enables integration with databases, APIs, and any external system.

With streaming, the function call is returned in whole in a single chunk, not streamed across chunks.

---

## How It Works

![Function call request/response flow](https://docs.x.ai/_next/image?url=%2Fassets%2Fdocs%2Fdevelopers%2Ftools%2Ffunction-calling%2Ffunction-call.png&w=384&q=75)

Function call request/response flow

1. Define tools with a name, description, and JSON schema for parameters
2. Include tools in your request
3. Model returns a `tool_call` when it needs external data
4. Execute the function locally and return the result
5. Model continues with your result
![Local server handling function calls](https://docs.x.ai/_next/image?url=%2Fassets%2Fdocs%2Fdevelopers%2Ftools%2Ffunction-calling%2Flocal-server.png&w=1920&q=75)

Local server handling function calls

---

## Quick Start

```python
import os
import json
from xai_sdk import Client
from xai_sdk.chat import user, tool, tool_result
client = Client(api_key=os.getenv("XAI_API_KEY"))
# Define tools
tools = [
    tool(
        name="get_temperature",
        description="Get current temperature for a location",
        parameters={
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City name"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"], "default": "fahrenheit"}
            },
            "required": ["location"]
        },
    ),
]

chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=tools,
)

chat.append(user("What is the temperature in San Francisco?"))
response = chat.sample()

# Handle tool calls
if response.tool_calls:
    chat.append(response)
    for tc in response.tool_calls:
        args = json.loads(tc.function.arguments)
        # Execute your function
        result = {"location": args["location"], "temperature": 59, "unit": args.get("unit", "fahrenheit")}
        chat.append(tool_result(json.dumps(result)))
    response = chat.sample()
print(response.content)
```

---

## Defining Tools with Pydantic

Use Pydantic models for type-safe parameter schemas:

```python
from typing import Literal
from pydantic import BaseModel, Field
from xai_sdk.chat import tool

class TemperatureRequest(BaseModel):
    location: str = Field(description="City and state, e.g. San Francisco, CA")
    unit: Literal["celsius", "fahrenheit"] = Field("fahrenheit", description="Temperature unit")

class CeilingRequest(BaseModel):
    location: str = Field(description="City and state, e.g. San Francisco, CA")

# Generate JSON schema from Pydantic models
tools = [
    tool(
        name="get_temperature",
        description="Get current temperature for a location",
        parameters=TemperatureRequest.model_json_schema(),
    ),

    tool(
        name="get_ceiling",
        description="Get current cloud ceiling for a location",
        parameters=CeilingRequest.model_json_schema(),
    ),
]
```

---

## Handling Tool Calls

When the model wants to use your tool, execute the function and return the result:

```python
import json

def get_temperature(location: str, unit: str = "fahrenheit") -> dict:
    # In production, call a real weather API
    temp = 59 if unit == "fahrenheit" else 15
    return {"location": location, "temperature": temp, "unit": unit}

def get_ceiling(location: str) -> dict:
    return {"location": location, "ceiling": 15000, "unit": "ft"}

tools_map = {
    "get_temperature": get_temperature,
    "get_ceiling": get_ceiling,
}

chat.append(user("What's the weather in Denver?"))
response = chat.sample()

# Process tool calls
if response.tool_calls:
    chat.append(response)

    for tool_call in response.tool_calls:
        name = tool_call.function.name
        args = json.loads(tool_call.function.arguments)
        result = tools_map[name](**args)
        chat.append(tool_result(json.dumps(result)))

    response = chat.sample()
print(response.content)
```

---

## Combining with Built-in Tools

Function calling works alongside built-in agentic tools. The model can use web search, then call your custom function:

```python
from xai_sdk.chat import tool
from xai_sdk.tools import web_search, x_search

tools = [
    web_search(),                    # Built-in: runs on xAI servers
    x_search(),                      # Built-in: runs on xAI servers  
    tool(                            # Custom: runs on your side
        name="save_to_database",
        description="Save research results to the database",
        parameters={
            "type": "object",
            "properties": {
                "data": {"type": "string", "description": "Data to save"}
            },
            "required": ["data"]
        },
    ),
]

chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=tools,
)
```

When mixing tools:

- **Built-in tools** execute automatically on xAI servers
- **Custom tools** pause execution and return to you for handling

See [Advanced Usage](https://docs.x.ai/developers/tools/advanced-usage#mixing-server-side-and-client-side-tools) for complete examples with tool loops.

---

## Tool Choice

Control when the model uses tools:

| Value | Behavior |
| --- | --- |
| `"auto"` | Model decides whether to call a tool (default) |
| `"required"` | Model must call at least one tool |
| `"none"` | Disable tool calling |
| `{"type": "function", "function": {"name": "..."}}` | Force a specific tool |

---

## Parallel Function Calling

By default, parallel function calling is enabled — the model can request multiple tool calls in a single response. Process all of them before continuing:

```python
# response.tool_calls may contain multiple calls

for tool_call in response.tool_calls:
    result = tools_map[tool_call.function.name](**json.loads(tool_call.function.arguments))
    # Append each result...
```

Disable with `parallel_tool_calls: false` in your request.

---

## Tool Schema Reference

| Field | Required | Description |
| --- | --- | --- |
| `name` | Yes | Unique identifier (max 200 tools per request) |
| `description` | Yes | What the tool does — helps the model decide when to use it |
| `parameters` | Yes | JSON Schema defining function inputs |

### Parameter Schema

```json
{
  "type": "object",
  "properties": {
    "location": {
      "type": "string",
      "description": "City name"
    },
    "unit": {
      "type": "string",
      "enum": ["celsius", "fahrenheit"],
      "default": "celsius"
    }
  },
  "required": ["location"]
}
```

