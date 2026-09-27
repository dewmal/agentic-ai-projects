# Project 1 - Work Request Triage Agent with Pydantic AI

## Complete implementation guide

This project implements the first project in the A.G.E.N.T. design portfolio: a single-agent, no-business-tool system that converts an unstructured work request into exactly one of two typed outcomes:

1. `ClarificationRequest` when critical information is missing or contradictory.
2. `WorkPlan` when enough critical information exists to plan responsibly.

The project demonstrates the minimum viable agentic loop:

```text
Reason -> Decide -> Clarify -> Plan -> Stop
```

It is deliberately small. The autonomy comes from deciding what should happen next, not from calling external tools.

---

## 1. A.G.E.N.T. design summary

### Aim

Convert ambiguous work requests into clear, actionable outcomes without inventing requirements.

The agent may structure work, identify assumptions, prioritize tasks, record dependencies, and recommend next steps. It must not invent deadlines, stakeholders, business facts, requirements, or completed actions.

### Gather

Use only information available in the conversation. Relevant information can include:

- objective;
- desired deliverable;
- deadline;
- constraints;
- stakeholders;
- priority;
- success criteria.

Missing information is classified as either:

- **Critical:** planning would require inventing an important requirement.
- **Optional:** useful context is absent, but a responsible plan can still be produced if the uncertainty is recorded.

### Execution

```text
Understand -> Extract -> Check gaps -> Clarify or plan -> Prioritize -> Complete
```

### Navigation

```text
UNDERSTAND
    |
    v
CHECK_INFORMATION
    |
    +-- critical gap ----------> CLARIFY
    |                               |
    |                               | user answers
    |                               v
    |                         CHECK_INFORMATION
    |
    +-- blocking conflict ------> CLARIFY
    |
    +-- impossible objective ---> PLAN a feasible alternative
    |
    +-- sufficient context -----> PLAN
                                    |
                                    v
                                 COMPLETE
```

Navigation rules:

- unclear objective -> ask focused clarification;
- missing essential information -> ask for it;
- conflicting requirements -> identify the conflict instead of silently choosing;
- missing optional information -> continue and record the unknown;
- impossible objective -> explain the constraint and propose a realistic path;
- sufficient context -> return a prioritized plan;
- changed requirement -> re-evaluate from the new conversation state and re-plan.

### Test

The seven source-defined scenarios are:

| Scenario | Expected behavior |
|---|---|
| Complete request | Produce a structured plan directly. |
| Very vague request | Ask focused clarification. |
| Missing optional information | Continue without blocking. |
| Missing essential information | Ask for the missing information. |
| Contradictory requirements | Identify and explain the conflict. |
| User changes requirement | Re-plan from the new state. |
| Impossible objective | Explain the limitation and propose a realistic path. |

---

## 2. Architecture

```text
                    +-------------------+
                    | User work request |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    | Pydantic AI Agent |
                    | single agent      |
                    | no business tools |
                    +---------+---------+
                              |
                      reason + classify
                              |
                 +------------+------------+
                 |                         |
                 v                         v
      +-----------------------+   +------------------+
      | ClarificationRequest  |   | WorkPlan         |
      | typed Pydantic model  |   | typed Pydantic   |
      +-----------+-----------+   +---------+--------+
                  |                         |
            user supplies                  |
            missing context                |
                  |                         |
                  +----------> Agent <------+
                              |
                        updated history
```

There are no `@agent.tool` functions. `ToolOutput` is used only as the structured-output mechanism. Naming the two output tools makes the terminal decision explicit:

```text
ask_for_clarification
return_work_plan
```

Pydantic validates the returned JSON against the selected output model before the run completes.

---

## 3. Project structure

```text
work-request-triage/
├── .env.example
├── pyproject.toml
├── README.md
├── src/
│   └── triage_agent/
│       ├── __init__.py
│       ├── models.py
│       ├── agent.py
│       └── cli.py
└── tests/
    ├── test_models.py
    └── test_scenarios.py
```

Save this guide as `README.md` inside the implementation repository, or replace the `readme` value in `pyproject.toml` with the name you choose.

---

## 4. `pyproject.toml`

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "work-request-triage"
version = "0.1.0"
description = "A Pydantic AI agent that clarifies or plans unstructured work requests."
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
    "pydantic-ai>=1.0,<2.0",
]

[project.scripts]
triage = "triage_agent.cli:main"

[dependency-groups]
dev = [
    "pytest>=8.0",
]

