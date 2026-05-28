---
name: system:verify-closure
description: Run five closure tests to verify the system design is complete
allowed-tools:
  - Read
  - Bash
  - Grep
  - Glob
  - Write
  - Agent
---

<objective>

Run five closure tests against the system design. Verify every output is consumed, every trigger is sourced, every accumulation is regulated, every failure is handled, and every decision has data.

This is the intellectual core of the /system skill chain. A system that passes all five tests has every loop closed.

**Requires:** `.system/MAP.md`, `.system/flows/`, `.system/feedback/` must exist.

**Produces:**
- `.system/CLOSURE-REPORT.md`
- `.system/CLOSURE-REPORT.html`

**After this command:** Run `/system:build-plan` to generate the build plan.

</objective>

<execution_context>

@.claude/system-design/references/closure-test.md
@.claude/system-design/templates/closure-report.md

</execution_context>

<process>

## Step 1: Resolve Active System and Check Prerequisites

```bash
ACTIVE=$(cat .system/ACTIVE 2>/dev/null)
[ -z "$ACTIVE" ] && echo "ERROR: No active system. Run /system:new-system [name] first." && exit 1
[ ! -f ".system/$ACTIVE/MAP.md" ] && echo "ERROR: Run /system:map-system first" && exit 1
[ ! -d ".system/$ACTIVE/flows" ] && echo "ERROR: Run /system:design-flows first" && exit 1
[ ! -d ".system/$ACTIVE/feedback" ] && echo "ERROR: Run /system:design-feedback first" && exit 1
[ ! -f ".system/$ACTIVE/INPUT-AUDIT.md" ] && echo "ERROR: Run /system:input-audit first. Closure verification blocks until every input boundary is classified per .claude/rules/garbage-in-garbage-out.md." && exit 1
echo "Active system: $ACTIVE | Working directory: .system/$ACTIVE/"
```

Count flow files and feedback files:
```bash
FLOW_COUNT=$(ls ".system/$ACTIVE/flows/"*.md 2>/dev/null | wc -l)
FEEDBACK_COUNT=$(ls ".system/$ACTIVE/feedback/"*.md 2>/dev/null | wc -l)
echo "Flows: $FLOW_COUNT | Feedback mechanisms: $FEEDBACK_COUNT"
```

If either count is zero, tell the operator which skill to run.

From this point, all file paths use `.system/{active-system}/` as the root.

## Step 2: Display Stage Banner

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ◉  SYSTEM DESIGN >> VERIFYING CLOSURE  ·  05/06
  Five tests. Every loop must close.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Step 3: Load Context

Read all design artifacts from `.system/{active-system}/`:
- `.system/{active-system}/SYSTEM-MAP.md`
- `.system/{active-system}/MAP.md`
- `.system/{active-system}/OUTCOMES.md`
- `.system/{active-system}/DESIGN.md`
- All files in `.system/{active-system}/flows/`
- All files in `.system/{active-system}/feedback/`

## Step 3: Spawn Closure Verifier

Spawn sys-closure-verifier agent with full context:

```
Agent(prompt="
Read your agent definition first:
@.claude/agents/sys-closure-verifier.md

<system_artifacts>
[Include content of all .system/ files]
</system_artifacts>

<instructions>
Run all 5 closure tests against the system design.
Write .system/CLOSURE-REPORT.md using the template.
Return CLOSURE VERIFIED or CLOSURE GAPS FOUND with results.
</instructions>
", subagent_type="general-purpose", description="Closure verification")
```

## Step 4: Generate HTML Closure Report

After the agent writes CLOSURE-REPORT.md, generate a self-contained interactive HTML report.

Use the report pattern from `.claude/skills/html-templates/report.md` (read that file before generating HTML).

Brand spec (white theme):
```css
--bg: #ffffff; --bg-secondary: #f8f8fb; --text: #0f0f0f; --text-muted: #6b7280;
--accent: #7c6af7; --accent-light: #ede9fe;
--success: #22c55e; --success-bg: #f0fdf4;
--error: #ef4444; --error-bg: #fef2f2;
--warning: #f59e0b; --warning-bg: #fffbeb;
--border: #e5e7eb; --max-width: 1100px;
--font: system-ui, -apple-system, sans-serif;
```

