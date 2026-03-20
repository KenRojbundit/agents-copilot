# Copilot Agent Instructions

## Sub-Agent Routing (Max Capability)

All sub-agents are free regardless of model. Always use the strongest available model for the job.

### Agents and Model Assignments

| Agent | Model | Fallback | When to Route |
|-------|-------|----------|---------------|
| **architect** | `claude-opus-4.6-fast` | `claude-sonnet-4.6` → `claude-sonnet-4` | Architecture, planning, research, scaffolding, code review, complex multi-step logic |
| **surgeon** | `gpt-5.4` | `gpt-4.1` | Targeted edits, bug fixes, refactoring, test writing, YAML/config surgery |
| **designer** | `gemini-3-pro-preview` | — | Screenshot/image evaluation, E2E testing, UI review, documentation |
| **guardian** | `claude-opus-4.6-fast` | `claude-sonnet-4.6` → `claude-sonnet-4` | Security auditing, infrastructure ops, CI/CD, health checks, deployment |
| **orchestrator** | Session default | — | Triage, route, synthesize (invoke with @orchestrator for complex multi-step work) |

Use the primary model. If it errors as unavailable, use the fallback.

### Signal Words

| Route to | Keywords |
|----------|----------|
| **surgeon** | "fix", "bug", "edit", "change this", "refactor", "rename", "test for", "YAML", "config", "patch", "tweak" |
| **designer** | "screenshot", "looks like", "UI", "frontend", "visual", "layout", "design review", "docs", "document" |
| **guardian** | "security", "audit", "deploy", "CI/CD", "health", "OWASP", "secrets", "infra ops", "kubectl", "pipeline" |
| **architect** | Everything else — "design", "plan", "scaffold", "review", "research", "migrate", "how should we" |

### Delegation Pattern
```
task(agent_type="general-purpose", model="<agent-model>", prompt="<full context + task>")
```

### Compound Tasks
For requests spanning multiple roles, decompose into parallel sub-agents:
1. **architect** (Opus Fast) designs the approach
2. **surgeon** (GPT 5.4) executes precise edits (one per file if parallel-safe)
3. **designer** (Gemini) validates visual output or writes docs (if applicable)
4. **guardian** (Opus Fast) validates security and runs health checks

### Escalation
- surgeon edit fails → escalate to architect for deeper analysis
- designer critique requires structural changes → architect plans, surgeon implements
- guardian flags security issue → architect redesigns, surgeon patches
- Any agent uncertain → orchestrator handles directly

## Session Continuity

Every turn must end with exactly one `ask_user` call containing a brief summary and a required free-text string field named `next_instruction`. Structured options may supplement, never replace, this field. Empty or declined → `ask_user` again.
