---
name: acceptance-test-driven-development
description: Turn human-written feature drafts into executable acceptance specifications before implementation. Use when asked to write a spec from requirements, implement features with acceptance-test-driven development (ATDD), or constrain agent drift through tests. Treat runnable behavior tests as the acceptance specification and enforce one test per behavior.
---

# Acceptance Test Driven Development

## Establish the contract

Treat the user's Markdown feature draft as the source of intent. Make runnable acceptance tests the executable specification and completion gate. Do not substitute a Markdown specification, Gherkin without step implementations, a test plan, or a passing-test claim for actual executable tests.

Preserve the user's intent throughout translation. Tests cannot authorize an omission or contradict an explicit requirement. When prose and tests disagree, expose and resolve the discrepancy rather than declaring the implementation correct because tests pass.

## Load the testing skill first

Before designing or writing tests, discover and read the companion skill named `testing` through the available skill catalog or repository skill directories. Follow its applicable testing conventions and references. If the user explicitly identifies a replacement testing skill, load that instead.

Keep framework details in that companion skill; keep this skill responsible for the ATDD workflow and acceptance integrity. Do not create a second testing skill as part of an ordinary feature task.

If `testing` is unavailable, explicitly report that it is missing, then continue using this skill's baseline rules and existing repository conventions. Do not claim to have loaded it. Follow explicit user instructions when resolving conflicts and call out consequential conflicts between skill instructions.

## 1. Read intent and inspect the system

- Read the feature draft, relevant code, existing tests, repository instructions, and test/CI commands.
- Identify observable behaviors, inputs, outputs, state changes, permissions, failure cases, and important boundaries.
- Ask only about ambiguities that materially change expected behavior. Resolve routine implementation choices using existing conventions.
- Do not invent business rules, error semantics, numerical thresholds, or out-of-scope features.
- Preserve existing valid tests. Reuse coverage that already proves a requested behavior.

## 2. Translate behaviors into executable tests

Write acceptance tests before changing production behavior. Add only the minimum harness or interface scaffolding needed to run them; disclose scaffolding separately from implementation.

Maintain a lightweight mapping from each requested behavior to its test file and test name. Use test names or nearby comments for identifiers when useful. This mapping is an index, not an alternative specification. Mark genuinely nonautomatable requirements as gaps with a concrete manual check; do not silently drop them or claim complete executable coverage.

Enforce these rules:

- **One test, one behavior:** express one scenario and its observable outcome per test. Split unrelated success, failure, authorization, and boundary behaviors into separate tests.
- Allow multiple assertions when they jointly prove that single behavior. Do not confuse one behavior with one assertion, one method, or one production class.
- Use descriptive names that identify the condition and expected outcome. Use Arrange/Act/Assert or equivalent structure.
- Make each test independent, deterministic, and runnable alone. Isolate data and clean up resources. Do not depend on test order, arbitrary sleeps, or uncontrolled external services.
- Assert public behavior and relevant persisted state. Avoid coupling acceptance to private methods, internal call sequences, implementation snapshots, or the implementation's own calculation of expected results.
- Choose the smallest test boundary that faithfully proves the requirement. Use browser E2E tests when browser-to-backend integration is part of the behavior; do not force every case through a browser.
- For full-stack acceptance, exercise the real frontend, backend, and relevant infrastructure. Do not mock the behavior under test or bypass the security/persistence path the test claims to verify.
- For Spring Boot persistence behavior, prefer application integration tests using the production database engine through Testcontainers and real migrations when supported. Do not present an in-memory substitute or mocked repository as proof of database integration.
- Stub uncontrolled third-party services at explicit boundaries. Report what is real, stubbed, or unverified. Test error paths with controlled fault injection without replacing the behavior being verified.
- Use executable Gherkin only when the repository already supports it or the user requests it; readable native test code is sufficient.

Example decomposition for cart removal:

