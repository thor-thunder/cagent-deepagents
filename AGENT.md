# AGENT.md

This file documents the Deep Agents framework architecture and key concepts for developers building and extending agents.

**See also:** [`CLAUDE.md`](./CLAUDE.md) for development environment setup and project-wide standards.

## Deep Agents Framework Overview

Deep Agents is a Python agent harness built on LangGraph that provides batteries-included capabilities for complex, multi-step tasks:

- **Planning** — `write_todos` tool for task breakdown and progress tracking
- **Filesystem** — `read_file`, `write_file`, `edit_file`, `ls`, `glob`, `grep` for context management
- **Shell Access** — `execute` tool for running commands in a sandbox
- **Sub-agents** — `task` tool for delegating work with isolated context windows
- **Smart Defaults** — Pre-tuned prompts and middleware for effective tool use
- **Context Management** — Auto-summarization when conversations exceed context limits

The core entry point is `create_deep_agent()`, which returns a compiled LangGraph graph ready to invoke.

## Core Architecture

### Graph Structure (LangGraph)

The Deep Agents graph is defined in `libs/deepagents/deepagents/graph.py` and follows a standard agentic loop:

1. **Input:** Messages with user query
2. **LLM Call:** Model decides which tool to call (or returns final response)
3. **Tool Execution:** Middleware handles tool invocation
4. **Loop:** Repeat until model returns stop signal
5. **Output:** Compiled LangGraph graph supporting streaming, persistence, checkpointing

The graph is **provider-agnostic** — works with any LLM supporting tool calling (Claude, GPT-4, Gemini, open models, etc.).

### Middleware System

Middleware wraps the core graph and provides tool implementations and context management.

**Built-in Middleware:**

- **`FilesystemMiddleware`** — Virtual file system. Manages `read_file`, `write_file`, `edit_file`, `ls`, `glob`, `grep`.
  - Path isolation: agents cannot escape the working directory
  - Large outputs saved to files automatically
  - See: `libs/deepagents/deepagents/middleware/filesystem.py`

- **`SubAgentMiddleware`** — Spawns sub-agents with isolated context windows via `task` tool
  - Creates a new agent instance with a subset of the conversation
  - Useful for delegating specialized work
  - See: `libs/deepagents/deepagents/middleware/subagents.py`

- **`AsyncSubAgentMiddleware`** — Async variant for parallel sub-agent execution
  - See: `libs/deepagents/deepagents/middleware/async_subagents.py`

- **`MemoryMiddleware`** — Context management and auto-summarization
  - Tracks conversation length and summarizes when approaching token limits
  - See: `libs/deepagents/deepagents/middleware/memory.py`

### Base Prompt

The agent's system prompt is in `libs/deepagents/deepagents/base_prompt.md`. It teaches the model:
- How to use each tool effectively
- When to delegate vs. handle locally
- How to break down complex tasks (via `write_todos`)
- Reasoning patterns for multi-step workflows

Custom prompts can be passed to `create_deep_agent(system_prompt="your prompt")`.

### Models & Backends

The SDK includes pre-configured chat models in `libs/deepagents/deepagents/backends/`:
- Anthropic Claude (claude-opus, claude-sonnet, claude-haiku)
- OpenAI (gpt-4, gpt-4o)
- Google Gemini

Models are initialized via LangChain's `init_chat_model()`:

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("openai:gpt-4o")
agent = create_deep_agent(model=model)
```

## Building Agents

### Basic Usage

```python
from deepagents import create_deep_agent

# Create with defaults (uses Anthropic Claude)
agent = create_deep_agent()

# Invoke
result = agent.invoke({
    "messages": [{"role": "user", "content": "Research LangGraph and write a summary"}]
})
```

### Customization

```python
from deepagents import create_deep_agent, SubAgentMiddleware
from langchain.chat_models import init_chat_model

agent = create_deep_agent(
    model=init_chat_model("openai:gpt-4o"),
    system_prompt="You are a specialized research agent.",
    tools=[custom_tool_1, custom_tool_2],  # Add custom tools
    middleware=[SubAgentMiddleware()],      # Include sub-agent support
)
```

### Adding Custom Tools

Tools are LangChain `BaseTool` instances. Define using the decorator pattern:

```python
from langchain.tools import tool

@tool
def get_weather(location: str) -> str:
    """Get current weather for a location.
    
    Args:
        location: City or region name.
        
    Returns:
        Weather description.
    """
    # Implementation
    return f"Sunny in {location}"

agent = create_deep_agent(tools=[get_weather])
```

Tools are automatically bound to the agent and exposed to the LLM.

## Sub-Agents

Sub-agents delegate work with isolated context windows, useful for:
- Delegating specialized tasks (research, coding, analysis)
- Parallel execution (via `AsyncSubAgentMiddleware`)
- Reducing context pressure on the main agent

### Using Sub-Agents

When the main agent invokes the `task` tool:

```
User: Research and compare three ML frameworks

Agent decides to delegate:
  -> task: "Research TensorFlow and write a summary"
  -> task: "Research PyTorch and write a summary"
  -> task: "Research JAX and write a summary"
