# Project 4 — AI Incident Response Team

## Implementation Plan with Three Pydantic AI Agents and Typed Handoffs

**Source alignment:** Project 4 in *Four Practical Agentic AI Projects* (AI Incident Response Team — Three-Agent System, pp. 15–18).

## 1. Objective

Build a multi-agent incident-response system with **exactly three specialized agents**:

1. **Triage Agent** — determines what appears to be happening and where to investigate.
2. **Investigator Agent** — gathers evidence and develops supported hypotheses.
3. **Response Coordinator** — converts the evidence into a safe response plan and approval request.

### Core invariant

> Multi-agent value comes from specialization and controlled handoffs, not from adding more model calls.

The agents should not have an unstructured conversation. They exchange typed contracts.

## 2. System architecture

```text
Incident Report
      |
      v
+------------------+
|  Triage Agent    |
+---------+--------+
          |
      TriageReport
          |
          v
+------------------+
| Investigator     |
| Agent            |
+---------+--------+
          |
      EvidenceReport
          |
          v
+------------------+
| Response         |
| Coordinator      |
+---------+--------+
          |
      ResponsePlan
          |
   +------+------+
   |             |
 approval      complete
```

Conceptual orchestration:

```text
Incident -> Triage -> Investigate -> Assess Evidence -> Propose Response
-> Human Approval -> Complete
```

## 3. Suggested project structure

```text
incident-response-team/
├── pyproject.toml
├── src/
│   └── incident_team/
│       ├── __init__.py
│       ├── models/
│       │   ├── incident.py
│       │   ├── triage.py
│       │   ├── investigation.py
│       │   └── response.py
│       ├── agents/
│       │   ├── triage_agent.py
│       │   ├── investigator_agent.py
│       │   └── response_agent.py
│       ├── tools/
│       │   ├── logs.py
│       │   ├── metrics.py
│       │   ├── deployments.py
│       │   └── runbooks.py
│       ├── dependencies.py
│       ├── orchestrator.py
│       └── cli.py
└── tests/
    ├── fixtures/
    ├── test_triage_agent.py
    ├── test_investigator_agent.py
    ├── test_response_agent.py
    └── test_orchestration.py
```

Install:

```bash
uv add pydantic-ai
uv add --dev pytest pytest-asyncio
```

## 4. Incident input contract

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

Reported symptoms are input claims, not automatically established root-cause evidence.

# Agent 1 — Triage Agent

## 5. Responsibility

The Triage Agent answers:

> What appears to be happening and where should we investigate?

It identifies symptoms, affected systems, severity/impact, incident start time if known, missing information, and initial investigation areas. It must **not declare the root cause**.

## 6. Triage contract

```python
from typing import Literal
from pydantic import BaseModel, Field


class InvestigationArea(BaseModel):
    area: str
    reason: str
    priority: Literal['high', 'medium', 'low']


class TriageReport(BaseModel):
    incident_id: str
    summary: str
    symptoms: list[str]
    affected_systems: list[str]
    severity: Literal['SEV1', 'SEV2', 'SEV3', 'SEV4', 'unknown']
    possible_impact: list[str]
    incident_start_time: str | None = None
    missing_information: list[str] = Field(default_factory=list)
    investigation_areas: list[InvestigationArea]
    root_cause: None = None
```

## 7. Triage agent

```python
from pydantic_ai import Agent

triage_agent = Agent(
    'openai:gpt-5.6-sol',
    output_type=TriageReport,
    instructions='''
You are an Incident Triage Agent.
Determine what appears to be happening, which systems may be affected,
likely impact/severity, what context is missing, and where investigation should begin.
Do not declare root cause.
A plausible explanation is not an established cause.
Return a structured TriageReport.
''',
)
```

Triage should generally have no operational evidence tools in the MVP.

# Agent 2 — Investigator Agent

## 8. Responsibility

The Investigator answers:

> What evidence supports or contradicts possible causes?

It must distinguish facts from hypotheses and may challenge the triage framing.

## 9. Evidence and hypothesis contracts

```python
class EvidenceItem(BaseModel):
    source_type: Literal[
        'log', 'metric', 'deployment', 'runbook', 'configuration', 'incident_record'
    ]
    source_reference: str
    observation: str
    observed_at: str | None = None


class Hypothesis(BaseModel):
    hypothesis: str
    supporting_evidence: list[EvidenceItem] = Field(default_factory=list)
    contradicting_evidence: list[EvidenceItem] = Field(default_factory=list)
    confidence: Literal['low', 'medium', 'high']
    missing_evidence: list[str] = Field(default_factory=list)


class EvidenceReport(BaseModel):
    incident_id: str
    observed_facts: list[EvidenceItem]
    hypotheses: list[Hypothesis]
    strongest_supported_hypothesis: str | None
    unresolved_questions: list[str]
    additional_evidence_needed: list[str]
    investigation_complete: bool
```

