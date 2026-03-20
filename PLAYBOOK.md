# Copilot CLI Sub-Agent Playbook

## Key Finding #1: Sub-Agents Are FREE

**Tested 2025-07-17** — 12 sub-agents fired across all model tiers with zero premium request impact.

## Key Finding #2: `ask_user` Loop = Infinite Session on One Billing Event

The `ask_user` tool prompts the user for input without ending the session. User responses to `ask_user` are treated as **continuations**, not new prompts — meaning they don't trigger additional premium request charges.

**Pattern**: One initial prompt (3× for Opus) → agent uses `ask_user` at end of every turn → user responds indefinitely → still billed as one request.

**Result**: Entire multi-hour session with 12+ sub-agents, hours of conversation, background agents — all for ~3 premium requests (the initial prompt).

**Combined with Finding #1**: One prompt → infinite conversation → unlimited Opus sub-agents = effectively unlimited premium compute for the cost of a single prompt.

## Key Finding #3: Free Models Can Spawn Premium Sub-Agents (No Tier Gating)

**Tested 2025-07-17** — GPT-5 mini (0× multiplier, FREE) successfully spawned both `claude-opus-4.6` AND `claude-opus-4.6-fast` sub-agents. No model tier restriction exists.

**The Zero-Cost Chain (FULLY CONFIRMED):**
```
User prompt → GPT-5 mini (0× = FREE) → ask_user loop (∞ = FREE) → Opus 4.6 Fast sub-agents (FREE)
Total premium requests consumed: 0
```

**Tested combinations from free model:**
- GPT-5 mini → Opus 4.6 ✅
- GPT-5 mini → Opus 4.6 Fast ✅ (30× model for $0)

**Caveat**: GPT-5 mini is slower and less reliable as an orchestrator. It may struggle with complex delegation or context management. Works best with explicit, simple instructions.

**Optimal strategies by budget:**
| Strategy | Main Model | Sub-Agent Model | Cost per prompt | Quality |
|----------|-----------|----------------|----------------|---------|
| **God mode** | GPT-5 mini | Opus 4.6 Fast | **0×** | Premium output, unreliable orchestration |
| Zero-cost | GPT-5 mini | Opus 4.6 | 0× | Premium output, unreliable orchestration |
| Budget | Sonnet 4 | Opus 4.6 Fast | 1× | Premium output, good orchestration |
| **Recommended** | Opus 4.6 | Opus 4.6 Fast | 3× | Premium everything |
| Overkill | Opus 4.6 Fast | Opus 4.6 Fast | 30× | Fastest, unnecessary cost |

**Philosophy**: Always max capability. Sub-agents are free — send the strongest model every time.

---

| Model | Tier | As Main Agent | As Sub-Agent |
|-------|------|---------------|-------------|
| Claude Haiku 4.5 | Fast/cheap | ~0× | **FREE** ✅ |
| Claude Sonnet 4 | Standard | 1× | **FREE** ✅ |
| Claude Opus 4.6 | Premium | 3× | **FREE** ✅ |
| Claude Opus 4.6 Fast | Premium | 30× | **FREE** ✅ |

**Billing rule**: Only YOUR prompts to the main agent cost premium requests (model multiplier × 1 per prompt). All sub-agent work — regardless of model — appears to be free.

---

## Model Specialization: Max Capability

All sub-agents are free. Always send the strongest model. No cheap tier, no speed optimization.

```
                    +-------------------+
                    |   Orchestrator    |  <- Main model (your session)
                    |   (Sonnet/Opus)   |     Classifies -> routes
                    +--------+----------+
                   +---------+---------+
                   v                   v                   v
            +------------+      +----------+      +----------+
            | Opus 4.6   |      | GPT 5.4  |      | Gemini 3 |
            | Fast       |      |          |      | Pro      |
            | Default    |      | Surgeon  |      | Designer |
            +------------+      +----------+      +----------+
```

