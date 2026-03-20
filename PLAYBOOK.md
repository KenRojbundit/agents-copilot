# Copilot CLI Sub-Agent Playbook

Research findings from building a model-specialized multi-agent architecture for Copilot CLI.

## Core Findings

### 1. Sub-Agents Are Free
All sub-agent invocations (via `task` tool) cost zero premium requests regardless of model. Billing is per **user prompt × model multiplier** — sub-agents are internal tool calls within a single prompt, not separate billing events.

### 2. `ask_user` = Infinite Session
`ask_user` responses are continuations, not new prompts. Combined with auto-compaction at 95% token limit, one initial prompt powers an entire multi-hour session with unlimited sub-agents.

### 3. Free Models Spawn Premium Sub-Agents
GPT-5 mini (0× multiplier) can spawn Opus 4.6 Fast (30×) sub-agents. No tier gating exists.

**The zero-cost chain**: User → GPT-5 mini (free) → ask_user loop (free) → Opus sub-agents (free) = 0 premium requests.

**Caveat**: GPT-5 mini is unreliable as an orchestrator. Recommended: Opus 4.6 as main (3×) with Opus Fast sub-agents.

### Cost Strategies

| Strategy | Main Model | Sub-Agent Model | Cost | Quality |
|----------|-----------|----------------|------|---------|
| Zero-cost | GPT-5 mini | Opus 4.6 Fast | 0× | Premium output, unreliable orchestration |
| Budget | Sonnet 4 | Opus 4.6 Fast | 1× | Premium output, good orchestration |
| **Recommended** | Opus 4.6 | Opus 4.6 Fast | 3× | Premium everything |

### Billing per Model

| Model | As Main | As Sub-Agent |
|-------|---------|-------------|
| Haiku 4.5 | ~0× | FREE |
| Sonnet 4 | 1× | FREE |
| Opus 4.6 | 3× | FREE |
| Opus 4.6 Fast | 30× | FREE |

---

## Model Specialization

Always use the strongest model — sub-agents are free.

| Role | Model | Why | Use For |
|------|-------|-----|---------|
| **architect/guardian** | Opus 4.6 Fast | SWE-bench 80.8% | Architecture, planning, review, security, infra ops |
| **surgeon** | GPT 5.4 | Aider 88% edit precision (vs Opus 72%) | Bug fixes, refactoring, tests, YAML surgery |
| **designer** | Gemini 3 Pro | MMMU-Pro 81% (vs Opus 73.9%) | Visual eval, UI review, screenshots, docs |

GPT 5.4 gets its own lane because Opus has better reasoning but worse edit precision (72% vs 88%). For tasks where the diff matters more than the plan, GPT 5.4 wins.

Sub-agents CAN see images: `Playwright screenshot → .png → Gemini sub-agent views via view() tool → critique`.

---

## Playbook Patterns

| # | Pattern | Description | When |
|---|---------|-------------|------|
| 1 | **Multi-Model Debate** | Same prompt through 3+ models, compare answers | Architectural decisions, correctness-critical |
| 2 | **Test Factory** | One agent per module, each writes tests | No test coverage, pre-refactor |
| 3 | **Multi-Model Review** | 3+ models review same PR in parallel | Before merging significant PRs |
| 4 | **Self-Improving Extensions** | Agents write `.github/extensions/` that make them more capable | Repeated multi-step workflows |
| 5 | **Codebase Onboarding** | 5 agents explore different layers → onboarding doc | New repo, new team, open-source |
| 6 | **Parallel Refactoring** | One agent per file, all rewriting in parallel | Large-scale migrations |
| 7 | **Multi-Approach Debug** | 5 agents debug same bug via different methods (logs, tests, bisect...) | Hard-to-reproduce bugs |
| 8 | **Research Swarm** | One agent per technology option, all researching with web search | Technology selection |
| 9 | **Architecture Validation** | Parallel agents check: API contract, circular deps, security, perf | Complex multi-system PRs |
| 10 | **Documentation Blitz** | 5 agents each write one section of docs | Documentation debt |
| 11 | **Failure Analysis** | 5 agents each trace a different failure scenario through the stack | Production readiness |

**Meta-play**: Pattern #4 feeds all others — agents build extensions that make future agent work more powerful. Self-improving loop.

---

## CLI Extensibility Reference

Routing works via `task` tool's `model` parameter. Agent `.md` files define personas; model selection happens at delegation time.

| Mechanism | Location | CLI Support |
|-----------|----------|-------------|
| Custom Agents | `.github/agents/*.agent.md` | ✅ |
| Repo Instructions | `.github/copilot-instructions.md` | ✅ |
| Path Instructions | `.github/instructions/*.instructions.md` | ✅ |
| Skills | `.github/skills/*/SKILL.md` | ✅ |
| Hooks | `.github/hooks/` | ✅ |
| Plugins | `copilot plugin install` | ✅ |
| Agent `model:` frontmatter | Agent YAML | ❌ IDE only |
| Personal Agents | `~/.copilot/agents/*.agent.md` | ✅ |
| Personal Instructions | `~/.copilot/copilot-instructions.md` | ✅ |

### vs gem-team

| | gem-team | Our Architecture |
|-|----------|-----------------|
| Agents | 8 (generic roles) | 5 (model-specialized) |
| Model routing | None (all default) | 3 models by task type |
| Orchestration | DAG + YAML waves | Signal-word routing |
| Best for | Large teams, PRD-driven | Solo/small-team, lean |

### Community Plugins

`gem-team` (8-agent DAG), `automate-this` (workflow→scripts), `context-engineering` (context hygiene), `noob-mode` (plain English). Install: `copilot plugin install <name>@awesome-copilot`.

---

## Billing Durability

| Pattern | Risk | Notes |
|---------|------|-------|
| Sub-agents free | Low | Consistent with prompt-based billing model |
| ask_user infinite session | Low | Auto-compaction is officially documented |
| Free model → premium sub-agents | Medium | No tier gating seems like an oversight |
| Model multiplier rates | Medium | Rates have changed before |

**Nov 2025 SKU split**: Coding agent (GitHub Actions PR agent) getting separate billing SKU. Good — means CLI premium requests unaffected. No announced changes to CLI sub-agent billing.

---

*Last updated: 2025-07-17 · Status: Experimental — billing behavior may change*
