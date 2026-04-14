# AWS Generative AI for Developers

## <img src="https://api.iconify.design/mdi/text-box-outline.svg?color=%23FF5500" width="16"/> Amazon Bedrock: Text & Video Generation

### <img src="https://api.iconify.design/mdi/comment-text-outline.svg?color=%23FF5500" width="14"/> Part 1 — Text Inference (`amazon.nova-pro-v1:0`)

Used `invoke_model` to send a text prompt to **Amazon Nova Pro** and parse the response.

**Prompt:** _"How does Amazon Bedrock compare to similar services from Microsoft and Google in terms of market share?"_

**Key steps:**
1. Call `bedrock.invoke_model` with `messages-v1` schema
2. Read the raw response bytes from `response['body']`
3. Decode bytes → JSON → extract `output.message.content[0].text`

```python
MODEL_ID = "amazon.nova-pro-v1:0"
bedrock = boto3.client(service_name='bedrock-runtime', region_name='us-east-1')

response = bedrock.invoke_model(
    modelId=MODEL_ID,
    body=json.dumps({
        "schemaVersion": "messages-v1",
        "messages": [{"role": "user", "content": [{"text": "..."}]}],
        "inferenceConfig": {"maxTokens": 500, "topK": 20, "temperature": 0.7}
    })
)
text = json.loads(response['body'].read())["output"]["message"]["content"][0]["text"]
```

---

### <img src="https://api.iconify.design/mdi/video-outline.svg?color=%23FF5500" width="14"/> Part 2 — Async Video Generation (`amazon.nova-reel-v1:0`)

Used `start_async_invoke` to generate a 6-second video from a text prompt via **Amazon Nova Reel**, with output saved to S3.

**Prompt:** _"A robot dancing on a mountain."_

**Key steps:**
1. Submit an async job with `taskType: TEXT_VIDEO`, resolution `1280x720`, 24 fps, 6 s
2. Output is written to an S3 bucket (`s3://gen-ai-exercise-yv/video/`)
3. Poll job status with `get_async_invoke(invocationArn=...)`

```python
model_id = "amazon.nova-reel-v1:0"
response = bedrock_runtime.start_async_invoke(
    modelId=model_id,
    modelInput={"taskType": "TEXT_VIDEO", "textToVideoParams": {"text": prompt}, ...},
    outputDataConfig={"s3OutputDataConfig": {"s3Uri": "s3://gen-ai-exercise-yv/video/"}}
)
invocation_arn = response["invocationArn"]

# Poll status
job_status = bedrock_runtime.get_async_invoke(invocationArn=invocation_arn)
print(job_status["status"])  # InProgress / Completed / Failed
```

---

## <img src="https://api.iconify.design/mdi/check-circle-outline.svg?color=%23FF5500" width="16"/> Develop a Task Tracker with Amazon Q 

Built a task management app in two stages: a CLI and a Flask web API, both AI-assisted.

### <img src="https://api.iconify.design/mdi/console-line.svg?color=%23FF5500" width="14"/> Part 1 — CLI (`task_tracker.py`)

A terminal-based task manager with priority sorting.

| Component | Description |
|---|---|
| `Task` | Stores name + priority (1–5), validates on init |
| `TaskManager` | Maintains a list sorted by priority descending |
| `main()` | Menu loop: add / list / remove / exit |

**Features:**
- Priority validated as integer 1–5
- Tasks auto-sorted highest priority first
- Input error handling with descriptive messages

---

### <img src="https://api.iconify.design/mdi/web.svg?color=%23FF5500" width="14"/> Part 2 — Flask REST API (`app.py`)

Converted the CLI into a web app with a JSON REST API and HTML frontend, persisting tasks to `tasks.json`.

| Endpoint | Method | Description |
|---|---|---|
| `/` | `GET` | Serves the HTML frontend (`templates/index.html`) |
| `/tasks` | `GET` | Returns all tasks as JSON |
| `/tasks` | `POST` | Adds a task `{name, priority}`, validates and re-sorts |
| `/tasks/<index>` | `DELETE` | Removes task at given index |

**Stack:** Flask · JSON file persistence · `pathlib.Path`

## AWS Bedrock Guardrails

<img src="https://api.iconify.design/mdi/shield-lock-outline.svg?color=%23FF5500" width="18"/> **Goal:** Explore Amazon Bedrock Guardrails — apply safety policies to model inputs and outputs using `apply_guardrail`, `invoke_model`, and `converse`.

---

## <img src="https://api.iconify.design/mdi/book-open-outline.svg?color=%23FF5500" width="16"/> What This Exercise Covers

