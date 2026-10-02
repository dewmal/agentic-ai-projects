# AI Incident Response Team — Groq + Pydantic AI

## Step-by-Step Workshop Build Guide

This guide builds the **AI Incident Response Team** as a working three-agent project using:

- Python 3.11+
- Pydantic
- Pydantic AI
- Groq
- Typed agent outputs
- Agent tools
- Deterministic application-controlled handoffs
- Mock observability data
- Human approval for high-risk actions

The architecture contains **exactly three agents**:

1. **Triage Agent** — understands the incident and decides where to investigate.
2. **Investigator Agent** — gathers evidence using tools and forms supported hypotheses.
3. **Response Coordinator** — creates a safe response plan and identifies actions requiring approval.

The agents do **not** freely chat with one another.

The Python application controls the handoffs:

```text
IncidentReport
      |
      v
Triage Agent
      |
      v
TriageReport
      |
      v
Investigator Agent
      |
      v
EvidenceReport
      |
      v
Response Coordinator
      |
      v
ResponsePlan
      |
      +----> Human Approval
      |
      +----> Complete
```

---

# Step 0 — Prerequisites

You need:

- Python 3.11 or later
- `uv` installed
- A Groq account
- A Groq API key

Create a Groq API key from the Groq Console.

We will use this environment variable:

```bash
GROQ_API_KEY
```

The default model in this workshop is:

```text
llama-3.3-70b-versatile
```

Pydantic AI supports Groq models using the `groq:` model prefix.

---

# Step 1 — Create the Project

Create the project folder:

```bash
mkdir incident-response-groq
cd incident-response-groq
```

Initialize it:

```bash
uv init
```

Create the source folders:

```bash
mkdir -p src/incident_team/models
mkdir -p src/incident_team/agents
mkdir -p src/incident_team/tools
mkdir -p tests
```

Create empty package files:

```bash
touch src/incident_team/__init__.py
touch src/incident_team/models/__init__.py
touch src/incident_team/agents/__init__.py
touch src/incident_team/tools/__init__.py
```

Your project will eventually look like this:

```text
incident-response-groq/
├── .env
├── .env.example
├── pyproject.toml
├── src/
│   └── incident_team/
│       ├── __init__.py
│       ├── config.py
│       ├── dependencies.py
│       ├── orchestrator.py
│       ├── cli.py
│       ├── models/
│       │   ├── __init__.py
│       │   ├── incident.py
│       │   ├── triage.py
│       │   ├── investigation.py
│       │   ├── response.py
│       │   └── state.py
│       ├── agents/
│       │   ├── __init__.py
│       │   ├── triage_agent.py
│       │   ├── investigator_agent.py
│       │   └── response_agent.py
│       └── tools/
│           ├── __init__.py
│           ├── logs.py
│           ├── metrics.py
│           ├── deployments.py
│           └── runbooks.py
└── tests/
    ├── test_models.py
    └── test_tools.py
```

---

# Step 2 — Install the Dependencies

Replace `pyproject.toml` with:

```toml
[project]
name = "incident-response-groq"
version = "0.1.0"
description = "Three-agent AI incident response workshop using Groq and Pydantic AI"
requires-python = ">=3.11"
dependencies = [
    "pydantic>=2",
    "pydantic-ai-slim[groq]",
    "python-dotenv>=1",
]

[project.optional-dependencies]
dev = [
    "pytest>=8",
    "pytest-asyncio>=0.23",
]

[project.scripts]
incident-team = "incident_team.cli:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/incident_team"]

[tool.pytest.ini_options]
pythonpath = ["src"]
asyncio_mode = "auto"
```

Install everything:

```bash
uv sync --extra dev
```

### Checkpoint

Run:

```bash
uv run python -c "import pydantic_ai; print('Pydantic AI installed')"
```

Expected:

```text
Pydantic AI installed
```

---

# Step 3 — Configure Groq

Create `.env.example`:

```env
GROQ_API_KEY=your-groq-api-key
GROQ_MODEL=llama-3.3-70b-versatile
```

Copy it:

```bash
cp .env.example .env
```

Open `.env` and add your actual Groq API key:

```env
GROQ_API_KEY=gsk_your_actual_key_here
GROQ_MODEL=llama-3.3-70b-versatile
```

Do not commit `.env` to source control.

Create `src/incident_team/config.py`:

```python
import os

from dotenv import load_dotenv


load_dotenv()

GROQ_MODEL = os.getenv(
    "GROQ_MODEL",
    "llama-3.3-70b-versatile",
)

MODEL_NAME = f"groq:{GROQ_MODEL}"
```

### What this does

Every agent can now use:

```python
MODEL_NAME
```

which becomes:

```text
groq:llama-3.3-70b-versatile
```

Pydantic AI automatically uses `GROQ_API_KEY`.

---

# Step 4 — Define the Incident Input

Create:

```text
src/incident_team/models/incident.py
```

Add:

```python
from datetime import datetime

from pydantic import BaseModel, Field


class IncidentReport(BaseModel):
    incident_id: str
    description: str
    reported_at: datetime
    reporter: str | None = None
    affected_service: str | None = None
    observed_symptoms: list[str] = Field(default_factory=list)
```

This is the starting contract.

Example:

```json
{
  "incident_id": "INC-1001",
  "description": "Payments API error rate increased after deployment",
  "affected_service": "payments-api",
  "observed_symptoms": [
    "HTTP 500 errors",
    "increased latency",
    "database connection errors"
  ]
}
```

Important:

> User-reported symptoms are input claims. They are not automatically root-cause evidence.

---

# Step 5 — Define the Triage Contract

Create:

```text
src/incident_team/models/triage.py
```

Add:

```python
from typing import Literal

from pydantic import BaseModel, Field


class InvestigationArea(BaseModel):
    area: str
    reason: str
    priority: Literal["high", "medium", "low"]


class TriageReport(BaseModel):
    incident_id: str
    summary: str
    symptoms: list[str]
    affected_systems: list[str]

    severity: Literal[
        "SEV1",
        "SEV2",
        "SEV3",
        "SEV4",
        "unknown",
    ]

    possible_impact: list[str]

    incident_start_time: str | None = None

    missing_information: list[str] = Field(
        default_factory=list
    )

    investigation_areas: list[InvestigationArea]

    # Triage is intentionally prevented from
    # declaring a root cause.
    root_cause: None = None
```

The most important rule is:

```text
Triage identifies where to investigate.
Triage does NOT declare the root cause.
```

---

# Step 6 — Define the Investigation Contracts

Create:

```text
src/incident_team/models/investigation.py
```

Add:

```python
from typing import Literal

from pydantic import BaseModel, Field


class EvidenceItem(BaseModel):
    source_type: Literal[
        "log",
        "metric",
        "deployment",
        "runbook",
        "configuration",
        "incident_record",
    ]

    source_reference: str
    observation: str
    observed_at: str | None = None


class Hypothesis(BaseModel):
    hypothesis: str

    supporting_evidence: list[EvidenceItem] = Field(
        default_factory=list
    )

    contradicting_evidence: list[EvidenceItem] = Field(
        default_factory=list
    )

    confidence: Literal[
        "low",
        "medium",
        "high",
    ]

    missing_evidence: list[str] = Field(
        default_factory=list
    )


class EvidenceReport(BaseModel):
    incident_id: str

    observed_facts: list[EvidenceItem]

    hypotheses: list[Hypothesis]

    strongest_supported_hypothesis: str | None

    unresolved_questions: list[str]

    additional_evidence_needed: list[str]

    investigation_complete: bool
```

The key separation is:

```text
FACT
    comes from a tool result

HYPOTHESIS
    interpretation of one or more facts
```

A strong hypothesis is still not automatically an established root cause.

---

# Step 7 — Define the Response Contract

Create:

```text
src/incident_team/models/response.py
```

Add:

```python
from typing import Literal

from pydantic import BaseModel


class ResponseAction(BaseModel):
    order: int
    action: str
    reason: str

    risk: Literal[
        "low",
        "medium",
        "high",
    ]

    requires_approval: bool
    verification: str


class ResponsePlan(BaseModel):
    incident_id: str

    incident_summary: str

    evidence_summary: list[str]

    immediate_actions: list[ResponseAction]

    mitigation_or_rollback: list[str]

    monitoring_steps: list[str]

    escalation_conditions: list[str]

    stakeholder_communication: list[str]

    approval_required: bool

    approval_reason: str | None = None

    unresolved_risks: list[str]
```

The Response Coordinator proposes operational actions.

It does **not** claim that production changes have already happened.

---

# Step 8 — Define the Workflow State

Create:

```text
src/incident_team/models/state.py
```

Add:

```python
from typing import Literal

from pydantic import BaseModel

from .incident import IncidentReport
from .investigation import EvidenceReport
from .response import ResponsePlan
from .triage import TriageReport


class IncidentState(BaseModel):
    incident: IncidentReport

    triage: TriageReport | None = None

    investigation: EvidenceReport | None = None

    response: ResponsePlan | None = None

    status: Literal[
        "triage",
        "investigating",
        "response_planning",
        "awaiting_approval",
        "complete",
        "needs_more_context",
    ]
```

Our first implementation follows:

