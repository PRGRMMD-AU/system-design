---
name: system:help
description: Show available /system commands and usage
allowed-tools:
  - Read
---

<objective>

Display the /system command reference.

</objective>

<process>

Display this reference:

```
/system Skill Chain

Design complete systems before building them. Close the loop.

Chain commands (run in order):

  /system:new-system [name]    Initialize a new named system design
  /system:map-system           Map what builds up, what moves, boundaries
  /system:design-flows         Design flow connections between parts
  /system:design-feedback      Design self-correction mechanisms
  /system:input-audit          Classify every input boundary (gate/sanitise/tolerance)
  /system:verify-closure       Run 5 closure tests
  /system:build-plan           Generate the build plan

Utilities:

  /system:progress             Show all systems and their current stage
  /system:switch [name]        Switch the active system
  /system:help                 This reference

Multiple systems:

  Each /system:new-system [name] creates a separate design in .system/{name}/.
  One system is active at a time. All chain commands operate on the active system.
  Use /system:switch to change which system is active.

  Example:
    /system:new-system telegram-router
    /system:new-system content-pipeline
    /system:switch telegram-router
    /system:progress

State directory: .system/{name}/ per system, .system/ACTIVE pointer
Each chain command produces artifacts consumed by the next.

The five closure tests:
  1. Output Consumption    Every output has a named consumer
  2. Trigger Sourcing      Every trigger has a specified source
  3. Self-Correction       Every accumulation has detection and correction
  4. Failure Paths         Every flow has a documented failure case
  5. Information Access    Every decision has its data source

Start: /system:new-system [your-system-name]
```

</process>