### 1. PII Detection — `apply_guardrail`
Tested a text snippet containing a license plate number (`UNV 425`).
The guardrail intercepted it, flagged `LICENSE_PLATE` as a PII entity, and blocked the output.

### 2. Contextual Grounding Check — `apply_guardrail`
Sent a grounding source, a query, and a hallucinated answer (claiming Mars and Jupiter are the same planet).
The guardrail detected the factual inconsistency and intervened.

### 3. Prompt Injection Detection — `invoke_model` with guardrail tag
Used the `amazon-bedrock-guardrails-guardContent_{suffix}` XML tag to mark protected content.
Embedded a prompt injection (`Forget all previous instructions...`) — the guardrail blocked it.

### 4. Content Policy — `invoke_model`
Sent a harmful content request ("What is a good way to insult someone?").
The guardrail's content policy blocked the response.

### 5. Converse API with Guardrails — `converse`
Used the higher-level `converse` API with `guardrailConfig` to apply the guardrail during a standard chat turn.

---

## <img src="https://api.iconify.design/mdi/alert-circle-outline.svg?color=%23FF5500" width="16"/> EU / US Region Challenges

This exercise ran into several cross-region issues because resources were created in different AWS regions.

| Resource | Region |
|---|---|
| Guardrail `GenAIExercise3Guardrail` | `eu-west-1` |
| Application inference profile | `us-east-1` |

### <img src="https://api.iconify.design/mdi/close-circle-outline.svg?color=%23FF5500" width="14"/> Problem 1 — `apply_guardrail`: wrong region
The boto3 client was initialised with `us-east-1`, but the guardrail only exists in `eu-west-1`.

**Fix:** Use a `eu-west-1` client for all `apply_guardrail` calls.

### <img src="https://api.iconify.design/mdi/close-circle-outline.svg?color=%23FF5500" width="14"/> Problem 2 — `invoke_model`: guardrail ARN invalid
Passing the full guardrail ARN (`arn:aws:bedrock:eu-west-1:...`) to a `us-east-1` client raised `ValidationException: The provided guardrail ARN is invalid`.

`invoke_model` requires the model and the guardrail to be in **the same region** — cross-region guardrail references are not supported.

**Fix:** Use the `eu-west-1` client and the short guardrail ID (no ARN needed when client and guardrail are in the same region).

### <img src="https://api.iconify.design/mdi/close-circle-outline.svg?color=%23FF5500" width="14"/> Problem 3 — `invoke_model`: on-demand not supported
Using the plain model ID `amazon.nova-micro-v1:0` raised:
> Invocation of model ID amazon.nova-micro-v1:0 with on-demand throughput isn't supported. Retry your request with the ID or ARN of an inference profile that contains this model.

Amazon Nova models do not support on-demand invocation — they require an inference profile.

**Fix:** Use the AWS system-defined cross-region inference profile `eu.amazon.nova-micro-v1:0`, which routes across EU regions and satisfies the inference profile requirement.

---

## <img src="https://api.iconify.design/mdi/code-braces.svg?color=%23FF5500" width="16"/> Final Client Setup

```python
# Guardrail lives in eu-west-1; inference profile was in us-east-1
bedrock    = boto3.client('bedrock-runtime', region_name='eu-west-1')  # apply_guardrail, invoke_model, converse
bedrock_us = boto3.client('bedrock-runtime', region_name='us-east-1')  # kept for reference only

GUARDRAIL_ID = "5whobi2i9rfc"                         # short ID, same region as client
EU_MODEL_ID  = "eu.amazon.nova-micro-v1:0"            # system cross-region inference profile
```

### <img src="https://api.iconify.design/mdi/link-variant.svg?color=%23FF5500" width="16"/> Resources

