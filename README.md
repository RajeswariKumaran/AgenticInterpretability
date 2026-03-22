# Agentic Interpretability

# Implementation Analysis
## How Semantic Conventions Solve AI Interpretability Challenges

> Based on `langgraph_agent_flow_with_tools-2.py`

---

## Table of Contents

1. [The Two-Layer Architecture](#the-two-layer-architecture)
2. [Layer 1: `record_decision()` — The Semantic Convention](#layer-1-record_decision---the-semantic-convention)
3. [Layer 2: `@agent_span()` — The Enforcement Wrapper](#layer-2-agent_span---the-enforcement-wrapper)
4. [How It Solves Each Interpretability Challenge](#how-it-solves-each-interpretability-challenge)
5. [The Net Result](#the-net-result)
6. [Agent Decision Summary](#agent-decision-summary)

---

## The Two-Layer Architecture

The implementation solves interpretability through two Python constructs that work together: `record_decision()` and the `@agent_span()` decorator. Neither changes what the agents *do* — they instrument *how* agents document themselves.

---

## Layer 1: `record_decision()` — The Semantic Convention

Every agent calls this function before returning. It captures a structured decision object with eight fields:

```python
record_decision(
    state,
    agent="estimation_agent",
    type="velocity_adjustment",
    context_facts={"base_velocity": 13, "new_members": 1},
    commitment_action={"adjusted_velocity": 15, "flagged_issues": 2},
    rules_fired=["velocity_adjustment_heuristic"],
    alternatives=["static_velocity"],
    confidence=0.9,
    status="committed"
)
```

Each field maps directly to an interpretability question:

| Field | Interpretability question answered |
|---|---|
| `context_facts` | What did the agent know when it decided? |
| `rules_fired` | What logic or heuristic produced this outcome? |
| `commitment_action` | What did it actually do? |
| `alternatives` | What else could it have done? |
| `confidence` | How certain is it? |
| `status` | Is this decision final, pending human review, or failed? |

This is the **semantic convention** — a shared vocabulary that every agent in the pipeline uses, making all decisions comparable and queryable.

---

## Layer 2: `@agent_span()` — The Enforcement Wrapper

This decorator wraps every agent function in an OpenTelemetry span and does three things automatically.

### 1. Captures inputs and outputs

It reads the state before and after the agent runs and writes key fields as span attributes:

```python
# Written automatically for every agent:
input.issues.count   → number of Jira issues passed in
output.next_agent    → which agent runs next
agent.name           → identifies the span in Honeycomb
```

### 2. Enforces the interpretability contract

After the agent returns, the decorator calls:

```python
enforce_decision_recording(span_name, result, decision_count_before)
```

If the `decisions` list has not grown, it raises `InterpretabilityViolation`. No agent can exit silently — interpretability is a runtime requirement, not a convention that can be skipped.

### 3. Handles failures visibly

Exceptions are caught, recorded on the span with `ERROR` status, and re-raised. The defect alignment agent goes further, with an explicit `try/except` that calls `record_decision` even on failure:

```python
except Exception as e:
    record_decision(state,
        agent="defect_backlog_semantic_alignment_agent",
        context_facts={"error": str(e)},
        rules_fired=["exception_path"],
        confidence=0.2,
        status="failed")
    raise
```

Even errors are traceable.

---

## How It Solves Each Interpretability Challenge

### Challenge 1: Black-box decisions
> *"The AI's reasoning might not be clear"*

**How solved:** The `rules_fired` field makes internal logic explicit. The `sprint_health_agent` fires `blocked_penalty`, `oversized_penalty`, and `dependency_penalty` — each subtracting a known amount from the health score. The score is not a mysterious number; it is fully reconstructable from the recorded rules. A project manager can see exactly why the score dropped from 1.0 to 0.55.

---

### Challenge 2: Dynamic decisions
> *"As new patterns emerge, recommendations shift"*

**How solved:** Every decision is timestamped and appended to the `decisions` list in state. Because all agents write to the same list, you can reconstruct the full decision timeline in sequence:

```python
def print_decision_timeline(state):
    for d in state.get("decisions", []):
        print(f"[{d['agent']}] {d['type']} conf={d['confidence']}")
```

When a recommendation changes between sprint cycles, you can diff the decision lists and identify exactly which agent changed which rule, and when.

---

### Challenge 3: Cascading errors
> *"Later agents depend on earlier ones — errors compound invisibly"*

**How solved:** The trace tree in Honeycomb shows parent-child span relationships, so a bad value propagating from `estimation_agent` to `sprint_health_agent` to `planning_agent` is visible as a chain. Each span carries `input.*` attributes, so you can see what each agent received, not just what it produced. The source of a downstream error can be pinpointed to a specific upstream decision.

---

### Challenge 4: Lack of defined features
> *"Models learn abstractions that lack human interpretability"*

**How solved:** The semantic convention sidesteps this problem entirely. Rather than trying to explain what a neural network's internal activations mean, each agent *declares* its own reasoning in human-readable terms. The defect alignment agent records:

```python
rules_fired=[
    "semantic_similarity_threshold",
    "module_affinity_boost",
    "human_review_threshold"
]
```

These are the actual named heuristics in the code — keyword overlap score, module affinity boost of 0.2, and an alignment threshold of 0.5. The abstraction layer is bypassed by making the agent itself the source of explanation.

---

### Challenge 5: Unpredictability in novel situations
> *"Emergent behaviour from agent interactions"*

**How solved:** The `alternatives` field makes the decision boundary explicit. The `prioritization_agent` records that it chose `weighted_priority_plus_size_bias` over `fifo`, `wsjf`, and `pure_weighted`. When a novel situation produces an unexpected outcome, you can inspect which alternative was *not* chosen and understand what the agent was optimising for. This makes edge-case behaviour interrogable rather than surprising.

---

### Challenge 6: The performance-interpretability trade-off

**How solved:** This is the most important structural point. The semantic convention does **not** constrain the model or simplify it. The LLM inside `defect_backlog_semantic_alignment_agent` is still a full GPT-4o-mini making free-form decisions about which tools to call. The sprint health scoring is still a weighted heuristic with penalties. The interpretability is **additive** — layered on top via the decorator and the record function, not baked into the model architecture. You get the full performance of the underlying models and full auditability of their decisions.

---

### Challenge 7: Human oversight and trust
> *"Team members will not rely on estimates they cannot validate"*

**How solved:** The `status` field creates a formal human-in-the-loop protocol. When the defect alignment agent's confidence is below threshold or too many defects match, it sets `status="proposed"` and routes to `human_alignment_review_agent`:

```python
decision_status = "proposed" if requires_human else "committed"
state["next_agent"] = (
    "human_alignment_review"
    if requires_human
    else "unplanned_work_prediction"
)
```

The human reviewer reads the `proposed` decision from the decisions list, approves or modifies it, and records a new decision with `confidence=1.0` and `rules_fired=["human_authority_override"]`. The human's choice is itself part of the audit trail — not a black-box override, but a traceable decision like any other.

---

## The Net Result

The full `print_decision_timeline` at the end of the pipeline prints every agent's decision in sequence — who decided, what type of reasoning, which rules fired, and with what confidence. Combined with the Honeycomb trace tree showing parent-child spans and all attributes, you have two complementary views of the same pipeline:

- **A logical audit trail** in the state object — the `decisions` list, readable in Python
- **A temporal execution trace** in Honeycomb — spans linked by parent-child relationships, every attribute queryable

Neither alone is sufficient. Together they give a complete answer to the question any project manager, auditor, or reviewer might ask:

> *Why does this sprint plan look the way it does?*

---

## Agent Decision Summary

Every agent in the pipeline calls `record_decision()`. The table below summarises the key fields for each:

| Agent | Type | Key `rules_fired` | Confidence | Status |
|---|---|---|---|---|
| `jira_data_agent` | `load_issues` | _(none)_ | 0.99 | committed |
| `estimation_agent` | `velocity_adjustment` | `velocity_adjustment_heuristic` | 0.90 | committed |
| `defect_backlog_semantic_alignment_agent` | `scrum_analysis_with_tool_orchestration` | `llm_multi_tool_orchestration` | 0.85 | committed / proposed |
| `human_alignment_review_agent` | `confirm_defect_alignment` | `human_authority_override` | 1.00 | committed |
| `unplanned_work_prediction_agent` | `predict_unplanned_work` | `mid_sprint_start_frequency` | 0.70 | committed |
| `sprint_health_agent` | `compute_sprint_health` | `blocked_penalty`, `oversized_penalty`, `dependency_penalty` | 0.85 | committed |
| `prioritization_agent` | `rank_backlog` | `focus_weight_priority`, `small_story_bias` | 0.88 | committed |
| `planning_agent` | `sprint_plan` | `oversized_split`, `blocked_removal`, `dependency_resequence` | 0.80 | committed |
| `llm_story_split_agent` | `generate_story_splits` | `story_point_constraint` | None (LLM) | committed |

---

## Pipeline Overview

```
jira_data_agent
    └── estimation_agent
            └── defect_backlog_semantic_alignment_agent
                    ├── (low confidence) → human_alignment_review_agent
                    │                           └── unplanned_work_prediction_agent
                    └── (high confidence) → unplanned_work_prediction_agent
                                                └── sprint_health_agent
                                                        └── prioritization_agent
                                                                └── planning_agent
                                                                        └── llm_story_split_agent
```

Each node is wrapped with `@agent_span()` and must call `record_decision()` before exiting — enforced at runtime by `enforce_decision_recording()`.

---

## References

| Paper | Relevance |
|---|---|
| Lipton, Z.C. (2016). *The Mythos of Model Interpretability* | Foundational taxonomy: transparency vs post-hoc |
| Doshi-Velez & Kim (2017). *Towards a Rigorous Science of Interpretable ML* | Scientific framework for evaluating interpretability |
| Zhang et al. (2021). *A Survey on Neural Network Interpretability* | Comprehensive survey; covers GDPR Art. 22 |
| Chaduvula et al. (2026). *From Features to Actions: Explainability in Agentic AI* | Directly relevant to this work |
| Nanda, N. (2025). *Mechanistic Interpretability Explainer & Glossary* | Terminology reference |
| Han et al. (2023). *Is Ignorance Bliss? Faithfulness and Alignment of Post Hoc Explanations* | Critical perspective on expert bias in explanations |
