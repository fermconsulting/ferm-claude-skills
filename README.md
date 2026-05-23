# ferm-claude-skills

Reusable Claude Code skills extracted from real FERM Consulting projects.
Two families:

## 1. Reliable multi-agent LangGraph pipelines

For any agent system with a Validator/Critic node, multi-step tool calls, and an
LLM-generated final answer:

| Skill | What it prevents |
|---|---|
| **langgraph-validator-with-evidence** | Blind Validator rejecting good answers / approving hallucinations because it sees only prose, not raw tool outputs |
| **langgraph-loop-prevention** | Agent repeating the same SQL/tool call · Validator ↔ Agent ping-pong on vague feedback · `MAX_ITER` hits on simple queries |
| **langgraph-output-guard** | Final pipeline sink rendering an unvalidated draft → fabricated values like inventing IDs or numbers the SQL never returned |

These three compose: evidence → specific feedback → loops broken → guard ensures no hallucination ships if the pipeline still fails.

## 2. Production deployment on a shared Linux server

For deploying Dockerized apps behind Nginx Proxy Manager (the FERM Hetzner/Ubuntu box):

| Skill | What it covers |
|---|---|
| **deploy-docker-behind-npm** | Full recipe: Docker network connectivity, internal-vs-host port trap, NPM proxy host + Let's Encrypt SSL, private-repo cloning, recovering data/env from a running container, the 502/400/404 gotcha catalogue |
| **nginx-sse-and-subpath-gotchas** | SSE/EventSource streaming through nginx (`proxy_buffering off`), subpath base-href/relative-URL fixes, subdomain-vs-subpath decision, browser-cache traps |

## Install (per machine, once)

```bash
git clone https://github.com/ogarciab/ferm-claude-skills ~/.claude/skills-ferm

# Symlink each skill into the active skills directory
mkdir -p ~/.claude/skills
for s in langgraph-output-guard langgraph-validator-with-evidence langgraph-loop-prevention \
         deploy-docker-behind-npm nginx-sse-and-subpath-gotchas; do
  ln -sf ~/.claude/skills-ferm/$s ~/.claude/skills/$s
done
```

After this, **any project you open with Claude Code on this machine** has access to all three skills. They activate automatically when the description matches the task you're describing.

## Update

```bash
cd ~/.claude/skills-ferm && git pull
```

Symlinks point at the live directory, so an update propagates immediately to all machines that pulled.

## Manual activation

If you want to apply a specific pattern proactively:

> "Use the langgraph-output-guard pattern to add an anti-hallucination guard to my run()"

> "Apply langgraph-validator-with-evidence — my Validator can't see the tool outputs"

> "I need langgraph-loop-prevention — the agent is repeating the same query"

## Origin

Extracted from production patches to `sql-explorer` (the `feature/anti-hallucination` branch), May 2026. Each skill includes the concrete failure mode that motivated it ("Validator rejected with 'verify the data' three times then the answer said WO-0001 — a value not in any SQL result").

## License

MIT. Use freely.
