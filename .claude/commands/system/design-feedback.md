---
name: system:design-feedback
description: Design self-correction mechanisms and detect system trap patterns
allowed-tools:
  - Read
  - Write
  - Bash
  - Agent
  - AskUserQuestion
---

<objective>

Design the self-correction mechanisms that keep the system on target. For everything that builds up, specify how drift gets detected and corrected. Check the design against 7 common trap patterns that cause systems to fail over time.

Optionally generates a system diagram showing what builds up, what moves, and what self-corrects.

Spawns the sys-feedback-designer agent.

**Reads:** `.system/{active}/MAP.md`, `.system/{active}/flows/`, `.system/{active}/OUTCOMES.md`
**Creates:** `.system/{active}/feedback/{mechanism-name}.md`, optionally `.system/{active}/system-diagram.excalidraw`
**Updates:** `.system/{active}/STATE.md`

**After this command:** Run `/system:input-audit` to classify every input boundary before closure verification.

</objective>

<execution_context>

@.claude/agents/sys-feedback-designer.md
@.claude/system-design/references/system-traps.md

</execution_context>

<process>

## Step 1: Resolve Active System and Validate

```bash
ACTIVE=$(cat .system/ACTIVE 2>/dev/null)
[ -z "$ACTIVE" ] && echo "ERROR: No active system. Run /system:new-system [name] first." && exit 1
[ ! -d ".system/$ACTIVE/flows" ] && echo "ERROR: No flows designed. Run /system:design-flows first." && exit 1
ls ".system/$ACTIVE/flows/"*.md 2>/dev/null | wc -l | grep -q "^0$" && echo "ERROR: No flow files found. Run /system:design-flows first." && exit 1
echo "Active system: $ACTIVE | Working directory: .system/$ACTIVE/"
```

From this point, all file paths use `.system/{active-system}/` as the root. Read:
- `.system/{active-system}/MAP.md`
- `.system/{active-system}/OUTCOMES.md`
- `.system/{active-system}/DESIGN.md`
- `.system/{active-system}/config.json`
- All files in `.system/{active-system}/flows/`

Extract: all accumulations from MAP.md, all flows, all outcomes, config preferences.

## Step 2: Display Stage Banner

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ↺  SYSTEM DESIGN >> DESIGNING FEEDBACK  ·  04/06
  What self-corrects? What amplifies? What traps exist?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Designing self-correction mechanisms for [system name]...
