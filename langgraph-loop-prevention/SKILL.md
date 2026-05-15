---
name: langgraph-loop-prevention
description: Use when a LangGraph or ReAct agent gets stuck calling the same tool with the same arguments repeatedly, when validators trigger infinite revision loops, or when iteration counters cap out without progress. Solves the "blind retry" anti-pattern with three layered defences — failure memory, duplicate detection, and actionable retry context. Apply when you see "same SQL ran 3 times" in traces, when `iteration >= MAX_ITER` is hit on simple queries, or when Validator → Agent → Validator bounces with identical content.
---

# Loop Prevention — Failure Memory + Duplicate Detection + Retry Context

## The bug this kills

Three loop shapes recur in agent pipelines:

1. **Tool retry loop** — Agent calls `run_sql(x)`, gets `ERROR: column not found`, retries `run_sql(x)` unchanged. The error never reaches the prompt that builds the next call.
2. **Successful-duplicate loop** — Agent calls `run_sql(x)`, gets a valid result, then on the next ReAct round forgets, calls `run_sql(x)` again. Burns tokens.
3. **Validator ping-pong** — Validator rejects with "wrong data". Agent has no specific fix to apply, regenerates the same answer (perhaps with cosmetic edits). Validator rejects again. Round-trip until `MAX_ITER`.

All three share a root cause: **state has no memory of what was already tried**, and feedback to the Agent is not actionable. Three complementary defences fix this.

## Defence 1 — Failure memory in state

```python
class AgentState(TypedDict):
    sqls_executed: Annotated[list, _append]   # successes
    sqls_failed: Annotated[list, _append]     # {sql, sql_fp, error}
    columns_not_found: Annotated[list, _append]
    errors_seen: Annotated[list, _append]
    schema_discovered: Annotated[list, _append]
    validation_history: Annotated[list, _append]
```

After every tool call, append. `_append` is the merge function:

```python
def _append(left: list, right: list) -> list:
    return (left or []) + (right or [])
```

The Agent's next ReAct prompt **includes** these lists verbatim — failed SQLs, missing columns, errors. The system prompt enforces that this memory is binding:

```text
SQLs ALREADY FAILED (do NOT repeat):
  - SELECT ... FROM "foo" WHERE bar = 'x' → No field named bar

COLUMNS NOT FOUND (do NOT reference):
  - bar
  - baz

ERRORS SEEN (do NOT trigger again):
  - no field named bar
```

## Defence 2 — Duplicate detection at the dispatcher

Fingerprint every operation. If a duplicate hits, short-circuit with a directive instead of re-executing:

```python
import re

def _fingerprint(s: str) -> str:
    return re.sub(r"\s+", " ", s.strip())

def _dispatch_tool(name: str, args: dict, state: dict) -> tuple[str, dict]:
    if name == "run_sql":
        sql = args["sql"]
        sql_fp = _fingerprint(sql)

        # Already failed → refuse, don't re-execute
        if any(f["sql_fp"] == sql_fp for f in state.get("sqls_failed", [])):
            return "ERROR: This SQL already failed. Use a different approach.", {}

        # Already succeeded → refuse, push Agent toward answering
        prior = [s for s in state.get("sqls_executed", []) if _fingerprint(s) == sql_fp]
        if prior:
            return ("DUPLICATE: This exact SQL was already executed. "
                    "You already have the data — write your answer now."), {}

        # First time → execute
        result = _tool_run_sql(sql)
        # ... record success/failure ...
```

## Defence 3 — Loop budget at the ReAct level

The dispatcher's "DUPLICATE" string is a hint, not a hard stop. The Agent may emit another duplicate next round. Cap it:

```python
MAX_REACT_ROUNDS = 12
MAX_DUPLICATES = 2

def _react(system, user, tools, state, model=None):
    duplicate_count = 0
    for _ in range(MAX_REACT_ROUNDS):
        msg = _llm_call(messages, tools, model=model)
        if msg.tool_calls:
            for tc in msg.tool_calls:
                result_str, updates = _dispatch_tool(tc.function.name, json.loads(tc.function.arguments), state)

                if tc.function.name == "run_sql" and "DUPLICATE" in result_str:
                    duplicate_count += 1
                    if duplicate_count >= MAX_DUPLICATES:
                        # Hard exit — inject a "STOP" tool result and let the Agent close out
                        messages.append({
                            "role": "tool",
                            "tool_call_id": tc.id,
                            "content": ("STOP: You are in a loop. You already have the data "
                                        "you need. Write your final answer NOW using your "
                                        "previous query results."),
                        })
                        continue
                # ... else accumulate updates and continue ...
```

