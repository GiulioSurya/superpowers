# Writing Good Tests

**Load this reference when:** writing, changing or reviewing tests, adding
mocks, or adding cleanup/helper methods for tests.

## Overview

A test exists to catch a specific break. Every test is also a cost: it is
read, run on every change, and rewritten whenever the decision it pins
changes. A suite of tests that each catch a real break beats a larger one
that restates the code. Three principles govern everything here:

```
1. Every test names the break it catches
2. Every test exercises the real thing
3. Every test earns its cost: one per behavior, at the lowest level, fast
```

Strict TDD produces the first two naturally: a test written first and
watched failing against real code has already proven it can fail, and only
earns a mock when the real dependency proves slow or external. The third
is a choice you make before writing each test.

## What Gets a Test

**Worth a test**, one per behavior:

- **A behavior someone depends on**, seen at the public interface: a
  return value, a state change, a row written, a response, an error or a
  refusal, text whose content depends on data or permission.
- **A rule your code enforces**: validation, a computed default,
  normalization, a permission or tenant check, a limit.
- **A boundary your code branches on**: empty, zero, one, the cap, cap+1,
  missing row, wrong owner. Only the boundaries this code actually treats
  differently; a value that takes the same branch as one already tested
  adds nothing.
- **A fixed bug**: the test that reproduces it on the user's path, naming
  the bug or incident.
- **A contract between two parts that can drift**: the test reads both
  sides (the other file, schema or config), never a copy of one of them.
- **A property the code exists to provide** — concurrency, ordering,
  timeout, isolation of a crashing child process: one real test per
  property.

**Not worth a test.** Don't write these; when you touch a test file that
has them, delete them and list them in your report:

| Pattern | Why it catches nothing | Instead |
|---------|------------------------|---------|
| Text presence: a prompt, skill, message, doc, label or log "contains" a phrase | Proves the text is the text; every rewording breaks it | Test the code that branches on it; instructions for agents are tested by the agent's behavior; fixed UI labels get no test |
| Echoed values: the expected value copied from a constant, config, price list or model list | Fails only on intentional edits | Test the behavior that uses the value; if two sources must agree, the test reads both |
| Language, framework or library features: frozen, `__slots__`, a required field, enum membership, "the router calls my handler" | Their maintainers' test | Test your own rule (a validator, a default) once |
| Import or export smoke: the module loads, the name exists | Any real test that imports it already fails | Nothing |
| Tests of your own fakes and helpers | The fake is not the product | Nothing; keep fakes small |
| Mock-call assertions on internal collaboration: called-with, call order, fan-out to N sinks, `close()` called | Passes while the outcome is wrong; breaks on every refactor | Assert the end state on an in-memory fake |
| Reading the source: `read_text`, `getsource`, `readFileSync` or a regex over code to pin names, counts, imports or order | Change detector: fails on every redesign, never on a bug | An architecture rule goes in a linter (ruff banned-api, import-linter, ESLint no-restricted-imports) |
| "Does not raise", with no decision behind it | Cannot fail for a meaningful reason | Assert what the call returns or changes |
| The same case at two levels | Double maintenance for one break | Keep the lowest level that sees it; delete the other |
| Several bodies that differ only in literals | Many tests for one behavior | One parametrized or table test |
| A live external service answers (a provider ping, an API is reachable) | Tests their uptime, not your code | Test your adapter against a recorded response; a smoke check is a script outside the suite |
| The text of a one-shot migration that has already run everywhere | Pins history | Test the resulting schema or behavior against a database, or delete it once applied |
| Needs a local environment variable, an absolute path, a file outside the repo, or the network | Protects one machine | A hermetic fixture; a skip only with a stated reason |

**Never cut, even when they look like a row above** — they are the tests
the table exists to protect:

