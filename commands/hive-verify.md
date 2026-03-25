# /hive-verify

Run the **Verify Hub** in isolation. Validates the composed solution before delivery — checking compliance, accessibility, technical feasibility, and consistency.

## Usage

```
/hive-verify
/hive-verify --load-state <path to state file>
/hive-verify --strict
/hive-verify --check compliance,accessibility
```

## Arguments

- `--load-state` (optional): Load an existing state file; Verify reads from `state.compose` and `state.ground`
- `--strict` (optional): Treat warnings as errors (no delivery if warnings exist)
- `--check` (optional): Run only specific checks, comma-separated. Options: `compliance`, `accessibility`, `feasibility`, `consistency`

---

## Hub Purpose

The Verify Hub is the **quality gate**. No output reaches the user without passing through Verify. This is a non-negotiable step in HIVE — the entire framework's reliability depends on it.

Verify answers:
- Does the output respect domain rules and business conventions?
- Is it design system compliant?
- Is it accessible?
- Is it technically feasible?
- Is it internally consistent?

---

## Entry Conditions

Before running, verify:
- `state.compose.blueprint` is non-empty
- `state.ground.rules` is non-empty (to have rules to check against)
- `state.meta.status` is `"compose_done"` or later

If these conditions are not met, warn the user and suggest running `/hive-compose` first.

---

## Agents to Invoke

### 1. Compliance Agent

**Mission**: Check that the composition complies with all domain rules and design system constraints.

**Skills**: `validate_ds_compliance`

**What to check**:
- Every component used is in the approved component list from `state.ground.components`
- No deprecated components are used
- Tokens used match those in `state.ground.tokens`
- Layout follows the rules in `state.ground.rules.layout`
- No inline styles or hardcoded values that bypass the token system
- Governance rules from `state.ground.rules.governance` are respected

**Output**:
```json
{
  "compliance_check": {
    "passed": true|false,
    "issues": [
      "Component 'CustomButton' is not in the approved component list",
      "Hardcoded color '#FF0000' used — must use token 'color.error.primary'"
    ],
    "warnings": [
      "Component 'DataTable' has an updated variant available: 'DataTable v2'"
    ]
  }
}
```

### 2. Accessibility Agent

**Mission**: Verify that the composition meets minimum accessibility requirements.

**Skills**: `validate_accessibility`

**What to check** (minimum WCAG AA):
- All form inputs have associated `<label>` elements or `aria-label`
- All interactive elements are keyboard-focusable
- Focus order is logical
- Error messages are announced via `aria-live` regions
- Images have alt text
- Color contrast meets 4.5:1 for normal text, 3:1 for large text
- No content relies solely on color to convey meaning

**Output**:
```json
{
  "accessibility_check": {
    "passed": true|false,
    "level": "WCAG AA",
    "issues": [
      "PaymentForm: card-number input has no associated label",
      "CTA Button: missing accessible name"
    ],
    "warnings": [
      "Consider adding aria-describedby to CVV field for additional context"
    ]
  }
}
```

### 3. Technical Feasibility Agent

**Mission**: Verify that the composition can be implemented with the available stack and platform constraints.

**Skills**: `validate_platform_constraints`

**What to check**:
- All components are available in the target framework (from `state.ground.platform`)
- No composition decision relies on browser features not supported in the target browser range
- State management approach (from `state.compose.state_map`) is feasible
- No circular dependencies or impossible interactions

**Output**:
```json
{
  "feasibility_check": {
    "passed": true|false,
    "issues": [
      "CSS Grid subgrid is not supported in Safari 15 (target includes Safari 16+, so this is acceptable)"
    ],
    "warnings": [
      "Modal implementation requires focus-trap library — confirm it is available in project dependencies"
    ]
  }
}
```

### 4. Consistency Agent

**Mission**: Verify that the composition is internally consistent — no contradictions between components, interactions, or data model.

**Skills**: `detect_inconsistencies`

**What to check**:
- Component variants are consistently used (e.g., not mixing primary and secondary CTAs with same purpose)
- Data referenced in `state.compose.state_map` is actually used by the components
- Interaction flows don't contradict each other (e.g., two conflicting success handlers)
- Layout and component placement are coherent

**Output**:
```json
{
  "consistency_check": {
    "passed": true|false,
    "issues": [
      "Two primary Button components in the same section — only one primary CTA allowed per section"
    ],
    "warnings": []
  }
}
```

### 5. Remediation Agent

**Mission**: For each issue identified, propose a concrete, actionable fix.

**Skills**: `suggest_fixes`

**Rules**:
- Every `issue` must have a corresponding `fix`
- Fixes must be specific — not "fix the accessibility issue" but "Add `<label for='card-number'>Card number</label>` to the card number input"
- Warnings may have suggested improvements but do not require fixes
- If a fix cannot be determined, flag the issue as `requires_manual_review`

**Output**:
```json
{
  "fixes": [
    {
      "issue": "card-number input has no associated label",
      "fix": "Add <label for='card-number'>Card number</label> before the input element",
      "applies_to": "state.compose.components[id='card-number-input']"
    }
  ]
}
```

---

## Final Verify Output

Assemble all check results into the final verification report. Write to `state.verify`:

```json
{
  "valid": true|false,
  "checks_run": ["compliance", "accessibility", "feasibility", "consistency"],
  "warnings": [
    "Consider adding aria-describedby to CVV field for additional context",
    "Modal implementation requires focus-trap library"
  ],
  "issues": [
    "PaymentForm: card-number input has no associated label",
    "Two primary Button components in payment-form section"
  ],
  "fixes": [
    {
      "issue": "PaymentForm: card-number input has no associated label",
      "fix": "Add <label for='card-number'>Card number</label>",
      "applies_to": "state.compose.components[id='card-number-input']"
    }
  ],
  "remediation_available": true|false
}
```

**`valid` is `true` only when ALL of the following are satisfied:**
- `compliance_check.passed = true`
- `accessibility_check.passed = true`
- `feasibility_check.passed = true`
- `consistency_check.passed = true`

**`valid` is `false` if any check fails.** Warnings do not make `valid` false (unless `--strict` mode is active).

Update `state.meta.status` to `"verify_done"`.

---

## Remediation Loop

If `state.verify.valid = false` and `state.verify.fixes` is non-empty:

1. The Main Orchestrator (or the user, if running in isolation) applies fixes to `state.compose`
2. `/hive-verify` is re-run
3. If still `valid = false` after one remediation cycle, surface all remaining issues and halt

When running `/hive-verify` in isolation and issues are found, present the fixes clearly and ask if the user wants to apply them to the composition before re-running verification.

---

## Hub Rules

1. **Verify must always run before Deliver.** This is the core governance rule of HIVE.
2. **Never mark valid = true if any check fails.** Partial validity is not valid.
3. **Every issue must have a fix.** If a fix cannot be determined, mark it `requires_manual_review`.
4. **Warnings are informational, not blocking** — unless `--strict` mode is active.
5. **Run all checks by default.** Skipping a check requires an explicit `--check` argument.

---

## Anti-Patterns

- Do not return `valid: true` when issues exist, even minor ones
- Do not skip the Accessibility Agent — accessibility violations are delivery blockers
- Do not produce only a pass/fail result without listing issues and fixes
- Do not perform composition changes here — Verify Hub only diagnoses, it does not redesign