```text
START
  |
  v
TRIAGE
  |
  v
INVESTIGATE
  |
  v
RESPONSE_PLAN
  |
  +------> AWAITING_APPROVAL
  |
  +------> COMPLETE
```

Later we can add investigation loops.

---

# Step 9 — Create Mock Log Data

We do not want workshop participants connecting directly to real production systems.

For the MVP, the Investigator receives safe, read-only mock tools.

Create:

```text
src/incident_team/tools/logs.py
```

Add:

```python
from incident_team.models.investigation import EvidenceItem


class MockLogClient:
    async def query(
        self,
        service: str,
        query: str,
        minutes: int = 30,
    ) -> list[EvidenceItem]:

        if service != "payments-api":
            return []

        return [
            EvidenceItem(
                source_type="log",
                source_reference=(
                    f"logs://{service}"
                    f"?query={query}"
                    f"&minutes={minutes}"
                ),
                observation=(
                    "Payments API recorded repeated "
                    "'connection pool exhausted' errors."
                ),
                observed_at="2026-10-02T09:06:00Z",
            ),
            EvidenceItem(
                source_type="log",
                source_reference=(
                    f"logs://{service}"
                    f"?query={query}"
                    f"&minutes={minutes}"
                ),
                observation=(
                    "HTTP 500 responses increased shortly "
                    "after the connection errors appeared."
                ),
                observed_at="2026-10-02T09:07:00Z",
            ),
        ]
```

This simulates a log system.

Notice that the tool returns **typed evidence**.

---

# Step 10 — Create Mock Metrics

Create:

```text
src/incident_team/tools/metrics.py
```

Add:

```python
from incident_team.models.investigation import EvidenceItem


class MockMetricsClient:
    async def query(
        self,
        service: str,
        metric: str,
        minutes: int = 60,
    ) -> list[EvidenceItem]:

        if service != "payments-api":
            return []

        metric_name = metric.lower()

        if "error" in metric_name:
            observation = (
                "HTTP 5xx error rate increased from "
                "0.4% to 12.8%."
            )

        elif "latency" in metric_name:
            observation = (
                "P95 latency increased from "
                "240 ms to 2.8 seconds."
            )

        elif "connection" in metric_name:
            observation = (
                "Database connection-pool utilization "
                "rose above 95%."
            )

        else:
            observation = (
                f"Metric '{metric}' changed materially "
                "during the incident window."
            )

        return [
            EvidenceItem(
                source_type="metric",
                source_reference=(
                    f"metrics://{service}/{metric}"
                    f"?minutes={minutes}"
                ),
                observation=observation,
                observed_at="2026-10-02T09:08:00Z",
            )
        ]
```

---

# Step 11 — Create Mock Deployment Data

Create:

```text
src/incident_team/tools/deployments.py
```

Add:

```python
from incident_team.models.investigation import EvidenceItem


class MockDeploymentClient:
    async def recent(
        self,
        service: str,
    ) -> list[EvidenceItem]:

        if service != "payments-api":
            return []

        return [
            EvidenceItem(
                source_type="deployment",
                source_reference=(
                    "deploy://payments-api/release-2026.10.02.1"
                ),
                observation=(
                    "Version 2026.10.02.1 was deployed "
                    "approximately six minutes before "
                    "the first reported errors."
                ),
                observed_at="2026-10-02T09:00:00Z",
            )
        ]
```

Important:

```text
deployment happened before errors
```

does **not** automatically mean:

```text
deployment caused errors
```

The Investigator still needs evidence.

---

# Step 12 — Create a Mock Runbook Tool

Create:

```text
src/incident_team/tools/runbooks.py
```

Add:

```python
from incident_team.models.investigation import EvidenceItem


class MockRunbookClient:
    async def search(
        self,
        service: str,
        topic: str,
    ) -> list[EvidenceItem]:

        if service != "payments-api":
            return []

        return [
            EvidenceItem(
                source_type="runbook",
                source_reference=(
                    "runbook://payments-api/"
                    "database-connection-exhaustion"
                ),
                observation=(
                    "For connection-pool exhaustion, "
                    "verify pool utilization, recent "
                    "configuration/deployment changes, "
                    "and database health before rollback."
                ),
            )
        ]
```

The runbook is guidance.

It is not evidence that an event actually happened.

---

# Step 13 — Combine Tool Dependencies

Create:

```text
src/incident_team/dependencies.py
```

Add:

```python
from dataclasses import dataclass

from incident_team.tools.deployments import (
    MockDeploymentClient,
)
from incident_team.tools.logs import MockLogClient
from incident_team.tools.metrics import MockMetricsClient
from incident_team.tools.runbooks import MockRunbookClient


@dataclass
class InvestigatorDependencies:
    logs: MockLogClient
    metrics: MockMetricsClient
    deployments: MockDeploymentClient
    runbooks: MockRunbookClient
```