[tool.hatch.build.targets.wheel]
packages = ["src/triage_agent"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"
```

Why a bounded major version? Pydantic AI evolves quickly. Allowing compatible `1.x` updates avoids freezing the tutorial to one patch release, while `<2.0` prevents an automatic major-version migration. Commit the generated `uv.lock` for reproducible builds.

### `.env.example`

```dotenv
OPENAI_API_KEY=replace-me
TRIAGE_MODEL=openai:gpt-5-mini
```

Use a model identifier supported by your configured provider. The environment variable keeps model selection outside application code.

---

## 5. Output contracts: `models.py`

Create `src/triage_agent/models.py`:

```python
from __future__ import annotations

from typing import Literal

from pydantic import BaseModel, Field


class ClarificationQuestion(BaseModel):
    """One focused question needed before responsible planning can continue."""

    field: str = Field(
        description="The missing or conflicting information represented by the question."
    )
    question: str = Field(description="A concise question to ask the user.")
    reason: str = Field(
        description="Why the answer is required before a plan can be produced."
    )


class ClarificationRequest(BaseModel):
    """Return when critical information is missing or requirements conflict."""

    kind: Literal["clarification"] = "clarification"
    understood_goal: str = Field(
        description="The agent's current understanding of the requested outcome."
    )
    missing_critical_information: list[str] = Field(
        min_length=1,
        description="Critical gaps that prevent responsible planning.",
    )
    questions: list[ClarificationQuestion] = Field(
        min_length=1,
        description="Focused questions required to resolve the critical gaps.",
    )
    conflicts: list[str] = Field(
        default_factory=list,
        description="Contradictory requirements found in the request.",
    )


class PlanTask(BaseModel):
    """One actionable, ordered task in a work plan."""

    order: int = Field(ge=1)
    task: str = Field(description="A concrete action to perform.")
    priority: Literal["high", "medium", "low"]
    rationale: str = Field(description="Why the task has this priority.")
    depends_on: list[str] = Field(
        default_factory=list,
        description="Exact names of earlier tasks on which this task depends.",
    )


class WorkPlan(BaseModel):
    """Return when enough critical information exists to plan responsibly."""

    kind: Literal["plan"] = "plan"
    goal: str
    assumptions: list[str] = Field(
        default_factory=list,
        description="Explicit assumptions used only for non-critical uncertainty.",
    )
    constraints: list[str] = Field(default_factory=list)
    deadline: str | None = Field(
        default=None,
        description="A deadline supplied by the user. Never invent one.",
    )
    stakeholders: list[str] = Field(
        default_factory=list,
        description="Stakeholders supplied by the user. Never invent them.",
    )
    tasks: list[PlanTask] = Field(min_length=1)
    next_steps: list[str] = Field(min_length=1)
    unresolved_optional_information: list[str] = Field(
        default_factory=list,
        description="Useful but non-blocking information that remains unknown.",
    )


TriageOutput = ClarificationRequest | WorkPlan
```

The clarification question list intentionally has no hard `max_length=3`. The prompt asks the model to prefer one to three questions, but an otherwise useful response with four questions should not exhaust retries merely because of a brittle schema limit.

### `__init__.py`

Create `src/triage_agent/__init__.py`:

```python
from .agent import triage_agent
from .models import ClarificationRequest, TriageOutput, WorkPlan

__all__ = [
    "ClarificationRequest",
    "TriageOutput",
    "WorkPlan",
    "triage_agent",
]
```

---

## 6. Agent implementation: `agent.py`

Create `src/triage_agent/agent.py`:

```python
from __future__ import annotations

import os

from pydantic_ai import Agent, ModelRetry, RunContext, ToolOutput

from .models import ClarificationRequest, TriageOutput, WorkPlan


MODEL = os.getenv("TRIAGE_MODEL", "openai:gpt-5-mini")


INSTRUCTIONS = """
You are a Work Request Triage Agent.

Transform the user's work request into exactly one final structured output:

- ask_for_clarification
- return_work_plan

Do not return ordinary conversational text as the final result.

PROCESS

UNDERSTAND -> CHECK_INFORMATION -> CLARIFY or PLAN -> COMPLETE

AIM

Convert ambiguous work requests into clear, actionable outcomes without
inventing requirements.

GATHER

Use only information available in the conversation.

Relevant information may include the objective, desired deliverable, deadline,
constraints, stakeholders, priority, and success criteria.

Classify missing information as CRITICAL or OPTIONAL.

CRITICAL means that responsible planning would require inventing an important
requirement.

OPTIONAL means planning can continue if the uncertainty is made explicit.

ASK FOR CLARIFICATION

Use ask_for_clarification when:

- the objective is unclear;
- a critical requirement is missing;
- conflicting requirements prevent a valid plan;
- the intended deliverable is unclear.

Ask the minimum number of focused questions. Prefer one to three.

RETURN A WORK PLAN

Use return_work_plan when enough critical information exists.

Do not block because optional information is absent. Put optional unknowns in
unresolved_optional_information and use assumptions only when they are safe,
explicit, and non-critical.

If the objective is impossible, do not pretend it can be completed. Record the
limitation as a constraint and propose a feasible alternative path.

When the user changes a requirement, use the latest requirement and re-plan.
Do not continue presenting the superseded requirement as current.

BOUNDARIES

You may structure work, decompose it, prioritize tasks, identify dependencies,
record assumptions, and suggest next steps.

Never invent deadlines, stakeholders, business facts, requirements, or actions
that have already been completed.

If no deadline was explicitly supplied, set deadline to null.
If no stakeholders were explicitly supplied, set stakeholders to an empty list.

PLAN QUALITY

A work plan must contain a clear goal, explicit assumptions, known constraints,
ordered and prioritized tasks, dependencies where relevant, and concrete next
steps. Never state or imply that proposed work has already happened.
"""


triage_agent: Agent[None, TriageOutput] = Agent(
    MODEL,
    instructions=INSTRUCTIONS,
    output_type=[
        ToolOutput(
            ClarificationRequest,
            name="ask_for_clarification",
            description=(
                "Use when critical information is missing or contradictory "
                "and responsible planning cannot continue."
            ),
            max_retries=3,
        ),
        ToolOutput(
            WorkPlan,
            name="return_work_plan",
            description=(
                "Use when enough critical information exists to return an "
                "actionable, prioritized plan."
            ),
            max_retries=3,
        ),
    ],
)


@triage_agent.output_validator
def validate_output(
    ctx: RunContext[None], output: TriageOutput
) -> TriageOutput:
    """Reject structurally valid but internally inconsistent work plans."""

    if isinstance(output, ClarificationRequest):
        return output

    expected_orders = list(range(1, len(output.tasks) + 1))
    actual_orders = [task.order for task in output.tasks]
    if actual_orders != expected_orders:
        raise ModelRetry(
            "Task order must be sequential, beginning at 1, with no duplicates."
        )

    previous_tasks: set[str] = set()
    for task in output.tasks:
        unknown_dependencies = set(task.depends_on) - previous_tasks
        if unknown_dependencies:
            names = ", ".join(sorted(unknown_dependencies))
            raise ModelRetry(
                "Every dependency must exactly name an earlier task. "
                f"Invalid dependencies: {names}."
            )
        previous_tasks.add(task.task)

    return output
```

### Why named `ToolOutput` objects?

Passing `[ClarificationRequest, WorkPlan]` directly is valid, but explicit output names communicate the routing decision more clearly. Each output tool also receives its own retry allowance.

### Why keep the output validator narrow?

Schema and output validators should enforce objective invariants:

- required fields exist;
- task order is sequential;
- dependencies refer to earlier tasks;
- enum values are legal.

They should not pretend to prove subjective properties such as whether an objective is genuinely clear or whether a plan is strategically good. Those belong in scenario tests, evaluation datasets, and human review.

The prompt says not to invent deadlines and stakeholders, but a plain output validator cannot reliably compare those values with the user's request. For a production-grade hard guarantee, parse explicit request facts into typed dependencies or validation context and compare the final output with those facts.

---

## 7. Custom CLI with conversation history

Create `src/triage_agent/cli.py`:

```python
from __future__ import annotations

from pydantic_ai import (
    ModelMessage,
    UnexpectedModelBehavior,
    capture_run_messages,
)

from .agent import triage_agent


def main() -> None:
    message_history: list[ModelMessage] = []

    print("Work Request Triage Agent")
    print("Enter a request, answer follow-up questions, or type 'exit'.\n")

    while True:
        try:
            user_input = input("You: ").strip()
        except (EOFError, KeyboardInterrupt):
            print("\nGoodbye.")
            break

        if user_input.lower() in {"exit", "quit"}:
            print("Goodbye.")
            break

        if not user_input:
            continue

        with capture_run_messages() as messages:
            try:
                result = triage_agent.run_sync(
                    user_input,
                    message_history=message_history,
                )
            except UnexpectedModelBehavior as exc:
                print("\n[Pydantic AI output/retry error]")
                print(exc)
                print("\nCause:")
                print(repr(exc.__cause__))
                print("\nMessages exchanged with the model:")
                for message in messages:
                    print()
                    print(message)
                print()
                continue
            except Exception as exc:
                print("\n[Error]")
                print(repr(exc))
                print()
                continue

        print("\nAgent:")
        print(result.output.model_dump_json(indent=2))
        print()

        # all_messages() includes prior history plus this completed run. Passing
        # it back lets a clarification answer continue the original request.
        message_history = result.all_messages()


if __name__ == "__main__":
    main()
```

The custom loop is preferable to a generic CLI because it:

- renders both typed outcomes as readable JSON;
- carries history between turns;
- lets a clarification answer continue the original request;
- exposes retry and validation details during development;
- handles exit, empty input, Ctrl-C, and end-of-input cleanly.

For a server-backed application, persist `result.new_messages()` per turn rather than rewriting the entire history. Validate and sanitize any history crossing an untrusted client boundary before passing it as `message_history`.

---

## 8. Complete request that should return `WorkPlan`

Paste this into the CLI as one request:

```text
We need to improve the customer onboarding process for our SaaS product.

The main problem is that many new customers abandon the setup process before
completing onboarding. We want to reduce onboarding friction and increase the
percentage of customers who successfully complete setup.

The expected deliverable is a prioritized implementation plan identifying the
main onboarding problems, recommended improvements, validation steps, and
rollout activities.

The work should be completed before October 30, 2026.

Stakeholders are:
- Product Manager
- UX Designer
- Engineering Team
- Customer Success Team

Constraints:
- We cannot redesign the entire product.
- Changes should focus only on onboarding and initial setup.
- Engineering capacity is limited, so prioritize by impact and effort.
- Existing customers should not be affected.

Priority: High.

Success criteria:
- Identify the main points where customers abandon onboarding.
- Prioritize the highest-impact onboarding improvements.
- Define measurable validation criteria for each major improvement.
- Produce a clear sequence of implementation and rollout tasks.

Create a prioritized work plan with tasks, dependencies, assumptions,
constraints, and concrete next steps.
```

This request supplies the objective, problem, deliverable, deadline, stakeholders, constraints, priority, and success criteria. The agent should select `return_work_plan`, not `ask_for_clarification`.

---

## 9. Example `WorkPlan` output

Exact wording can vary, but a good result should resemble:

```json
{
  "kind": "plan",
  "goal": "Reduce customer abandonment during SaaS onboarding and improve setup completion.",
  "assumptions": [
    "Existing onboarding analytics or customer feedback can be reviewed.",
    "The team can make targeted onboarding changes without redesigning the entire product."
  ],
  "constraints": [
    "Changes are limited to onboarding and initial setup.",
    "Engineering capacity is limited.",
    "Existing customers must not be negatively affected."
  ],
  "deadline": "October 30, 2026",
  "stakeholders": [
    "Product Manager",
    "UX Designer",
    "Engineering Team",
    "Customer Success Team"
  ],
  "tasks": [
    {
      "order": 1,
      "task": "Analyze the existing onboarding flow and identify abandonment points.",
      "priority": "high",
      "rationale": "The team needs evidence about where users encounter friction before choosing improvements.",
      "depends_on": []
    },
    {
      "order": 2,
      "task": "Collect qualitative feedback about onboarding friction.",
      "priority": "high",
      "rationale": "Customer feedback can explain why abandonment occurs at identified points.",
      "depends_on": [
        "Analyze the existing onboarding flow and identify abandonment points."
      ]
    },
    {
      "order": 3,
      "task": "Prioritize onboarding issues by customer impact and implementation effort.",
      "priority": "high",
      "rationale": "Limited engineering capacity requires focusing on the highest-value improvements.",
      "depends_on": [
        "Analyze the existing onboarding flow and identify abandonment points.",
        "Collect qualitative feedback about onboarding friction."
      ]
    },
    {
      "order": 4,
      "task": "Design targeted improvements for the highest-priority onboarding issues.",
      "priority": "medium",
      "rationale": "Solutions should address validated friction points rather than redesigning the entire product.",
      "depends_on": [
        "Prioritize onboarding issues by customer impact and implementation effort."
      ]
    },
    {
      "order": 5,
      "task": "Define validation metrics and acceptance criteria for each improvement.",
      "priority": "medium",
      "rationale": "Each change needs measurable evidence of whether onboarding completion improves.",
      "depends_on": [
        "Design targeted improvements for the highest-priority onboarding issues."
      ]
    },
    {
      "order": 6,
      "task": "Implement and test the prioritized onboarding improvements.",
      "priority": "medium",
      "rationale": "Validated improvements need controlled implementation before rollout.",
      "depends_on": [
        "Define validation metrics and acceptance criteria for each improvement."
      ]
    },
    {
      "order": 7,
      "task": "Roll out the changes and monitor onboarding completion metrics.",
      "priority": "medium",
      "rationale": "Post-release monitoring confirms whether the changes achieved the intended outcome.",
      "depends_on": [
        "Implement and test the prioritized onboarding improvements."
      ]
    }
  ],
  "next_steps": [
    "Map the current onboarding journey.",
    "Gather onboarding funnel metrics.",
    "Identify the highest abandonment points.",
    "Schedule a review with the named stakeholders."
  ],
  "unresolved_optional_information": []
}
```

Treat the output as proposed work. It must not claim that analysis, interviews, implementation, rollout, or measurement has already occurred.

---

## 10. Max-retry troubleshooting with `capture_run_messages`

`UnexpectedModelBehavior: exceeded maximum retries` usually means the model repeatedly produced output that failed one of these layers:

1. JSON/tool argument parsing;
2. Pydantic schema validation;
3. the custom output validator;
4. the requirement to select a structured output instead of plain text.

Pydantic AI's output retry budget defaults to one. This implementation gives each named `ToolOutput` a budget of three. More retries can help with occasional formatting mistakes, but they should not conceal a schema or prompt that is systematically rejecting reasonable answers.

The CLI wraps each run in:

```python
with capture_run_messages() as messages:
    try:
        result = triage_agent.run_sync(
            user_input,
            message_history=message_history,
        )
    except UnexpectedModelBehavior as exc:
        print(exc)
        print(repr(exc.__cause__))
        for message in messages:
            print(message)
```

Look for a retry prompt containing details such as:

```text
RetryPromptPart(
    content=[
        {
            "type": "missing",
            "loc": ["tasks"],
            ...
        }
    ]
)
```

or:

```text
Every dependency must exactly name an earlier task.
```

Troubleshooting sequence:

1. Read the last retry prompt and locate the rejected field or validator message.
2. Decide whether the output is truly invalid or the contract is unnecessarily strict.
3. If the contract is too strict, loosen it. Prefer prompt guidance for style preferences.
4. If the model chose ordinary text, make the two named output choices more explicit in the instructions.
5. If a validator is rejecting most reasonable plans, reduce it to objective invariants.
6. Test a smaller model-independent contract using `TestModel` or `FunctionModel`.
7. Only then increase retries, and keep the increase bounded.

Common causes in this project:

- enforcing a maximum of three clarification questions in the schema;
- dependencies that do not exactly match task names;
- duplicated or skipped task order numbers;
- asking for a plan while the prompt still tells the model to explain its choice in prose;
- using a model/provider combination without compatible tool calling;
- an invalid or unavailable model name;
- missing provider credentials.

`capture_run_messages()` is diagnostic. Do not log full captured prompts in production without considering confidential user data.

---

## 11. Output validation strategy

Use three validation layers.

### Layer 1: Pydantic schema validation

Good for facts that are local to one field or object:

- `kind` is a legal literal;
- priority is `high`, `medium`, or `low`;
- task order is at least one;
- at least one task and next step exist;
- clarification contains at least one gap and question.

### Layer 2: Pydantic AI output validation

Good for cross-field invariants:

- order values are exactly `1..n`;
- dependencies refer only to earlier tasks;
- output-specific semantic checks that can return `ModelRetry`.

Keep validator error messages actionable because the message is sent back to the model for correction.

### Layer 3: Behavioral evaluation

Good for requirements that depend on meaning:

- did the agent clarify only when necessary?
- did it preserve explicit constraints?
- did it avoid fabricated deadlines or stakeholders?
- did it surface contradictions?
- did it replace a superseded requirement?
- did it propose a feasible path for an impossible objective?
- is the plan useful and prioritized?

Deterministic pytest tests should verify application wiring and contracts. A separate live evaluation suite should run representative prompts against the selected production model, record outputs, and score them with deterministic checks plus human or rubric-based review. Do not make normal unit tests depend on live model calls.

---

## 12. Contract tests: `test_models.py`

Create `tests/test_models.py`:

```python
import pytest
from pydantic import ValidationError

from triage_agent.models import (
    ClarificationRequest,
    PlanTask,
    WorkPlan,
)


def test_clarification_requires_a_gap_and_question() -> None:
    with pytest.raises(ValidationError):
        ClarificationRequest(
            understood_goal="Improve onboarding",
            missing_critical_information=[],
            questions=[],
        )


def test_plan_defaults_do_not_invent_deadline_or_stakeholders() -> None:
    plan = WorkPlan(
        goal="Reduce onboarding friction",
        tasks=[
            PlanTask(
                order=1,
                task="Review the onboarding flow",
                priority="high",
                rationale="Identify friction before selecting changes.",
            )
        ],
        next_steps=["Review the current flow"],
    )

    assert plan.kind == "plan"
    assert plan.deadline is None
    assert plan.stakeholders == []


def test_plan_requires_at_least_one_task() -> None:
    with pytest.raises(ValidationError):
        WorkPlan(
            goal="Improve onboarding",
            tasks=[],
            next_steps=["Start"],
        )
```

---

## 13. Pytest coverage for all seven A.G.E.N.T. scenarios

Create `tests/test_scenarios.py`:

```python
from __future__ import annotations

from collections.abc import Callable
from typing import Any

from pydantic_ai import (
    ModelMessage,
    ModelResponse,
    ToolCallPart,
    models,
)
from pydantic_ai.models.function import AgentInfo, FunctionModel

from triage_agent.agent import triage_agent
from triage_agent.models import ClarificationRequest, WorkPlan


# Prevent an accidental billable/network model call anywhere in this test module.
models.ALLOW_MODEL_REQUESTS = False


def task(name: str, *, constraint: str | None = None) -> dict[str, Any]:
    rationale = "This is the first actionable step."
    if constraint:
        rationale = f"This step respects the constraint: {constraint}"
    return {
        "order": 1,
        "task": name,
        "priority": "high",
        "rationale": rationale,
        "depends_on": [],
    }


def plan(
    goal: str,
    *,
    task_name: str = "Assess the current state",
    constraints: list[str] | None = None,
    unresolved: list[str] | None = None,
) -> dict[str, Any]:
    return {
        "kind": "plan",
        "goal": goal,
        "assumptions": [],
        "constraints": constraints or [],
        "deadline": None,
        "stakeholders": [],
        "tasks": [task(task_name)],
        "next_steps": [task_name],
        "unresolved_optional_information": unresolved or [],
    }


def clarification(
    goal: str,
    missing: str,
    question: str,
    *,
    conflicts: list[str] | None = None,
) -> dict[str, Any]:
    return {
        "kind": "clarification",
        "understood_goal": goal,
        "missing_critical_information": [missing],
        "questions": [
            {
                "field": "request",
                "question": question,
                "reason": "This answer is required for responsible planning.",
            }
        ],
        "conflicts": conflicts or [],
    }


def fake_model(
    route: Callable[[list[ModelMessage]], tuple[str, dict[str, Any]]]
) -> FunctionModel:
    def respond(
        messages: list[ModelMessage], info: AgentInfo
    ) -> ModelResponse:
        del info
        output_name, payload = route(messages)
        return ModelResponse(parts=[ToolCallPart(output_name, payload)])

    return FunctionModel(respond)


def fixed(
    output_name: str, payload: dict[str, Any]
) -> FunctionModel:
    return fake_model(lambda messages: (output_name, payload))


def test_complete_request_returns_structured_plan() -> None:
    model = fixed(
        "return_work_plan",
        plan(
            "Reduce customer onboarding abandonment",
            task_name="Identify onboarding abandonment points",
        ),
    )

    with triage_agent.override(model=model):
        result = triage_agent.run_sync(
            "Reduce onboarding abandonment and return an implementation plan."
        )

    assert isinstance(result.output, WorkPlan)
    assert result.output.goal == "Reduce customer onboarding abandonment"


def test_very_vague_request_asks_focused_clarification() -> None:
    model = fixed(
        "ask_for_clarification",
        clarification(
            "Improve something",
            "The objective is unclear.",
            "What outcome do you want to achieve?",
        ),
    )

    with triage_agent.override(model=model):
        result = triage_agent.run_sync("Make it better.")

    assert isinstance(result.output, ClarificationRequest)
    assert len(result.output.questions) == 1


def test_missing_optional_information_does_not_block() -> None:
    model = fixed(
        "return_work_plan",
        plan(
            "Reduce onboarding friction",
            unresolved=["Exact implementation budget"],
        ),
    )

    with triage_agent.override(model=model):
        result = triage_agent.run_sync(
            "Create a plan to reduce onboarding friction. The budget is not known yet."
        )

    assert isinstance(result.output, WorkPlan)
    assert "Exact implementation budget" in (
        result.output.unresolved_optional_information
    )


def test_missing_essential_information_asks_for_it() -> None:
    model = fixed(
        "ask_for_clarification",
        clarification(
            "Prepare a deliverable",
            "The intended deliverable is missing.",
            "What deliverable should this work produce?",
        ),
    )

    with triage_agent.override(model=model):
        result = triage_agent.run_sync(
            "Please handle the onboarding issue for leadership."
        )

    assert isinstance(result.output, ClarificationRequest)
    assert "deliverable" in result.output.questions[0].question.lower()


def test_contradictory_requirements_surface_conflict() -> None:
    conflict = (
        "The request requires changing the onboarding flow while also "
        "prohibiting every onboarding change."
    )
    model = fixed(
        "ask_for_clarification",
        clarification(
            "Improve onboarding without changing onboarding",
            "A blocking contradiction must be resolved.",
            "Which requirement takes precedence?",
            conflicts=[conflict],
        ),
    )

    with triage_agent.override(model=model):
        result = triage_agent.run_sync(
            "Redesign onboarding, but do not change any onboarding behavior or content."
        )

    assert isinstance(result.output, ClarificationRequest)
    assert result.output.conflicts == [conflict]


def test_changed_requirement_replans_from_new_history() -> None:
    calls = 0

    def route(
        messages: list[ModelMessage],
    ) -> tuple[str, dict[str, Any]]:
        nonlocal calls
        calls += 1
        if calls == 1:
            return (
                "return_work_plan",
                plan("Prepare a webinar", task_name="Outline the webinar"),
            )
        return (
            "return_work_plan",
            plan("Prepare a written guide", task_name="Outline the written guide"),
        )

    with triage_agent.override(model=fake_model(route)):
        first = triage_agent.run_sync("Prepare a customer education webinar.")
        second = triage_agent.run_sync(
            "Change the deliverable to a written guide instead.",
            message_history=first.all_messages(),
        )

    assert isinstance(first.output, WorkPlan)
    assert isinstance(second.output, WorkPlan)
    assert first.output.goal == "Prepare a webinar"
    assert second.output.goal == "Prepare a written guide"
    assert "webinar" not in second.output.goal.lower()


def test_impossible_objective_explains_limit_and_feasible_path() -> None:
    limitation = "A zero-defect guarantee cannot be established for all future releases."
    model = fixed(
        "return_work_plan",
        plan(
            "Reduce defects and establish measurable release quality gates",
            task_name="Define a measurable defect-reduction target",
            constraints=[limitation],
        ),
    )

    with triage_agent.override(model=model):
        result = triage_agent.run_sync(
            "Guarantee that every future release has zero defects."
        )

    assert isinstance(result.output, WorkPlan)
    assert limitation in result.output.constraints
    assert "measurable" in result.output.goal.lower()
```

These are deterministic application tests. `FunctionModel` replaces the network model and returns controlled output-tool calls, so the tests prove that:

- both routes deserialize to the correct Pydantic model;
- the agent accepts the expected result for each source scenario;
- conversation history supports changed requirements;
- optional gaps remain non-blocking;
- contradictions and infeasibility have explicit output locations;
- no real model request can occur accidentally.

They do **not** prove that every production model will reason correctly from unseen wording. Add a live evaluation dataset for that purpose.

---

## 14. Optional live behavioral evaluation

Keep live evaluation separate from unit tests. A useful case record contains:

```json
{
  "name": "contradictory_requirements",
  "input": "Redesign onboarding, but do not change any onboarding behavior or content.",
  "expected_kind": "clarification",
  "checks": [
    "conflicts is not empty",
    "question asks which requirement takes precedence",
    "no work is claimed complete"
  ]
}
```

For each of the seven scenarios, evaluate:

- output type accuracy;
- critical versus optional gap classification;
- fabricated deadline count;
- fabricated stakeholder count;
- contradiction detection;
- preservation of the latest requirement;
- task actionability and ordering;
- feasible-alternative quality;
- consistency across repeated runs;
- latency and token usage.

Run live evaluations manually or in a controlled CI job with credentials and cost limits. Store the model identifier, dependency lockfile, prompts, outputs, and scoring results so regressions can be investigated.

---

## 15. Install and run

### Create the project

```bash
mkdir work-request-triage
cd work-request-triage
uv init --bare
```

Add the files from this guide, then install and lock dependencies:

```bash
uv sync --dev
```

### Configure credentials

macOS or Linux:

```bash
export OPENAI_API_KEY="your-api-key"
export TRIAGE_MODEL="openai:gpt-5-mini"
```

PowerShell:

```powershell
$env:OPENAI_API_KEY = "your-api-key"
$env:TRIAGE_MODEL = "openai:gpt-5-mini"
```

### Run the CLI

Using the installed console entry point:

```bash
uv run triage
```

Or run the module directly:

```bash
uv run python -m triage_agent.cli
```

### Run tests

```bash
uv run pytest
```

Run only the seven scenario tests:

```bash
uv run pytest tests/test_scenarios.py -q
```

Run one scenario while debugging:

```bash
uv run pytest tests/test_scenarios.py::test_changed_requirement_replans_from_new_history -q
```

---

## 16. Expected interactive flow

```text
$ uv run triage
Work Request Triage Agent
Enter a request, answer follow-up questions, or type 'exit'.

You: We need to improve onboarding.

Agent:
{
  "kind": "clarification",
  "understood_goal": "Improve the onboarding process.",
  "missing_critical_information": [
    "The problem to solve and desired outcome are unclear."
  ],
  "questions": [
    {
      "field": "objective",
      "question": "What problem in onboarding are you trying to solve, and what outcome should improve?",
      "reason": "A specific outcome is required to create an actionable plan."
    }
  ],
  "conflicts": []
}

You: New customers abandon setup because it takes too long. We want to reduce setup friction before our October launch.

Agent:
{
  "kind": "plan",
  "goal": "Reduce setup friction that causes new customers to abandon onboarding before the October launch.",
  "assumptions": [],
  "constraints": [
    "Changes need to be prepared before the October launch."
  ],
  "deadline": null,
  "stakeholders": [],
  "tasks": [
    {
      "order": 1,
      "task": "Map the current onboarding flow and identify the highest-friction steps.",
      "priority": "high",
      "rationale": "The team needs evidence about where setup friction occurs before choosing changes.",
      "depends_on": []
    }
  ],
  "next_steps": [
    "Document the current onboarding flow.",
    "Collect setup abandonment data."
  ],
  "unresolved_optional_information": [
    "Exact October launch date",
    "Available implementation capacity"
  ]
}
```

Notice that “October launch” is not converted into an invented October 1, 15, or 31 deadline. Therefore `deadline` remains `null` while the timing is preserved as a constraint.

---

## 17. Production hardening checklist

Before using this beyond a learning project:

- pin and commit the dependency lockfile;
- select and test a production model/provider combination;
- add timeouts and application-level usage limits;
- redact secrets and personal data from diagnostic logs;
- persist message history by authenticated conversation ID;
- sanitize history received from an untrusted client;
- add observability for retries, output route, latency, tokens, and failures;
- version prompts and schemas together;
- build a live evaluation dataset from real, approved examples;
- add regression cases whenever the agent makes a premature assumption;
- define human review rules for high-impact plans;
- document data retention and access controls.

---

## 18. Definition of done

The MVP is complete when all of the following are true:

### Behavior

- A complete request returns `WorkPlan` directly.
- A vague request returns a minimal `ClarificationRequest`.
- Missing optional information does not unnecessarily block planning.
- Missing essential information produces a focused question.
- Contradictory requirements are explicitly surfaced.
- A changed requirement produces a revised plan based on current history.
- An impossible objective records the limitation and proposes a feasible path.

### Correctness and boundaries

- Every run ends with exactly one typed output.
- No generic free-text terminal response is accepted.
- The agent does not invent a deadline or stakeholders.
- The plan contains at least one actionable task and next step.
- Task order is sequential.
- Dependencies name earlier tasks.
- Proposed actions are not described as completed work.

### CLI

- `uv run triage` starts the interactive application.
- Follow-up answers reuse the original message history.
- `exit`, `quit`, Ctrl-C, and EOF end cleanly.
- Retry failures show captured messages in development.

### Tests

- `uv run pytest` passes.
- Model requests are disabled in deterministic tests.
- All seven source-defined scenarios are represented.
- Contract tests cover required lists and safe defaults.

### Reproducibility

- `pyproject.toml` is valid.
- `uv.lock` is committed.
- `.env.example` documents required configuration without containing a secret.
- A fresh checkout can be installed, tested, and run using the commands in this guide.

---

## 19. Design rationale and next step

This implementation is a genuine minimum viable agent rather than a chatbot with a large prompt. It has:

- a defined job;
- two legal terminal outcomes;
- explicit state transitions;
- typed contracts;
- bounded retries;
- message history;
- validation boundaries;
- deterministic scenario coverage;
- clear stop conditions.

The strongest next improvement is not adding tools. It is building a small live evaluation dataset for the seven scenarios, running it against the chosen production model, and tightening prompts or validators only when measured failures justify the change.

---

## References

- A.G.E.N.T. source design: *Four Practical Agentic AI Projects Using the A.G.E.N.T. Framework*, Project 1, “Work Request Triage Agent.”
- Pydantic AI output documentation: <https://pydantic.dev/docs/ai/core-concepts/output/>
- Pydantic AI message-history documentation: <https://pydantic.dev/docs/ai/core-concepts/message-history/>
- Pydantic AI testing documentation: <https://pydantic.dev/docs/ai/guides/testing/>
