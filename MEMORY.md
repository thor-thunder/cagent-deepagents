# MEMORY.md

MemoryMiddleware loads project context from AGENTS.md files and injects it into the system prompt.

## Overview

Unlike skills (on-demand workflows), memory is always loaded and provides persistent context to the agent. MemoryMiddleware supports the [AGENTS.md specification](https://agents.md/), enabling agents to work effectively with project-specific instructions, architecture notes, build commands, and style guidelines.

## Quick Start

```python
from deepagents import MemoryMiddleware, create_deep_agent
from deepagents.backends.filesystem import FilesystemBackend

backend = FilesystemBackend(root_dir="/")
middleware = MemoryMiddleware(
    backend=backend,
    sources=[
        "~/.deepagents/AGENTS.md",
        "./.deepagents/AGENTS.md",
    ],
)

agent = create_deep_agent(middleware=[middleware])
```

## Memory Sources

Sources are paths to AGENTS.md files loaded in order and concatenated. Later sources appear after earlier ones.

```python
MemoryMiddleware(
    sources=[
        "~/.deepagents/AGENTS.md",      # Global defaults
        "./.deepagents/AGENTS.md",      # Project-specific overrides
    ]
)
```

## File Format

AGENTS.md files are standard Markdown with no required structure. Common sections:
- Project overview
- Build/test commands
- Code style guidelines
- Architecture notes
- Development environment setup

Example:
```markdown
# Project AGENTS

## Build
make build

## Test
pytest tests/

## Code Style
- Type hints required
- Use Black for formatting
```

## Security

FilesystemBackend allows reading/writing from the entire filesystem. Ensure either:
- Agent runs in a sandbox, OR
- Add human-in-the-loop (HIL) approval to file operations

## State Management

Memory contents are stored in `MemoryState.memory_contents` as a `dict[path: str]` and marked private (not included in final agent state).