This dependency object will be injected into the Investigator Agent.

---

# Step 14 — Build Agent 1: Triage Agent

Create:

```text
src/incident_team/agents/triage_agent.py
```

Add:

```python
from pydantic_ai import Agent, ModelRetry

from incident_team.config import MODEL_NAME
from incident_team.models.triage import TriageReport


triage_agent = Agent(
    MODEL_NAME,
    output_type=TriageReport,
    instructions="""
You are the Incident Triage Agent.

Your job is to determine:

1. What appears to be happening.
2. Which systems may be affected.
3. The likely impact and severity.
4. What important information is missing.
5. Where investigation should begin.

RULES

- Do not declare a root cause.
- Do not invent logs, metrics, deployments, or system state.
- A plausible explanation is not an established cause.
- Use "unknown" when the available incident report is insufficient.
- Keep investigation areas concrete and useful.
- Return a structured TriageReport.
""",
)


@triage_agent.output_validator
async def validate_triage(
    output: TriageReport,
) -> TriageReport:

    if output.root_cause is not None:
        raise ModelRetry(
            "Triage must not declare a root cause."
        )

    return output
```

### Agent responsibility

The Triage Agent answers:

> What appears to be happening, and where should we investigate?

It does not have operational tools.

That is intentional.

---

# Step 15 — Build Agent 2: Investigator Agent

Create:

```text
src/incident_team/agents/investigator_agent.py
```

Add:

```python
from pydantic_ai import Agent, ModelRetry, RunContext

from incident_team.config import MODEL_NAME
from incident_team.dependencies import (
    InvestigatorDependencies,
)
from incident_team.models.investigation import (
    EvidenceItem,
    EvidenceReport,
)


investigator_agent = Agent(
    MODEL_NAME,
    deps_type=InvestigatorDependencies,
    output_type=EvidenceReport,
    instructions="""
You are the Investigator Agent.

You receive:

- the original IncidentReport
- a typed TriageReport

Your responsibility is to determine what evidence
supports or contradicts possible causes.

PROCESS

1. Review the incident and triage report.
2. Do not blindly accept triage assumptions.
3. Use tools to gather relevant evidence.
4. Separate observed facts from hypotheses.
5. Create one or more hypotheses.
6. Search for supporting evidence.
7. Look for contradicting evidence.
8. Preserve uncertainty.
9. Identify missing evidence.
10. Decide whether investigation is sufficiently complete.

TOOL USE

For an affected service, normally inspect:

- logs
- error-rate metrics
- latency metrics
- connection/resource metrics when relevant
- recent deployments
- relevant runbooks

FACT VS HYPOTHESIS

Observed facts must come from retrieved evidence.

Never present a hypothesis as an observed fact.

A deployment preceding an incident is correlation,
not automatically causation.

CONFIDENCE

High confidence requires strong supporting evidence
and no major unresolved contradiction.

MISSING DATA

Never fabricate logs, metrics, deployments,
configuration, or monitoring data.

Return a structured EvidenceReport.
""",
)


@investigator_agent.tool
async def query_logs(
    ctx: RunContext[InvestigatorDependencies],
    service: str,
    query: str,
    minutes: int = 30,
) -> list[EvidenceItem]:
    """Query application logs for incident evidence."""

    return await ctx.deps.logs.query(
        service=service,
        query=query,
        minutes=minutes,
    )


@investigator_agent.tool
async def query_metrics(
    ctx: RunContext[InvestigatorDependencies],
    service: str,
    metric: str,
    minutes: int = 60,
) -> list[EvidenceItem]:
    """Query monitoring metrics for incident evidence."""

    return await ctx.deps.metrics.query(
        service=service,
        metric=metric,
        minutes=minutes,
    )


@investigator_agent.tool
async def get_recent_deployments(
    ctx: RunContext[InvestigatorDependencies],
    service: str,
) -> list[EvidenceItem]:
    """Get recent deployments for a service."""

    return await ctx.deps.deployments.recent(
        service=service
    )


@investigator_agent.tool
async def get_runbook(
    ctx: RunContext[InvestigatorDependencies],
    service: str,
    topic: str,
) -> list[EvidenceItem]:
    """Retrieve read-only operational guidance."""

    return await ctx.deps.runbooks.search(
        service=service,
        topic=topic,
    )


@investigator_agent.output_validator
async def validate_investigation(
    output: EvidenceReport,
) -> EvidenceReport:

    for hypothesis in output.hypotheses:

        if (
            hypothesis.confidence == "high"
            and not hypothesis.supporting_evidence
        ):
            raise ModelRetry(
                "High-confidence hypotheses require "
                "supporting evidence."
            )

    return output
```

