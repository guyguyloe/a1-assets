---
name: write-tests
description: Audit and fill test gaps in an existing codebase by mapping each behavior to the test type that can actually falsify it. Use when adding tests to code that already exists, hardening coverage on an untested module, or diagnosing "tests stayed green but it broke at runtime" failures. This skill is not to be used for Test-Driven Development.
---

# Write Tests

For code that already exists and has no issue spec. The complement of `tdd`: TDD starts from an acceptance criterion and writes the test before the code; this skill starts from the code and asks what tests it deserves. Shared vocabulary: [TEST-TYPES.md](../_references/TEST-TYPES.md) (unit / integration / component-level integration / e2e, and how to pick), [LANGUAGE.md](../_references/LANGUAGE.md) (module / interface / seam / adapter), [REQUIREMENTS.md](../_references/REQUIREMENTS.md) (behavior / acceptance criterion).

## Why this skill exists

Coverage tools count *files touched by a test*. They cannot tell you the test is at the **wrong seam** — a unit test that mocks a store and asserts a method was called stays green while the component that composes that store breaks at runtime. "Tests exist" ≠ "behavior is falsifiable." This skill hunts for the invisible gap: tests that pass because they test the wrong thing, not because the behavior is correct.

## Two kinds of gap

- **Missing seam** — a seam where behavior is observable has no test at all.
- **Wrong-type test** — a seam has a test, but the type cannot falsify the behavior there (a unit test at an integration seam; a mocked-collaborator test where composition is the behavior). This is the gap coverage reports hide. See the "green tests, broken composition" failure mode in [TEST-TYPES.md](../_references/TEST-TYPES.md).

## Two kinds of fill — per gap, user-owned, no silent default

For each gap, the fill is one of:

- **Characterization test** — pins *current* behavior as-is. Runs green. Use when the user wants to lock in what exists before changing it. Risk: it ossifies a bug as "the spec."
- **Specification test** — asserts *intended* behavior recovered from AGENTS.md, comments, README, or the user. Runs red if the code is wrong → surfaces a bug for the user to decide on. Use when intent is recoverable.

Do **not** default to either. Classify each gap and let the user choose. A characterization test that goes green on broken composition is worse than no test — it manufactures false confidence. When intent is recoverable from the repo's own docs, bias the recommendation toward specification; when the user explicitly says "pin what's there, we'll fix later," use characterization.

## Workflow

### 1. Read the repo

Read the repo's `AGENTS.md` first. Note every file it references — conventions, architecture, domain glossary, test setup, framework, and the exact command to run tests. If `AGENTS.md` is absent, say so and gather what you can from `package.json` / config / existing test files, flagging the absence as a gap.

**Scope.** If the user pointed at an area or module, target it. If not, ask whether they want a whole-repo pass or a specific area. Both are valid; the skill is neutral. Whole-repo can be large and noisy — surface a count before diving in.

### 2. Discover existing tests

Find every test file in scope. For each, record:

- **Type** — unit / integration / component-level integration / e2e ([TEST-TYPES.md](../_references/TEST-TYPES.md)).
- **Seam** — where its behavior is observable (a function's return value, a module interaction, the rendered UI, a full journey).
- **What it actually falsifies** — read the assertions, not the test name. Names lie.

### 3. Map seams in the target

For the area in scope, list where behavior is genuinely observable. A store method's seam is its return value; the seam for "switching departments shows the right view" is the rendered UI. The seam determines the minimal test type that can falsify the behavior.

### 4. Identify gaps

Compare mapped seams against discovered tests. Report both kinds:

- **Missing seam** — name the seam and the minimal type that falsifies behavior there.
- **Wrong-type test** — name the existing test, the seam it *should* cover, why its type can't falsify the behavior there, and the type that would.

### 5. Report and get approval (gate)

Present gaps, prioritized (critical paths and composition seams first). For each gap show: the seam, the minimal type needed, the existing test (if wrong-type), and your characterization-vs-specification recommendation with the reason. Ask the user which gaps to fill and which kind. Do not write any test until this is approved — same gate discipline as TDD's planning step.

### 6. Write tests

For each approved gap, write the minimal test at the right seam and type. **Run it.** Report the result honestly:

- **Green** — behavior confirmed (specification) or pinned (characterization). Say which.
- **Red** — for a specification test, red is a *signal*, not a failure to fix. Surface the failing assertion and your read of the likely bug to the user. Do **not** weaken the test, broaden the matcher, or mock the collaborator to force green — that recreates the exact failure mode this skill exists to prevent. For a characterization test, red means your understanding of current behavior was wrong; revise the assertion to match actual behavior and re-run.

## When the repo has no test setup

Do not pick a framework silently. Present options ranked by utility for the stack and area, drawing from `AGENTS.md` conventions and surrounding context (language, package manager, existing dependencies, build tool). Give a one-line reason per option. Ask the user to choose. Framework selection is a meaning-changing decision the user owns.

## Checklist per test

```
[ ] Test targets a seam where the behavior is actually observable
[ ] Test type can falsify the behavior at that seam (minimal type, not higher)
[ ] Gap classified as characterization or specification with user sign-off
[ ] Test runs; result reported honestly — red on a specification test is a signal, not a fix target
[ ] No collaborator mocked away to force green at an integration seam
```