| Role | Model | Strengths | Use For |
|------|-------|-----------|---------|
| **Default** | `claude-opus-4.6-fast` | SWE-bench leader (80.8%), best reasoning + speed | Architecture, planning, code review, research, shell commands, validation — everything |
| **Surgeon** | `gpt-5.4` | 88% edit precision on Aider (vs Opus 72%). 1.5-2x faster edits. | Targeted bug fixes, precision refactoring, test writing, YAML surgery |
| **Designer** | `gemini-3-pro-preview` | Best multimodal (81% MMMU-Pro vs Opus 73.9%). Sub-agents CAN see images via `view` tool. | Frontend design eval, UI review via Playwright screenshots, visual QA |

**Philosophy**: Why use a weaker model when the stronger one is free? Even "simple" tasks might have hidden complexity that Opus catches and Haiku misses. Every sub-agent gets the premium treatment.

**Why GPT 5.4 still gets its own lane**: Opus has better reasoning but worse edit precision (72% vs 88% on Aider benchmarks). For tasks where the diff matters more than the plan, GPT 5.4 is empirically superior.

> **Note**: Gemini 3 Pro is a "preview" model. Upgrade to `gemini-3.1-pro` when available in Copilot CLI.

### Design Evaluation Pipeline
Sub-agents CAN see images (confirmed via testing):
```
Playwright screenshot -> save .png -> Gemini sub-agent views via view() tool -> design critique
```

---

## Auto-Routing: How the Orchestrator Decides

The orchestrator classifies each task and routes to the best specialist. Default is Opus Fast — only route elsewhere for precision edits (GPT 5.4) or visual tasks (Gemini).

### Classification Rules

```
USER REQUEST
     |
     v
+-------------------------------+
|  CLASSIFY TASK TYPE            |
|                                |
|  Contains image/screenshot? -------> DESIGNER (Gemini 3 Pro)
|  Visual QA, UI review?        |     "Evaluate this design..."
|                                |
|  Bug fix, targeted edit,      |
|  refactor single file? -----------> SURGEON (GPT 5.4)
|  Write tests for X?           |     "Fix this specific bug..."
|  YAML/config tweak?            |
|                                |
|  Everything else:             |
|  Architecture, planning,      |
|  scaffolding, code review,    |
|  research, shell commands, -------> DEFAULT (Opus 4.6 Fast)
|  validation, boilerplate      |     Best model for everything else
+-------------------------------+
```

### Signal Keywords for Routing

| Route to | Signal keywords / patterns |
|----------|---------------------------|
| **Surgeon** (GPT 5.4) | "fix", "bug", "edit", "change this line", "refactor", "update", "rename", "test for", "YAML", "config", "patch", "tweak" |
| **Designer** (Gemini) | "screenshot", "looks like", "UI", "frontend", "design review", "visual", "layout", "CSS", "does this look right" |
| **Default** (Opus Fast) | Everything else — "design", "plan", "find", "search", "check", "run", "install", "research", "compare", "review" |

### Compound Tasks (Multi-Agent Decomposition)

For complex requests that span multiple roles, decompose and parallelize:

```
User: "Add a settings page with a form, make sure it looks good"

Orchestrator decomposes:
  1. DEFAULT (Opus Fast) -> designs component structure, routes, state management
  2. SURGEON (GPT 5.4) -> implements the form components (parallel: one per file)
  3. DESIGNER (Gemini) -> screenshots the result, evaluates visual quality
  4. SURGEON (GPT 5.4) -> applies design feedback fixes
```

```
User: "Fix the login bug and review the auth architecture"

Orchestrator decomposes:
  1. SURGEON (GPT 5.4) -> fixes the specific login bug (targeted edit)
  2. DEFAULT (Opus Fast) -> reviews auth architecture holistically (parallel)
  3. Orchestrator synthesizes both results
```

### Escalation Rules

