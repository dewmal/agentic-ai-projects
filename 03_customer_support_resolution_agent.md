# Project 3 — Customer Support Resolution Agent

## Implementation Plan with Pydantic AI + External Data + Policy + Human Approval

**Source alignment:** Project 3 in *Four Practical Agentic AI Projects* (Customer Support Resolution Agent with External Data, pp. 12–14).

## 1. Objective

Build a support agent that investigates a customer request using authoritative business data and policy sources, then returns a grounded resolution, clarification, escalation, or approval request.

### Core invariant

> No consequential recommendation without verified business evidence and applicable authoritative policy.

The MVP should **recommend** high-impact actions such as refunds rather than execute them.

## 2. Agentic flow

```text
UNDERSTAND CASE
      |
      v
IDENTIFY RECORD
      |
      v
RETRIEVE DATA
      |
      v
RETRIEVE POLICY
      |
      v
VALIDATE ELIGIBILITY
      |
      v
RESOLVE
      |
      v
RESPOND
```

Key branches:

```text
record not found       -> clarify
multiple records       -> clarify
external API down      -> do not invent; escalate/clarify
policy ambiguous       -> escalate
high-impact action     -> human approval
facts conflict         -> surface discrepancy
sufficient evidence    -> grounded resolution
```

## 3. Suggested project structure

```text
customer-support-agent/
├── pyproject.toml
├── src/
│   └── support_agent/
│       ├── __init__.py
│       ├── models.py
│       ├── dependencies.py
│       ├── agent.py
│       ├── tools.py
│       ├── service.py
│       └── clients/
│           ├── customer_client.py
│           ├── order_client.py
│           ├── shipment_client.py
│           ├── policy_client.py
│           └── support_history_client.py
└── tests/
    ├── fixtures/
    │   ├── customers.json
    │   ├── orders.json
    │   ├── shipments.json
    │   └── policies.json
    ├── test_models.py
    ├── test_tools.py
    ├── test_agent.py
    └── test_resolution_cases.py
```

Install:

```bash
uv add pydantic-ai httpx
uv add --dev pytest pytest-asyncio
```

## 4. Authoritative domain models

```python
from datetime import date, datetime
from decimal import Decimal
from typing import Literal
from pydantic import BaseModel


class CustomerRecord(BaseModel):
    customer_id: str
    name: str
    account_status: Literal['active', 'restricted', 'closed']


class OrderRecord(BaseModel):
    order_id: str
    customer_id: str
    status: Literal['pending', 'processing', 'shipped', 'delivered', 'cancelled']
    total: Decimal
    currency: str
    ordered_at: datetime


class ShipmentRecord(BaseModel):
    order_id: str
    status: Literal['not_shipped', 'in_transit', 'delivered', 'delayed', 'lost', 'unknown']
    carrier: str | None = None
    expected_delivery: date | None = None
    delivered_at: datetime | None = None


class PolicyRule(BaseModel):
    policy_id: str
    title: str
    version: str
    effective_from: date
    rule: str
    source: str
```

## 5. Evidence contract

```python
class EvidenceItem(BaseModel):
    source: Literal[
        'customer',
        'order',
        'shipment',
        'policy',
        'support_history',
    ]
    reference: str
    fact: str
```

Customer claims must not be silently promoted into verified facts.

## 6. Typed terminal outcomes

```python
class ClarificationRequest(BaseModel):
    kind: Literal['clarification'] = 'clarification'
    understood_issue: str
    missing_information: list[str]
    questions: list[str]


class EscalationRequest(BaseModel):
    kind: Literal['escalation'] = 'escalation'
    reason: str
    evidence: list[EvidenceItem]
    unresolved_questions: list[str]
    customer_message: str


class ApprovalRequest(BaseModel):
    kind: Literal['approval_required'] = 'approval_required'
    proposed_action: Literal['refund', 'replacement', 'credit']
    order_id: str
    rationale: str
    evidence: list[EvidenceItem]
    policy_basis: list[str]
    customer_message: str


class SupportResolution(BaseModel):
    kind: Literal['resolution'] = 'resolution'
    intent: str
    verified_facts: list[EvidenceItem]
    applicable_policy: list[str]
    eligibility: Literal['eligible', 'not_eligible', 'not_applicable']
    recommended_action: str
    requires_human_approval: bool
    customer_message: str
```

## 7. Dependencies

```python
from dataclasses import dataclass


@dataclass
class SupportDependencies:
    customers: 'CustomerClient'
    orders: 'OrderClient'
    shipments: 'ShipmentClient'
    policies: 'PolicyClient'
    history: 'SupportHistoryClient | None' = None
```

```python
from pydantic_ai import Agent

support_agent = Agent(
    'openai:gpt-5.6-sol',
    deps_type=SupportDependencies,
    output_type=[
        ClarificationRequest,
        SupportResolution,
        ApprovalRequest,
        EscalationRequest,
    ],
    instructions='...',
)
```

## 8. Read-only business tools

```python
from pydantic_ai import RunContext


@support_agent.tool
async def get_customer(
    ctx: RunContext[SupportDependencies],
    customer_id: str,
) -> CustomerRecord | None:
    return await ctx.deps.customers.get_customer(customer_id)


@support_agent.tool
async def get_order(
    ctx: RunContext[SupportDependencies],
    order_id: str,
) -> OrderRecord | None:
    return await ctx.deps.orders.get_order(order_id)


@support_agent.tool
async def get_shipment(
    ctx: RunContext[SupportDependencies],
    order_id: str,
) -> ShipmentRecord | None:
    return await ctx.deps.shipments.get_shipment(order_id)


@support_agent.tool
async def search_policies(
    ctx: RunContext[SupportDependencies],
    topic: str,
) -> list[PolicyRule]:
    return await ctx.deps.policies.search(topic)
```