### Important design decision

The Investigator has **read-only evidence tools**.

We do not give it:

```text
restart_service()
rollback_deployment()
delete_resource()
change_configuration()
deploy_version()
```

The agent investigates.

It does not mutate production.

---

# Step 16 — Build Agent 3: Response Coordinator

Create:

```text
src/incident_team/agents/response_agent.py
```

Add:

```python
from pydantic_ai import Agent, ModelRetry

from incident_team.config import MODEL_NAME
from incident_team.models.response import ResponsePlan


response_agent = Agent(
    MODEL_NAME,
    output_type=ResponsePlan,
    instructions="""
You are the Incident Response Coordinator.

You receive:

- concise incident context
- the Investigator's typed EvidenceReport

Your job is to create the safest operational
response plan supported by the evidence.

Include:

1. Immediate actions.
2. Verification for every action.
3. Mitigation or rollback recommendations.
4. Monitoring steps.
5. Escalation conditions.
6. Stakeholder communication.
7. Remaining risks.
8. Approval requirements.

SAFETY RULES

- Do not claim that a rollback occurred.
- Do not claim that a restart occurred.
- Do not claim that a deployment occurred.
- Do not claim that a traffic shift occurred.
- Do not claim that a configuration change occurred
  unless an authoritative execution system confirms it.

- High-risk production actions require human approval.

- When evidence is uncertain, prefer:
  * reversible actions
  * additional verification
  * lower-risk mitigation

Return a structured ResponsePlan.
""",
)


@response_agent.output_validator
async def validate_response(
    output: ResponsePlan,
) -> ResponsePlan:

    risky_actions = [
        action
        for action in output.immediate_actions
        if action.risk == "high"
    ]

    if risky_actions and not output.approval_required:
        raise ModelRetry(
            "High-risk remediation requires "
            "human approval."
        )

    for action in output.immediate_actions:
        if not action.verification.strip():
            raise ModelRetry(
                "Every response action must include "
                "a verification step."
            )

    return output
```

The Response Coordinator answers:

> Given the evidence, what is the safest next operational plan?

It recommends.

It does not silently execute dangerous actions.

---

# Step 17 — Build the Deterministic Orchestrator

Create:

```text
src/incident_team/orchestrator.py
```

Add:

```python
from incident_team.agents.investigator_agent import (
    investigator_agent,
)
from incident_team.agents.response_agent import (
    response_agent,
)
from incident_team.agents.triage_agent import triage_agent
from incident_team.dependencies import (
    InvestigatorDependencies,
)
from incident_team.models.incident import IncidentReport
from incident_team.models.state import IncidentState


async def handle_incident(
    incident: IncidentReport,
    investigator_deps: InvestigatorDependencies,
) -> IncidentState:

    state = IncidentState(
        incident=incident,
        status="triage",
    )

    # ---------------------------------
    # 1. TRIAGE
    # ---------------------------------

    triage_result = await triage_agent.run(
        incident.model_dump_json(indent=2)
    )

    state.triage = triage_result.output

    # If the triage agent identifies important
    # missing information, we still allow this
    # workshop MVP to investigate available data.
    # A later version can branch to
    # "needs_more_context".

    state.status = "investigating"

    # ---------------------------------
    # 2. INVESTIGATION
    # ---------------------------------

    investigation_prompt = f"""
ORIGINAL INCIDENT

{incident.model_dump_json(indent=2)}

TRIAGE REPORT

{state.triage.model_dump_json(indent=2)}

Use the available read-only evidence tools before
forming conclusions.

Investigate the incident and return an EvidenceReport.
"""

    investigation_result = await investigator_agent.run(
        investigation_prompt,
        deps=investigator_deps,
    )

    state.investigation = investigation_result.output

    state.status = "response_planning"

    # ---------------------------------
    # 3. RESPONSE PLANNING
    # ---------------------------------

    response_prompt = f"""
INCIDENT

{incident.model_dump_json(indent=2)}

INVESTIGATION EVIDENCE

{state.investigation.model_dump_json(indent=2)}

Create the safest response plan supported by the
evidence.

Do not claim any production action has already been
executed.
"""

    response_result = await response_agent.run(
        response_prompt
    )

    state.response = response_result.output

    # ---------------------------------
    # 4. APPROVAL STATE
    # ---------------------------------

    if state.response.approval_required:
        state.status = "awaiting_approval"
    else:
        state.status = "complete"

    return state
```

This is a critical architectural point:

```text
The application owns routing.
The agents do not arbitrarily call one another.
```

The handoff is:

```text
IncidentReport
    ↓
TriageReport
    ↓
EvidenceReport
    ↓
ResponsePlan
```

---

