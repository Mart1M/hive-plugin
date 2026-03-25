# /hive-compose

Run the **Compose Hub** in isolation. Builds a structured solution from the intent and grounded knowledge.

## Usage

```
/hive-compose
/hive-compose --load-state <path to state file>
/hive-compose --output blueprint
/hive-compose --output code-plan
```

## Arguments

- `--load-state` (optional): Load an existing state file; Compose reads from `state.intent` and `state.ground`
- `--output` (optional): Hint the output format. Options: `blueprint` (default), `code-plan`, `spec`, `component-tree`

---

## Hub Purpose

The Compose Hub **builds the solution**. It takes the structured intent and the grounded knowledge package and produces a concrete, structured composition: a blueprint, a component tree, a layout structure, and an interaction map.

Compose Hub does not invent — it assembles. Every element in the composed output must trace back to something identified during grounding.

---

## Entry Conditions

Before running, verify:
- `state.intent.goal` is non-empty
- `state.ground` is non-empty (contains at least `components` and `rules`)
- `state.meta.status` is `"ground_done"` or later

If these conditions are not met, warn the user and suggest running `/hive-ground` first.

---

## Agents to Invoke

### 1. Layout Composer Agent

**Mission**: Build the overall page or screen structure — major sections, layout grid, hierarchy.

**Skills**: `generate_layout_structure`

**Rules**:
- Use layout patterns from `state.ground.patterns`
- Apply layout rules from `state.ground.rules.layout`
- Produce a named, structured layout with clearly labeled sections
- Use the grid system defined in tokens (e.g., 8px grid, 12-column grid)

**Output**:
```json
{
  "layout": {
    "type": "two-column",
    "sections": [
      { "id": "cart-summary", "position": "left", "width": "40%" },
      { "id": "payment-form", "position": "right", "width": "60%" }
    ],
    "grid": "12-column",
    "max_width": "1200px"
  }
}
```

### 2. Component Composition Agent

**Mission**: Select and arrange the specific components for each section of the layout.

**Skills**: `compose_component_tree`

**Rules**:
- Use only components from `state.ground.components`
- Respect component usage constraints (e.g., "Primary Button for main CTA only")
- For each component instance, specify: name, variant, purpose, section it belongs to, any required props
- Do not invent new components — if a needed component wasn't grounded, flag it as a gap

**Output**:
```json
{
  "components": [
    {
      "id": "cta-button",
      "component": "Button",
      "variant": "primary",
      "section": "payment-form",
      "purpose": "Submit payment",
      "props": { "label": "Pay now", "type": "submit" }
    }
  ]
}
```

### 3. Interaction Flow Agent

**Mission**: Define the interactions and behaviors — what happens when the user acts.

**Skills**: `map_interactions`

**Rules**:
- Define interactions in terms of trigger → action → outcome
- Include form validation behaviors, error states, loading states
- Reference accessibility rules from `state.ground.rules.accessibility` for interaction patterns

**Output**:
```json
{
  "interactions": [
    {
      "trigger": "User submits payment form",
      "validation": "Validate all required fields before submission",
      "success": "Show order confirmation",
      "error": "Show inline error messages on invalid fields, announce via aria-live"
    }
  ]
}
```

### 4. State Mapper Agent

**Mission**: Map the data model — what data is needed, where it comes from, and how it flows.

**Rules**:
- Identify all data entities needed by the composition (cart items, user info, payment data)
- Map each entity to a component
- Note which data is user-provided vs. pre-loaded

**Output**:
```json
{
  "state_map": {
    "cart_items": { "source": "cart store", "used_by": ["cart-summary"] },
    "payment_data": { "source": "user input", "used_by": ["payment-form"] },
    "order_total": { "source": "computed from cart_items", "used_by": ["cart-summary", "cta-button"] }
  }
}
```

### 5. Output Blueprint Agent

**Mission**: Assemble the complete, structured blueprint from all agent outputs. This is the single source of truth for Verify and Deliver.

**Skills**: `create_blueprint_json`

**Rules**:
- The blueprint must be self-contained — it must make sense without reading the agent outputs separately
- Include all elements: layout, component tree, interactions, data map
- Add a `metadata` section describing what was used from grounding
- The blueprint must be machine-readable and human-readable

---

## Final Compose Output

Write the complete blueprint to `state.compose`:

```json
{
  "blueprint": {
    "id": "<uuid>",
    "title": "...",
    "goal": "<from state.intent.goal>",
    "layout": {},
    "component_tree": [],
    "interactions": [],
    "state_map": {},
    "metadata": {
      "patterns_used": [],
      "components_used": [],
      "tokens_applied": [],
      "grounding_gaps": []
    }
  },
  "layout": {},
  "components": [],
  "interactions": [],
  "state_map": {}
}
```

Include a `grounding_gaps` list for any component or pattern the composition needed but wasn't found in grounding. These will surface as issues in the Verify Hub.

Update `state.meta.status` to `"compose_done"`.

---

## Hub Rules

1. **Do not invent elements.** Every component, pattern, and token in the composition must come from `state.ground`.
2. **Respect layout rules.** Use the grid, spacing, and alignment rules from `state.ground.rules.layout`.
3. **Log grounding gaps.** If a needed component was not grounded, do not silently invent it — log the gap and use the closest available alternative.
4. **The blueprint is the contract.** What is in the blueprint is what Verify will check and Deliver will build from.
5. **Do not validate here.** Do not perform DS compliance or accessibility checking — that is Verify Hub's responsibility.

---

## Anti-Patterns

- Do not use components not in `state.ground.components` (this is the primary source of hallucination in composition)
- Do not produce a narrative description instead of a structured blueprint
- Do not skip the State Mapper — an untested data model causes delivery failures
- Do not combine multiple unrelated solutions in one composition — one composition per goal
