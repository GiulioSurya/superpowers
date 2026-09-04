---
name: requesting-plan-review
description: Use when an implementation plan is written and before it is executed (subagent-driven-development or executing-plans), when a plan is rewritten after its spec changed, or when your human partner asks for a plan review, a review panel on the plan, or a lean-up pass. Not for specs (requesting-spec-review) and not for code diffs (requesting-code-review).
---

# Requesting a plan review

Five read-only Opus lenses read the plan against the spec and the real code, in parallel, each blind to the
others. The loop — dispatch, consolidate, next round, stop — is the one in superpowers:requesting-spec-review;
this skill states only what changes when the document is a plan.

**REQUIRED BACKGROUND:** superpowers:requesting-spec-review. Read it first. Everything it says about the mechanism
(the lenses are agents; never pass `model`; never paste a lens), what the loop produces, the consolidation order,
the round-2 message and the stop reasons applies unchanged.

**Core principle:** a plan fails differently from a spec. A spec fails on mechanisms and decisions; a plan fails on
text that does not compile, tests that cannot pass, and fixtures that do not exist. "Compiles but does not run"
has been the dominant class in every plan round so far.

## The fifth lens

| Lens | Agent | Looks for |
|---|---|---|
| Ponytail | `panel-ponytail` | reinvented stdlib, speculative abstractions, duplicated helpers, tests that test the framework |

Its ruleset is the real `ponytail-review` skill, injected into the agent by the `skills:` line of its frontmatter
— never a paraphrase written by you. Two house rules travel with it: every "reuse X instead" cites the existing
thing as `repo/file:line`; a lean-up that touches a mechanism the spec mandates is rejected by citing the spec
section, and the lens says so under "Rejected lean-ups" instead of proposing it.

All five run on every plan. Ponytail and security have had zero overlap with the other three: each found a class
of its own.

## 1. Gather — the author's self-check for a plan

What `sed` on every anchor is for a spec, **running the literal text** is for a plan:
- Extract every code block and hand it to the compiler or parser it claims to be for: `ast.parse` or `py_compile`
  for Python, `esbuild` or `tsc` for TypeScript, `swiftc -typecheck` for Swift. Count the blocks you extracted
  against the blocks in the file — a block introduced by prose instead of a path escapes a naive regex.
- Where the Python blocks form a package, run them with `pytest` in a scratch directory. A typecheck does not
  catch a fixture with wrong arithmetic, an epoch constant a year off, a `now` injected but ignored, or a fake
  that patches the wrong module; executing the tests does.
- Every file marked "new": `ls` it. A plan said "new" of a test file that already existed; four lenses reported it.
- Every "delete lines a-b": `sed -n` on a-1 through b+1.
- Every "Expected:" of a DB check: run the check first. One was written against a tenant with zero rows.
- Every expected number in a test: recompute it yourself before the panel does.
- Every test: read it asking "can this fail?" — cleanup, double render, a mock that returns what the assertion
  checks.
- The plan's verification commands against the spec's rollout section, verbatim. Plans narrow the final
  verification ("13 files" called "the full suite"), and a bare `npm run test` may launch integration projects
  against a shared database.
- Every mechanism the plan adds that the spec did not name gets the spec loop's question: who reads it, and on
  which path is its value undefined.

The plan ends with **claims to attack hardest**. The ones that paid: a task order that must be executable
red-first without a broken intermediate state; a closure that must have a name in scope as written; an upsert
against an index rather than a constraint; a redundant loop you kept because it was correct; the sweep of
intentional test breaks against `grep` reality.

## 2. Dispatch — what the text carries for a plan

The four parts of the spec dispatch, plus three:

5. **The spec's path and closing revision** — ponytail cites its sections to reject a lean-up, quality checks the
   plan's verification against its rollout, critical hunts silent deviations from it.
6. **A scratch directory and the permission to execute there**, for the adversarial and quality lenses: write the
   plan's code blocks to it and run them; nothing is written anywhere else. In the runs that granted it, the
   lenses with a mandate to execute found every real blocker.
7. **Parts written at contract level**, if any (another engineer will detail them): say so, and that they are
   reviewed for consistency with the spec, not for missing literal code. Otherwise the quality lens spends its
   budget there.

Same text for all five agents, one message, no `model` parameter.

## 3. Consolidate — what changes for a plan

The order is the spec loop's. Three rules earned on plans:
- **The file wins, not the lens.** A lens reported `243-245` where the file says `242-244`. The Iron Law applies to
  ranges as it does to lines: `sed` before folding.
- **The repo's own comment beats the generic reuse rule.** Two ponytail lean-ups contradicted a convention
  declared in the files themselves and were rejected with the citation. Read the file around a lean-up's target
  before accepting it.
- **Accept a scalability fold for its scenario, not its recipe.** "A light projection for list rows" would have
  broken the cards' headers; a gate that skips the query while the panel is closed closed the same scenario at
  zero cost.

A plan's consolidation is often a rewrite rather than patches. A full rewrite is not riskier than patches, but
every mechanism it adds gets the §1 bar before the next round. Make substitutions exact and checked — assert that
each one applied, report the count — then re-run the §1 extraction and parse on the whole revision: a fold that
breaks a block is invisible until the implementer hits it.

Ponytail findings fold by the same bar as any other: read the existing thing it cites, then decide. Two ponytail
reuses have closed findings of other lenses (a shim renamed to the existing helper; a refresh routed through the
existing token manager); one lean-up introduced a latent throw and was caught by two other lenses converging.

## 4. Rounds and stop

Same messages, same stop reasons. Plans converge faster than specs — the author arrives calibrated by the spec
loop — and your human partner may set the cap at one round; name the cap you closed under. Residual MINOR/NIT
become notes inside the plan's tasks. Then superpowers:subagent-driven-development or
superpowers:executing-plans.

Five lenses on a plan's first round cost as much as a spec's first round: in the order of 1.1-1.7M agent tokens
and 10-17 minutes of wall clock.

## Red flags

- A plan block you did not parse before the panel.
- "The tests are in the plan, they will pass."
- A ponytail lens whose ruleset came from your own words.
- Folding a lean-up without reading the file it targets.
- A range folded without `sed` on its neighbours.
- A verification command in the plan narrower than the spec's rollout.

## Reference files

- `agents/panel-ponytail.md` — the fifth lens, at the plugin root; its ruleset arrives through
  `skills: [ponytail-review]`, its persona is vendored verbatim in the file.
- `agents/panel-adversarial.md`, `agents/panel-critical.md`, `agents/panel-quality-scalability.md`,
  `agents/panel-security-tenancy.md` — shared with the spec loop.
