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

Sub-agents delegate work with isolated context windows for specialized tasks, parallel execution, and reduced context pressure.

```python
User: Research and compare three ML frameworks

Agent delegates:
  -> task: "Research TensorFlow"
  -> task: "Research PyTorch"
  -> task: "Research JAX"
```

Each spawns a sub-agent with fresh context. For parallel execution, use `AsyncSubAgentMiddleware`:

```python
from deepagents import AsyncSubAgentMiddleware, create_deep_agent

agent = create_deep_agent(middleware=[AsyncSubAgentMiddleware()])
```

## Parallel Tool Calling (npm/npx)

Run multiple tools concurrently using npm/npx scripts:

```bash
# Run multiple tool calls in parallel
npx concurrently "npm run tool:research" "npm run tool:analyze" "npm run tool:summarize"

# or with xargs (shell built-in)
echo -e "tool:research\ntool:analyze\ntool:summarize" | xargs -P 3 -I {} npm run {}
```

The agent can invoke these as a single tool that handles parallelization:

```python
@tool
def run_parallel_tools(tasks: list[str]) -> str:
    """Run multiple tasks concurrently.
    
    Args:
        tasks: List of task names to run in parallel.
        
    Returns:
        Combined results from all tasks.
    """
    import subprocess
    cmd = ["npx", "concurrently"] + [f"npm run {t}" for t in tasks]
    result = subprocess.run(cmd, capture_output=True, text=True)
    return result.stdout
```

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

Use `pytest` for unit tests (mocked tools) and integration tests (real tools/models). Agents are non-deterministic — test for key phrases, not exact output. Mocked tools control behavior in unit tests.

```python
@pytest.fixture
def agent():
    return create_deep_agent(tools=[my_custom_tool])

def test_agent(agent):
    result = agent.invoke({
        "messages": [{"role": "user", "content": "Use my_custom_tool with X"}]
    })
    assert "expected_output" in result["messages"][-1]["content"]
```

## LangSmith Integration

```python
import os
os.environ["LANGSMITH_API_KEY"] = "your-key"
os.environ["LANGSMITH_PROJECT"] = "my-project"

agent = create_deep_agent()  # Automatically traced
```

Traces show tool calls, intermediate outputs, and conversation flow for debugging and monitoring.

## Extending Deep Agents

### Custom Middleware

```python
class CustomMiddleware:
    def __init__(self):
        self.name = "custom_tool"
        self.description = "Tool description"
    
    async def __call__(self, input_str: str) -> str:
        return f"Result: {input_str}"

agent = create_deep_agent(middleware=[CustomMiddleware()])
```

### Custom Prompts & Model Swapping

```python
from langchain.chat_models import init_chat_model

agent = create_deep_agent(
    model=init_chat_model("openai:gpt-4o"),
    system_prompt="You are an expert data analyst. Be concise."
)
```

## Performance & Troubleshooting

**Performance:** Monitor context length; use async tools and `AsyncSubAgentMiddleware` for parallel work. File system middleware auto-saves outputs >1MB.

**Agent loops indefinitely:** Check tool definitions, verify LLM receives tool results, add max-iterations limit.

**Out of context:** Use `MemoryMiddleware` for summarization or split tasks into sub-agents.

**Tool not called:** Verify tool is in `tools` parameter, ensure clear description, add examples to system prompt.