`strongest_supported_hypothesis` is not automatically an established root cause.

## 10. Investigator dependencies and tools

```python
from dataclasses import dataclass
from pydantic_ai import Agent, RunContext


@dataclass
class InvestigatorDependencies:
    logs: 'LogClient'
    metrics: 'MetricsClient'
    deployments: 'DeploymentClient'
    runbooks: 'RunbookClient'


investigator_agent = Agent(
    'openai:gpt-5.6-sol',
    deps_type=InvestigatorDependencies,
    output_type=EvidenceReport,
    instructions='...',
)


@investigator_agent.tool
async def query_logs(
    ctx: RunContext[InvestigatorDependencies],
    service: str,
    query: str,
    minutes: int = 30,
) -> list[EvidenceItem]:
    return await ctx.deps.logs.query(service=service, query=query, minutes=minutes)


@investigator_agent.tool
async def query_metrics(
    ctx: RunContext[InvestigatorDependencies],
    service: str,
    metric: str,
    minutes: int = 60,
) -> list[EvidenceItem]:
    return await ctx.deps.metrics.query(service=service, metric=metric, minutes=minutes)


@investigator_agent.tool
async def get_recent_deployments(
    ctx: RunContext[InvestigatorDependencies],
    service: str,
) -> list[EvidenceItem]:
    return await ctx.deps.deployments.recent(service)
```

Optional read-only tools: `get_configuration_changes`, `search_incidents`, `get_runbook`.

Do not expose restart, rollback, deployment, configuration mutation, or deletion tools in the MVP.

## 11. Investigator instructions

```python
INVESTIGATOR_INSTRUCTIONS = '''
You are the Investigator Agent.
You receive a typed TriageReport.
Do not blindly accept triage assumptions.

PROCESS
1. Review triage.
2. Gather relevant evidence.
3. Form hypotheses.
4. Search for supporting evidence.
5. Search for contradicting evidence.
6. Preserve uncertainty.
7. Request additional evidence when necessary.

FACT VS HYPOTHESIS
Observed facts must come from retrieved evidence.
Never present a hypothesis as an observed fact.

CONFIDENCE
High confidence requires strong supporting evidence and no major unresolved contradiction.
Plausibility alone is insufficient.

MISSING DATA
Do not fabricate logs, metrics, deployments, or configuration state.
'''
```

# Agent 3 — Response Coordinator

## 12. Responsibility

The Response Coordinator answers:

> Given the evidence, what is the safest next operational plan?

It creates immediate actions, verification steps, rollback/mitigation recommendations, monitoring steps, escalation conditions, stakeholder communication, and approval requirements.

## 13. Response contracts

```python
class ResponseAction(BaseModel):
    order: int
    action: str
    reason: str
    risk: Literal['low', 'medium', 'high']
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

## 14. Response agent

```python
response_agent = Agent(
    'openai:gpt-5.6-sol',
    output_type=ResponsePlan,
    instructions='''
You are the Incident Response Coordinator.
Use the Investigator's typed EvidenceReport to build the safest operational response plan.

Do not claim that a rollback, restart, deployment, traffic shift, or configuration change occurred
unless execution is confirmed by an authoritative system.

High-risk production actions require human approval.
When evidence is uncertain, prefer reversible actions and additional verification.
Return a ResponsePlan.
''',
)
```

## 15. Context separation

```text
Triage Agent
  receives: IncidentReport

Investigator Agent
  receives: IncidentReport + TriageReport
  tools: logs, metrics, deployments, runbooks

Response Coordinator
  receives: concise incident context + EvidenceReport
  does not need uncontrolled raw logs
```

## 16. Programmatic handoff

Pydantic AI supports programmatic handoff where application code runs agents in succession:

```python
async def handle_incident(
    incident: IncidentReport,
    investigator_deps: InvestigatorDependencies,
) -> ResponsePlan:
    triage_result = await triage_agent.run(incident.model_dump_json())
    triage = triage_result.output

    investigation_result = await investigator_agent.run(
        f'Incident:
{incident.model_dump_json()}

Triage:
{triage.model_dump_json()}',
        deps=investigator_deps,
    )
    evidence = investigation_result.output

    response_result = await response_agent.run(
        f'Incident:
{incident.model_dump_json()}

Evidence:
{evidence.model_dump_json()}'
    )

    return response_result.output
