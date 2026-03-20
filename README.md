# Copilot CLI Sub-Agent Playbook

A model-specialized multi-agent architecture for GitHub Copilot CLI. Routes tasks to the optimal AI model based on task type — not the default model, not the cheapest, but the **best**.

## The Idea

All Copilot CLI sub-agents are free regardless of model tier. So why use a weak model when a strong one costs nothing? This playbook defines 5 specialized agents, each pinned to the model that benchmarks highest for its role.

## Agent Team

| Agent | Model Line | Role | Why This Model |
|-------|-----------|------|----------------|
| **orchestrator** | Session default | Triage complexity, route to specialists, synthesize results | Only agent that costs a premium request |
| **architect** | Latest Claude Opus (fast) | Research, plan, design, review | SWE-bench leader, best reasoning + speed |
| **surgeon** | Latest GPT | Precision edits, bug fixes, TDD implementation | Best edit precision on Aider benchmarks |
| **designer** | Latest Gemini Pro | Visual evaluation, E2E testing, documentation | Best multimodal (MMMU-Pro leader) |
| **guardian** | Latest Claude Opus (fast) | Security audit, infrastructure ops, CI/CD | Deep reasoning for security analysis |

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

## Installation

### Option A: User-level (all repos, recommended)

Copy agents and instructions to your Copilot config directory. They'll be loaded automatically in **every** repository.

```bash
# Clone the repo
git clone https://github.com/KenRojbundit/copilot-agents.git
cd copilot-agents

# Copy agents to user-level config
mkdir -p ~/.copilot/agents
cp .github/agents/*.agent.md ~/.copilot/agents/
cp .github/copilot-instructions.md ~/.copilot/copilot-instructions.md
```

**Verify:** `ls ~/.copilot/agents/` should show all 5 agent files.

### Option B: Repo-level (single repo)

Copy `.github/` into a specific repository. Agents will only be available when working in that repo.

```bash
# From the target repo root
cp -r /path/to/copilot-agents/.github/agents .github/agents
cp /path/to/copilot-agents/.github/copilot-instructions.md .github/copilot-instructions.md
```

### Option C: Let Copilot do it

Point Copilot CLI at this repo and ask:

> "Read https://github.com/KenRojbundit/copilot-agents and install the agents at user level (~/.copilot/agents/)"

Copilot will fetch the files from GitHub and place them in the right location.

### Updating

Pull latest and re-copy:

```bash
cd copilot-agents && git pull
cp .github/agents/*.agent.md ~/.copilot/agents/
cp .github/copilot-instructions.md ~/.copilot/copilot-instructions.md
```

## Usage

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

## Copilot Plan Compatibility

The instructions specify model **lines** (e.g., "Latest Claude Opus") not exact versions. The agent resolves to the best available model at runtime.

**Note**: "Fast" variants (e.g., `claude-opus-4.6-fast`) are only available on Pro+ and Enterprise. On lower tiers, the agent will use the standard Opus/Sonnet variant instead.

| Tier | Price | Premium Requests | architect/guardian | surgeon | designer |
|------|-------|-----------------|-------------------|---------|----------|
| **Pro+** | $39/mo | 1,500/mo | Opus Fast | Latest GPT | Latest Gemini Pro |
| **Pro** | $10/mo | 300/mo | Sonnet | Latest GPT | Latest Gemini Pro |
| **Free** | $0 | 50/mo | Sonnet (older) | GPT (older) | Gemini Pro |

### Model Freshness

The system self-maintains — when routing, it checks available models for newer versions in the same family and prefers them. If `web_search` is available, it verifies benchmarks before switching. No manual model updates needed.

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
