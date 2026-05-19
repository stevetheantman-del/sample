# OpenAI Agents SDK Reference Guide (2026)

## Overview

The OpenAI Agents SDK is a lightweight framework for building multi-agent workflows. It wraps the Responses API and provides clean abstractions for agents, tools, handoffs, guardrails, and tracing.

**Key Primitives:**
- **Agents** - LLMs with instructions, tools, and configuration
- **Runner** - Execution loop that orchestrates agent calls
- **Tools** - Functions, MCP servers, web search, file search, computer use, etc.
- **Handoffs** - Agent-to-agent delegation
- **Guardrails** - Input/output validation
- **Sessions** - Conversation state persistence

---

## Agent Definition

### Basic Agent

```python
from agents import Agent, Runner

agent = Agent(
    name="Assistant",
    instructions="You are a helpful assistant.",
    model="gpt-5.5",  # Default: gpt-5.5
    tools=[],         # Tools available to the agent
)

result = Runner.run_sync(agent, "Your input here")
print(result.final_output)
```

### Agent Parameters

```python
agent = Agent(
    name: str,                                    # Agent name
    instructions: str,                            # System instructions
    model: str = "gpt-5.5",                      # Model identifier
    tools: list = [],                            # Available tools
    
    # Tool behavior
    tool_choice: str = "auto",                   # "auto", "required", or tool name
    tool_use_behavior: str = "run_llm_again",   # "run_llm_again" or "return_tool_output"
    
    # Model settings
    model_settings: ModelSettings = None,        # Override temperature, max_tokens, etc.
    
    # Prompts
    prompt: dict | Callable = None,              # Named prompt or dynamic generator
    
    # Output
    output_type: type = str,                     # Dataclass for structured output
    
    # Safety
    input_guardrail: Guardrail = None,          # Validate inputs
    output_guardrail: Guardrail = None,         # Validate outputs
    
    # Hooks
    hooks: AgentHooks = None,                    # Observe agent events
)
```

### ModelSettings

```python
from agents import ModelSettings

agent = Agent(
    name="Weather Agent",
    instructions="Retrieve weather details.",
    tools=[get_weather],
    model_settings=ModelSettings(
        temperature=0.7,                          # 0.0-2.0
        max_tokens=1024,                         # Max output
        top_p=0.9,                               # Nucleus sampling
        frequency_penalty=0.0,                   # -2.0 to 2.0
        presence_penalty=0.0,                    # -2.0 to 2.0
        tool_choice="get_weather",               # Force specific tool
    )
)
```

---

## Runner

### Basic Execution

```python
from agents import Runner

# Synchronous
result = Runner.run_sync(agent, "What's the weather?")
print(result.final_output)

# Asynchronous
import asyncio
result = await Runner.run(agent, "What's the weather?")

# Streaming events
async for event in Runner.run_streamed(agent, "What's the weather?"):
    print(event)
```

### RunConfig Parameters

```python
from agents.run import RunConfig

result = await Runner.run(
    agent,
    input="Your task here",
    
    # Conversation state
    conversation_id="conv_123",                  # Resume existing conversation
    
    # Context (for dependency injection)
    context={"user_id": "usr_123"},             # Custom context object
    
    # Model overrides
    run_config=RunConfig(
        model_provider=None,                     # Override model lookup
        global_model_settings=ModelSettings(     # Override all agents
            temperature=0.5
        ),
        max_turns=20,                            # Max conversation turns
    ),
    
    # Hooks (observe entire workflow)
    run_hooks=RunHooks(
        on_run_start=callback,
        on_run_end=callback,
        on_tool_start=callback,
        on_tool_end=callback,
        on_handoff=callback,
    ),
)
```

### Result Object

```python
# Accessing the result
result.final_output              # String output
result.final_output_as(MyClass)  # Parse as dataclass
result.messages                  # Full conversation history
result.usage                     # Token usage stats
result.trace_id                  # For observability
```

---

## Function Tools

### Basic Function Tool

