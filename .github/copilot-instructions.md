# Copilot Agent Instructions

## Sub-Agent Routing

Sub-agents are free. Prefer delegation over doing work directly. Only handle trivially simple tasks yourself.

### Model Resolution

Resolve model IDs from the available-models list at runtime. Never hardcode versions.

- **architect, guardian** → highest-version `claude-opus-*`. Prefer `-fast` over base at the same version.
- **surgeon** → highest-version **unsuffixed** GPT (`gpt-{version}` only). Ignore `-codex`, `-mini`, `-max` — they are variants, not upgrades. Fallback to newest non-mini variant only if no unsuffixed GPT exists.
- **designer** → highest-version Gemini Pro (ID contains `gemini-` and `-pro`). Prefer stable over `-preview` at the same version.

Version order is numeric descending (`5.4` > `5.3` > `5.1`). Sub-agents are free — always pick the strongest.

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
Prefer `mode="background"` for blocking/long-running sub-agent tasks (builds, tests, large refactors) so you stay responsive to the user. Continue communicating while background agents work — you'll be notified on completion.

### Compound Tasks
Decompose multi-role requests: architect designs → surgeon implements → designer validates visuals → guardian checks security. Parallelize independent steps.

### Escalation
surgeon fails → architect analyzes. designer needs structural changes → architect plans, surgeon implements. guardian flags issue → architect redesigns. Uncertainty → handle directly.

## Session Continuity

Every turn must end with exactly one `ask_user` call containing a brief summary and a required free-text string field named `next_instruction`. Structured options may supplement, never replace, this field. Empty or declined → `ask_user` again.