# Step 18 — Create the CLI Demo

Create:

```text
src/incident_team/cli.py
```

Add:

```python
import asyncio
import os
from datetime import datetime, timezone

from dotenv import load_dotenv

from incident_team.dependencies import (
    InvestigatorDependencies,
)
from incident_team.models.incident import IncidentReport
from incident_team.orchestrator import handle_incident
from incident_team.tools.deployments import (
    MockDeploymentClient,
)
from incident_team.tools.logs import MockLogClient
from incident_team.tools.metrics import MockMetricsClient
from incident_team.tools.runbooks import MockRunbookClient


async def run_demo() -> None:

    load_dotenv()

    if not os.getenv("GROQ_API_KEY"):
        raise RuntimeError(
            "GROQ_API_KEY is missing. "
            "Add it to your .env file."
        )

    incident = IncidentReport(
        incident_id="INC-1001",
        description=(
            "Payments API is returning elevated "
            "HTTP 500 errors and high latency."
        ),
        reported_at=datetime.now(timezone.utc),
        reporter="monitoring-system",
        affected_service="payments-api",
        observed_symptoms=[
            "HTTP 500 error rate increased",
            "P95 latency increased",
            "database connection errors observed",
        ],
    )

    dependencies = InvestigatorDependencies(
        logs=MockLogClient(),
        metrics=MockMetricsClient(),
        deployments=MockDeploymentClient(),
        runbooks=MockRunbookClient(),
    )

    state = await handle_incident(
        incident=incident,
        investigator_deps=dependencies,
    )

    print()
    print("=" * 70)
    print("FINAL INCIDENT STATE")
    print("=" * 70)
    print()

    print(state.model_dump_json(indent=2))


def main() -> None:
    asyncio.run(run_demo())


if __name__ == "__main__":
    main()
```

---

# Step 19 — Run the Complete Multi-Agent System

Run:

```bash
uv run incident-team
```

Or:

```bash
uv run python -m incident_team.cli
```

The expected flow is:

```text
Incident Report
      |
      v
Triage Agent
      |
      | TriageReport
      v
Investigator Agent
      |
      | calls:
      |-- query_logs()
      |-- query_metrics()
      |-- get_recent_deployments()
      |-- get_runbook()
      |
      | EvidenceReport
      v
Response Coordinator
      |
      | ResponsePlan
      v
Awaiting Approval / Complete
```

Because LLM output is probabilistic, the exact text will vary.

The **contract and safety rules** should remain stable.

---

# Step 20 — What a Good Investigation Should Discover

With our mock data, the system has access to facts such as:

```text
FACT 1
Payments API logged connection-pool-exhausted errors.

FACT 2
HTTP 5xx errors increased.

FACT 3
P95 latency increased.

FACT 4
Database connection-pool utilization exceeded 95%.

FACT 5
A deployment occurred about six minutes before the
first reported errors.
```

A reasonable hypothesis is:

```text
The recent payments-api release may have caused or
exposed database connection-pool exhaustion, which
then contributed to latency and HTTP 500 errors.
```

But the Investigator should preserve the distinction:

```text
Supported hypothesis
!=
proven root cause
```

It may still request evidence such as:

- connection pool configuration before and after deployment
- request volume changes
- database health
- query latency
- resource saturation
- comparison with previous release behavior

---

# Step 21 — Add Basic Model Tests

Create:

```text
tests/test_models.py
```

Add:

```python
from datetime import datetime, timezone

from incident_team.models.incident import IncidentReport
from incident_team.models.triage import (
    InvestigationArea,
    TriageReport,
)


def test_incident_report() -> None:

    incident = IncidentReport(
        incident_id="INC-TEST",
        description="Example incident",
        reported_at=datetime.now(timezone.utc),
    )

    assert incident.incident_id == "INC-TEST"


def test_triage_cannot_have_root_cause() -> None:

    triage = TriageReport(
        incident_id="INC-TEST",
        summary="Errors detected",
        symptoms=["HTTP 500"],
        affected_systems=["payments-api"],
        severity="SEV2",
        possible_impact=[
            "Payment processing may fail"
        ],
        investigation_areas=[
            InvestigationArea(
                area="application logs",
                reason="Check error patterns",
                priority="high",
            )
        ],
    )

    assert triage.root_cause is None
```

---

# Step 22 — Test the Mock Tools

Create:

```text
tests/test_tools.py
```

Add:

```python
import pytest

from incident_team.tools.deployments import (
    MockDeploymentClient,
)
from incident_team.tools.logs import MockLogClient
from incident_team.tools.metrics import MockMetricsClient


@pytest.mark.asyncio
async def test_logs_return_evidence() -> None:

    client = MockLogClient()

    evidence = await client.query(
        service="payments-api",
        query="error",
    )

    assert evidence
    assert evidence[0].source_type == "log"


@pytest.mark.asyncio
async def test_metrics_return_evidence() -> None:

    client = MockMetricsClient()

    evidence = await client.query(
        service="payments-api",
        metric="error rate",
    )

    assert evidence
    assert evidence[0].source_type == "metric"


@pytest.mark.asyncio
async def test_deployments_return_evidence() -> None:

    client = MockDeploymentClient()

    evidence = await client.recent(
        "payments-api"
    )

    assert evidence
    assert evidence[0].source_type == "deployment"


@pytest.mark.asyncio
async def test_unknown_service_returns_no_logs() -> None:

    client = MockLogClient()

    evidence = await client.query(
        service="unknown-service",
        query="error",
    )

    assert evidence == []
```

Run:

```bash
uv run pytest -q
```

These tests do not require Groq because they test deterministic Python components.

---

# Step 23 — Add `.gitignore`

Create `.gitignore`:

```gitignore
.venv/
__pycache__/
.pytest_cache/
*.pyc
.env
dist/
build/
```

Important:

```text
Never commit GROQ_API_KEY.
```

---

# Step 24 — Understand the Three Agent Boundaries

## Agent 1 — Triage

### Input

```text
IncidentReport
```

### Output

```text
TriageReport
```

### Tools

```text
None
```

### Allowed

- classify symptoms
- identify systems
- assess likely severity
- identify missing information
- suggest investigation areas

### Not allowed

- declare root cause
- invent telemetry
- change production

---

## Agent 2 — Investigator

### Input

```text
IncidentReport
+
TriageReport
```

### Output

```text
EvidenceReport
```

### Tools

```text
Logs
Metrics
Deployments
Runbooks
```

### Allowed

- collect evidence
- challenge triage assumptions
- build hypotheses
- compare supporting evidence
- compare contradicting evidence
- preserve uncertainty

### Not allowed

- fabricate evidence
- restart services
- deploy code
- rollback production
- alter configuration

---

## Agent 3 — Response Coordinator

### Input

```text
Incident context
+
EvidenceReport
```

### Output

```text
ResponsePlan
```

### Allowed

- recommend mitigation
- recommend rollback
- define verification
- define monitoring
- define escalation
- request approval

### Not allowed

- claim an action occurred when it did not
- automatically perform a high-risk production change

---

# Step 25 — Map the Project to A.G.E.N.T.

This project can also be explained with the A.G.E.N.T. design framework.

## A — Aim

Goal:

```text
Help an operations team investigate an incident and
produce a safe evidence-based response plan.
```

Boundaries:

```text
The system investigates and recommends.
It does not autonomously execute dangerous changes.
```

---

## G — Gather

The system needs:

```text
Incident report
Logs
Metrics
Deployment history
Runbooks
```

These are exposed through controlled read-only tools.

---

## E — Execution

The workflow is:

```text
Triage
  ↓
Investigate
  ↓
Form hypotheses
  ↓
Assess evidence
  ↓
Create response plan
  ↓
Request approval when required
```

---

## N — Navigation

The current MVP navigation is:

```text
TRIAGE
  ↓
INVESTIGATING
  ↓
RESPONSE_PLANNING
  ↓
AWAITING_APPROVAL
        or
COMPLETE
```

A later version should support:

```text
TRIAGE
    ↓
NEEDS_MORE_CONTEXT

INVESTIGATE
    ↓
MORE_EVIDENCE_NEEDED
    ↓
INVESTIGATE

NEW_EVIDENCE
    ↓
REINVESTIGATE
```

---

## T — Test

Test at least these behaviors:

1. Obvious incident pattern produces a useful response.
2. Insufficient evidence causes the system to request more evidence.
3. Conflicting evidence preserves uncertainty.
4. Investigator can challenge a poor triage assumption.
5. Unsupported claims are rejected or corrected.
6. Dangerous remediation requires human approval.
7. Missing monitoring data is never fabricated.
8. Resolved incidents stop unnecessary investigation.
9. New evidence causes reassessment.

---

# Step 26 — Next Improvement: Add a Missing-Evidence Loop

The first workshop version is intentionally linear:

```text
Triage
  ↓
Investigator
  ↓
Response
```

The next version can inspect:

```python
state.investigation.investigation_complete
```

If it is `False`, route back into investigation:

```text
INVESTIGATE
     |
     | investigation_complete = false
     v
GET MORE EVIDENCE
     |
     v
INVESTIGATE
```

Do this in Python first.

Do not move to a graph framework until the workflow genuinely needs more complex branching, persistence, resumability, or parallel execution.

---