- Successful removal deletes the selected item and updates its total: one behavior, potentially several assertions.
- Failed removal retains the item and its total: a separate behavior.
- Another customer's item cannot be removed: a separate behavior.
- Removing the last item disables checkout: a separate behavior.

## 3. Demonstrate RED

Run the new tests against the pre-implementation behavior and inspect the actual failures. Require evidence that each new or changed behavior test detects the missing or incorrect behavior.

Do not count compilation errors, missing tools, failed container startup, broken selectors, or unavailable services as a meaningful RED. Fix the harness or minimum scaffolding, then rerun until the behavioral assertion fails for the expected reason.

If a test already passes, investigate: the behavior may already exist, the test may be weak, or its setup may bypass the target. Do not manufacture a failure for existing correct behavior. Report existing coverage accurately. For a regression fix, demonstrate that the test reproduces the original defect before fixing it where feasible.

For high-risk or suspiciously weak assertions, temporarily introduce a targeted behavioral fault and confirm the test catches it. Restore the fault completely; do not leave mutation changes in the final diff. Use mutation tools when already available and appropriate rather than requiring them for every task.

If execution is blocked, state the precise blocker and missing evidence. Continue useful independent work, but do not report RED, GREEN, or completion without execution evidence.

## 4. Implement GREEN and refactor

Implement the requested behavior against the executable specification. Keep acceptance expectations stable while resolving implementation failures. Refactor while keeping the acceptance suite passing; add focused lower-level tests where they improve coverage or diagnosis.

For a request to write the spec only, stop after producing executable tests and honest RED/existing-behavior evidence. Do not implement the production feature unless authorized by the task.

## 5. Protect acceptance integrity

Never obtain GREEN by removing assertions, loosening expected values, skipping or disabling tests, filtering out failures, catching and swallowing assertion failures, excessive retries, mocking away the target behavior, hard-coding fixture answers, or changing CI to avoid running the specification.

Distinguish a genuine test/harness defect from an implementation defect using the original requirements and independent evidence. Correct a demonstrably wrong test transparently and rerun it; do not freeze known-bad tests. Explain any change to acceptance meaning and obtain clarification if it changes unresolved business intent. A changed user requirement can legitimately require changed tests.

Inspect the final diff for assertion changes, skips, retry changes, configuration exclusions, and replacement of real dependencies with mocks. Report acceptance changes separately from production changes. A prompt cannot enforce file protection: use independent review and CI when available; do not claim this skill makes tests tamper-proof.

## 6. Verify and report

Run the affected acceptance tests and relevant regression/build checks using the repository's normal commands. Ensure CI discovers the new tests through existing conventions; add scoped wiring when necessary. Do not expand into unrelated CI infrastructure changes.

Report concisely:

- Requested behaviors and their executable test locations/names.
- Observed RED evidence, or which behaviors already existed.
- Final commands, actual outcomes, and tests skipped or not run.
- Test boundaries, dependency substitutions, and unresolved coverage gaps.
- Any acceptance expectation changes and their justification.

For implementation tasks, mark complete only when requested automatable behaviors are covered and the required checks actually pass. For spec-only tasks, report the executable spec as ready while clearly identifying expected failing tests and any blockers. Do not imply that a finite test suite proves the absence of all defects.

## Testing principles and references

Apply Google's guidance to test behaviors rather than methods, favor behavior over implementation details, and choose the smallest faithful test scope. These references motivate the baseline; delegate detailed testing guidance to `testing` once available.

- Software Engineering at Google, Testing Overview: https://abseil.io/resources/swe-book/html/ch11.html
- Software Engineering at Google, Unit Testing (including testing behaviors): https://abseil.io/resources/swe-book/html/ch12.html
- Test Behaviors, Not Methods: https://testing.googleblog.com/2014/04/testing-on-toilet-test-behaviors-not.html
- Test Behavior, Not Implementation: https://testing.googleblog.com/2013/08/testing-on-toilet-test-behavior-not.html
