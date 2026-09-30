```markdown
# enhancements Development Patterns

> Auto-generated skill from repository analysis

## Overview

This skill teaches you the core development patterns, coding conventions, and operational workflows used in the `enhancements` repository. The repository is primarily written in TypeScript (no framework detected) and focuses on managing Kubernetes Enhancement Proposals (KEPs) and their associated documentation and readiness reviews. You'll learn how to structure files, follow commit and code style conventions, and participate in the main KEP proposal and iteration workflows.

## Coding Conventions

**File Naming**
- Use PascalCase for file names.
  - Example: `MyFeatureProposal.md`, `KepAnalysis.ts`

**Import Style**
- Use relative imports for TypeScript modules.
  - Example:
    ```typescript
    import { analyzeFeature } from './FeatureAnalysis';
    ```

**Export Style**
- Use named exports.
  - Example:
    ```typescript
    export function analyzeFeature() { ... }
    ```

**Commit Patterns**
- Commit messages are freeform, typically short (average 49 characters).
  - Example: `Update prod-readiness for feature X`

## Workflows

### kep-proposal-workflow
**Trigger:** When someone wants to propose a new Kubernetes Enhancement Proposal (KEP).
**Command:** `/new-kep`

1. Create a new directory under `keps/sig-api-machinery/<kep-number>-<feature-name>/`.
2. Add a `README.md` and `kep.yaml` to the new directory.
3. Optionally, add `ANALYSIS.md`, `BEFORE.md`, and `AFTER.md` for supporting documentation.
4. Add a corresponding production readiness YAML under `keps/prod-readiness/sig-api-machinery/`.

**Example Directory Structure:**
```
keps/
  sig-api-machinery/
    1234-my-feature/
      README.md
      kep.yaml
      ANALYSIS.md
      BEFORE.md
      AFTER.md
  prod-readiness/
    sig-api-machinery/
      1234-my-feature.yaml
```

### kep-feedback-and-iteration-workflow
**Trigger:** When someone wants to revise a KEP based on feedback or new requirements.
**Command:** `/update-kep`

1. Edit `README.md` in the relevant KEP directory.
2. Optionally update `kep.yaml` and/or the corresponding prod-readiness YAML.
3. Commit with a message referencing the feedback or comments addressed.

**Example Commit Message:**
```
Address review comments on KEP-1234: clarify rollout plan
```

## Testing Patterns

- Test files follow the pattern `*.test.*`.
- The specific testing framework is unknown, but test files are colocated with the code or documentation they verify.

**Example Test File:**
```
FeatureAnalysis.test.ts
```

## Commands

| Command     | Purpose                                                      |
|-------------|--------------------------------------------------------------|
| /new-kep    | Start a new Kubernetes Enhancement Proposal workflow         |
| /update-kep | Revise an existing KEP based on feedback or new requirements |
```