The Agent receives an explicit "STOP" tool message — its next turn is forced to be a plain text answer, not another tool call. That breaks the cycle.

## Defence 4 — Actionable validator-to-agent retry context

This is the layer most teams skip. When the Validator rejects and the loop returns to the Agent, the Agent's input MUST contain:

1. Its **last executed operation** (verbatim, so it can't "rewrite from memory")
2. The Validator's feedback (constrained to be specific — see `langgraph-validator-with-evidence`)
3. Explicit rules about what to do if the feedback is vague

```python
def node_sql_agent(state):
    feedback = state.get("validation_feedback", "")
    user_parts = [f"Question: {question}"]

    if feedback:
        last_sqls = state.get("sqls_executed", [])
        last_sql_block = ""
        if last_sqls:
            last_sql_block = f"\n\nYOUR LAST EXECUTED SQL:\n{last_sqls[-1]}"

        user_parts.append(
            f"\nVALIDATOR FEEDBACK (fix this — run a NEW, DIFFERENT query before answering):"
            f"\n{feedback}"
            f"{last_sql_block}"
            f"\n\nRETRY RULES:"
            f"\n- Do NOT re-run the exact same SQL. Write a NEW query addressing the specific fix."
            f"\n- If the feedback is vague or you cannot identify a concrete change, respond with:"
            f"\n  'Unable to determine a better query — the previous SQL appears correct. "
            f"   Rejecting validator feedback.'"
            f"\n  and write your final answer using the data you already have."
            f"\n- NEVER fabricate a value not present in your SQL results."
        )

    user = "\n".join(user_parts)
    answer, updates = _react(system, user, TOOL_SPECS, state, model=agent_model)
```

The escape clause ("if feedback is vague, reject it and answer") is critical. Without it, the Agent has no way out of a Validator that produces useless feedback — the loop is structurally inevitable.

## Anti-patterns to refuse

1. **Stripping comments/whitespace before fingerprinting is good — stripping `ORDER BY` or filters is not.** Two SQLs that differ only by `ORDER BY ASC` vs `ORDER BY DESC` are different queries with different results. Fingerprint = normalised whitespace, lowercase keywords if you like, but NEVER drop clauses.
2. **Capping iterations without an exit message** — Agent's last turn is a tool call that gets nothing back; it crashes or hangs. Always inject a final tool response like "STOP: write answer now" when capping.
3. **Failure memory that grows unbounded** — keep `sqls_failed` to last 5–10. Old failures matter less than recent ones and they bloat the prompt.
4. **Trusting the Agent to "remember on its own"** — LLM context is not memory. If a fact must persist across iterations, it lives in `AgentState`, not in the model's mental model.
5. **Same fingerprint function for SQL and tool args** — SQL fingerprint normalises whitespace; tool args fingerprint should be a stable JSON serialisation (sorted keys). Don't share the helper.

## Implementation checklist

- [ ] `AgentState` has all 4 memory lists: `sqls_executed`, `sqls_failed`, `columns_not_found`, `errors_seen`
- [ ] `_append` reducer is wired via `Annotated[list, _append]`
- [ ] Agent system prompt includes failure memory verbatim in each iteration
- [ ] `_dispatch_tool` short-circuits duplicates and already-failed operations BEFORE executing
- [ ] `MAX_REACT_ROUNDS` and `MAX_DUPLICATES` constants exist with sensible defaults (12 / 2)
- [ ] When `MAX_DUPLICATES` hits, the loop injects an explicit "STOP" tool response
- [ ] On Validator rejection retry, the Agent receives its last executed operation in the prompt
- [ ] On Validator rejection retry, the Agent has an escape clause for vague feedback
- [ ] Trace shows: a duplicate triggers "DUPLICATE: …" once; second duplicate within same ReAct loop triggers "LOOP DETECTED — stopping"
- [ ] Smoke test: a query that the Validator rejects 3× with vague feedback exits the graph in `aborted` state (not `success`), and pairs with `langgraph-output-guard` to emit "UNVERIFIED RESULT" — not a hallucinated answer

## Calibration

Track these counters in production:

- `duplicate_per_query` — should average < 1. If > 2, the Agent prompt isn't surfacing the memory clearly enough.
- `validator_rejection_chain_length` — should average ≤ 2. If > 2, Validator feedback is too vague; apply the `langgraph-validator-with-evidence` skill.
- `max_iter_hit_rate` — should be < 5% of queries. If higher, the Contractor (if you have one) is producing bad contracts, or the Validator is too strict.

These three skills (`output-guard`, `validator-with-evidence`, `loop-prevention`) compose: evidence makes feedback specific → specific feedback breaks loops → unbroken loops trigger the guard → guard refuses to ship hallucinations. Use them together.
