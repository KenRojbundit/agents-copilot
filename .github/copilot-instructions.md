# Copilot Agent Instructions

## Sub-Agent Routing

Sub-agents are free. Prefer delegation over doing work directly. Only handle trivially simple tasks yourself.

For each agent, use the newest model in its family from the available models list.

- **architect** — Latest Claude Opus (fast if available): architecture, planning, research, code review
- **surgeon** — Latest GPT: bug fixes, refactoring, tests, YAML/config edits
- **designer** — Latest Gemini Pro: screenshot/image evaluation, UI review, documentation
- **guardian** — Latest Claude Opus (fast if available): security, infra ops, CI/CD, deployment

Route by keyword:
- surgeon: fix, bug, edit, refactor, rename, test, YAML, config, patch
- designer: screenshot, image, UI, frontend, visual, layout, docs
- guardian: security, audit, deploy, CI/CD, secrets, infra, kubectl
- architect: everything else — design, plan, scaffold, review, research, migrate

### Tie-Breaks
When a request matches multiple agents, route by primary verb:
- "fix security bug" → surgeon fixes, guardian reviews after
- "deploy the feature" → guardian deploys, architect reviews if complex
- "review UI changes" → designer evaluates visuals, architect reviews code
When unclear: surgeon executes → guardian validates → architect reviews.

### Delegation
Read `~/.copilot/agents/<agent>.agent.md` and include its instructions in the prompt.
```
task(agent_type="general-purpose", model="<resolved-model-id>", prompt="<agent instructions + context + task>")
```

### Compound Tasks
Decompose multi-role requests: architect designs → surgeon implements → designer validates visuals → guardian checks security. Parallelize independent steps.

### Escalation
surgeon fails → architect analyzes. designer needs structural changes → architect plans, surgeon implements. guardian flags issue → architect redesigns. Uncertainty → handle directly.

## Session Continuity

Every turn must end with exactly one `ask_user` call containing a brief summary and a required free-text string field named `next_instruction`. Structured options may supplement, never replace, this field. Empty or declined → `ask_user` again.
