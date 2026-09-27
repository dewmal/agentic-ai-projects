# Project 2 — Project Workspace Navigator Using MCP

## Implementation Plan with Pydantic AI + Read-Only MCP

**Source alignment:** Project 2 in *Four Practical Agentic AI Projects* (Project Workspace Navigator Using MCP, pp. 9–11).

## 1. Objective

Build a read-only agent that investigates an unfamiliar codebase through MCP-exposed capabilities, gathers the minimum relevant evidence, and returns an evidence-based explanation.

### Core invariant

> The agent may investigate and recommend, but it must not create, edit, delete, or execute project files.

The new capability compared with Project 1 is autonomous **capability selection**.

## 2. Agentic flow

```text
User question
    |
    v
DISCOVER
    |
    v
SEARCH
    |
    v
INSPECT
    |
    v
VERIFY
    |
    v
EXPLAIN
```

Execution loop:

```text
Think -> Select MCP Tool -> Execute -> Observe -> Decide Next Action -> Repeat or Answer
```

## 3. Suggested project structure

```text
project-workspace-navigator/
├── pyproject.toml
├── src/
│   └── workspace_navigator/
│       ├── __init__.py
│       ├── models.py
│       ├── agent.py
│       ├── mcp_server.py
│       ├── safety.py
│       └── cli.py
└── tests/
    ├── fixtures/
    │   └── sample_project/
    ├── test_mcp_tools.py
    ├── test_agent.py
    └── test_scenarios.py
```

Install:

```bash
uv add pydantic-ai fastmcp
uv add --dev pytest pytest-asyncio
```

## 4. Output contracts

```python
from typing import Literal
from pydantic import BaseModel, Field


class Evidence(BaseModel):
    path: str
    line_start: int | None = None
    line_end: int | None = None
    finding: str


class InvestigationStep(BaseModel):
    action: str
    reason: str


class WorkspaceAnswer(BaseModel):
    kind: Literal['answer'] = 'answer'
    answer: str
    evidence: list[Evidence] = Field(min_length=1)
    related_files: list[str] = Field(default_factory=list)
    investigation_summary: list[InvestigationStep] = Field(default_factory=list)
    uncertainty: list[str] = Field(default_factory=list)


class ClarificationRequest(BaseModel):
    kind: Literal['clarification'] = 'clarification'
    understood_question: str
    ambiguity: str
    question: str
```

## 5. MCP capability surface

Expose only read-only capabilities in v1:

```text
list_directory
search_files
search_content
read_file
inspect_project_metadata
```

Do **not** expose:

```text
write_file
apply_patch
delete_file
run_command
git_commit
```

The safety boundary should exist at the MCP server level, not only in the prompt.

## 6. Workspace path safety

```python
from pathlib import Path

PROJECT_ROOT = Path.cwd().resolve()


def resolve_safe_path(relative_path: str) -> Path:
    path = (PROJECT_ROOT / relative_path).resolve()
    if path != PROJECT_ROOT and PROJECT_ROOT not in path.parents:
        raise ValueError('Path escapes project workspace.')
    return path
```

## 7. FastMCP server skeleton

```python
from fastmcp import FastMCP

mcp = FastMCP('workspace-navigator')
```

Example tools:

```python
@mcp.tool()
def list_directory(path: str = '.', max_depth: int = 2) -> list[str]:
    root = resolve_safe_path(path)
    if not root.exists():
        return []

    results: list[str] = []
    base_depth = len(root.parts)
    for item in root.rglob('*'):
        depth = len(item.parts) - base_depth
        if depth <= max_depth:
            results.append(str(item.relative_to(PROJECT_ROOT)))
    return sorted(results)


@mcp.tool()
def search_files(query: str, limit: int = 30) -> list[str]:
    query_lower = query.lower()
    matches: list[str] = []
    for path in PROJECT_ROOT.rglob('*'):
        if not path.is_file():
            continue
        relative = str(path.relative_to(PROJECT_ROOT))
        if query_lower in relative.lower():
            matches.append(relative)
        if len(matches) >= limit:
            break
    return matches
```

`search_content` should return path, line number, and a bounded preview. `read_file` should support bounded line ranges.

## 8. Sensitive-content policy

At minimum, block known secret-bearing files:

