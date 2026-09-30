---
name: kep-proposal-workflow
description: Workflow command scaffold for kep-proposal-workflow in enhancements.
allowed_tools: ["Bash", "Read", "Write", "Grep", "Glob"]
---

# /kep-proposal-workflow

Use this workflow when working on **kep-proposal-workflow** in `enhancements`.

## Goal

Proposing a new Kubernetes Enhancement Proposal (KEP), including initial documentation and production readiness review.

## Common Files

- `keps/sig-api-machinery/*/README.md`
- `keps/sig-api-machinery/*/kep.yaml`
- `keps/sig-api-machinery/*/AFTER.md`
- `keps/sig-api-machinery/*/ANALYSIS.md`
- `keps/sig-api-machinery/*/BEFORE.md`
- `keps/prod-readiness/sig-api-machinery/*.yaml`

## Suggested Sequence

1. Understand the current state and failure mode before editing.
2. Make the smallest coherent change that satisfies the workflow goal.
3. Run the most relevant verification for touched files.
4. Summarize what changed and what still needs review.

## Typical Commit Signals

- Create a new directory under keps/sig-api-machinery/<kep-number>-<feature-name>/
- Add README.md and kep.yaml to the new directory
- Add related analysis/BEFORE/AFTER documentation if needed
- Add a corresponding prod-readiness YAML under keps/prod-readiness/sig-api-machinery/

## Notes

- Treat this as a scaffold, not a hard-coded script.
- Update the command if the workflow evolves materially.