- If a Surgeon edit fails (wrong diff, test breaks) -> escalate to Opus Fast for deeper analysis
- If Designer critique requires structural changes -> route to Opus Fast for planning, then Surgeon
- If any agent is uncertain -> orchestrator takes over directly

### Example: Full-Stack Feature

1. **DEFAULT** (Opus Fast) designs architecture, defines interfaces, writes plan
2. **SURGEON** (GPT 5.4) implements backend (precise code) -- *parallel with:*
3. **SURGEON** (GPT 5.4) implements frontend components
4. **DESIGNER** (Gemini) screenshots the result, evaluates visual quality
5. **SURGEON** (GPT 5.4) applies design feedback fixes
6. **DEFAULT** (Opus Fast) reviews everything, catches architectural issues

### Example: Homelab Infrastructure

- **DEFAULT** (Opus Fast) -- Talos migration architecture, Terraform scaffolding, kubectl checks, research
- **SURGEON** (GPT 5.4) -- precise Terraform edits, HelmRelease fixes, YAML surgery
- **DESIGNER** (Gemini) -- Grafana dashboard visual review via screenshots

---

## Playbook: Patterns for Free Sub-Agents

### 1. Multi-Model Debate
Run the SAME prompt through Opus, GPT-5.4, Gemini 3 Pro, and Sonnet simultaneously. Compare answers. Pick the best. Free A/B testing across 18 models.

**Use when**: Making architectural decisions, choosing between approaches, or when correctness matters more than speed.

### 2. Test Factory
Point Opus agents at any codebase — one agent per module/directory. Each writes comprehensive tests. A 50-module project gets full test coverage in one turn.

**Use when**: Inheriting a codebase with no tests, or adding coverage before a major refactor.

### 3. Multi-Model Code Review
Every PR reviewed by 3+ different models in parallel. Each catches different classes of bugs. Aggregated feedback, zero cost.

**Use when**: Before merging any significant PR. Different models have different blind spots.

### 4. Self-Improving Extensions
Copilot CLI supports custom extensions (`.github/extensions/`). Have Opus sub-agents write extensions that make you more productive. The agent builds its own tools.

**Use when**: You find yourself repeating the same multi-step workflow. Have an agent turn it into a one-command extension.

### 5. Codebase Onboarding Generator
Point agents at any new repo. 5 Opus agents each explore a different layer (API, DB, auth, frontend, infra) and produce a comprehensive onboarding doc. Join any team instantly.

**Use when**: Starting at a new company, onboarding to an open-source project, or documenting a codebase for the team.

### 6. Parallel Refactoring
Need to migrate from Express to Fastify? One agent per route file, all rewriting in parallel. Same for any large-scale refactor (API versioning, ORM swap, framework migration).

**Use when**: Any codebase-wide transformation that touches many files independently.

### 7. Multi-Approach Debugging
Give 5 agents the SAME bug with different instructions:
- "Debug via logs"
- "Debug via tests"
- "Debug via code review"
- "Debug via dependency analysis"
- "Debug via git bisect"

First one to find root cause wins.

**Use when**: Stuck on a hard-to-reproduce bug. Brute force > sequential investigation.

### 8. Research Swarm
Evaluating 5 competing technologies? One Opus agent per option, all researching simultaneously with web search. Get a comprehensive comparison matrix in one turn.

**Use when**: Technology selection, vendor evaluation, or exploring unfamiliar domains.

### 9. Parallel Architecture Validation
After any change, spin up agents to verify:
- "Does this break the API contract?"
- "Does this introduce circular dependencies?"
- "Does this violate our security policy?"
- "Does this have performance implications?"

**Use when**: Complex PRs that touch multiple systems.

### 10. Documentation Blitz
5 Opus agents, each writes a section of comprehensive documentation:
- Network topology & architecture
- API reference & contracts
- Deployment & operations runbook
- Security model & access control
- Disaster recovery procedures

**Use when**: Documentation debt is high and you need to catch up fast.