```python
SENSITIVE_NAMES = {
    '.env',
    '.env.local',
    '.env.production',
    'credentials.json',
    'secrets.json',
    'id_rsa',
    'id_ed25519',
}
```

The MCP layer should refuse content access and searches should skip these files.

## 9. Connect MCP to Pydantic AI

Current Pydantic AI uses `MCPToolset`:

```python
from pydantic_ai import Agent
from pydantic_ai.mcp import MCPToolset

workspace_tools = MCPToolset(
    'src/workspace_navigator/mcp_server.py'
)

navigator_agent = Agent(
    'openai:gpt-5.6-sol',
    output_type=[WorkspaceAnswer, ClarificationRequest],
    instructions='...',
    toolsets=[workspace_tools],
)
```

For explicit stdio process configuration:

```python
from fastmcp.client.transports import StdioTransport
from pydantic_ai.mcp import MCPToolset

workspace_tools = MCPToolset(
    StdioTransport(
        command='python',
        args=['src/workspace_navigator/mcp_server.py'],
    )
)
```

## 10. Agent instructions

```python
INSTRUCTIONS = '''
You are a Project Workspace Navigator.

BOUNDARY
You may list directories, search filenames/content, read selected files,
and inspect project documentation/metadata.
You must never create, modify, delete, or execute project files.

PROCESS
DISCOVER -> SEARCH -> INSPECT -> VERIFY -> EXPLAIN

DISCOVER
Determine the minimum repository orientation needed.
Do not automatically scan the whole repository.

SEARCH
Broaden when results are empty; narrow when results are noisy.

INSPECT
Read only files and line ranges needed for the current question.

VERIFY
Confirm important conclusions using repository evidence.
Do not infer implementation merely from a filename.

AMBIGUITY
Investigate cheap likely interpretations or ask one focused clarification question.

SENSITIVE DATA
Never expose secrets.

FINAL ANSWER
Return the answer, evidence, related files, investigation summary,
and unresolved uncertainty.
'''
```

## 11. Investigation budgets

```python
MAX_SEARCH_RESULTS = 30
MAX_FILE_LINES = 250
MAX_FILES_PER_INVESTIGATION = 12
MAX_TOOL_CALLS = 20
```

Treat these as configurable defaults that reinforce selective context gathering.

## 12. Required behavior tests

| Scenario | Expected behavior |
|---|---|
| Find authentication implementation | Locate and connect relevant files |
| Feature spans multiple modules | Explain relationships between files |
| Expected file does not exist | Search alternative locations/patterns |
| Large repository | Avoid reading the entire workspace |
| Search returns irrelevant results | Refine the search |
| Sensitive configuration encountered | Do not expose secrets |
| Ambiguous feature name | Clarify or investigate likely interpretations |

Additional tests:

- path traversal is rejected;
- read range is bounded;
- no mutation tools are registered;
- MCP failure becomes explicit uncertainty;
- final evidence paths correspond to retrieved files.

## 13. Testing strategy

- **MCP unit tests:** test each tool as ordinary Python behavior.
- **Agent tests:** use `TestModel`/`FunctionModel` for tool/output wiring.
- **Real-model evals:** evaluate search strategy, cross-file reasoning, evidence quality, and secret handling.

## 14. MVP build order

1. Define `WorkspaceAnswer` and `ClarificationRequest`.
2. Implement workspace path safety.
3. Build the read-only FastMCP server.
4. Add secret-file filtering.
5. Connect the server through `MCPToolset`.
6. Implement Navigator instructions.
7. Create a representative sample repository fixture.
8. Unit-test every MCP tool.
9. Add the seven source-defined behavior scenarios.
10. Add real-model evals for search strategy and evidence quality.

## 15. Definition of done

The MVP is complete when:

- the agent independently selects read-only MCP capabilities;
- it retrieves only relevant evidence;
- every important repository conclusion has evidence;
- failed/noisy searches are refined;
- workspace mutation and execution are impossible through the MCP surface;
- known secret files cannot be exposed;
- all source-defined scenarios pass.

## 16. Pydantic AI references

- MCP client and `MCPToolset`: https://pydantic.dev/docs/ai/mcp/client/
- Structured outputs: https://pydantic.dev/docs/ai/core-concepts/output/
- Unit testing: https://pydantic.dev/docs/ai/guides/testing/