- the regression test of a fixed bug, named after the bug;
- a security, permission or tenant-isolation test, even when its proof
  uses text (another tenant's data must not appear in the output);
- a contract test that reads both sides of the contract;
- byte-for-byte equality of two outputs (cache stability) — a behavior,
  not a text test;
- a mock-call assertion on an external effect that is otherwise
  unobservable (a message sent, an RPC whose name is the contract);
- a text check on an artifact with no runtime to exercise, whose text is
  what others consume (a Dockerfile, a CI workflow, a deploy definition),
  or a repo-wide guard for a security invariant no runtime test reaches
  (every view declares the security option, no secret in a deploy file).

"Another test already covers it" is a claim to check by reading that
test, never by its name.

## Principle 1: Name the Break

Before writing the test body, answer: **what production change should
make this test fail — and is that change a bug or a decision?** A test
earns its place by catching a wrong branch, missing side effect, wrong
argument, boundary case, or broken contract.

**Derive expectations independently.** Use literals and hand-checked
fixtures; table-driven tests with literal `want` values are the preferred
shape. An expectation computed by the code under test — or its helpers —
passes no matter what that code does:

```typescript
// ❌ Mirror assertion: the same builder computes both sides — always true
const expected = buildSearchQuery({ tag: 'urgent' });
expect(buildSearchQuery({ tag: 'urgent' })).toBe(expected);

// ✅ Hand-derived literal
expect(buildSearchQuery({ tag: 'urgent' })).toBe('tag:"urgent"');
```

**No change detectors.** If only intentional decisions can fail a test —
a constant's value, exact message wording, private structure — it fires
on redesign and sleeps through bugs. Test the behavior that depends on
the decision: not `expect(MAX_RETRIES).toBe(5)` but "a failing call is
retried 5 times and the 6th attempt never happens."

**Your code, not the framework.** Test the contract your code makes at
its boundaries — the route you register, the query you emit, the payload
you produce. Constructors, getters, constants, and trivial forwarding earn
tests only when they validate, normalize, default, derive, enforce, or
cause side effects — otherwise assert the first consumer-visible result
that depends on them. When upstream behavior genuinely surprised you,
write one narrow characterization test naming the assumption.

### Gate Function

```
BEFORE writing the test body:
  Name the production change that would make this test fail.

  Cannot name one            → no test; redesign around an observable behavior
  "The source text changed"  → run the artifact and assert its effects
  Only intentional decisions → change detector; test the behavior
                               that depends on the decision
  Already caught elsewhere   → no test, once you have read the test
                               that catches it (never for a bug's
                               regression test)

  Confirm the expected value is derived without the code under test.
```

## Principle 2: Exercise the Real Thing

**The mock earns no assertions** — except an external effect with no
other observable (see "Never cut"). A mock assertion passes when the mock
is present and fails when it is absent — it says nothing about the
component. Assert the real component's behavior; if the mock is what you
are checking, unmock it or delete the assertion.

```typescript
// ✅ Real behavior
expect(screen.getByRole('navigation')).toBeInTheDocument();

// ❌ Mock existence
expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
```

**Prefer real components, then fakes, then mocks.** Learn every side
effect of the real method before replacing it; replace only the slow or
external level and keep what the test depends on real. A fake in memory
(a dict for a repository, a list for a message bus) lets you assert the
end state; a mock only lets you assert calls.

**Make doubles specific and complete.** When arguments or ordering are
the contract, assert them; give each branch (success, error, malformed)
its own fixture. Mock the complete structure as it exists in reality —
all documented fields — not just the ones your test reads.

**Production classes carry production methods only.** Cleanup that only
tests need lives in test utilities, never as a method on the production
class.

### Gate Function

```
BEFORE adding a mock or test helper:
  List the real method's side effects; keep the ones the test
  depends on real — replace the slow/external level below them.

  A fake whose end state you can assert beats a mock.
  Mock responses mirror the complete real structure.
  About to assert on the mock itself? Unmock it or delete the assertion.
```

## Principle 3: Earn the Cost

**One test per behavior, not per function.** A function with no behavior
of its own is covered by the first consumer-visible result that uses it.
Several cases of one behavior are one parametrized test. A test whose name
needs "and" tests two behaviors.

**The lowest level that sees the defect.** A higher-level test (several
components, HTTP, a browser) is for what only it can see — the wiring
between parts, one per seam — not for cases a unit test already covers.

**Fast by default.** A unit test runs in milliseconds and touches no
network, disk outside a temp directory, clock or sleep:

- Slow machinery — a child process, a converter (LibreOffice, pandoc,
  OCR), a browser, a real network call, a real wait — gets **one real
  test per property it guarantees**. Every other case runs on a pure
  function, a fake, an injected clock, or deadlines of milliseconds.
- Never sleep to wait: await the event or poll for the condition, and
  inject the clock instead of waiting for it.
- An expensive fixture that tests only read (one conversion, many
  assertions) is built once per module, not once per test.
- A test that takes more than a second says in a comment which property
  it proves that no fast test proves.

**Hermetic.** A test passes on a clean checkout with no local environment
variables, absolute paths or network. A skip names its reason.

## When Code Changes, Tests Change Too

- A test that pinned a decision you just changed is rewritten to assert
  the new behavior, or deleted if it was a change detector — never patched
  to the new literal.
- When you touch a test file, delete the not-worth tests in it and list
  them in your report. A sweep beyond the files you touch is its own task,
  with an inventory your human partner approves.
- Never delete or weaken a test to make the suite green. A red test you
  believe is wrong goes in your report with your reasoning.

## The Mutation Check

Before finishing, mentally mutate the production code; at least one test
should fail for each realistic mutation:

- Wrong constant or argument
- Wrong branch handler
- Missing state change or side effect
- Empty or default return
- Missing validation for zero, empty, nil, unauthorized, or malformed input

A mutation nothing catches marks the behavior as unprotected. A test that
no realistic mutation can fail is not protecting anything: delete it.

## Quick Reference

| When you... | Do |
|-------------|-----|
| Write any test | Name the break it catches — a bug, not a decision |
| Find the break is already caught | Don't write the test |
| Build an expected value | Derive it by hand; never with the code under test |
| Test a script, prompt or document | Run it / test its consumer's behavior; never grep its text |
| Reach for a dependency test | Test your boundary contract, not their documented mechanics |
| Want to assert on a mock call | Use a fake and assert the end state |
| Have several cases of one behavior | One parametrized test |
| Need a child process, converter, browser or wait | One real test per property; the rest fast |
| Touch a test file with not-worth tests | Delete them and list them in your report |
| Finish a test file | Run the mutation check, mentally |

## Warning Signs

- The test fails on every intentional change, never on accidental breakage
- The expected value is a copy of a constant, config or list in the code
- Setup and assertion share the same object, guaranteeing equality
- The test greps source text, or asserts a removed symbol stays removed
- The test would still matter if only the framework remained
- An assertion checks a mock, or fails only if you remove the mock
- Mock setup is more than half the test
- Many assertions on unrelated outcomes in one test
- Three test bodies that differ only in literals
- The test sleeps, or takes more than a second without saying why
- The test passes only on your machine
- The test exists for coverage, checking no side effect or outcome