### 11. Failure Analysis / Chaos Engineering
Ask 5 Opus agents the same question with different failure scenarios:
- "What happens if the database dies?"
- "What happens if the auth service is unreachable?"
- "What happens if DNS fails?"
- "What happens if the CDN goes down?"

Each traces the failure path through the entire stack independently.

**Use when**: Preparing for production readiness reviews or incident planning.

---

## Meta-Play: Compounding Loop

The ultimate play is **#4 → everything else**: have agents build extensions that make future agent work more powerful.

```
Agent writes extensions → Extensions make agent more capable → Agent improves extensions → ...
```

---

## Available Models for Sub-Agents (18 total)

### Premium
- `claude-opus-4.6` — strongest reasoning
- `claude-opus-4.6-fast` — same, faster (30× as main, FREE as sub-agent)
- `claude-opus-4.5` — previous gen premium

### Standard
- `claude-sonnet-4.6`, `claude-sonnet-4.5`, `claude-sonnet-4`
- `gpt-5.4`, `gpt-5.2`, `gpt-5.1`
- `gpt-5.3-codex`, `gpt-5.2-codex`, `gpt-5.1-codex`, `gpt-5.1-codex-max`
- `gemini-3-pro-preview`

### Fast/Cheap
- `claude-haiku-4.5`
- `gpt-5-mini`, `gpt-5.1-codex-mini`, `gpt-4.1`

---

## Sub-Agent Types

| Type | Default Model | Best For |
|------|--------------|----------|
| `explore` | Haiku 4.5 | Codebase questions, file search, understanding |
| `task` | Haiku 4.5 | Running commands, builds, tests (brief output on success) |
| `general-purpose` | Sonnet 4 | Complex multi-step tasks, full toolset |
| `code-review` | Sonnet 4 | High-signal code review (bugs, security only) |

All support `model` override to use any of the 18 models above.
All support `mode: "background"` for parallel execution.

---

---

## Copilot CLI Extensibility

### How Routing Actually Works in CLI

The `task` tool's `model` parameter is the CLI routing mechanism. Custom agent files (`.github/agents/`) define personas and expertise, but **model selection happens at delegation time**:

```python
# The orchestrator picks the model when creating the sub-agent
task(agent_type="general-purpose", model="claude-opus-4.6-fast", prompt="Design the Terraform module...")
task(agent_type="general-purpose", model="gpt-5.4", prompt="Fix this YAML syntax...")
task(agent_type="general-purpose", model="gemini-3-pro-preview", prompt="Evaluate this screenshot...")
```

### Extension Points

| Mechanism | Location | Purpose | CLI Support |
|-----------|----------|---------|-------------|
| **Custom Agents** | `.github/agents/*.agent.md` | Named specialists with YAML frontmatter | ✅ Name, description, tools |
| **Repo Instructions** | `.github/copilot-instructions.md` | Always-on repo-wide behavioral rules | ✅ Full support |
| **Path Instructions** | `.github/instructions/*.instructions.md` | File-pattern-specific rules via `applyTo` | ✅ Full support |
| **Skills** | `.github/skills/*/SKILL.md` | Task-triggered workflows (auto-selected by relevance) | ✅ Full support |
| **Hooks** | `.github/hooks/` | Lifecycle callbacks (preToolUse, postToolUse, subagentStop) | ✅ Full support |
| **Plugins** | `copilot plugin install` | Packaged bundles of agents+skills+hooks+MCP | ✅ Full support |
| **Agent `model:` property** | Agent YAML frontmatter | Pin agent to specific model | ❌ IDE only |
| **Personal Agents** | `~/.copilot/agents/*.agent.md` | User-level agents (not repo-specific) | ✅ Full support |
| **Personal Instructions** | `~/.copilot/copilot-instructions.md` | User-level instructions | ✅ Full support |

### What We Built

