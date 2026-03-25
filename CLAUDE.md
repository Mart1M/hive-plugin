# HIVE System — Claude Code Plugin Context

You are operating within a project that uses the **HIVE architecture framework** (Hub-based Intelligence & Verified Execution).

## What HIVE Is

HIVE is an agentic architecture framework that organizes AI work into five specialized hubs, orchestrated by a central coordinator, all sharing a structured JSON state. Its purpose is to make agentic systems **modular, governed, explainable, maintainable, versionable, and reusable**.

## Hub-and-spoke model

HIVE follows a **hub-and-spoke** pattern: each hub is the single center of responsibility for its stage. Agents inside a hub are specialized spokes — they consolidate outputs into that hub’s state section and hand off through the orchestrator. Avoid unconstrained many-to-many agent collaboration outside this structure; it weakens traceability and blurs boundaries.

## The Five Hubs and Their Roles

| Hub | Purpose | Writes to state section |
|---|---|---|
| **Intent** | Parse raw request → structured intention | `state.intent` |
| **Ground** | Gather all knowledge needed before composition | `state.ground` |
| **Compose** | Build a structured solution from intent + ground | `state.compose` |
| **Verify** | Validate solution before delivery | `state.verify` |
| **Deliver** | Produce the final actionable artifact | `state.deliver` |

## Canonical Execution Flow

```
Intent → Ground → Compose → Verify → Deliver
```

If Verify returns `valid: false`, trigger remediation then re-verify before proceeding to Deliver.

## Pipeline variants

- **Full pipeline (default):** Intent → Ground → Compose → Verify → Deliver.
- **Simplified pipeline:** When intent is already clear or `state.intent` is pre-filled (ticket, brief, prior `/hive-intent`), you may run **Ground → Compose → Verify** only — e.g. `/hive-run --only ground,compose,verify` or `/hive-run --from ground` with a loaded state. **Never** skip Verify before Deliver when a deliverable is required.

## The Shared State

The shared state is the common memory of the system. It is structured JSON — **never free text as the primary structure**. Each hub writes only to its designated section.

```json
{
  "request": { "id": "", "source": "", "raw_input": "" },
  "intent": {},
  "ground": {},
  "compose": {},
  "verify": {},
  "deliver": {},
  "meta": {
    "version": "1.0.0",
    "status": "initialized",
    "hub_runs": []
  }
}
```

**Traceability (`meta.hub_runs`):** After each hub completes in a run, append an entry so execution stays inspectable: `hub` (e.g. `intent`), `started_at` / `completed_at` (ISO 8601), `input_snapshot` (e.g. keys or short summary of sections read), `output_snapshot` (e.g. keys written), `warnings_count` (optional). No disk file is required when state lives only in the session.

When running any HIVE hub or orchestrator command, maintain this state in the working session. If `.hive/` exists in the project, load state from `.hive/state.schema.json` and the active run context.

## Hub Definitions

### Intent Hub
- **Goal**: Transform a raw request into a structured intention
- **Agents**: Goal Interpreter, Scope Clarifier, Constraint Extractor, Success Criteria
- **Output shape**:
```json
{
  "goal": "...",
  "domain": "...",
  "constraints": [],
  "success_criteria": []
}
```

### Ground Hub
- **Goal**: Gather all necessary knowledge before composition — limit hallucination, connect to project reality
- **Agents**: Pattern Resolver, Component Resolver, Token Resolver, Rule Resolver, Documentation Resolver, Example Resolver, Platform Context, Governance Precheck, Context Aggregator
- **Output shape**:
```json
{
  "patterns": [],
  "components": [],
  "tokens": {},
  "rules": {},
  "docs": [],
  "examples": []
}
```

### Compose Hub
- **Goal**: Build a structured solution from intent and grounding
- **Agents**: Layout Composer, Component Composition, Interaction Flow, State Mapping, Output Blueprint
- **Output shape**: blueprint JSON, component tree, layout structure, interaction map

### Verify Hub
- **Goal**: Validate composed solution before delivery
- **Agents**: Compliance, Accessibility, Technical Feasibility, Consistency, Remediation
- **Output shape**:
```json
{
  "valid": true,
  "warnings": [],
  "issues": [],
  "fixes": []
}
```

