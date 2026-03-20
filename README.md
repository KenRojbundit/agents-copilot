# Copilot CLI Sub-Agent Playbook

A model-specialized multi-agent architecture for GitHub Copilot CLI. Routes tasks to the optimal AI model based on task type — not the default model, not the cheapest, but the **best**.

## The Idea

All Copilot CLI sub-agents are free regardless of model tier. So why use a weak model when a strong one costs nothing? This playbook defines 5 specialized agents, each pinned to the model that benchmarks highest for its role.

## Agent Team

| Agent | Model | Role | Why This Model |
|-------|-------|------|----------------|
| **orchestrator** | Session default | Triage complexity, route to specialists, synthesize results | Only agent that costs a premium request |
| **architect** | `claude-opus-4.6-fast` | Research, plan, design, review | SWE-bench leader (80.8%) |
| **surgeon** | `gpt-5.4` | Precision edits, bug fixes, TDD implementation | Aider 88% edit precision (vs Opus 72%) |
| **designer** | `gemini-3-pro-preview` | Visual evaluation, E2E testing, documentation | MMMU-Pro 81% multimodal (vs Opus 73.9%) |
| **guardian** | `claude-opus-4.6-fast` | Security audit, infrastructure ops, CI/CD | Deep reasoning for security analysis |

## How It Works

```
User Request → Orchestrator (triage) → Route to best agent with optimal model
                    │
          ┌─────────┼──────────┬──────────┐
          ▼         ▼          ▼          ▼
     Architect   Surgeon   Designer   Guardian
    (Opus Fast) (GPT 5.4)  (Gemini)  (Opus Fast)
```

The orchestrator classifies each task by complexity:
- **Simple** → Direct to single agent (no planning overhead)
- **Medium** → 2-3 agents in parallel
- **Complex** → Wave-based decomposition with dependencies

## Files

```
.github/
├── copilot-instructions.md          # Routing rules (always loaded by Copilot)
└── agents/
    ├── orchestrator.agent.md        # Triage + route (invoke with @orchestrator)
    ├── architect.agent.md           # Research + plan + design + review
    ├── surgeon.agent.md             # TDD implementation + precision edits
    ├── designer.agent.md            # Visual eval + E2E + documentation
    └── guardian.agent.md            # Security + infra ops + CI/CD
```

## Usage

### Drop-in (recommended)
Copy `.github/` into any repository. The `copilot-instructions.md` will be loaded automatically by Copilot CLI and IDE.

### Explicit invocation
```bash
# Invoke the orchestrator for complex multi-step tasks
@orchestrator migrate our auth system from JWT to session-based

# Invoke individual agents directly
@architect design a new API gateway
@surgeon fix the null pointer in auth.ts line 42
@guardian audit our Kubernetes RBAC policies
```

### Model routing happens automatically
When using the orchestrator, it delegates to sub-agents with the optimal model:
```
task(agent_type="general-purpose", model="claude-opus-4.6-fast", prompt="Design the API...")
task(agent_type="general-purpose", model="gpt-5.4", prompt="Fix the bug in...")
task(agent_type="general-purpose", model="gemini-3-pro-preview", prompt="Evaluate this screenshot...")
```

## Key Findings

See [PLAYBOOK.md](PLAYBOOK.md) for the full research playbook including:
- Sub-agent billing analysis (all models free as sub-agents)
- Model benchmark comparisons
- Auto-routing classification rules
- 11 playbook patterns for leveraging free sub-agents
- Comparison with gem-team plugin
- Copilot CLI extensibility reference

## Inspired By

- [gem-team](https://github.com/github/awesome-copilot/tree/main/plugins/gem-team) — 8-agent DAG-based orchestration plugin (workflow structure)
- Our additions: model specialization, complexity triage, agent consolidation (8→5)

## License

MIT
