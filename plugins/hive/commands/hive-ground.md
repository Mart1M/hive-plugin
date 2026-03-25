# /hive-ground

Run the **Ground Hub** in isolation. Gathers all knowledge necessary for composition — connecting the system to the project's reality before any solution is built.

## Usage

```
/hive-ground
/hive-ground --from-intent <intent JSON or file>
/hive-ground --load-state <path to state file>
```

## Arguments

- `--from-intent` (optional): Provide intent JSON directly if state hasn't been initialized
- `--load-state` (optional): Load an existing state file; Ground will read from `state.intent`

If run in isolation without a prior Intent Hub result, Ground will attempt to infer a minimal intent from context. A warning will be logged.

---

## Hub Purpose

The Ground Hub is the **knowledge foundation** of HIVE. Before any composition happens, the system must know what it is working with. Ground Hub eliminates hallucination by forcing the system to anchor itself in verified, project-specific knowledge.

It answers: **What do we actually know, and what do we have available, to solve this?**

Sources include: project components, design patterns, design tokens, rules, documentation, existing examples, platform constraints, governance requirements.

---

## Agents to Invoke

Run all relevant agents. Skip agents only if explicitly configured as disabled in `hive.config.json`.

### 1. Pattern Resolver Agent

**Mission**: Identify applicable UX/design/architecture patterns for this goal.

**Skills**: `search_patterns_by_use_case`

**Rules**:
- Only return patterns that are relevant to the stated goal
- Prefer patterns documented in `.hive/skills/patterns/` if present
- Include pattern name, description, and applicability rationale

**Output**:
```json
{
  "patterns": [
    {
      "name": "two-column-checkout",
      "description": "Cart summary on left, payment form on right",
      "applicable_because": "Goal involves checkout with two distinct content areas"
    }
  ]
}
```

### 2. Component Resolver Agent

**Mission**: Identify all components available and relevant to the goal.

**Skills**: `get_component_docs`

**Rules**:
- Only include components from the approved design system or component library
- Flag any deprecated components — do not use them
- For each component, note: name, purpose, variants, usage constraints

**Output**:
```json
{
  "components": [
    {
      "name": "Button",
      "variants": ["primary", "secondary", "ghost"],
      "constraints": ["Must have accessible label", "Primary variant for main CTA only"]
    }
  ]
}
```

### 3. Token Resolver Agent

**Mission**: Retrieve design tokens applicable to the goal's context.

**Skills**: `get_tokens_for_context`

**Rules**:
- Include spacing, color, typography, and elevation tokens relevant to the context
- Note the platform and mode (light/dark) the tokens apply to
- Do not invent tokens — only return tokens that exist in the project's token system

**Output**:
```json
{
  "tokens": {
    "spacing": { "small": "4px", "medium": "8px", "large": "16px" },
    "color": { "primary": "#0066CC", "surface": "#FFFFFF" },
    "typography": { "body": "16px/1.5 Inter" }
  }
}
```

### 4. Rule Resolver Agent

**Mission**: Retrieve all relevant rules: layout, composition, accessibility, platform, governance.

**Skills**: `get_accessibility_rules`, `get_framework_constraints`

**Rules to look for**:
- Layout rules: grid, spacing, alignment
- Accessibility rules: WCAG requirements, ARIA patterns, focus management
- Platform rules: browser/device constraints
- Governance rules: what is allowed/forbidden in this domain

**Output**:
```json
{
  "rules": {
    "layout": ["Use 8px grid", "Max content width 1200px"],
    "accessibility": ["All form fields must have associated labels", "Error messages must be announced via aria-live"],
    "platform": ["Support iOS 16+, Android 12+, Chrome/Firefox/Safari latest"],
    "governance": ["No inline styles", "No deprecated components"]
  }
}
```

### 5. Documentation Resolver Agent

**Mission**: Retrieve relevant documentation sections from project docs, design system docs, or domain references.

**Skills**: `search_docs`

**Rules**:
- Only include documentation directly relevant to the goal
- Include section title, source, and a short summary
- Prefer project-specific docs over generic ones

**Output**:
```json
{
  "docs": [
    {
      "title": "Checkout Pattern",
      "source": ".hive/skills/patterns/checkout.md",
      "summary": "Standard two-step checkout: cart review then payment"
    }
  ]
}
```

### 6. Example Resolver Agent

**Mission**: Find existing examples in the codebase or documentation that are relevant to the goal.

**Skills**: `get_existing_examples`

**Rules**:
- Examples must be relevant — do not include unrelated examples for completeness
- Include file path or reference so Compose can use them as models

**Output**:
```json
{
  "examples": [
    {
      "name": "Existing PaymentForm component",
      "path": "src/components/PaymentForm/PaymentForm.tsx",
      "relevance": "Existing implementation of payment form to reuse or reference"
    }
  ]
}
```

### 7. Platform Context Agent

**Mission**: Inject platform, framework, and stack constraints.

**Skills**: `get_framework_constraints`

**Output**:
```json
{
  "platform": {
    "framework": "React",
    "version": "18",
    "css": "Tailwind CSS",
    "target": "web",
    "browser_support": ["Chrome 110+", "Firefox 110+", "Safari 16+"]
  }
}
```

### 8. Governance Precheck Agent

**Mission**: Filter out any deprecated, non-compliant, or out-of-scope elements discovered by previous agents.

**Rules**:
- Remove deprecated components from the `components` list; log them as warnings
- Flag any rules that conflict with each other
- Do not silently drop items — always log what was filtered and why

**Output**:
```json
{
  "governance_warnings": [
    "Component 'LegacyCheckbox' is deprecated — replaced with 'Checkbox'. Removed from component list."
  ]
}
```

### 9. Context Aggregator Agent

**Mission**: Assemble all agent outputs into a single, clean knowledge package for Compose Hub.

**Rules**:
- Deduplicate entries
- Resolve conflicts (flag unresolvable ones as warnings)
- Ensure the final package is self-contained and ready for Compose

---

## Final Ground Output

Write the assembled knowledge package to `state.ground`:

```json
{
  "patterns": [],
  "components": [],
  "tokens": {},
  "rules": {},
  "docs": [],
  "examples": [],
  "platform": {},
  "governance_warnings": []
}
```

Update `state.meta.status` to `"ground_done"`.

---

## Hub Rules

1. **Ground Hub must run before Compose.** Compose must not invent components or patterns that weren't identified in Ground.
2. **Never hallucinate knowledge.** If a component or pattern is not in the project, do not include it. Mark the gap as a warning.
3. **Be exhaustive but relevant.** Include everything useful for the goal, but exclude unrelated knowledge that would bloat the state.
4. **Deprecated items must be flagged, not silently included.** The Governance Precheck Agent exists specifically for this.

---

## Anti-Patterns

- Do not include components or tokens not present in the actual project (hallucination)
- Do not skip the Governance Precheck — it prevents downstream compliance failures
- Do not produce a single large text blob — every item must be in its typed, structured section
- Do not perform composition reasoning here — that is Compose Hub's job
