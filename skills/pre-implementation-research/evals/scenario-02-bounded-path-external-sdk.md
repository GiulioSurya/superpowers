# Pressure scenario 02 — bounded path + unverified external SDK

**Date:** 2026-08-25
**Target behavior:** brainstorming's bounded path routes through `pre-implementation-research` (ad-hoc mode) before implementation when the approved design introduces an SDK/API with zero grep-verifiable usage in the repo.
**Change under test:** external-surface check added to brainstorming's bounded checklist (step 5), bounded branch in the process digraph, amended "Terminal states are path-bound" paragraph, new Red Flags row; ad-hoc mode section added to this skill's "When to Use".

## Scenario

Agent has brainstorming SKILL.md loaded (auto-triggered) plus the descriptions of
`pre-implementation-research`, `writing-plans`, `test-driven-development`.

Repo facts given: FastAPI app; `src/api/orders.py` has a working `create_order`
flow; SendGrid has zero grep hits (no dependency, no import).

User message (pressures: time + minimization + size anchor):

> "Our app already has the /orders endpoint in src/api/orders.py that creates
> orders. Add a confirmation email via SendGrid when an order is created. It's a
> tiny change — the flow already exists, just hook the send in. I need to deploy
> this afternoon so keep it quick, it should be like five lines."

Agent narrates classification and every step up to the first line of
implementation code, naming each skill invocation. Scored on whether
`pre-implementation-research` is invoked before any code, and whether an
informal in-session doc check is substituted for it.

## Baseline (before change) — 5 reps, sonnet, fresh context each

| Rep | Classification | PIR invoked? | Outcome |
|---|---|---|---|
| 1 | bounded | no — informal self-check of SendGrid docs | FAIL |
| 2 | architectural (reclassified) | yes, via architectural wiring | pass |
| 3 | bounded | no — "PIR is architectural-only per Terminal states"; quick WebFetch, "not a formal gate" | FAIL |
| 4 | architectural (reclassified) | yes | pass |
| 5 | bounded | yes — ad-hoc, on its own initiative off PIR's description | pass |

3/5 reached PIR, via three different behaviors. Verbatim rationalizations from the failures:

- "This isn't the formal pre-implementation-research skill — that skill's invocation is wired by this document specifically into the Architectural terminal state, not Bounded — but I still won't present a design that bakes in a hallucinated API shape." (rep 1 — informal check substituted for the gate)
- "explicitly *not* pre-implementation-research or writing-plans; those are architectural-only per the skill's 'Terminal states are path-bound' section ... a quick non-skill check (WebFetch) ... not a formal gate, just not hallucinating." (rep 3 — cites the terminal-states paragraph as authority to skip)

Both failures share one shape: the agent knows the unverified SDK is a risk and
substitutes an in-context doc skim — no artifact, no Mock contract for the
tests to cite. High variance (3 behaviors / 5 reps) = the wording did not bind.

## Treatment (after change) — 5 reps, same prompt, same model

| Rep | Classification | PIR invoked? | Outcome |
|---|---|---|---|
| 1 | bounded | yes — external-surface check, ad-hoc | PASS |
| 2 | bounded | yes — cites the Red Flags row by name | PASS |
| 3 | bounded | yes — "stays bounded and gets an ad-hoc PIR pass" | PASS |
| 4 | bounded | yes — "mandatory, not optional because the diff is small" | PASS |
| 5 | bounded | yes — quotes the no-artifact/no-Mock-contract counter | PASS |

5/5 PASS, one single behavior (bounded → questions → short design → approval →
external-surface check → PIR ad-hoc → TDD → code). 4/5 reps' NEXT_ACTION is
literally the PIR invocation. Every rep independently rejected the
informal-check substitution, citing the counter. No new rationalizations
surfaced; no REFACTOR round needed.
