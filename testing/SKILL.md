---
name: testing
description: How to write tests in this repo — prefer state-based testing over interaction testing, prefer fakes over mocks, skip unit tests for trivial code, integration-test every backend feature with Testcontainers and every data-fetching React page against the MSW fake backend, write component specs (*.spec.test.tsx) for presentational components and browser specs (*.browser.test.tsx, real Chromium) only where jsdom cannot render the behaviour, keep Playwright for key user journeys, keep relevant test data next to the assertion, and assert one behaviour per test. Use when writing or reviewing tests (backend or frontend), adding a feature that needs test coverage, deciding which level a test belongs at, converting a test that vi.mocks our own hooks, choosing between a component spec, an integration test and a browser spec, adding a Playwright scenario, writing a Storybook story (planning-fe/STORYBOOK.md), writing or debugging Jest tests in an Expo / React Native app (jest-expo, expo-router/testing-library, MSW), or when a test is brittle, slow or failing after a refactor. Language specifics in references/{java,typescript,react-native,python,rust}.md.
---

# Testing

The goal is **refactor-proof tests**: if the observable behaviour did not
change, the test must not fail. A test that breaks when you rename a method,
reorder a call, or extract a class was testing the implementation, not the
behaviour.

These principles follow the testing chapters of *Software Engineering at
Google* (ch. 11–14): test via public APIs, test state rather than
interactions, prefer real implementations and fakes to mocks, and keep large
tests few and focused on the journeys that matter.

Language specifics — idioms, frameworks, full examples:

| Language | Reference |
|---|---|
| Java (Spring Boot) | [`references/java.md`](references/java.md) |
| TypeScript | [`references/typescript.md`](references/typescript.md) |
| React Native / Expo (after `typescript.md`) | [`references/react-native.md`](references/react-native.md) |
| Python | [`references/python.md`](references/python.md) |
| Rust | [`references/rust.md`](references/rust.md) |

Read the language file before writing tests. The principles below apply to all
of them.

---

## 1. State over interaction

Assert on **resulting state**, not on which calls happened.

```
BAD   verify(repository).save(any());          ← passes even if save() saved garbage
GOOD  assertThat(repository.findAll()).extracting(Deck::title).containsExactly("Spring");
```

Interaction assertions couple the test to the implementation. Rename the
collaborator, inline it, batch two calls into one — the behaviour is identical
and the test fails anyway.

**The exception**, and it is narrow: when the behaviour under test *is* the
absence or presence of an interaction, and no observable state can reveal it.
The canonical case is caching:

```java
// The behaviour IS "does not hit the database on a cache hit".
// No amount of state inspection can express that.
service.get(id);                       // populates cache
clearInvocations(repository);
service.get(id);                       // second call
verifyNoInteractions(repository);
```

If you reach for `verify(...)` and can name a state assertion that would prove
the same thing, use the state assertion.

## 2. Do not unit-test trivial code

No unit tests for mappers, getters, DTO conversion, or plumbing that just moves
fields around. They cost maintenance and prove nothing — a mapping bug is
caught by the integration test that exercises the feature.

**Do** unit-test genuine logic:

- non-trivial calculations (streaks, scheduling intervals, pricing)
- dynamic programming and other algorithms
- custom data structures
- graph traversal and ordering
- parsing, tokenising, reconciliation rules

Rule of thumb: if you could get it wrong on paper, unit-test it. If the only
possible bug is a typo the compiler nearly catches, do not.

## 3. Integration-test every feature

Every backend feature gets an integration test. Use **Testcontainers** so the
database is real, and isolate everything else.

**One real component at a time** — usually the database. The app talks to a
real PostgreSQL; external services are not brought up. Do not stand up Kafka,
Quickwit, S3, and the app together and call it an integration test; that is an
E2E test with worse failure messages.

These tests are state-driven: drive the API, then assert on what is in the
database (or what the API returns), never on internal calls.

**Frontend**: the equivalent is a page rendered with its real hooks against
the **fake backend** (`src/test-support/` in `planning-fe`, served by MSW).
The fake is the one component standing in for the backend; everything of ours
above the network runs for real. See `references/typescript.md` → "Frontend".

## 4. End-to-end when a flow spans features

When a whole user flow needs covering, add a **Cucumber** scenario in
`Java-monorepo/E2E-TEST-BE-planning-app/`. Also state-driven — steps describe
what the user did and what is then true, never how the code did it.

Reach for E2E only after the integration test exists. E2E is for flows across
features; it is a bad place to test a single endpoint's edge cases.

**Frontend**: **Playwright**, in `e2e/*.e2e.test.ts`, for key user journeys only — signing
in, working the daily todo list, filing a mistake. It runs the real app
against the same fake backend, and each scenario ends by reloading the page,
because proving the change was *sent* is what a component test cannot do.

## 5. Avoid mocks

Every mock is a guess about how a collaborator behaves, and the guess rots
silently.

In order of preference: the **real** implementation; a **fake** — a
lightweight working implementation holding real state, like the frontend's
fake backend; a **stub** returning canned data; and a **mock** that records
calls, last. A fake must mirror what it replaces, so change it in the same PR
as the thing it fakes.

**Legitimate**: things that cannot run in a test, or whose setup dwarfs the
value. In this repo the standard case is **authentication** — integration tests
mock `CurrentUserService` rather than standing up Cognito:

```java
@MockitoBean private CurrentUserService currentUserService;
// ...
when(currentUserService.getCurrentUsersId()).thenReturn(userId);
```

That is a boundary you own and control, not a collaborator whose behaviour you
are asserting. Also acceptable: clocks, random sources, third-party HTTP.

