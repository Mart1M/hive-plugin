# /hive-status

Display the current HIVE state for the active session — showing what has been processed, what is pending, and any issues found.

## Usage

```
/hive-status
/hive-status --load-state <path to state file>
/hive-status --verbose
/hive-status --section intent
```

## Arguments

- `--load-state` (optional): Load state from a specific file path instead of the active session state
- `--verbose` (optional): Show full content of each state section, not just a summary
- `--section` (optional): Show only a specific section. Options: `request`, `intent`, `ground`, `compose`, `verify`, `deliver`, `meta`

---

## What This Command Does

`/hive-status` gives a clear, at-a-glance view of where a HIVE run is in its lifecycle. It reads the current `state` object (from the active session or a file) and renders a formatted summary.

---

## Status Display Format

Render the status in the following format:

### Header

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  HIVE STATUS
  Run ID: <state.meta.run_id>
  Status: <state.meta.status>
  Version: <state.meta.version>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Request

```
REQUEST
  Input: "<state.request.raw_input>"
  Source: <state.request.source>
  Timestamp: <state.request.timestamp>
```

### Hub Pipeline Status

Show each hub with a status indicator:

```
PIPELINE
  ✅ Intent Hub     — Done
  ✅ Ground Hub     — Done
  ✅ Compose Hub    — Done
  ⚠️  Verify Hub    — Done (2 warnings)
  ⏳ Deliver Hub    — Pending
```

**Status indicator rules**:
- `✅` — Hub completed, no issues
- `⚠️` — Hub completed with warnings
- ❌ — Hub completed with issues/failures
- `⏳` — Hub not yet run
- `🔄` — Hub currently running

Derive hub status from `state.meta.status` and the content of each hub's state section.

### Intent Summary (if `state.intent` is populated)

```
INTENT
  Goal: <state.intent.goal>
  Domain: <state.intent.domain>
  Scope: <state.intent.scope>
  Constraints: <comma-separated list>
  Success Criteria: <count> defined
```

### Ground Summary (if `state.ground` is populated)

```
GROUND
  Patterns: <count> resolved
  Components: <count> resolved
  Tokens: <count> token groups
  Rules: <count> rule sets
  Docs: <count> documents
  Examples: <count> examples
  Governance Warnings: <count>
```

### Compose Summary (if `state.compose` is populated)

```
COMPOSE
  Blueprint: <state.compose.blueprint.title>
  Layout: <state.compose.layout.type>
  Components: <count> components placed
  Interactions: <count> defined
  Grounding Gaps: <count> (if any)
```

### Verify Summary (if `state.verify` is populated)

```
VERIFY
  Result: PASSED ✅ | FAILED ❌
  Checks Run: <list>
  Issues: <count>
  Warnings: <count>
  Fixes Available: <count>
```

If `state.verify.valid = false`, list all issues:
```
  Issues:
    • <issue 1>
    • <issue 2>
```

### Deliver Summary (if `state.deliver` is populated)

```
DELIVER
  Status: Delivered
  Artifacts: <count>
  <artifact type>: <artifact name>
  Delivered At: <timestamp>
```

### Footer

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Next step: <suggested next command>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Next step suggestion logic**:
- If `status = "initialized"` → suggest `/hive-intent` or `/hive-run <task>`
- If `status = "intent_done"` → suggest `/hive-ground`
- If `status = "ground_done"` → suggest `/hive-compose`
- If `status = "compose_done"` → suggest `/hive-verify`
- If `status = "verify_done"` and `verify.valid = true` → suggest `/hive-deliver`
- If `status = "verify_done"` and `verify.valid = false` → suggest reviewing issues and re-running `/hive-compose` with fixes applied, then `/hive-verify`
- If `status = "delivered"` → suggest reviewing artifacts or running a new pipeline

---

## Verbose Mode (`--verbose`)

When `--verbose` is active, also show the full JSON content of each populated state section after the summary block.

---

## No State Found

If no active session state is found and no `--load-state` path is provided:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  HIVE STATUS — No active session
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
No HIVE state found in the current session.

To start:
  /hive-init         — Set up HIVE in your project
  /hive-run <task>   — Run the full pipeline
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Notes

- `/hive-status` is read-only — it never modifies state
- It is safe to run at any point in a pipeline without affecting the run
- Use it after each hub to confirm the expected outputs are present before proceeding
