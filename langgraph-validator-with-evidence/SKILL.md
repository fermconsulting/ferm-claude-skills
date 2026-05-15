---
name: langgraph-validator-with-evidence
description: Use when designing or fixing a multi-agent LangGraph pipeline that has a Validator / Critic / Judge node reviewing another agent's output. The Validator must see the same evidence the Agent saw (raw tool outputs, schema, samples) — not just the final answer. Apply when validators reject good answers without evidence, when feedback like "wrong data, please verify" appears without specifics, when iteration loops trigger on valid queries, or when the Validator says "approved" for hallucinated answers. Solves the "blind auditor" anti-pattern.
---

# Validator With Evidence — Peer Review, Not Blind Audit

## The bug this kills

A Validator that only sees the Agent's prose answer + the SQL/operation strings cannot tell whether values in the answer came from real tool output or were invented. Two failure modes:

1. **False rejection of valid answers.** Agent uses `sample_values` to discover values, writes a correct listing answer. Validator (not seeing the samples) thinks no SQL ran, says "wrong data, run a query first". Agent loops.
2. **False approval of hallucinations.** Agent runs a correct SQL, but in the final prose substitutes `099455` → `WO-0001` to make the answer "cleaner". Validator (seeing only prose + SQL text) finds them coherent and approves.

Root cause: **the Validator has strictly less information than the Agent.** It's auditing books it can't open. The fix is symmetric access: snapshot every tool call's raw output into shared state and pipe it into the Validator's prompt.

## The pattern

### 1. Extend state with evidence

```python
class AgentState(TypedDict):
    question: str
    current_answer: str
    sqls_executed: Annotated[list, _append]
    sqls_failed: Annotated[list, _append]
    tool_evidence: Annotated[list, _append]  # NEW — what the Agent actually saw
    validation_feedback: str
    validation_history: Annotated[list, _append]
    iteration: int
    status: str
```

### 2. Record evidence at every tool dispatch

```python
def _record_evidence(tool: str, args: dict, result: str, max_chars: int = 800) -> dict:
    """Snapshot a tool call so the Validator can audit it later."""
    snippet = result if len(result) <= max_chars else result[:max_chars] + "… [truncated]"
    return {
        "tool_evidence": [{
            "tool": tool,
            "args": {k: v for k, v in args.items() if k != "reason"},
            "result": snippet,
        }]
    }

def _dispatch_tool(name: str, args: dict, state: dict) -> tuple[str, dict]:
    if name == "list_tables":
        result = _tool_list_tables()
        return result, _record_evidence(name, args, result)

    if name == "sample_values":
        result = _tool_sample_values(args["table"], args["column"])
        return result, _record_evidence(name, args, result)

    if name == "run_sql":
        # ... existing logic ...
        updates = {"sqls_executed": [sql]}
        updates.update(_record_evidence(name, {"sql": sql}, result, max_chars=1500))
        return result, updates

    # ... etc for describe_table, profile_column, check_sql ...
```

Truncation rules of thumb: schema introspection 800 chars, query results 1500 chars. Truncate the **middle** of large tables, not the tail, so column headers and a sample of rows both survive.

### 3. Feed evidence into the Validator prompt

```python
def node_validator(state: AgentState) -> dict:
    evidence = state.get("tool_evidence", [])
    evidence_section = ""
    if evidence:
        evidence_section = "RAW TOOL EVIDENCE (data the Agent actually saw — verify every value in the ANSWER against this):\n"
        for i, e in enumerate(evidence[-8:], 1):  # last 8 calls is plenty
            args_str = ", ".join(f"{k}={v}" for k, v in e["args"].items())
            evidence_section += f"  [{i}] {e['tool']}({args_str})\n      → {e['result']}\n"

    user = (
        f"ORIGINAL QUESTION:\n{question}\n\n"
        f"CONTRACT:\n{json.dumps(contract, indent=2)}\n\n"
        f"{evidence_section}"
        f"ANSWER:\n{answer}"
    )
```

### 4. Rewrite the Validator system prompt around evidence

```text
You receive: the question, the contract, the SQL queries executed, RAW TOOL EVIDENCE
(actual data the Agent saw from tools), and the Agent's ANSWER.

EVIDENCE-BASED VALIDATION:
The "RAW TOOL EVIDENCE" section is ground truth. Use it to verify every claim.
  • All values in the answer appear in the evidence (or are valid aggregations) → APPROVE.
  • Any value not in the evidence → REJECT, quoting the mismatch.
  • Listing/enumeration grounded in sample_values or profile_column is VALID
    (no run_sql required when the data is already there).
  • Evidence is empty → REJECT (ungrounded).

The contract is a GUIDE. The question is what matters. Do not reject for
contract shape/operation mismatches if the answer addresses the question.

When rejecting, the 'feedback' field MUST:
  (a) Quote the specific wrong value/filter from the answer or SQL.
  (b) Propose a concrete fix the Agent can act on
      ("change ORDER BY ASC to DESC", "add WHERE event_year = 2025",
       "use table_v2 instead of table_v1").
  (c) NEVER use vague phrases ("verify the data", "check results", "make sure").
If you cannot articulate a specific, actionable fix — APPROVE with a cautionary
note instead. Vague feedback causes infinite loops.

COHERENCE: Stay coherent across iterations. If the Agent has already repeated
the same query twice after your feedback, your feedback was not actionable —
APPROVE now with a note. Never contradict your previous iteration.
```

## Anti-patterns to refuse

1. **"Just trust the Validator's verdict"** — without evidence access, its verdicts are guesses. Coin flip with extra steps.
2. **Passing the full transcript instead of structured evidence** — token-expensive and noisy. The Validator drowns. Keep `tool_evidence` to last 8 calls, each truncated.
3. **Evidence in `current_answer`** — pollutes the answer string. Use a separate state field. Concerns of "evidence vs prose" must remain separable.
4. **Letting the Validator call tools itself in early iterations** — sounds appealing ("auditor with hands"), but doubles cost, adds latency, and introduces a second loop to debug. Start with shared evidence (read-only). Only add tool access after the basic Validator-with-evidence flow is stable for weeks.
5. **One global "is_valid" boolean instead of structured verdicts** — you need `verdict`, `feedback`, `issues[]`, optionally `notes` for the success path. Lose any of these and you lose the ability to debug rejections.

## Implementation checklist

- [ ] `tool_evidence: Annotated[list, _append]` added to `AgentState`
- [ ] `_record_evidence(tool, args, result)` helper exists and is called by every branch of `_dispatch_tool`
- [ ] Truncation cap configured (suggest 800 chars / 1500 for SQL results)
- [ ] Validator user prompt includes the last 8 evidence entries with a clear header
- [ ] Validator system prompt forbids vague feedback strings (list them: "verify", "check", "make sure")
- [ ] Validator must APPROVE when it cannot articulate a specific fix
- [ ] Coherence rule: Validator cannot contradict previous iterations' verdicts
- [ ] Validator approves listing answers grounded in `sample_values` without requiring `run_sql`
- [ ] Smoke test: lookup query that only uses `sample_values` is APPROVED on the first iteration

## Diagnostic to run after implementing

```python
# Trace a successful run and confirm the Validator's user prompt contains:
# 1. The original question
# 2. The contract JSON
# 3. >= 1 entry in RAW TOOL EVIDENCE
# 4. The Agent's final answer
# All four must be present. If evidence section is empty, _record_evidence
# isn't being called for the tool branches the Agent used.
```

If the symptom was "Validator rejects valid listings", confirm by logging `evidence_section` for each Validator call. A non-empty evidence section + an APPROVE verdict on listing queries is the success criterion.