**Not legitimate**: mocking your own repository, service, or mapper to avoid
writing an integration test. On the frontend: mocking your own hooks, `Api`
client or stores to avoid rendering against the fake backend.

## 6. One behaviour per test

Each test asserts **one** behaviour. If a function does two things, write two
tests. Side effects get their own tests.

```
BAD   createDeck_worksCorrectly()
        → asserts the response, the DB row, the audit log, and the search index

GOOD  createDeck_persistsDeckWithGivenTitle()
      createDeck_writesAuditLogEntry()
      createDeck_indexesDeckForSearch()
```

A test asserting everything fails for many unrelated reasons, and its name
cannot tell you which. Naming is the tell: if you need "and" in the test name,
it is two tests.

## 7. Keep relevant data next to the assertion

The fields a test depends on must be **visible in the test body**, right where
you can read them against the assertion. Everything else is noise and belongs
in a builder.

```
BAD                                    GOOD
──────────────────────────────────     ─────────────────────────────────────
Deck deck = TestData.aDeck();          Deck deck = aDeck().title("Spring").build();
deckRepository.save(deck);             deckRepository.save(deck);
// ...                                 // ...
assertThat(result.title())             assertThat(result.title())
    .isEqualTo("Spring");                  .isEqualTo("Spring");
                                       
Where does "Spring" come from?         The input is right there, next to
Somewhere in TestData. The reader      the expectation. Nothing to look up.
has to go find it.
```

Write a **default builder in the test class** that fills mandatory fields with
throwaway values. Tests then override only what they assert on:

```java
private AnkiDeck.AnkiDeckBuilder aDeck() {
  return AnkiDeck.builder()
      .id(UUID.randomUUID())
      .ownerId(userId)
      .status(AnkiDeckStatus.HOT)   // irrelevant to most tests — hidden
      .title("untitled");           // overridden when it matters
}
```

Structure every test **Arrange → Act → Assert**, in that order, with the
relevant data at the top.

## 8. Assert fluently

Use the language's fluent assertion library (AssertJ in Java) and chain
assertions rather than writing many separate ones. Prefer extracting the fields
you care about over asserting whole objects:

```java
assertThat(decks)
    .extracting(AnkiDeck::getTitle, AnkiDeck::getStatus)
    .containsExactly(tuple("Spring", HOT), tuple("Kafka", HOT));
```

`containsExactly` when order is part of the behaviour;
`containsExactlyInAnyOrder` when it is not. Asserting order that the behaviour
does not actually guarantee is a classic flaky test.

Never assert an entire object with `isEqualTo` unless every field is genuinely
part of the behaviour — that is the brittleness of principle 6 in another form.

## 9. Coverage is a signal, not a target

Coverage is reported on every PR and **never enforced**. There is no threshold,
deliberately: the cheapest way to hit a coverage target is a test that executes
code without asserting anything, which is worse than no test — it costs
maintenance and creates false confidence.

Use the report to answer "is there behaviour here nobody exercises?", then go
and write the test that principle 2 says is worth writing. Do not use it to
answer "are we at 80% yet?".

Generated and behaviour-free code (MapStruct impls, entities, DTOs, config) is
excluded from the report, so the number reflects logic rather than plumbing.

```bash
cd Java-monorepo/BE-common-be
./gradlew test jacocoAggregateReport
open build/reports/jacoco/aggregate/html/index.html
```

Read the **aggregate**, not per-module reports: the feature modules hold the
code but most of it is exercised by integration tests living in `:application`,
so a module read in isolation looks far less tested than it is.

## 10. A skipped test suite is worse than a failing one

Integration tests skip locally when Docker is not running, so the rest of the
suite still works. **In CI they fail instead.** A green build that quietly ran
no integration tests is the worst outcome available — it reports success while
testing nothing.

If you add a condition that can disable tests, make sure it cannot do so
silently in CI. Applies to every language: an `it.skip`, an `#[ignore]`, a
`@Disabled` left in place, or an environment probe that degrades to "skip".

---

## Quick decision table

| Situation | What to write |
|---|---|
| Mapper, DTO conversion, getter | Nothing |
| Streak calc, DP, graph, parser, custom structure | Unit test |
| New endpoint / feature | Integration test with Testcontainers |
| Flow spanning several features | Integration tests + one Cucumber scenario |
| "Does not hit the DB when cached" | Integration test + `verifyNoInteractions` |
| Needs the logged-in user | Integration test, mock `CurrentUserService` only |
| Tempted to mock your own repository | Write the integration test instead |
| React page or component that fetches data | `Foo.integration.test.tsx`, real hooks, fake backend |
| Presentational React component with branching | `Foo.spec.test.tsx`, props from a factory, no network |
| SVG map clicks, layout/overflow, canvas, Monaco | `Foo.browser.test.tsx` in real Chromium, as well as the above |
| Pure frontend logic (URL state, filters, dates) | Unit test |
| Key user journey, or "survives a reload" | One Playwright scenario in `e2e/foo.e2e.test.ts` |
| New React component, or documenting an existing one | `Foo.stories.tsx` beside it, ticked in `STORYBOOK.md` |
| Tempted to `vi.mock` our own hook, `Api` or store | Lint rejects it in any test; seed the fake backend (typescript.md → "Converting a test that mocks our code") |

## Reviewing a test

- Would this still pass after a pure refactor? If not, it tests implementation.
- Can you tell what broke from the test name alone?
- Is every asserted field actually part of the behaviour being tested?
- Is the data the assertion depends on visible in the test body?
- Is every mock either auth, a clock, randomness, or a third party?
- Frontend: is the component found by role and name? If it cannot be, is that
  an accessibility bug to fix in the component?