Optional: `get_prior_cases`, but only when prior support history materially affects the current decision.

## 9. Do not expose mutation tools in the MVP

Do not register:

```text
issue_refund
apply_credit
cancel_order
change_account
ship_replacement
```

A consequential action should become an `ApprovalRequest`, not an executed action.

## 10. Agent instructions

```python
INSTRUCTIONS = '''
You are a Customer Support Resolution Agent.

Use authoritative business data and current policy.
Never treat a customer statement as a verified system fact.

PROCESS
UNDERSTAND_CASE -> IDENTIFY_RECORD -> RETRIEVE_DATA -> RETRIEVE_POLICY
-> VALIDATE_ELIGIBILITY -> RESOLVE -> RESPOND

IDENTIFY RECORD
Never invent identifiers or arbitrarily select between multiple matches.

RETRIEVE DATA
Retrieve only the data needed for the current case.

RETRIEVE POLICY
Use the current authoritative policy rather than remembered generic rules.

VALIDATE ELIGIBILITY
Clearly distinguish customer claims, verified facts, policy requirements,
and the eligibility conclusion.

CONFLICTS
When customer statements conflict with system data, surface the discrepancy.

MISSING DATA
If an external service is unavailable, do not fabricate status.
Use another authoritative source if available; otherwise clarify or escalate.

POLICY AMBIGUITY
Do not guess. Escalate.

ACTIONS
You may provide information, clarify, determine eligibility, recommend a refund,
replacement or escalation, and request human approval.
You must not issue refunds, credits, account changes, cancellations, or replacements.
'''
```

## 11. Output validator

```python
from pydantic_ai import ModelRetry, RunContext


@support_agent.output_validator
async def validate_resolution(
    ctx: RunContext[SupportDependencies],
    output,
):
    if isinstance(output, SupportResolution):
        if output.eligibility != 'not_applicable' and not output.applicable_policy:
            raise ModelRetry('Eligibility decisions require an applicable policy.')

        if output.requires_human_approval:
            raise ModelRetry(
                'Actions requiring human approval must be returned as ApprovalRequest.'
            )

    if isinstance(output, ApprovalRequest):
        if not output.evidence:
            raise ModelRetry('Approval requests require verified evidence.')
        if not output.policy_basis:
            raise ModelRetry('Approval requests require policy justification.')

    return output
```

## 12. Example resolution path

```text
Customer: delayed shipment + refund request
      |
      v
get_order(order_id)
      |
      v
get_shipment(order_id)
      |
      v
search_policies('delayed delivery refund')
      |
      v
compare verified facts with current policy
      |
      +---- eligible + approval required ---> ApprovalRequest
      +---- eligible + low-risk -----------> SupportResolution
      +---- not eligible ------------------> SupportResolution
      +---- missing/ambiguous evidence ----> Clarification/Escalation
```

## 13. Mock clients first

Start with fixture-backed service clients instead of production APIs:

```text
tests/fixtures/customers.json
tests/fixtures/orders.json
tests/fixtures/shipments.json
tests/fixtures/policies.json
```

This isolates agent behavior from external integration complexity.

## 14. Required behavior tests

| Scenario | Expected behavior |
|---|---|
| Normal delayed delivery | Investigate and provide grounded response |
| Order does not exist | Ask for verification rather than guessing |
| External API unavailable | State that status cannot be verified |
| Refund eligible | Recommend refund and approval if required |
| Refund not eligible | Explain applicable policy |
| Policy conflicts with request | Follow current authoritative policy |
| Ambiguous request | Clarify intent |
| High-value refund | Require human approval |
| System data conflicts with customer statement | Surface discrepancy explicitly |

Additional tests:

- stale policy is not selected as current;
- tool timeout does not become invented evidence;
- customer claim is not copied into `verified_facts` without confirmation;
- approval-required action cannot be returned as completed.

## 15. Testing layers

- **Client unit tests:** lookup behavior, error mapping, timeouts, policy version selection.
- **Agent integration tests:** `TestModel`/`FunctionModel` for deterministic wiring.
- **Behavioral evals:** policy selection, evidence grounding, discrepancy handling, approval routing.

## 16. MVP build order

1. Define customer/order/shipment/policy models.
2. Define the four output branches.
3. Implement fixture-backed service clients.
4. Create `SupportDependencies`.
5. Register read-only Pydantic AI tools.
6. Implement agent instructions.
7. Add output validators.
8. Implement the nine source-defined test scenarios.
9. Add failure simulations for APIs and policy ambiguity.
10. Only then replace mock adapters with real external systems.

## 17. Definition of done

The MVP is complete when:

- consequential decisions are evidence-grounded;
- applicable policy is explicit;
- external failures never produce fabricated facts;
- customer/system conflicts are surfaced;
- high-impact actions are recommendations or approval requests, not executions;
- all nine source-defined scenarios pass.

## 18. Pydantic AI references

- Dependencies and `RunContext`: https://pydantic.dev/docs/ai/core-concepts/dependencies/
- Function tools: https://pydantic.dev/docs/ai/tools/
- Output types and validators: https://pydantic.dev/docs/ai/core-concepts/output/
- Unit testing: https://pydantic.dev/docs/ai/guides/testing/