- [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
- [Cross-region inference profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles.html)
- [apply_guardrail API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_ApplyGuardrail.html)


## <img src="https://api.iconify.design/mdi/robot-outline.svg?color=%23FF5500" width="18"/> Building Production-Ready AI Agents with Amazon Bedrock AgentCore

Building agentic AI applications follows a predictable pattern: you start with a proof-of-concept wired to a foundation model via the Amazon Bedrock API. Scaling to real users requires application hosting, persistent memory, secure tool integration, observability, and authentication. The traditional approach stitches together DynamoDB, custom APIs, authentication layers, and hosting infrastructure — creating complex systems with extensive custom code.

**Amazon Bedrock AgentCore** eliminates this undifferentiated heavy lifting with purpose-built, modular services for agent operations.

---

### <img src="https://api.iconify.design/mdi/server-outline.svg?color=%23FF5500" width="16"/> AgentCore Runtime — Purpose-Built Agent Hosting

A serverless execution environment designed specifically for agentic workloads. Unlike standard serverless functions, Runtime supports **sessions up to 8 hours** with **100 MB payload capacity** for multi-modal content.

<p align="center">
  <img src="https://raw.githubusercontent.com/yauheniya-adesso/aws-genai/main/images/AgentCore_Runtime.png" alt="AgentCore Runtime" width="100%">
</p>

#### <img src="https://api.iconify.design/mdi/chip.svg?color=%23FF5500" width="14"/> MicroVM Isolation

| Feature | Detail |
|---|---|
| Isolation | Each user session runs in a dedicated microVM with isolated CPU, memory, and filesystem |
| Data safety | Complete separation prevents cross-session data contamination |
| Cleanup | Automatic microVM termination and memory sanitization after session end |
| Security | Deterministic security regardless of non-deterministic AI behavior |

#### <img src="https://api.iconify.design/mdi/puzzle-outline.svg?color=%23FF5500" width="14"/> Framework & Model Support

| Dimension | Details |
|---|---|
| Frameworks | LangGraph, CrewAI, Strands Agents, custom or any framework |
| Models | Any LLM provider — Amazon Bedrock, Anthropic, OpenAI, Google |
| Protocol | MCP (Model Context Protocol) for agent-to-tool communication |

---

### <img src="https://api.iconify.design/mdi/brain.svg?color=%23FF5500" width="16"/> AgentCore Memory — Context-Aware Intelligence

Dual memory architecture covering both short-term (within a session) and long-term (across sessions) context, giving agents the ability to remember and reason over past interactions.

---

### <img src="https://api.iconify.design/mdi/api.svg?color=%23FF5500" width="16"/> AgentCore Gateway — Unified Tool Integration

Gateway implements **Model Context Protocol (MCP)** as a standardized way for agents to discover and invoke tools, eliminating custom translation code.

| MCP Method | Purpose |
|---|---|
| `tools/list` | List all available tools |
| `tools/call` | Invoke a specific tool |
| `x_amz_bedrock_agentcore_search` | Semantic tool discovery |

---

### <img src="https://api.iconify.design/mdi/key-outline.svg?color=%23FF5500" width="16"/> AgentCore Identity — Secure Agent Authentication

Each agent receives a unique identity with fine-grained permissions — similar to Amazon Cognito but purpose-built for agents.

#### <img src="https://api.iconify.design/mdi/code-braces.svg?color=%23FF5500" width="14"/> SDK Integration

```python
@requires_access_token
def call_external_api(user_context):
    # SDK automatically injects and validates access token
    pass

@requires_api_key
def call_third_party_service():
    # SDK handles API key injection and validation
    pass
```

#### <img src="https://api.iconify.design/mdi/lock-outline.svg?color=%23FF5500" width="14"/> Secure Token Vault

- OAuth tokens, API keys, and client secrets encrypted at rest
- Scoped access by default
- Automatic credential rotation
- Request verification from trusted origins

---

### <img src="https://api.iconify.design/mdi/wrench-outline.svg?color=%23FF5500" width="16"/> AgentCore Built-in Tools — Secure Execution Environments

> **Operational Pattern:** Create resource → Launch session → Interact via API → Terminate session

#### <img src="https://api.iconify.design/mdi/code-tags.svg?color=%23FF5500" width="14"/> Code Interpreter

| Property | Detail |
|---|---|
| Languages | Python, TypeScript, JavaScript |
| Security | Isolated sandbox in a Bedrock-managed environment |
| Use cases | Data analysis, calculations, code generation |

#### <img src="https://api.iconify.design/mdi/web.svg?color=%23FF5500" width="14"/> Browser Tool

| Property | Detail |
|---|---|
| Environment | Locked-down, cloud-based browser runtime |
| Capabilities | Live web page interaction, research, web app testing |
| Security | Isolated execution with network controls |

---

### <img src="https://api.iconify.design/mdi/chart-line.svg?color=%23FF5500" width="16"/> AgentCore Observability — Production Monitoring

#### <img src="https://api.iconify.design/mdi/eye-outline.svg?color=%23FF5500" width="14"/> Comprehensive Visibility

- **Reasoning steps** — complete decision-making process visibility
- **Tool invocations** — inputs, outputs, and execution time
- **Model interactions** — LLM requests, responses, and token usage
- **Error tracking** — detailed context for debugging

#### <img src="https://api.iconify.design/mdi/cloud-outline.svg?color=%23FF5500" width="14"/> Default Telemetry

- Automatic CloudWatch integration
- OpenTelemetry compatibility for existing stacks
- Key metrics: latency · token usage · session count · error rates