```python
from agents import function_tool

@function_tool
def get_weather(city: str) -> str:
    """Returns weather info for the specified city."""
    return f"The weather in {city} is sunny"

agent = Agent(
    name="Weather Agent",
    instructions="Retrieve weather details.",
    tools=[get_weather],
)
```

### Advanced Function Tool

```python
@function_tool(
    name="custom_name",                # Override function name
    description="Custom description",  # Override docstring description
    defer_loading=False,               # Load immediately or on-demand
)
def get_weather(city: str) -> str:
    """Returns weather info for the specified city."""
    return f"The weather in {city} is sunny"
```

### Tool with Complex Parameters

```python
from pydantic import BaseModel
from agents import function_tool

class WeatherRequest(BaseModel):
    city: str
    units: str = "celsius"  # "celsius" or "fahrenheit"

@function_tool
def get_weather(request: WeatherRequest) -> str:
    """Get weather with unit preference."""
    return f"Weather in {request.city}: sunny, 25 {request.units}"
```

---

## Built-in Tools

### WebSearchTool

```python
from agents import Agent, WebSearchTool, Runner

agent = Agent(
    name="Research Assistant",
    tools=[
        WebSearchTool(
            user_location="San Francisco, CA",  # For local searches
            search_context_size=3,              # Results per search
        )
    ],
)
```

### FileSearchTool (Vector Store)

```python
from agents import FileSearchTool

agent = Agent(
    name="Document Assistant",
    tools=[
        FileSearchTool(
            vector_store_ids=["vs_abc123"],     # OpenAI vector store
            max_num_results=5,                  # Top K results
            filters={"document_type": "pdf"},   # Optional filters
        )
    ],
)
```

### CodeInterpreterTool

```python
from agents import CodeInterpreterTool

agent = Agent(
    name="Code Executor",
    tools=[CodeInterpreterTool()],
)
```

### ShellTool (Local)

```python
from agents import ShellTool

agent = Agent(
    name="Shell Agent",
    tools=[
        ShellTool(
            environment={"type": "local"},      # Local execution
            allow_dangerous_commands=False,     # Safety
        )
    ],
)
```

### ShellTool (Hosted Container)

```python
from agents import ShellTool, ShellToolSkillReference

csv_skill: ShellToolSkillReference = {
    "type": "skill_reference",
    "skill_id": "skill_698bb...",
    "version": "1",
}

agent = Agent(
    name="Container Agent",
    tools=[
        ShellTool(
            environment={
                "type": "container_auto",  # OpenAI-managed container
                "network_policy": {"type": "disabled"},
                "skills": [csv_skill],
            }
        )
    ],
)
```

### ImageGenerationTool

```python
from agents import ImageGenerationTool

agent = Agent(
    name="Image Creator",
    tools=[ImageGenerationTool()],
)
```

### HostedMCPTool (Remote MCP Server)

```python
from agents import HostedMCPTool

agent = Agent(
    name="MCP Agent",
    tools=[
        HostedMCPTool(
            url="https://mcp.example.com/sse",  # MCP server URL
            name="example_mcp",
        )
    ],
)
```

### Tool Search (Deferred Loading)

```python
from agents import ToolSearchTool, function_tool

@function_tool(defer_loading=True)  # Load on demand
def heavy_function_1():
    pass

@function_tool(defer_loading=True)
def heavy_function_2():
    pass

agent = Agent(
    name="Search Agent",
    tools=[
        ToolSearchTool(),  # Enables dynamic tool loading
        heavy_function_1,
        heavy_function_2,
    ],
)
```

---

## Sandbox Agents

### Basic Sandbox Agent

```python
from agents import Runner
from agents.sandbox import Manifest, SandboxAgent, SandboxRunConfig
from agents.sandbox.entries import GitRepo
from agents.sandbox.sandboxes import UnixLocalSandboxClient

agent = SandboxAgent(
    name="Workspace Assistant",
    instructions="Inspect the sandbox workspace before answering.",
    default_manifest=Manifest(
        entries={
            "repo": GitRepo(repo="openai/openai-agents-python", ref="main"),
        }
    ),
)

result = Runner.run_sync(
    agent,
    "Inspect the repo README and summarize what this project does.",
    run_config=RunConfig(
        sandbox=SandboxRunConfig(
            client=UnixLocalSandboxClient()
        )
    ),
)
```

