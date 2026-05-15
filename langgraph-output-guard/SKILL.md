---
name: langgraph-output-guard
description: Use when designing or debugging a LangGraph / multi-step / ReAct agent that produces a final answer to a user. Prevents the agent from rendering an unvalidated draft when validation failed, max iterations were reached, or a loop was detected — the root cause of "AI hallucinations" in pipelines that have a Validator/Critic node. Apply when you see fabricated values, "WO-0001"-style invented IDs, or answers that look plausible but contradict the SQL/tool output.
---

# Output Guard — Never Render Unvalidated Drafts

## The bug this kills

Pipelines with `Contractor → Agent → Validator → (loop)` happily print `state["current_answer"]` even when:

- `state["status"] == "drift"` (Validator rejected the last iteration)
- `state["iteration"] >= MAX_ITER` (gave up)
- Loop detector forced exit (`MAX_DUPLICATES` hit)

The final draft is whatever the Agent last produced — frequently **fabricated to placate vague validator feedback**. Examples from the field:

- Real query returned `wo_number = "099455"`. Final answer cited `"WO-0001"`.
- Validator rejected 3× with "verify the data". Agent rewrote its prose without re-querying. Numbers shifted to look "cleaner".

The fix is one guard at the sink, **not a smarter Validator**. If status ≠ success, the draft is by definition untrusted.

## The pattern

```python
# At the end of run() / the final sink that emits to user/UI:
answer = result.get("current_answer", "")
status = result.get("status", "unknown")

if status != "success":
    # NEVER show the draft. Show an honest receipt instead.
    feedback = result.get("validation_feedback", "")
    sqls = result.get("sqls_executed", [])  # or tool_calls, etc.
    print(f"└── ⚠  UNVERIFIED RESULT\n")
    print(f"    The answer could not be validated after {result.get('iteration', 0)} iteration(s).")
    if feedback:
        print(f"    Validator concern: {feedback[:300]}")
    if sqls:
        print(f"\n    Last operation:\n    {sqls[-1][:400]}")
    print(f"\n    Reformulate the question or verify manually.")
    answer = ""  # caller must not get the hallucinated draft

elif answer:
    print(f"└── 💬 ANSWER\n")
    _print_answer(answer)
    notes = result.get("validation_feedback", "")
    if notes:
        print(f"\n    📌 {notes}")  # success path may still carry caveats
```

For HTTP/SSE pipelines, return a structured envelope:

```python
return {
    "status": status,                       # success | unverified | error
    "answer": answer if status == "success" else None,
    "unverified_reason": feedback if status != "success" else None,
    "last_operation": sqls[-1] if sqls else None,
    "iterations": result.get("iteration", 0),
}
```

The frontend renders the answer only when `status == "success"`. For `unverified`, render a banner + the last operation so the user can audit.

## When to apply

| Symptom | Apply this skill? |
|---------|-------------------|
| Final response contains values not present in tool outputs | YES — root cause is missing guard |
| Loop detected message appears but user still gets an answer | YES |
| Validator says "wrong data" 3× then a plausible answer prints | YES |
| Agent runs cleanly in one pass, answer is correct | NO — guard adds nothing |
| Single-shot LLM call (no Validator) | NO — different problem; use temperature 0 + cite-the-source prompting |

## Anti-patterns to refuse

1. **"Just make the Validator approve so the user gets something"** — that's how hallucinations ship. The Validator is the smoke alarm; silencing it doesn't fix the fire.
2. **Hiding the unverified state behind a generic error message** — operators need the last operation + concern to debug. Show them.
3. **Auto-retry from the sink on `status != success`** — retries belong inside the graph (`edge_after_validator`). The sink only renders.
4. **Logging the draft "for debugging" then printing the unverified banner** — the draft still leaks via logs. Drop it from state entirely (`answer = ""`).
5. **One global "is_valid" boolean** — too coarse. Distinguish `success`, `drift` (recoverable), `error` (tool failure), `aborted` (max_iter / loop).

## Implementation checklist

- [ ] `AgentState` has `status: str` with values `pending | success | drift | error | aborted`
- [ ] The Validator node sets `status` (don't infer from `validation_feedback` truthiness)
- [ ] `edge_after_validator` returns `END` when `status == "success"` OR `iteration >= MAX_ITER`
- [ ] The sink (CLI print, HTTP handler, SSE final event) checks `status` BEFORE rendering `current_answer`
- [ ] Unverified path includes: last operation, validator concern (truncated), iteration count
- [ ] The unverified path zeros out `answer` so downstream consumers can't accidentally render it
- [ ] A test fires a known-bad query and asserts that the response contains "UNVERIFIED" and does NOT contain the hallucinated value

## Real-world calibration

Triggering this guard on legitimate complex queries is a failure mode. Two safeguards:

1. **Validator must give actionable feedback or approve** — see the `langgraph-validator-with-evidence` skill. Vague rejections cause the guard to misfire on valid answers.
2. **Track guard-trigger rate** — if it fires on > 10% of queries, the pipeline upstream is broken (probably the Contractor producing wrong `expected_shape`); don't tune down the guard, fix the source.