# Step 27 — Next Improvement: Add Human Approval

The current ResponsePlan already contains:

```python
approval_required: bool
approval_reason: str | None
```

The application can pause when:

```python
state.status == "awaiting_approval"
```

Later, add an approval object such as:

```python
from datetime import datetime

from pydantic import BaseModel


class ApprovalDecision(BaseModel):
    approved: bool
    approved_by: str
    decided_at: datetime
    comment: str | None = None
```

Then the application—not the model—decides whether an approved action is forwarded to a production execution system.

---

# Step 28 — Next Improvement: Replace Mock Tools

Once the workshop MVP works, replace one mock integration at a time.

For example:

```text
MockLogClient
    ↓
Datadog / Elastic / Splunk / CloudWatch

MockMetricsClient
    ↓
Prometheus / Grafana / Datadog

MockDeploymentClient
    ↓
GitHub Actions / Argo CD / Kubernetes / CI/CD

MockRunbookClient
    ↓
Git / Confluence / Notion / internal knowledge base
```

Keep the returned contract as:

```python
list[EvidenceItem]
```

That means the Investigator Agent does not need to know which vendor provides the data.

---

# Step 29 — Production Safety Rules

Before connecting this to real infrastructure:

## Keep evidence tools read-only

Good:

```text
get_logs
get_metrics
get_deployments
get_configuration
search_incidents
get_runbook
```

Higher-risk:

```text
restart_service
rollback
deploy
delete
scale
change_configuration
shift_traffic
```

Do not expose high-risk tools until you have:

- authentication
- authorization
- approval policies
- audit logs
- idempotency
- rollback mechanisms
- scoped credentials
- execution verification
- rate limits
- timeout handling
- incident ownership rules

---

# Step 30 — Final Architecture

```text
                         +--------------------+
                         |   IncidentReport   |
                         +---------+----------+
                                   |
                                   v
                         +--------------------+
                         |    Triage Agent    |
                         |       Groq         |
                         +---------+----------+
                                   |
                              TriageReport
                                   |
                                   v
                    +-----------------------------+
                    |      Investigator Agent     |
                    |            Groq             |
                    +-------------+---------------+
                                  |
             +--------------------+--------------------+
             |                    |                    |
             v                    v                    v
          Logs                 Metrics            Deployments
             |                    |                    |
             +--------------------+--------------------+
                                  |
                              Runbooks
                                  |
                                  v
                           EvidenceReport
                                  |
                                  v
                     +--------------------------+
                     |   Response Coordinator   |
                     |           Groq           |
                     +------------+-------------+
                                  |
                             ResponsePlan
                                  |
                     +------------+-------------+
                     |                          |
                     v                          v
             Human Approval                 Complete
```

---

# Step 31 — Commands Summary

Create environment:

```bash
uv sync --extra dev
```

Configure Groq:

```bash
cp .env.example .env
```

Edit:

```env
GROQ_API_KEY=gsk_your_actual_key_here
```

Run tests:

```bash
uv run pytest -q
```

Run the agents:

```bash
uv run incident-team
```

Or:

```bash
uv run python -m incident_team.cli
```

---

# Step 32 — Workshop Build Order

For a live workshop, build in this order:

```text
1. Project + dependencies
2. Groq API key
3. IncidentReport
4. TriageReport
5. Triage Agent
6. Run Triage Agent
7. Evidence models
8. Mock tools
9. Investigator dependencies
10. Investigator Agent
11. Run Investigator
12. Response models
13. Response Coordinator
14. Orchestrator
15. Run full workflow
16. Add validators
17. Add tests
18. Discuss human approval
19. Discuss production integrations
20. Map the solution to A.G.E.N.T.
```

This order is useful because participants see a working result early and then progressively add agent capabilities.

---

# References

The design follows the original three-agent incident-response example:

- Triage Agent
- Investigator Agent
- Response Coordinator
- typed handoffs
- evidence vs hypothesis separation
- application-controlled orchestration
- human approval for high-risk remediation

Current Pydantic AI Groq integration supports the Groq model prefix:

```python
Agent("groq:llama-3.3-70b-versatile")
```

and reads:

```text
GROQ_API_KEY
```

from the environment.

Groq also exposes an OpenAI-compatible API, but this workshop intentionally uses Pydantic AI's native Groq integration instead because it is clearer for teaching the provider architecture.

---

# End

At this point you have a complete first version of:

```text
AI Incident Response Team
+
Three Specialized AI Agents
+
Groq
+
Pydantic AI
+
Typed Handoffs
+
Read-Only Investigation Tools
+
Human Approval Boundary
```

The recommended next exercise is to build **Steps 1–6 first**, run the Triage Agent successfully with Groq, and then continue to the Investigator.