### Sandbox Manifest Entries

```python
from agents.sandbox.entries import (
    FileContent,      # Static file in workspace
    GitRepo,          # Clone a Git repository
    Directory,        # Mount a local directory
)

manifest = Manifest(
    entries={
        "code": Directory(path="/home/user/project"),
        "repo": GitRepo(repo="owner/repo", ref="main"),
        "config": FileContent(content="key=value\n"),
    }
)
```

---

## Handoffs (Agent-to-Agent Delegation)

### Basic Handoff

```python
from agents import Agent, Handoff

triage_agent = Agent(
    name="Triage Agent",
    instructions="Route to the appropriate specialist.",
    tools=[
        Handoff(
            agent=billing_agent,
            description="Transfer to billing specialist",
        ),
        Handoff(
            agent=support_agent,
            description="Transfer to support specialist",
        ),
    ],
)
```

### Handoff with Input Filter

```python
def filter_billing_input(context, original_input):
    """Edit input before handing off."""
    return f"CONTEXT: {context}\nUSER: {original_input}"

handoff = Handoff(
    agent=billing_agent,
    description="Transfer billing issues",
    input_filter=filter_billing_input,
)
```

---

## Guardrails (Input/Output Validation)

### Basic Guardrail

```python
from agents import Guardrail

def validate_input(input_text):
    if len(input_text) < 10:
        raise ValueError("Input too short")
    return input_text

input_guardrail = Guardrail(
    description="Validate minimum length",
    handler=validate_input,
)

agent = Agent(
    name="Guarded Agent",
    instructions="...",
    input_guardrail=input_guardrail,
    output_guardrail=Guardrail(
        description="Validate output format",
        handler=validate_output,
    ),
)
```

---

## Structured Output

### Define Output Schema

```python
from dataclasses import dataclass
from agents import Agent, Runner

@dataclass
class CalendarEvent:
    title: str
    date: str
    attendees: list[str]

agent = Agent(
    name="Event Extractor",
    instructions="Extract calendar event details. Return as JSON.",
    output_type=CalendarEvent,
)

result = Runner.run_sync(agent, "I have a meeting on 2026-04-10 with Sarah and Tom")
event = result.final_output_as(CalendarEvent)
print(event.title)      # "Q2 Planning Review"
print(event.attendees)  # ["Sarah", "Tom"]
```

---

## Tracing & Observability

### Enable Tracing

```python
from agents import Agent, Runner, trace

# Automatic tracing to OpenAI dashboard (openai.com/traces)
result = await Runner.run(agent, "Your input")
print(result.trace_id)  # Trace ID for debugging

# Disable tracing
# export OPENAI_AGENTS_DISABLE_TRACING=1

# Custom trace context
with trace("customer_support_session", metadata={"user_id": "usr_123"}):
    result1 = await Runner.run(triage_agent, "Billing issue")
    result2 = await Runner.run(billing_agent, "Follow-up")
    # Both runs appear as one trace in dashboard
```

### Access Trace Data

```python
from agents.tracing import get_current_trace

trace = get_current_trace()
print(trace.trace_id)
print(trace.metadata)
```

### Hooks (Observe Events)

```python
from agents import Agent, Runner, RunHooks

def on_tool_start(context):
    print(f"Tool started: {context.tool_call.function.name}")

def on_tool_end(context):
    print(f"Tool completed")

result = await Runner.run(
    agent,
    "Your input",
    run_hooks=RunHooks(
        on_tool_start=on_tool_start,
        on_tool_end=on_tool_end,
    ),
)
```

---

## Sessions (Conversation State)

### Server-Managed Sessions

```python
from agents import Runner

# Start new conversation
result1 = await Runner.run(agent, "Hello")

# Resume conversation
result2 = await Runner.run(
    agent,
    "What did I just say?",
    conversation_id="conv_123",  # From previous result
)
```

