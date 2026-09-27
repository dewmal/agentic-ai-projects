# Project 1 — Work Request Triage Agent

## Implementation Plan with Pydantic AI

**Source alignment:** Project 1 in *Four Practical Agentic AI Projects* (Work Request Triage Agent, pp. 6–8).

## 1. Objective

Build a **single-agent, no-tool** system that converts an unstructured work request into exactly one of two outcomes:

1. a focused clarification request, or
2. a structured, prioritized work plan.

### Core invariant

> Do not plan through critical ambiguity, and do not invent requirements.

The agent may structure and prioritize work, but it must not invent deadlines, stakeholders, business facts, or completed actions.

## 2. Agentic flow

```text
User request
    |
    v
UNDERSTAND
    |
    v
CHECK_INFORMATION
    |
    +---- critical gap / blocking conflict ----> CLARIFY
    |                                             |
    |                                             +--> user answers
    |                                                   |
    +---- enough critical information ------------------+
    |
    v
PLAN
    |
    v
COMPLETE
```

Conceptual execution loop:

```text
Understand -> Extract -> Check Gaps -> Clarify or Plan -> Prioritize -> Complete
```

## 3. Suggested project structure

```text
work-request-triage/
├── pyproject.toml
├── src/
│   └── triage_agent/
│       ├── __init__.py
│       ├── models.py
│       ├── agent.py
│       ├── service.py
│       └── cli.py
└── tests/
    ├── test_models.py
    ├── test_agent.py
    └── test_scenarios.py
```

Install:

```bash
uv add pydantic-ai
uv add --dev pytest pytest-asyncio
```

## 4. Output contracts

```python
from typing import Literal
from pydantic import BaseModel, Field


class ClarificationQuestion(BaseModel):
    field: str
    question: str
    reason: str


class ClarificationRequest(BaseModel):
    kind: Literal['clarification'] = 'clarification'
    understood_goal: str
    missing_critical_information: list[str] = Field(min_length=1)
    questions: list[ClarificationQuestion] = Field(min_length=1, max_length=3)
    conflicts: list[str] = Field(default_factory=list)


class PlanTask(BaseModel):
    order: int = Field(ge=1)
    task: str
    priority: Literal['high', 'medium', 'low']
    rationale: str
    depends_on: list[str] = Field(default_factory=list)


class WorkPlan(BaseModel):
    kind: Literal['plan'] = 'plan'
    goal: str
    assumptions: list[str] = Field(default_factory=list)
    constraints: list[str] = Field(default_factory=list)
    deadline: str | None = None
    stakeholders: list[str] = Field(default_factory=list)
    tasks: list[PlanTask] = Field(min_length=1)
    next_steps: list[str] = Field(min_length=1)
    unresolved_optional_information: list[str] = Field(default_factory=list)
```

Pydantic AI supports multiple structured output types:

```python
from pydantic_ai import Agent

triage_agent = Agent(
    'openai:gpt-5.6-sol',
    output_type=[ClarificationRequest, WorkPlan],
    instructions='...',
)
```

## 5. Agent instructions

```python
INSTRUCTIONS = '''
You are a Work Request Triage Agent.

Return exactly one of:
- ClarificationRequest
- WorkPlan

AIM
Convert ambiguous work requests into clear, actionable outcomes without inventing requirements.

GATHER
Use only information available in the conversation.
Classify missing information as CRITICAL or OPTIONAL.

NAVIGATION
Return ClarificationRequest when the objective is unclear, a critical requirement is missing,
a blocking contradiction exists, or the intended deliverable cannot be determined.
Ask only the minimum questions needed, preferably 1–3.

Return WorkPlan when enough critical information exists.
Optional gaps must not unnecessarily block planning; record them as assumptions or unresolved optional information.

BOUNDARIES
Never invent deadlines, stakeholders, business facts, user requirements,
or actions that have already been completed.
If no deadline was supplied, deadline must be null.
If no stakeholders were supplied, stakeholders must be an empty list.

IMPOSSIBLE REQUESTS
Do not pretend an impossible objective is achievable.
State the constraint and propose a feasible alternative when possible.
'''
```

## 6. Conversation state

Clarification must be able to re-enter the same agent:

```python
from pydantic_ai import ModelMessage

history: list[ModelMessage] = []

result = triage_agent.run_sync(
    user_input,
    message_history=history,
)

history = result.all_messages()
```

## 7. Output validation

Use output validators for application-level rules:

```python
from pydantic_ai import ModelRetry


@triage_agent.output_validator
async def validate_output(output):
    if isinstance(output, WorkPlan):
        orders = [task.order for task in output.tasks]
        if orders != sorted(orders):
            raise ModelRetry('Plan tasks must be ordered by execution sequence.')
    return output
```

Production validation should also compare structured fields against known conversation facts where feasible, especially deadlines and stakeholders.

## 8. Required behavior tests

| Scenario | Expected behavior |
|---|---|
| Complete request | Produce a structured plan directly |
| Very vague request | Ask focused clarification |
| Missing optional information | Continue without blocking |
| Missing essential information | Ask for missing information |
| Contradictory requirements | Identify the conflict |
| User changes requirement | Re-plan from the new state |
| Impossible objective | Explain the limitation and propose a feasible path |

Additional assertions:

- no fabricated deadlines;
- no fabricated stakeholders;
- clarification questions <= 3;
- plan contains at least one task and next step;
- agent never claims proposed work has already been completed.

## 9. Testing approach

- **Unit tests:** Pydantic schemas and pure helper functions.
- **Agent integration tests:** use `TestModel` or `FunctionModel` to exercise output wiring without real LLM calls.
- **Behavioral evals:** real-model cases for ambiguity detection, unnecessary clarification, contradiction handling, and plan usefulness.

## 10. MVP build order

1. Create Pydantic output models.
2. Create the single Pydantic AI agent.
3. Encode critical-vs-optional information rules in instructions.
4. Add multi-turn message history.
5. Add output validators.
6. Implement CLI/service wrapper.
7. Add the seven source-defined scenarios as tests.
8. Add real-model behavioral evals.
9. Add observability after behavior is stable.

## 11. Definition of done

The MVP is complete when:

- one Pydantic AI agent handles the task;
- it has no external tools;
- output is always `ClarificationRequest` or `WorkPlan`;
- critical ambiguity blocks planning;
- optional ambiguity does not unnecessarily block planning;
- contradictions are surfaced;
- invented requirements are rejected by tests;
- all seven behavioral scenarios pass.

## 12. Pydantic AI references

- Output types and structured outputs: https://pydantic.dev/docs/ai/core-concepts/output/
- Output validators: https://pydantic.dev/docs/ai/core-concepts/output/#output-validators
- Unit testing: https://pydantic.dev/docs/ai/guides/testing/