```

Each task spawns a sub-agent with a fresh context window. Results are collected and returned to the main agent.

### Async Sub-Agents

For parallel execution:

```python
from deepagents import AsyncSubAgentMiddleware, create_deep_agent

agent = create_deep_agent(
    middleware=[AsyncSubAgentMiddleware()]
)
```

Async sub-agents run concurrently, reducing total time.

## Context Management

The `MemoryMiddleware` handles context overflow:

1. **Tracking:** Monitor conversation length (token count)
2. **Summarization:** When approaching model's context limit, summarize earlier exchanges
3. **Persistence:** Summaries are prepended to the conversation to retain history

This allows agents to handle arbitrarily long conversations without truncation.

## File System Access

The `FilesystemMiddleware` provides sandboxed file operations:

- **`read_file(path)`** — Read file contents (returns as string)
- **`write_file(path, content)`** — Write/create file
- **`edit_file(path, old_string, new_string)`** — Search-and-replace within file
- **`ls(path)`** — List directory contents
- **`glob(pattern)`** — Find files matching pattern
- **`grep(pattern, path)`** — Search for pattern in files

All paths are isolated to the agent's working directory. The agent cannot access parent directories or escape the sandbox.

Large file outputs (>1MB) are automatically saved to disk rather than returned as strings, reducing memory pressure.

## Testing Agents

### Unit Tests

Test custom tools and middleware in isolation:

```python
import pytest
from deepagents import create_deep_agent

@pytest.fixture
def agent():
    return create_deep_agent(tools=[my_custom_tool])

def test_agent_uses_custom_tool(agent):
    result = agent.invoke({
        "messages": [{"role": "user", "content": "Use my_custom_tool with X"}]
    })
    assert "expected_output" in result["messages"][-1]["content"]
```

### Integration Tests

Test end-to-end workflows with real tools and models:

```python
@pytest.mark.integration
def test_research_workflow(agent):
    result = agent.invoke({
        "messages": [{"role": "user", "content": "Research and summarize LangGraph"}]
    })
    # Verify the agent completed the task
    assert len(result["messages"]) > 1
```

### Determinism & Flakiness

Agents are inherently non-deterministic (LLM outputs vary). For testing:
- Use mocked tools in unit tests to control behavior
- Accept some variance in integration tests (check for key phrases, not exact strings)
- Avoid flaky assertions on LLM output wording

## LangSmith Integration

For debugging, monitoring, and evals, agents can be traced to [LangSmith](https://smith.langchain.com):

```python
import os
from deepagents import create_deep_agent

os.environ["LANGSMITH_API_KEY"] = "your-key"
os.environ["LANGSMITH_PROJECT"] = "my-project"

agent = create_deep_agent()
# All invocations are automatically traced
```

Traces show tool calls, intermediate outputs, and full conversation flow.

## Extending Deep Agents

### Custom Middleware

Implement a middleware class extending `BaseTool` or a custom handler:

```python
from langchain.tools import BaseTool

class CustomMiddleware:
    def __init__(self):
        self.name = "custom_tool"
        self.description = "Custom tool description"
    
    async def __call__(self, input_str: str) -> str:
        # Handle tool invocation
        return f"Result: {input_str}"

agent = create_deep_agent(middleware=[CustomMiddleware()])
```

### Custom Prompts

Override the base prompt:

```python
agent = create_deep_agent(
    system_prompt="You are an expert data analyst. Be concise and focus on insights."
)
```

### Model Swapping

Test with different models:

```python
from langchain.chat_models import init_chat_model

models = [
    init_chat_model("openai:gpt-4o"),
    init_chat_model("anthropic:claude-opus-4-1"),
    init_chat_model("google_genai:gemini-2.0-flash"),
]

for model in models:
    agent = create_deep_agent(model=model)
    # Test or benchmark
```

## Performance Considerations

- **Context limits:** Monitor conversation length; use `MemoryMiddleware` for long sessions
- **Tool efficiency:** Keep tool implementations fast; async tools reduce latency
- **Sub-agent cost:** Each sub-agent incurs overhead; batch related tasks
- **Large files:** File system middleware auto-saves outputs >1MB; plan accordingly

## Troubleshooting

**Agent loops indefinitely:**
- Check tool definitions for correctness
- Verify LLM is receiving tool results
- Add a max-iterations limit in the graph configuration

**Out of context:**
- Enable `MemoryMiddleware` for summarization
- Reduce verbosity in tool outputs
- Split complex tasks into sub-agents

**Tool not called:**
- Verify tool is exposed in the graph (check `tools` parameter)
- Ensure tool description is clear and relevant
- Add examples to the system prompt showing tool usage

## Resources

- **Documentation:** https://docs.langchain.com/oss/python/deepagents/overview
- **API Reference:** https://reference.langchain.com/python/deepagents/
- **Chat with docs:** https://chat.langchain.com
- **LangGraph:** https://docs.langchain.com/oss/python/langgraph/overview
- **LangChain:** https://docs.langchain.com
- **GitHub:** https://github.com/langchain-ai/deepagents