HTML structure:
- **Header**: system name, overall score (e.g. "4/5 PASS"), date
- **Score grid**: 5 check cards — PASS = green badge, FAIL = red badge, WARNING = amber badge
- **Accordion sections**: one per closure test, auto-expanded if FAIL status
- **Failures section** (if any): each failure with plain-language explanation and recommended fix
- **Action items table** (if failures): Priority, Item, Recommended Fix columns

Interactivity (all self-contained JS):
- Accordion toggle on each test section
- Auto-open any FAIL sections on page load
- ToC scroll highlighting

Save to `.system/{active-system}/CLOSURE-REPORT.html`.

Add this as the first line of `.system/{active-system}/CLOSURE-REPORT.md` (after any frontmatter):
```
[Open Visual Report](CLOSURE-REPORT.html)
```

After saving, construct the absolute file URL so the operator can open it directly from chat:
```bash
echo "file://$(pwd)/.system/$ACTIVE/CLOSURE-REPORT.html"
```

Output this in your response as a clickable markdown link:
```
[Open Closure Report in Browser](file:///absolute/path/to/CLOSURE-REPORT.html)
```

### Publish to visible vault folder

Copy the final artifacts to `System-Designs/{active-system}/` so they are visible in Obsidian's file explorer, independent of whether the operator uses Cursor/VS Code:

```bash
mkdir -p "System-Designs/$ACTIVE"
cp ".system/$ACTIVE/CLOSURE-REPORT.md" "System-Designs/$ACTIVE/CLOSURE-REPORT.md"
cp ".system/$ACTIVE/CLOSURE-REPORT.html" "System-Designs/$ACTIVE/CLOSURE-REPORT.html"
echo "Published to System-Designs/$ACTIVE/"
```

## Step 5: Handle Results

**If `## CLOSURE VERIFIED`:**

Present the score and test summary.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ◉  SYSTEM DESIGN >> CLOSURE VERIFIED ✓  ·  05/06
  All five tests passed. Every loop is closed.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Score: 5/5 tests passed
Items checked: [total]
Warnings: [count]

[Test result table from agent return]

| Artifact              | Location                                        |
|-----------------------|-------------------------------------------------|
| Closure report        | System-Designs/{active-system}/CLOSURE-REPORT.md   |
| Visual report         | System-Designs/{active-system}/CLOSURE-REPORT.html |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Next: /system:build-plan -- classify components and generate the build sequence

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**If `## CLOSURE GAPS FOUND`:**

Present failures with fix recommendations.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ◉  SYSTEM DESIGN >> CLOSURE GAPS FOUND  ·  05/06
  Fix before proceeding. Gaps compound.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Score: [X]/5 tests passed
Failures: [count]

[Failure table with recommended fixes]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Fix the gaps, then re-run /system:verify-closure.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Step 6: Update State

Read `.system/{active-system}/STATE.md` and update:
- Current Position: verification
- Last completed: verify-closure (if passed)
- Mark verify-closure checkbox

Write updated STATE.md.

</process>

<output>

- `.system/CLOSURE-REPORT.md`
- `.system/CLOSURE-REPORT.html`
- `.system/STATE.md` (updated)

</output>

<success_criteria>

- [ ] Prerequisites checked (MAP.md, flows/, feedback/ exist)
- [ ] All design artifacts loaded
- [ ] sys-closure-verifier agent spawned with full context
- [ ] All 5 tests run with per-item results
- [ ] CLOSURE-REPORT.md written
- [ ] HTML report generated using report pattern from html-templates/report.md
- [ ] HTML has score header, 5 check cards, accordion detail sections, action items table
- [ ] FAIL sections auto-expand on page load
- [ ] CLOSURE-REPORT.html saved alongside markdown
- [ ] [Open Visual Report] link added to top of CLOSURE-REPORT.md
- [ ] Results presented clearly (pass or gaps)
- [ ] Failures paired with specific fix recommendations
- [ ] STATE.md updated
- [ ] Operator knows next step

</success_criteria>