### Local Session Storage

```python
from agents import LocalSessionStorage

storage = LocalSessionStorage(db_path="./sessions.db")

result = await Runner.run(
    agent,
    "Your input",
    session_storage=storage,
    conversation_id="conv_123",
)
```

### Redis Session Storage

```python
from agents import RedisSessionStorage

storage = RedisSessionStorage(url="redis://localhost:6379")

result = await Runner.run(
    agent,
    "Your input",
    session_storage=storage,
)
```

---

## Streaming

### Stream Agent Events

```python
async for event in Runner.run_streamed(agent, "Your input"):
    if event.type == "on_tool_start":
        print(f"Tool: {event.tool_name}")
    elif event.type == "on_message":
        print(f"Message: {event.content}")
    elif event.type == "on_final_output":
        print(f"Final: {event.output}")
```

### FastAPI Integration (SSE)

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from agents import Runner

app = FastAPI()

@app.get("/chat")
async def chat(input: str):
    async def event_generator():
        async for event in Runner.run_streamed(agent, input):
            # Convert to SSE format
            yield f"data: {event.to_json()}\n\n"
    
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

---

## Error Handling

```python
from agents import Runner
import openai

try:
    result = await Runner.run(agent, "Your input")
except openai.RateLimitError:
    print("Rate limited - back off")
except openai.AuthenticationError:
    print("Invalid API key")
except Exception as e:
    print(f"Agent error: {e}")
```

---

## Installation

```bash
# Basic installation
pip install openai-agents

# With voice support
pip install 'openai-agents[voice]'

# With Redis session support
pip install 'openai-agents[redis]'

# Verify
python -c "from agents import Agent, Runner; print('OK')"
```

**Requirements:** Python 3.10+ and openai>=1.65.0

---

## Key Differences: Agent vs Runner

| Aspect | Agent | Runner |
|--------|-------|--------|
| **Purpose** | Configuration object | Execution engine |
| **Execution** | Does not execute | Orchestrates full loop |
| **What it does** | Defines instructions, tools, settings | Calls model → runs tools → loops until done |
| **Stateless** | Yes | Can be stateful with sessions |

---

## Production Patterns

### Multi-Agent Workflow

```python
from agents import Agent, Handoff, Runner

# Specialized agents
triage_agent = Agent(name="Triage", ...)
billing_agent = Agent(name="Billing", ...)
support_agent = Agent(name="Support", ...)

# Connect via handoffs
triage_agent.tools.append(Handoff(agent=billing_agent, ...))
triage_agent.tools.append(Handoff(agent=support_agent, ...))

# Run workflow
result = await Runner.run(triage_agent, user_input)
```

### Structured Output + Validation

```python
from dataclasses import dataclass
from agents import Agent, Runner

@dataclass
class TicketDecision:
    issue_type: str  # "billing" | "bug" | "usage"
    priority: str    # "high" | "medium" | "low"
    assigned_to: str

agent = Agent(
    name="Triage",
    output_type=TicketDecision,
)

decision = Runner.run_sync(agent, ticket_description).final_output_as(TicketDecision)
```

### Streaming to Browser (SSE)

```python
# Backend with FastAPI + Runner.run_streamed()
# Frontend with EventSource()

const eventSource = new EventSource("/chat?input=...");
eventSource.onmessage = (e) => {
    const event = JSON.parse(e.data);
    console.log(event);
};
```

---

## Pricing (as of April 2026)

- **GPT-5.4-mini**: ~$0.0007 for 4-turn conversation with 2 tool calls
- **GPT-5**: Higher cost, better quality for complex tasks
- **Realtime API (voice)**: ~$0.40 per 5-minute call
- **Batch API**: 50% discount on standard pricing

---

## Resources

- **Docs**: https://openai.github.io/openai-agents-python/
- **GitHub**: https://github.com/openai/openai-agents-python
- **PyPI**: https://pypi.org/project/openai-agents/
- **Traces Dashboard**: https://openai.com/traces
