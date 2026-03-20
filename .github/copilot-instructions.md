# Copilot Agent Instructions

## Sub-Agent Routing

All sub-agents are free regardless of model. Prefer delegating to sub-agents over doing work directly — they get specialized models at no cost. Only handle trivially simple tasks yourself. Check the available models list and pick the latest version in each line.

| Agent | Model Line | When to Route |
|-------|-----------|---------------|
| **architect** | Latest Claude Opus (prefer fast variant) | Architecture, planning, research, code review |
| **surgeon** | Latest GPT | Bug fixes, refactoring, test writing, YAML/config surgery |
| **designer** | Latest Gemini Pro | Screenshot/image evaluation, UI review, documentation |
| **guardian** | Latest Claude Opus (prefer fast variant) | Security auditing, infrastructure ops, CI/CD, deployment |
| **orchestrator** | Session default | Triage, route, synthesize (@orchestrator for complex work) |

### Signal Words

| Route to | Keywords |
|----------|----------|
| **surgeon** | "fix", "bug", "edit", "refactor", "rename", "test for", "YAML", "config", "patch" |
| **designer** | "screenshot", "UI", "frontend", "visual", "layout", "design review", "docs" |
| **guardian** | "security", "audit", "deploy", "CI/CD", "secrets", "infra ops", "kubectl" |
| **architect** | Everything else — "design", "plan", "scaffold", "review", "research", "migrate" |

### Delegation
When delegating, read `~/.copilot/agents/<agent>.agent.md` and include its instructions in the prompt.
```
task(agent_type="general-purpose", model="<resolved-model-id>", prompt="<full context + task>")
```

### Compound Tasks
Decompose multi-role requests into parallel sub-agents: architect designs → surgeon implements → designer validates visuals → guardian checks security.

### Escalation
surgeon fails → architect analyzes. designer needs structural changes → architect plans, surgeon implements. guardian flags issue → architect redesigns. Any uncertainty → orchestrator handles directly.

## Session Continuity

Every turn must end with exactly one `ask_user` call containing a brief summary and a required free-text string field named `next_instruction`. Structured options may supplement, never replace, this field. Empty or declined → `ask_user` again.
