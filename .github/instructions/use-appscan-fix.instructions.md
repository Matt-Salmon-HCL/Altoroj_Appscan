---
description: "Use the appscan-fix skill for AppScan issue remediation, fix verification, and correlation-driven triage requests. Triggers: appscan, issue id, remediation, fix verification, DAST, IAST, correlation group."
applyTo: "**"
---

When a user asks to remediate, verify, or triage AppScan findings:

1. Use the appscan-fix skill as the primary workflow.
2. Prefer clue-first analysis from IAST contextual hints before manual repository lookup.
3. Use manual lookup only when IAST clues are insufficient or cannot map to editable source with high confidence.
4. Enforce evidence-first ordering: complete metadata and correlation gates before any source edit, compile, or boot.
5. Do not perform retroactive metadata backfill after patching. If gate order is violated, declare the run invalid and restart the workflow.
6. Output the required sections and windows defined by the appscan-fix skill, including:
- Workflow Status
- Unified Evidence Window
- Local Runtime Verification (when compile/boot verification is attempted)
- Contextual Clues Used
- Source location evidence with file and line

If no AppScan context exists, proceed with normal coding workflow.
