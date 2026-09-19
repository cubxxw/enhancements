---
name: kep-feedback-and-iteration-workflow
description: Workflow command scaffold for kep-feedback-and-iteration-workflow in enhancements.
allowed_tools: ["Bash", "Read", "Write", "Grep", "Glob"]
---

# /kep-feedback-and-iteration-workflow

Use this workflow when working on **kep-feedback-and-iteration-workflow** in `enhancements`.

## Goal

Addressing feedback and iterating on an existing KEP, typically by updating README.md and kep.yaml files, sometimes prod-readiness YAML.

## Common Files

- `keps/sig-api-machinery/*/README.md`
- `keps/sig-api-machinery/*/kep.yaml`
- `keps/prod-readiness/sig-api-machinery/*.yaml`

## Suggested Sequence

1. Understand the current state and failure mode before editing.
2. Make the smallest coherent change that satisfies the workflow goal.
3. Run the most relevant verification for touched files.
4. Summarize what changed and what still needs review.

## Typical Commit Signals

- Edit README.md in the relevant KEP directory
- Optionally update kep.yaml and/or prod-readiness YAML
- Commit with a message referencing feedback or comments

## Notes

- Treat this as a scaffold, not a hard-coded script.
- Update the command if the workflow evolves materially.