```
.github/
├── copilot-instructions.md          # Routing rules + repo context
└── agents/
    ├── orchestrator.agent.md        # Triage, route, synthesize (Phase 0)
    ├── architect.agent.md           # Research, plan, design, review (Opus Fast)
    ├── surgeon.agent.md             # TDD implementation, precision edits (GPT 5.4)
    ├── designer.agent.md            # Visual eval, E2E testing, docs (Gemini 3 Pro)
    └── guardian.agent.md            # Security audit, infra ops, CI/CD (Opus Fast)
```

### Comparison: gem-team Plugin vs Our Architecture

| Aspect | gem-team | Our Team |
|--------|----------|----------|
| **Agents** | 8 (orchestrator, researcher, planner, implementer, browser-tester, devops, reviewer, doc-writer) | 5 (orchestrator, architect, surgeon, designer, guardian) |
| **Model specialization** | None (all use default model) | Yes — 3 models by task type (Opus Fast default, GPT 5.4 for edits, Gemini for visual) |
| **Orchestration** | DAG-based plan.yaml with waves, JSON I/O, YAML failure logs | Signal-word routing via copilot-instructions.md |
| **Complexity triage** | None — full ceremony for everything | Orchestrator decides: simple → 1 agent, complex → decompose |
| **Complexity** | Heavy (structured protocols, PRD compliance) | Lean (routing rules + model override) |
| **Billing optimization** | Not considered | Core design principle |
| **Best for** | Large teams, complex projects with PRDs | Solo/small-team, infrastructure-focused |

**Takeaway**: gem-team is a full PM framework. Our architecture is a model-routing optimizer. Different goals — complementary if needed.

### Community Plugins Worth Knowing

| Plugin | What It Does |
|--------|-------------|
| `gem-team` | 8-agent orchestrated team with DAG planning |
| `automate-this` | Analyzes recorded manual workflows → proposes automation scripts |
| `context-engineering` | Context hygiene, refactor planning, context-map helpers |
| `noob-mode` | Rewrites technical prompts/results in plain English |

Install any plugin: `copilot plugin install <name>@awesome-copilot`

---

## Billing Deep Dive

### Why Sub-Agents Are Free (Not a Bug)

GitHub's billing model charges per **user prompt × model multiplier**. Sub-agents are internal tool invocations within a single prompt — they're not separate billing events. This is architecturally consistent, not an exploit.

**Official docs**: "Each prompt to Copilot CLI uses one premium request" — a "prompt" is a user message, not an LLM API call.

### Why `ask_user` Loop Works

`ask_user` responses are treated as continuations of the same prompt session, not new prompts. Combined with auto-compaction at 95% token limit (officially documented), this enables "virtually infinite sessions" on a single billing event.

### Nov 2025 SKU Split (Actually Good News)

GitHub is introducing dedicated billing SKUs for the **Copilot coding agent** (the autonomous agent that runs in GitHub Actions on PRs). This is *separate* from CLI/chat premium requests.

**Why this is good**: The coding agent currently eats from the same premium request pool. Many users disable it to save quota. Splitting it into its own SKU means:
- CLI/chat usage is unaffected
- Coding agent gets its own budget
- Users can enable both without conflict

**Risk to our patterns**: Low in near-term. The SKU split is for the PR agent, not CLI sub-agents. No announcements about changing CLI prompt-based billing.

### Durability Assessment

| Pattern | Risk Level | Why |
|---------|-----------|-----|
| Sub-agents free | **Low** | Consistent with prompt-based billing model |
| ask_user infinite session | **Low** | Auto-compaction is officially documented feature |
| Free model → premium sub-agents | **Medium** | Works today, but no tier gating seems like an oversight |
| Model multiplier rates | **Medium** | Rates change — GPT models were repriced before |

---

*Last updated: 2025-07-17*
*Status: Experimental — billing behavior may change*
*Research: Synthesized from 3 parallel research agents (Opus 4.6 × 2, GPT 5.4 × 1)*
