---
name: system:input-audit
description: Audit every input boundary in the system. Classify each as gate, sanitise, or accept-with-tolerance per garbage-in-garbage-out.
allowed-tools:
  - Read
  - Write
  - Bash
  - Agent
  - AskUserQuestion
---

<objective>

Enforce the garbage-in-garbage-out principle on the active system design. Identify every external input boundary the system depends on, classify each one, and define what happens when dirty input arrives. Block downstream verification until every boundary is classified.

Spawns the sys-input-auditor agent.

**Reads:** `.system/{active}/MAP.md`, `.system/{active}/flows/`, `.system/{active}/DESIGN.md`, `.system/{active}/STATE.md`
**Creates:** `.system/{active}/INPUT-AUDIT.md`
**Updates:** `.system/{active}/STATE.md`

**Runs after:** `/system:design-feedback`
**Runs before:** `/system:verify-closure`

</objective>

<execution_context>

@.claude/agents/sys-input-auditor.md

This stage exists because output quality is capped by input quality. A system that closes all internal loops but accepts dirty inputs at the boundary produces dirty outputs faster. Verify-closure cannot detect this -- it tests loop structure, not input integrity. This stage is the input integrity check.

</execution_context>

<process>

## Step 1: Resolve Active System and Validate

```bash
ACTIVE=$(cat .system/ACTIVE 2>/dev/null)
[ -z "$ACTIVE" ] && echo "ERROR: No active system. Run /system:new-system [name] first." && exit 1
[ ! -d ".system/$ACTIVE/flows" ] && echo "ERROR: No flows designed. Run /system:design-flows first." && exit 1
[ ! -d ".system/$ACTIVE/feedback" ] && echo "ERROR: No feedback designed. Run /system:design-feedback first." && exit 1
echo "Active system: $ACTIVE | Working directory: .system/$ACTIVE/"
```

From this point, all file paths use `.system/{active-system}/` as the root.

Read:
- `.system/{active-system}/MAP.md`
- `.system/{active-system}/DESIGN.md`
- `.system/{active-system}/STATE.md`
- `.system/{active-system}/OUTCOMES.md`
- `.system/{active-system}/config.json`
- All files in `.system/{active-system}/flows/`
- All files in `.system/{active-system}/feedback/`

Extract: every external dependency, every flow with an external trigger or data source, any input boundary pre-staged in STATE.md.

## Step 2: Display Stage Banner

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ▣  SYSTEM DESIGN >> AUDITING INPUTS  ·  05/07
  Garbage in = garbage out. Classify every boundary.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Auditing input boundaries for [system name]...
Spawning sys-input-auditor agent.
```

## Step 3: Spawn sys-input-auditor Agent

Build the agent prompt with full system context.

```
Agent(prompt="First, read .claude/agents/sys-input-auditor.md for your role and instructions.

<system_context>

**System name:** [name]

**External dependencies (from MAP.md):**
[Full list with current state and which flows consume them]

**Flows with external triggers or data sources:**
[Summary: flow ID, trigger type, data source, downstream consumers]

**Pre-staged input boundaries (from STATE.md if present):**
[Any IN[N] entries the operator already drafted]

**Outcomes:**
[Outcome list]

**Config:**
Mode: [interactive | yolo]
Depth: [quick | standard | comprehensive]

</system_context>

<instructions>
Follow your agent process:
1. Enumerate every input boundary
2. For each, ask the worst-case test
3. Classify as gate, sanitise, or accept-with-tolerance
4. Define the mechanism per classification
5. For accept-with-tolerance, require a fail-loud downstream mechanism
6. Flag any unresolved boundary
7. Write INPUT-AUDIT.md
8. Identify which flows and feedback mechanisms need reconciliation

If mode is 'yolo': propose all classifications in one batch, ask one confirmation, write.
If mode is 'interactive': walk one input at a time, confirm classification and mechanism before moving on.

Depth controls detail level:
- quick: classification and one-line mechanism per input. Skip cross-references.
- standard: full classification, mechanism, failure mode, and cross-references per input.
- comprehensive: everything in standard plus tolerance-range analysis for accept-with-tolerance inputs and gate-bypass scenarios for gated inputs.

Return your structured result with the design-change list when complete.
</instructions>
", description="Audit input boundaries")
```

## Step 4: Handle Agent Return

Read the agent's structured return. Extract:
- Total input boundary count
- Count per classification
- Unresolved boundary count
- Design-change list (flows + feedback to reconcile)
- Path to INPUT-AUDIT.md

Verify the audit file was written:
```bash
ls ".system/$ACTIVE/INPUT-AUDIT.md" 2>/dev/null
```

## Step 5: Update STATE.md

Read `.system/{active-system}/STATE.md` and update:
- Stage: input-audited
- Last completed: input-audit
- Next step:
  - If design-change list is empty -> `/system:verify-closure`
  - If design-change list is non-empty -> reconcile flows and feedback, then `/system:verify-closure`
- Mark `input-audit` checkbox complete
- Append the design-change list under a new "Reconciliation required" section

## Step 6: Present Results

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ▣  SYSTEM DESIGN >> INPUTS AUDITED ✓  ·  05/07
  Every boundary classified. Garbage stops here.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Input boundaries:** [count]
  - Gates: [n]
  - Sanitisers: [n]
  - Accept-with-tolerance: [n]
**Unresolved:** [count]

[Classification summary table from agent return]
```

If unresolved boundaries exist, present them prominently:

```
## Unresolved (blocks verify-closure)

[For each:]
- **IN[N] {input name}:** {reason it could not be classified}. Resolve before continuing.
```

If design-change list is non-empty:

```
## Reconciliation required

The audit identified design changes needed in flows and feedback:

**Flows to update:**
[list]

**Feedback mechanisms to add or update:**
[list]

Reconcile these before running /system:verify-closure.
```

```
| Artifact         | Location                                    |
|------------------|---------------------------------------------|
| Input audit      | .system/{active-system}/INPUT-AUDIT.md      |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Next: [verify-closure if clean | reconcile flows and feedback first]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

</process>

<output>

- `.system/{active-system}/INPUT-AUDIT.md` (created)
- `.system/{active-system}/STATE.md` (updated)

</output>

<success_criteria>

- [ ] Abort triggered if `.system/flows/` or `.system/feedback/` missing
- [ ] sys-input-auditor agent spawned with full system context
- [ ] Every external dependency from MAP.md classified
- [ ] Every flow with external trigger or data source audited
- [ ] Each input has exactly one classification (gate / sanitise / accept-with-tolerance)
- [ ] Each accept-with-tolerance input has a fail-loud mechanism defined
- [ ] Unresolved boundaries flagged prominently
- [ ] Design-change list captured for downstream reconciliation
- [ ] STATE.md updated with correct stage and next step
- [ ] Operator knows whether to reconcile or proceed to verify-closure

</success_criteria>