### Deliver Hub
- **Goal**: Transform validated solution into actionable artifact
- **Agents**: Code Generator, Specification Writer, Documentation, Delivery Formatter, Publisher
- **Output formats**: code scaffold, JSON blueprint, markdown documentation, dev ticket, config file

## Claude Code integration (interpretable layer)

HIVE need not be a standalone runtime. In Claude Code it is an **interpretable layer**: markdown and JSON under `.hive/` that you read, follow, and execute. **Read first** when present: `hive.config.json`, `state.schema.json`, `orchestrator/main-orchestrator.md`, `hubs/**/hub.md` and relevant agents, `skills/registry.json`, `rules/*.md`.

Some HIVE documentation uses example slash commands such as `/generate-hive` or `/run-ground`. In this plugin the **actual commands** are: `/hive-init`, `/hive-run`, `/hive-intent`, `/hive-ground`, `/hive-compose`, `/hive-verify`, `/hive-deliver`, `/hive-status`. Treat the doc examples as conceptual aliases of these.

For the full implementation checklist and migration path, see `hive_system_documentation.md` in the plugin (e.g. section 17).

## Core Rules You Must Follow

1. **Never conflate hub responsibilities.** Do not perform grounding inside Compose, or validation inside Deliver.
2. **Always write structured outputs.** Hub outputs must be JSON or well-structured markdown — not prose narration.
3. **Never skip Verify.** A composed solution must always pass Verify before reaching Deliver.
4. **Write only to your hub's state section.** Intent writes to `state.intent`, Ground to `state.ground`, etc.
5. **Keep the Main Orchestrator thin.** It controls flow and state transitions — it does not contain business logic.
6. **Skills must be explicit and limited.** Do not create catch-all skills that "do anything".
7. **Each hub must be runnable in isolation.** A hub invoked directly (via `/hive-intent`, `/hive-ground`, etc.) must produce a valid output without requiring other hubs to have run first.

## Skills (registry contract)

Each skill in `skills/registry.json` should be defined with: **name**, **purpose** (one line), **input_schema**, **output_schema**, **limits**, and **error_cases** (or equivalent fields). This keeps tooling auditable and prevents vague “do anything” capabilities.

## Anti-Patterns — Never Do These

- **Monolithic agent**: one agent that does everything (understand + ground + compose + validate + deliver)
- **Implicit state**: relying on conversational context as your primary state mechanism
- **Missing verification**: composing then delivering without a Verify step
- **Over-broad skills**: skills with no explicit input/output schema or guardrails
- **Blurry hub boundaries**: hub A performing work that belongs to hub B

## File Structure Reference

When `.hive/` exists in a project, it has this structure:

```
.hive/
  hive.config.json           ← domain, outputs, truth sources, governance
  state.schema.json          ← state schema definition
  orchestrator/
    main-orchestrator.md     ← orchestration rules
  hubs/
    intent/hub.md + agents/
    ground/hub.md + agents/
    compose/hub.md + agents/
    verify/hub.md + agents/
    deliver/hub.md + agents/
  skills/
    registry.json            ← all available skills
    patterns/ components/ tokens/ validation/ output/
  rules/
    naming.md
    handoffs.md
    verification.md
    governance.md
  prompts/
    conventions.md
  presets/
    design-system/
    product-spec/
    coaching/
```

## Available Commands

In **Claude Code** with this plugin installed from a marketplace, commands are **namespaced**: `/hive:hive-init`, `/hive:hive-run`, etc. The list below uses the short form; substitute `/hive:` when your environment requires it.

- `/hive-init` — scaffold `.hive/` in the current repo
- `/hive-run` — run the full HIVE pipeline
- `/hive-intent` — run the Intent Hub in isolation
- `/hive-ground` — run the Ground Hub in isolation
- `/hive-compose` — run the Compose Hub in isolation
- `/hive-verify` — run the Verify Hub in isolation
- `/hive-deliver` — run the Deliver Hub in isolation
- `/hive-status` — show current state
