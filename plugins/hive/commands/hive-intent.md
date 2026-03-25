# /hive-intent

Run the **Intent Hub** in isolation. Transforms a raw request into a fully structured intention object.

## Usage

```
/hive-intent <raw request or task description>
/hive-intent --load-state <path to state file>
```

## Arguments

- `<raw request>` (required): The raw task description to be interpreted
- `--load-state` (optional): Load an existing HIVE state file and run Intent on its `request.raw_input`

---

## Hub Purpose

The Intent Hub transforms ambiguous, raw user requests into a **structured intention** that downstream hubs can act upon precisely. It answers five questions:

1. What is the **real objective**?
2. What is the **business/project context**?
3. What is the **scope** of the task?
4. What **constraints** are already explicit?
5. What **ambiguities** must be resolved by inference or controlled hypothesis?

---

## Agents to Invoke

Work through each agent in sequence. Each enriches the intent object.

### 1. Goal Interpreter Agent

**Mission**: Extract the core, actionable goal from the raw input. Strip noise and ambiguity. Express the goal as a single, clear sentence.

**Rules**:
- Be specific, not generic ("Create a checkout page with cart summary and payment form" not "Build something for checkout")
- If the goal is ambiguous, choose the most reasonable interpretation and note the assumption
- Do not expand the scope beyond what is stated

**Output**:
```json
{ "goal": "..." }
```

### 2. Scope Clarifier Agent

**Mission**: Define the boundaries of the task. What is in scope? What is explicitly or implicitly out of scope?

**Rules**:
- Scope must be derived from the request — do not invent requirements
- Note any scope assumptions made
- Consider: which features, screens, user flows, platforms are included?

**Output**:
```json
{ "scope": "...", "in_scope": [], "out_of_scope": [], "assumptions": [] }
```

### 3. Constraint Extractor Agent

**Mission**: Identify all constraints — technical, business, design, accessibility, platform.

**Types of constraints to look for**:
- Platform constraints: web, mobile, desktop, responsive
- Technical constraints: specific framework, existing component library
- Business constraints: compliance, brand, audience
- Accessibility constraints: WCAG level, screen reader support
- Design system constraints: DS compliance, token usage only

**Output**:
```json
{ "constraints": ["responsive", "WCAG AA", "DS compliant"] }
```

### 4. Success Criteria Agent

**Mission**: Define what "done" looks like. What must be true for this task to be considered successfully resolved?

**Rules**:
- Success criteria must be verifiable — the Verify Hub will use these
- Be concrete: "Button uses primary color token" not "Button looks good"
- Include at least one functional criterion and one quality/compliance criterion

**Output**:
```json
{
  "success_criteria": [
    "All form fields have proper labels",
    "Layout uses only DS-approved components",
    "Payment form validates input before submission"
  ]
}
```

---

## Assembled Intent Output

After all agents have run, assemble the full intent object and write it to `state.intent`:

```json
{
  "goal": "...",
  "domain": "...",
  "scope": "...",
  "in_scope": [],
  "out_of_scope": [],
  "constraints": [],
  "success_criteria": [],
  "assumptions": []
}
```

Update `state.meta.status` to `"intent_done"`.

---

## Hub Rules

1. **Do not perform grounding here.** Do not look up components, patterns, or tokens. That is the Ground Hub's job.
2. **Do not make composition decisions.** Do not suggest layouts or component choices.
3. **Resolve ambiguity through controlled hypothesis**, not by asking the user for every detail. State your assumptions explicitly in `state.intent.assumptions`.
4. **If the request is fundamentally unresolvable** (truly contradictory or missing critical information), ask the user one single clarifying question and halt.

---

## Output Example

Input: `"Create a checkout page with cart summary and payment form"`

```json
{
  "goal": "Create a checkout page containing a cart summary section and a payment form",
  "domain": "design-system",
  "scope": "Single checkout page — cart summary, payment details, submit CTA",
  "in_scope": ["cart summary", "payment form", "submit button"],
  "out_of_scope": ["order confirmation page", "authentication flow"],
  "constraints": ["responsive", "DS compliant", "accessible"],
  "success_criteria": [
    "Cart summary displays item list, quantities, and total price",
    "Payment form includes card number, expiry, CVV fields with proper labels",
    "All interactive elements are keyboard-accessible",
    "Layout uses only DS-approved components and tokens"
  ],
  "assumptions": [
    "Target platform is web (responsive)",
    "User is already authenticated at this step",
    "Design system compliance is required"
  ]
}
```

---

## Anti-Patterns

- Do not return a vague goal like "create a page" — be specific
- Do not skip the Constraint Extractor even if no constraints seem obvious — default constraints (responsive, accessible) should always be present
- Do not invent requirements not mentioned or inferable from the request
- Do not produce prose — the output must be a structured JSON object