```

The application owns routing. The agents do not call each other arbitrarily.

## 17. Explicit system state

```python
class IncidentState(BaseModel):
    incident: IncidentReport
    triage: TriageReport | None = None
    investigation: EvidenceReport | None = None
    response: ResponsePlan | None = None
    status: Literal[
        'triage',
        'investigating',
        'response_planning',
        'awaiting_approval',
        'complete',
        'needs_more_context',
    ]
```

Suggested transitions:

```text
START -> TRIAGE
TRIAGE -> NEED_MORE_CONTEXT | INVESTIGATE
INVESTIGATE -> INVESTIGATE | RESPONSE_PLAN
RESPONSE_PLAN -> AWAITING_APPROVAL | COMPLETE
```

New evidence may route the system back to investigation.

## 18. Validators

```python
from pydantic_ai import ModelRetry


@triage_agent.output_validator
async def validate_triage(output: TriageReport) -> TriageReport:
    if output.root_cause is not None:
        raise ModelRetry('Triage must not declare root cause.')
    return output


@investigator_agent.output_validator
async def validate_investigation(output: EvidenceReport) -> EvidenceReport:
    for hypothesis in output.hypotheses:
        if hypothesis.confidence == 'high' and not hypothesis.supporting_evidence:
            raise ModelRetry('High-confidence hypotheses require supporting evidence.')
    return output


@response_agent.output_validator
async def validate_response(output: ResponsePlan) -> ResponsePlan:
    risky = any(action.risk == 'high' for action in output.immediate_actions)
    if risky and not output.approval_required:
        raise ModelRetry('High-risk remediation requires human approval.')
    return output
```

## 19. Required behavior tests

| Scenario | Expected behavior |
|---|---|
| Obvious incident pattern | Produce a response efficiently |
| Insufficient evidence | Request additional evidence |
| Conflicting evidence | Preserve uncertainty |
| Wrong triage hypothesis | Investigator may challenge it |
| Unsupported investigator claim | Flag or reject claim |
| Dangerous remediation | Require human approval |
| Monitoring unavailable | Do not fabricate metrics |
| Incident resolved | Stop unnecessary investigation |
| New evidence arrives | Reassess current conclusion |

## 20. Test each agent independently

**Triage:** never declares root cause; identifies missing context; produces useful investigation areas.

**Investigator:** separates facts and hypotheses; challenges weak triage assumptions; preserves conflicting evidence; never fabricates missing telemetry; confidence matches evidence.

**Response Coordinator:** high-risk actions require approval; every action includes verification; no remediation is claimed as executed; unresolved risk is preserved.

Then run all nine scenarios end to end through the orchestrator.

## 21. Observability

Record at minimum:

```text
incident_id
agent start/end
tool calls
tool failures
handoff payload type
hypothesis/confidence changes
approval state
final status
model usage/latency
```

## 22. Pydantic Graph — later

Start with plain Python programmatic handoffs. Move to Pydantic Graph only when richer branching, retries, persistence, or resumability justify an explicit graph/state-machine layer.

```text
V1: deterministic Python orchestrator
V2: graph/state-machine orchestration
```

## 23. MVP build order

1. Define `IncidentReport`.
2. Define and test `TriageReport`.
3. Implement Triage Agent.
4. Define `EvidenceItem`, `Hypothesis`, and `EvidenceReport`.
5. Implement mock log/metric/deployment/runbook clients.
6. Implement and test Investigator Agent.
7. Define `ResponseAction` and `ResponsePlan`.
8. Implement and test Response Coordinator.
9. Build deterministic programmatic orchestration.
10. Add missing-evidence loops.
11. Add approval states.
12. Add all nine end-to-end behavior tests.
13. Add tracing/observability.
14. Consider Pydantic Graph only after the MVP is stable.

## 24. Definition of done

The MVP is complete when:

- exactly three agents have distinct responsibilities;
- every handoff is typed;
- Triage never declares root cause;
- Investigator distinguishes evidence from hypothesis;
- Investigator may challenge triage assumptions;
- Response Coordinator recommends rather than executes high-impact changes;
- dangerous remediation requires approval;
- missing telemetry never becomes fabricated evidence;
- all nine source-defined scenarios pass end to end.

## 25. Pydantic AI references

- Multi-agent applications and programmatic handoff: https://pydantic.dev/docs/ai/guides/multi-agent-applications/
- Dependencies and `RunContext`: https://pydantic.dev/docs/ai/core-concepts/dependencies/
- Output validators: https://pydantic.dev/docs/ai/core-concepts/output/#output-validators
- Unit testing: https://pydantic.dev/docs/ai/guides/testing/
- Pydantic Graph: https://pydantic.dev/docs/graph/
