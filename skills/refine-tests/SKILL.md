---
name: refine-tests
description: >-
  Reviews existing test code for conformance to the test-designing-guide and
  test-writing-guide, then produces a refinement plan. Use this skill in plan
  mode when the user wants to review or refine existing test code (a single
  test file or a directory of tests) so it follows the project's test design
  and writing conventions. Typically invoked as `/refine-tests <path>`.
license: Unlicense
metadata:
  author: Koji Hasegawa
---

Reviews existing test code for conformance to the test-designing-guide and test-writing-guide, then produces a refinement plan.

## Mode Check

This skill requires **plan mode**. Before doing anything else, check the current mode:

- `ExitPlanMode` is in the deferred tools list → **not in plan mode** → stop immediately and tell the user:
  > "This skill (`/refine-tests`) requires plan mode. Enter plan mode first: use `/plan` or press Shift+Tab to toggle."
- `ExitPlanMode` is NOT in the deferred tools list (i.e., directly callable) → in plan mode → proceed.

## Scope Check

This skill is for refining **existing** tests for conformance to the guides. If the request is out of scope, redirect:

- Adding tests for a new feature or spec change → use `/plan-feature` instead
- A failing test, or a test that verifies incorrect behavior → use `/fix-bug` instead

## Input

One or more target path arguments. Each may be a single test file, a directory (resolved recursively to its test files), or a glob. Resolve the argument(s) to the concrete set of test files to review before proceeding.

## Workflow

### Phase 1: Read the Target Tests

Launch Explore agent(s) to read the target test file(s) and the production code they exercise. Reading the production code is necessary to judge layer-appropriateness and structural-vs-spec-based issues.

### Phase 2: Conformance Review

Load the `test-designing-guide` and `test-writing-guide` skills. Apply all rules that are **verifiable from the test code alone** — no requirements document is available.

The following sections of `test-designing-guide` require requirements input or production-design changes and are **out of scope**:
- Section 5 (requirements coverage / traceability / same-layer witness)
- Section 6 (design-document output format)
- Section 7 (Testability Assessment — remedies require production-design changes)

Produce a **Findings** list. Each finding records:
- Location: file path + test method name
- Category: which guide + rule violated, or *duplicate test* (see Phase 3)
- Concrete proposed change

### Phase 3: Duplicate Detection

Compare the target test files against each other and against other tests in the same test class.

A **true duplicate** has **both** of the following in common with another test:
- **Same condition** — identical setup / input
- **Same assertion** — identical observation / expected value

Do NOT flag tests that share only one:
- Different condition → not a duplicate
- Same condition but different assertion → not a duplicate

For each true duplicate pair, append a Finding to the Findings list from Phase 2:
- Proposed change: delete the redundant test (the less accurately named one) and keep the more accurately named one. Name both explicitly.
- Do NOT propose merging same-condition tests into a single multi-assert test.

### Phase 4: Review

Read the critical test files. Confirm the proposed changes in the Findings list are consistent with each other and that each change preserves what the test verifies. Also cross-check duplicate findings (Phase 3) against conformance findings (Phase 2): a test slated for rename must not also be the redundant side of a duplicate finding.

### Phase 5: Write the Plan File

Assemble the plan file with these sections:

1. **Context** — what is being refined and why
2. **Findings** — the Phase 2 list (location / rule / proposed change)
3. **Refine Workflow** — paste the **Template** from `## Refine Workflow` verbatim as the body of this section

### Phase 6: Call ExitPlanMode

---

## Refine Workflow

Paste the **Template** below verbatim as the body of the `## Refine Workflow` section in the plan file.

### Template

```markdown
### Step 1: Modify Tests

- [ ] Apply the test changes described in the Findings section
- [ ] Run tests with `/run-tests` and confirm **all pass**

### Step 2: Refactoring

- [ ] Resolve diagnostics at warning or higher for each modified file using `jb inspectcode` (it honors `.editorconfig` severity; the Unity compiler output does not). See the `code-writing-guide` skill's `diagnostics-review-feedback.md` for the command and suppression policy
- [ ] Run tests with `/run-tests` and confirm **all pass**
- [ ] Run the Claude Code built-in `/simplify` skill (`Skill({skill: "simplify"})` — not a plugin skill) to apply quality improvements to the modified code
- [ ] Run tests with `/run-tests` and confirm **all pass**
- [ ] Commit all remaining changes to git
```