Spawning sys-feedback-designer agent.
```

## Step 3: Create Feedback Directory

```bash
mkdir -p ".system/$ACTIVE/feedback"
```

## Step 4: Spawn sys-feedback-designer Agent

Build the agent prompt with full context.

```
Agent(prompt="First, read .claude/agents/sys-feedback-designer.md for your role and instructions.
Then read .claude/system-design/references/system-traps.md for the 7 trap patterns to check.

<system_context>

**System name:** [name]

**Accumulations (from MAP.md):**
[Full list of accumulations with current levels, targets, and boundaries]

**Flows:**
[Summary of all flows from flows/ directory: name, from, to, trigger type]

**Outcomes:**
[Outcome list from OUTCOMES.md]

**Config:**
Mode: [mode]
Depth: [depth]

</system_context>

<instructions>
Follow your agent process:
1. Design self-correction for each accumulation
2. Identify amplifiers (self-reinforcing cycles)
3. Check all 7 trap patterns against the design
4. Write feedback files
5. Flag unregulated accumulations

If mode is 'yolo': present your proposed mechanisms, ask one round of confirmation, then write.
If mode is 'interactive': question the operator on each accumulation's detection, correction, and delay.

Depth controls detail level:
- quick: detection method and correction action only. Skip delay analysis unless obvious risk.
- standard: full detection, correction, and delay for each accumulation. Check all 7 traps.
- comprehensive: everything in standard plus amplifier chain analysis, second-order trap interactions, and delay sensitivity assessment.

Write one file per mechanism to .system/feedback/{mechanism-name}.md.
Return your structured result when complete. Include trap warnings prominently.
</instructions>
", description="Design self-correction mechanisms")
```

## Step 5: Handle Agent Return

Read the agent's structured return. Extract:
- Mechanism count
- Amplifier count
- Trap warning count
- Unregulated accumulation count
- Mechanism summary table
- Trap warnings list

Verify feedback files were written:
```bash
ls ".system/$ACTIVE/feedback/"*.md 2>/dev/null | wc -l
```

## Step 6: Diagram Generation (Conditional)

Read `.system/{active-system}/config.json`. If `diagram` is `true`:

First verify the Excalidraw plugin is still installed:
```bash
[ -d ".obsidian/plugins/obsidian-excalidraw-plugin" ] && echo "excalidraw:installed" || echo "excalidraw:missing"
```

If `excalidraw:missing`: skip diagram generation, warn the operator ("Excalidraw plugin not found — skipping diagram. Install from Community Plugins to enable."), and continue to Step 7.

If `excalidraw:installed`:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ↺  SYSTEM DESIGN >> GENERATING DIAGRAM  ·  04/06
  Mapping what builds up, what moves, and what corrects.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Generate a lightweight Excalidraw diagram. Do not load external skill references or run the render pipeline — Obsidian renders the file natively. Just generate valid JSON and save it.

### Element mapping

| System element | Shape | Fill | Notes |
|----------------|-------|------|-------|
| Accumulation | rectangle | `#dbeafe` (light blue) | Label: name + target level |
| Inflow / outflow | arrow | none | Label: trigger type |
| Feedback loop | arrow, curved | none | Green stroke `#22c55e`, returns to source |
| System boundary | rectangle, dashed stroke | none | Gray `#9ca3af` |
| Trap warning | text | none | Red `#ef4444`, near affected element |

### JSON structure

Generate the diagram as a single valid Excalidraw JSON file:

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://excalidraw.com",
  "elements": [ ... ],
  "appState": { "viewBackgroundColor": "#ffffff" },
  "files": {}
}
```

Build elements in two passes — shapes first, then arrows. This is mandatory: arrows reference shape IDs, so shapes must exist first.

**Pass 1 — shapes:** accumulations (rectangles), system boundary (rectangle, dashed), trap warnings (text). Assign string IDs like `"acc-1"`, `"acc-2"`, `"boundary"`. Place shapes at explicit `x`/`y` coordinates. Layout left-to-right, 200px spacing between accumulations.

Rectangle template — include a `boundElements` array listing every arrow that connects to this shape:
```json
{
  "type": "rectangle",
  "id": "acc-1",
  "x": 100, "y": 200, "width": 160, "height": 60,
  "backgroundColor": "#dbeafe",
  "strokeColor": "#3b82f6",
  "fillStyle": "solid",
  "boundElements": [
    { "id": "flow-1", "type": "arrow" }
  ],
  "label": { "text": "Accumulation Name" }
}
```

**Pass 2 — arrows:** bind every arrow to its source and target using the IDs from Pass 1. Use `startBinding` and `endBinding` with `"mode": "orbit"` and `"fixedPoint": [x, y]` — this is the syntax Excalidraw requires for arrows to actually connect. Never use `focus` or `gap` fields.

`fixedPoint` controls where on the shape the arrow attaches. Use `[0.5, 1]` for bottom-center (arrow leaves downward), `[0.5, 0]` for top-center (arrow arrives from above), `[1, 0.5]` for right-center, `[0, 0.5]` for left-center. For left-to-right flows: source gets `fixedPoint: [1, 0.5]`, target gets `fixedPoint: [0, 0.5]`.

Arrow template:
```json
{
  "type": "arrow",
  "id": "flow-1",
  "x": 260, "y": 230,
  "width": 140, "height": 0,
  "points": [[0, 0], [140, 0]],
  "startBinding": {
    "mode": "orbit",
    "elementId": "acc-1",
    "fixedPoint": [1, 0.5]
  },
  "endBinding": {
    "mode": "orbit",
    "elementId": "acc-2",
    "fixedPoint": [0, 0.5]
  },
  "strokeColor": "#1e293b",
  "label": { "text": "trigger type" }
}
```

Feedback loop arrows: same binding syntax, `strokeColor: "#22c55e"`. For curved returns that loop back to the source, use `startBinding: null, endBinding: null` (unbound freehand arc) and route `points` manually below or above the elements — e.g. `[[0,0],[80,0],[80,80],[-200,80],[-200,0]]` to arc underneath.

`points` rule: first point is always `[0, 0]`. Subsequent points are relative offsets from the arrow's `x`/`y`. For a straight right-going arrow of width W: `[[0,0],[W,0]]`. For a curved feedback return, use 3–4 points to arc below or above the elements.

Use real system names from MAP.md, not placeholders.

### Save

```bash
mkdir -p Excalidraw
```

Write the complete JSON to `Excalidraw/{active-system-name}-system-diagram.excalidraw`. One write, no render loop.

After saving, construct the absolute file URL so the operator can open it directly from chat:
```bash
echo "file://$(pwd)/Excalidraw/{active-system-name}-system-diagram.excalidraw"
```

Output this in your response as a clickable markdown link:
```
[Open System Diagram](file:///absolute/path/to/{active-system-name}-system-diagram.excalidraw)
```

Opening the link launches the file in Obsidian (if Excalidraw plugin is installed) or in the system default handler.

## Step 7: Update STATE.md

Read `.system/{active-system}/STATE.md` and update:
- Stage: verification
- Last completed: design-feedback
- Next step: input-audit
- Mark `design-feedback` checkbox complete
- Add any decisions made during feedback design

## Step 8: Present Results

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ↺  SYSTEM DESIGN >> FEEDBACK DESIGNED ✓  ·  04/06
  Every accumulation watched. Every loop closed.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Mechanisms:** [count]
**Amplifiers identified:** [count]
**Trap warnings:** [count]
**Unregulated accumulations:** [count]

[Mechanism summary table from agent return]
```

If trap warnings exist, present them prominently:

```
## Trap Warnings

[For each trap warning:]
- **[Trap name]:** [Where detected in this system]. Fix: [suggested structural fix].
```

If unregulated accumulations exist:

```
## Unregulated (must address before verification)

[For each unregulated accumulation:]
- **[Name]:** No self-correction mechanism. It will drift unchecked.
```

If diagram was generated:

```
**Diagram:** .system/system-diagram.excalidraw
```

```
| Artifact         | Location                                              |
|------------------|-------------------------------------------------------|
| Feedback specs   | .system/{active-system}/feedback/                     |
| Diagram          | Excalidraw/{active-system}-system-diagram.excalidraw  |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Next: /system:input-audit -- classify every input boundary (gate / sanitise / accept-with-tolerance)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

</process>

<output>

- `.system/{active-system}/feedback/{mechanism-name}.md` (one per mechanism, created)
- `Excalidraw/{active-system}-system-diagram.excalidraw` (if diagram enabled and plugin installed)
- `.system/{active-system}/STATE.md` (updated)

</output>

<success_criteria>

- [ ] Abort triggered if `.system/flows/` missing or empty
- [ ] sys-feedback-designer agent spawned with full system context
- [ ] Every accumulation from MAP.md has a self-correction mechanism or is explicitly flagged
- [ ] Each mechanism has: detection, correction, delay
- [ ] Amplifiers identified and assessed (beneficial vs dangerous)
- [ ] All 7 trap patterns checked against the design
- [ ] Trap warnings presented with plain-language explanation and fix
- [ ] One file per mechanism in `.system/feedback/`
- [ ] Unregulated accumulations flagged prominently
- [ ] Diagram generated if config.diagram is true
- [ ] All four skill reference files read before generating JSON
- [ ] JSON built section by section with namespaced ID seeds
- [ ] Render pipeline run and diagram validated against 27-item checklist
- [ ] STATE.md updated with correct stage and next step
- [ ] Operator knows next step is `/system:input-audit`

</success_criteria>
