# /hive-init

Scaffold the full HIVE architecture directory structure (`.hive/`) in the current project repository.

## Usage

```
/hive-init
/hive-init --domain design-system
/hive-init --domain product-spec --preset minimal
```

## Arguments

- `--domain` (optional): The domain preset to use. Options: `design-system`, `product-spec`, `coaching`, `tech-docs`, `generic` (default: `generic`)
- `--preset` (optional): Scaffold detail level. Options: `full` (default), `minimal`

---

## What This Command Does

When invoked, create the `.hive/` directory structure in the project root. With **`--preset full`** (default), create every agent file listed under [Directory Structure to Create](#directory-structure-to-create). With **`--preset minimal`**, follow [Preset `minimal` (MVP)](#preset-minimal-mvp). Use the `--domain` argument to tailor example content in `hive.config.json`, `registry.json`, and agent stubs.

**Always confirm the target directory** before creating files. If `.hive/` already exists, warn the user and ask if they want to overwrite or merge.

---

## Directory Structure to Create

```
.hive/
  hive.config.json
  state.schema.json
  orchestrator/
    main-orchestrator.md
  hubs/
    intent/
      hub.md
      agents/
        goal-interpreter.md
        scope-clarifier.md
        constraint-extractor.md
        success-criteria.md
    ground/
      hub.md
      agents/
        pattern-resolver.md
        component-resolver.md
        token-resolver.md
        rule-resolver.md
        documentation-resolver.md
        example-resolver.md
        platform-context.md
        governance-precheck.md
        context-aggregator.md
    compose/
      hub.md
      agents/
        layout-composer.md
        component-composer.md
        interaction-flow.md
        state-mapper.md
        blueprint-agent.md
    verify/
      hub.md
      agents/
        compliance-agent.md
        accessibility-agent.md
        technical-feasibility.md
        consistency-agent.md
        remediation-agent.md
    deliver/
      hub.md
      agents/
        code-generator.md
        specification-writer.md
        documentation-agent.md
        delivery-formatter.md
        publisher-agent.md
  skills/
    registry.json
    patterns/
    components/
    tokens/
    validation/
    output/
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

---

## Preset `minimal` (MVP)

Use `--preset minimal` for a **small first version**: same top-level layout (`orchestrator/`, all five `hubs/`, `skills/` subfolders, `rules/`, `prompts/`, `presets/`) so you can grow without renaming paths. Differences from `full`:

| Area | `full` | `minimal` |
|------|--------|-----------|
| Intent agents | 4 files | `goal-interpreter.md`, `constraint-extractor.md`, `success-criteria.md` |
| Ground agents | 9 files | `pattern-resolver.md`, `component-resolver.md`, `context-aggregator.md` |
| Compose agents | 5 files | `layout-composer.md`, `blueprint-agent.md`, `component-composer.md` |
| Verify agents | 5 files | `compliance-agent.md`, `accessibility-agent.md`, `remediation-agent.md` |
| Deliver agents | 5 files | `code-generator.md`, `documentation-agent.md`, `delivery-formatter.md` |
| `skills/registry.json` | Full set for domain | **3–6** skills with name, description, `input_schema`, `output_schema`, `limits`, `error_cases` — only what’s needed for the chosen `--domain` |

Each hub’s `hub.md` should **list the omitted agents** as “deferred” (e.g. Example Resolver, Governance Precheck) so teams know what to add in phase 2. Empty skill category folders (`skills/patterns/`, etc.) are still created for later registration.

---

## File Contents to Generate

### `.hive/hive.config.json`

```json
{
  "version": "1.0.0",
  "domain": "<domain>",
  "project": "<project-name>",
  "description": "",
  "truth_sources": [],
  "output_formats": ["code", "markdown", "json"],
  "governance": {
    "require_verify": true,
    "require_grounding": true,
    "allow_direct_delivery": false
  },
  "hubs": {
    "intent": { "enabled": true },
    "ground": { "enabled": true },
    "compose": { "enabled": true },
    "verify":  { "enabled": true },
    "deliver": { "enabled": true }
  }
}
```

### `.hive/state.schema.json`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "HIVE Shared State",
  "type": "object",
  "properties": {
    "request": {
      "type": "object",
      "properties": {
        "id":        { "type": "string" },
        "source":    { "type": "string" },
        "raw_input": { "type": "string" },
        "timestamp": { "type": "string", "format": "date-time" }
      },
      "required": ["id", "raw_input"]
    },
    "intent": {
      "type": "object",
      "properties": {
        "goal":             { "type": "string" },
        "domain":           { "type": "string" },
        "constraints":      { "type": "array", "items": { "type": "string" } },
        "success_criteria": { "type": "array", "items": { "type": "string" } },
        "scope":            { "type": "string" }
      }
    },
    "ground": {
      "type": "object",
      "properties": {
        "patterns":    { "type": "array" },
        "components":  { "type": "array" },
        "tokens":      { "type": "object" },
        "rules":       { "type": "object" },
        "docs":        { "type": "array" },
        "examples":    { "type": "array" }
      }
    },
    "compose": {
      "type": "object",
      "properties": {
        "blueprint":    {},
        "layout":       {},
        "components":   { "type": "array" },
        "interactions": { "type": "array" },
        "state_map":    { "type": "object" }
      }
    },
    "verify": {
      "type": "object",
      "properties": {
        "valid":    { "type": "boolean" },
        "warnings": { "type": "array", "items": { "type": "string" } },
        "issues":   { "type": "array", "items": { "type": "string" } },
        "fixes":    { "type": "array", "items": { "type": "string" } }
      },
      "required": ["valid"]
    },
    "deliver": {
      "type": "object",
      "properties": {
        "artifacts": { "type": "array" },
        "format":    { "type": "string" },
        "output":    {}
      }
    },
    "meta": {
      "type": "object",
      "properties": {
        "version":    { "type": "string" },
        "status":     { "type": "string", "enum": ["initialized", "intent_done", "ground_done", "compose_done", "verify_done", "delivered", "error"] },
        "run_id":     { "type": "string" },
        "created_at": { "type": "string" },
        "updated_at": { "type": "string" },
        "hub_runs": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "hub": { "type": "string" },
              "started_at": { "type": "string" },
              "completed_at": { "type": "string" },
              "input_snapshot": { "type": "object" },
              "output_snapshot": { "type": "object" },
              "warnings_count": { "type": "integer" }
            }
          }
        }
      }
    }
  }
}
```

### `.hive/orchestrator/main-orchestrator.md`

Generate content that defines:
- The orchestration logic (Intent → Ground → Compose → Verify → Deliver)
- State initialization procedure
- Transition conditions between hubs
- Remediation loop: if `verify.valid = false`, run Remediation Agent then re-verify
- Error handling rules
- How to read/write state

### Hub `hub.md` files

Each hub's `hub.md` should define:
- Hub name and purpose
- Input requirements (which state sections it reads)
- Output contract (which state section it writes to)
- List of agents and their responsibilities
- Available skills
- Success criteria for hub completion

### Agent `.md` files

Each agent file should define:
- Agent name and mission
- Inputs (from state or hub context)
- Allowed skills
- Output format (JSON schema or example)
- Rules specific to this agent
- Error conditions

### `.hive/skills/registry.json`

Generate a skills registry appropriate for the domain, with entries like:
```json
{
  "skills": [
    {
      "name": "search_patterns_by_use_case",
      "hub": "ground",
      "agent": "pattern-resolver",
      "description": "Find applicable patterns given a use case",
      "input_schema": { "use_case": "string", "domain": "string" },
      "output_schema": { "patterns": "array" },
      "limits": "Returns empty array if no patterns found",
      "error_cases": ["unknown_domain", "missing_use_case"]
    }
  ]
}
```

### Rules files

- `rules/naming.md` — naming conventions for hubs, agents, skills, state keys
- `rules/handoffs.md` — how hubs hand off to each other, what must be present in state before a hub starts
- `rules/verification.md` — verification criteria, what constitutes a passing Verify result
- `rules/governance.md` — governance rules, deprecated elements, compliance requirements

### `.hive/prompts/conventions.md`

Global prompt conventions for the project: tone, format, language, output structure.

---

## Post-Scaffold Instructions

After creating all files, display:

1. A summary of what was created
2. The recommended next steps:
   - Edit `hive.config.json` to define truth sources and output formats
   - Fill in hub agent files with domain-specific rules
   - Add project-specific skills to `skills/registry.json`
   - Run `/hive-run <task>` to test the pipeline end-to-end

3. A reminder of the canonical flow:
   ```
   Intent → Ground → Compose → Verify → Deliver
   ```

---

## Constraints

- Do not overwrite existing `.hive/` files without explicit user confirmation
- Always validate that you are at a project root (look for `package.json`, `pyproject.toml`, `Cargo.toml`, `.git/`, or similar markers)
- If no project markers are found, warn the user and ask them to confirm the target directory
- Use the domain argument to pre-fill relevant agent names, skill names, and example content
