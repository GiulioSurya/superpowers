---
name: requesting-spec-review
description: Use when a spec or design document is drafted and about to go to planning or implementation, when a spec changes or removes existing behavior, or when your human partner asks for a spec review, a review panel, or a lens review. Not for code diffs (requesting-code-review) and not for implementation plans (requesting-plan-review).
---

# Requesting a spec review

Four read-only Opus lenses read the spec against the real code, in parallel, each blind to the others. You are
the consolidator: you gather what they need, dispatch them, validate what comes back, fold it, and name the stop
reason.

**Core principle:** a lens sees a file, never the intent. Two lenses agreeing is a strong signal, not a proof —
you own the verdict.

```
NO FINDING FOLDED WITHOUT READING THE LINE IT CITES
```

A lens that quotes `foo.py:243-245` may be off by one, and the fold then writes a wrong anchor into the spec that
every later round inherits. `sed -n` costs one tool call. Do it for every anchor you fold, including the ones you
are sure about.

## The lenses are agents, not prompts

Each lens lives in `agents/panel-<lens>.md` at the plugin root. The body of that file is the lens's system prompt;
Claude Code loads it when you dispatch `Agent(subagent_type: "panel-<lens>", prompt: ...)`. The lens text never
enters your context and you never write it: the only thing you write is the dispatch text in §2.

| Lens | Agent | Looks for |
|---|---|---|
| Adversarial | `panel-adversarial` | the input, race or sequence that makes a mechanism produce a wrong outcome |
| Critical | `panel-critical` | wrong or missing decisions, silent reversals, cheaper alternatives |
| Quality + scalability | `panel-quality-scalability` | anchors, symbols, test breaks, callers; load at 10×, growth, god-files |
| Security / tenancy | `panel-security-tenancy` | RLS as it would ship, cross-tenant reach, secrets, service-role writes |

Run all four when the spec touches a database or an edge function; the first three otherwise.

Two facts about the mechanism, each learned at the cost of a round:
- **Do not pass `model` to `Agent`.** The agent file pins `opus`; the tool's `model` parameter overrides the file
  silently, and a demoted lens still returns a well-formed verdict — the loss shows only as findings that were
  never made. This is the one case where "always pass the model explicitly" does not apply: the agent file is the
  explicit model.
- **Edit a lens in its agent file, never in a dispatch.** Text pasted into a prompt is a second copy that drifts
  from the first, and costs the panel's worth of tokens in your own context for no change in the result.

## What the loop produces

Exactly two things: revisions of the spec, and messages to the lenses. The resolution table is the body of the
next message; deferred suggestions go in a "Deferred to implementation" section of the spec; the stop reason is a
line in your reply. No brief files, no round results, no calibration notes, no document about the review.

## 1. Gather — before the first dispatch

**Measure before writing rev.1.** DB counts, `grep` blast radius, `sed` on every anchor the spec cites. The numbers
go into the spec's evidence section, where the lenses and the implementer both read them. Evidence that arrives
after rev.1 reverses decisions already written and costs a full round.

**Then self-check the draft** against the classes that keep coming back:
- Every query you reuse: re-read its filters line by line. A wrong filter is invisible to the lenses too — they
  check what the spec says, not what you meant.
- Every new field or flag: name its producer and its consumer in the same sentence.
- Every closed list ("the three callers are…"): sweep the repo by shape, not by the names you remember.
- Every new mechanism: who reads it, and on which paths is its value undefined? Checking that a symbol exists
  catches none of this — that check took 0 of 12 MAJOR in one round.
- Every file marked "new": `ls` it.
- Every open question that two greps would close: close it yourself. Left to the panel, three lenses pay for the
  same answer.

**The spec ends with "claims to attack hardest"** — the 3 to 6 places where you know you took a risk. The lenses
start there.

## 2. Dispatch — round 1

Four `Agent` calls in one message, same dispatch text for all. The text has four parts and nothing else:

1. **The spec** — path, revision, and the repos it touches, each on its branch.
2. **The files this wave lives in**, with approximate anchors: `legis_ai/core/agent/agent.py ~L47`. Approximate
   is enough — it removes the search, not the verification, and took first-round tool calls from 45-65 to 30-38
   per lens.
3. **The claims to attack hardest** — the spec's own list, numbered.
4. **Read-only DB access** for the adversarial and security lenses: project id and the exact MCP tool selector,
   SELECT only.

If the spec is a delta on a revision already shipped in part: name the commit that is the delta, and say the
untouched part is out of scope and does not drive the verdict.

Scope, decisions, evidence and rationale are in the spec, and the lens reads the spec. Restating them in the
dispatch buys nothing and starts a second source of truth that drifts from the first.

## 3. Consolidate — in this order

1. **Read every "Optional (not required)" section first.** They are quarantined proposals, not noise: one loop's
   best fix was sitting there, read after the fold.
2. **Validate every finding yourself** on the code or the DB: `sed -n` the anchor, read the gate that would
   prevent the claimed behaviour. FACT beats INFERENCE; a lens that measured beats a lens that reasoned. Two
   agreeing lenses have been contradicted by a third with evidence — the tag decides, not the count.
3. **Settle conflicts** without a third round: on what the spec says, the spec verbatim decides; on a fact of
   code, the lens that verified wins; when two lenses propose different fixes, choose one and say why in the
   spec's revision log — or pick the design that makes the disputed fact irrelevant.
4. **Promote** any Optional whose effect lands on a shared database or a deploy: it is a finding wearing the wrong
   label.
5. **Fold deletions and requested fixes. Nothing else.** Every spontaneous addition made during a consolidation
   has produced a finding in the next round. For each fix, first ask what already exists that does this; if the
   fix adds a mechanism, run the §1 self-check on it before writing it — round 2 exists to catch exactly these.
6. **Re-read the whole revision yourself.** Each lens sees its slice; the defect introduced by your own
   consolidation is visible only to you.
7. **`git status --short`** in every repo the lenses touched. Their tools deny edits; the check is one call.
8. Keep **one resolution line per finding** — finding → what you did → where. That is the body of the next
   message, not a file.

## 4. The next round

Same agents, via `SendMessage` (their names are in the `Agent` results; `ListAgents` lists them). They keep their
context and re-verify cumulatively, which gets stricter, not looser. An interruption during a `SendMessage` kills
the agent for good; a fresh dispatch works if the message is self-contained.

**Full round** to every lens that returned `NOT OK`. The message has these parts, in order:
1. The revision's commit range — `git diff <prev>..<new>` is exactly what changed.
2. The resolution lines for that lens's findings.
3. Decisions your human partner settled between rounds — attack the implementation, not the choice.
4. The mandate: a resolution table (RESOLVED / PARTIAL / NOT ADDRESSED, with the anchor that settles it, verified
   on the file); regressions from the consolidation — every mechanism the author added gets the same bar as
   original content; new claims only where the revision created them; **review the other lenses' fixes too** — a
   tenancy regression born from an adversarial fix is found by nobody otherwise.
5. Budget: about 40 tool calls.

**Delta check** to every lens that returned `OK`, and to all lenses on the last round: parts 1 and 2, then: check
your own findings, the root change, and genuinely new claims; do not re-read the spec end to end; about 15 tool
calls; `VERDICT: OK` with the resolution table is the expected outcome, not a failure; honest severities — MINOR
and NIT alone are an OK.

If a lens returns findings without severities, ask for them in the message rather than ranking them yourself.

Do not ask your human partner for confirmation between rounds. The loop converges on its own; a check-in
mid-loop interrupts it without adding information.

## 5. Stop — name the reason

| Reason | Condition | Then |
|---|---|---|
| `STOP-OK` | every lens `VERDICT: OK` in the same round or at the delta | spec approved |
| `STOP-CONVERGE` | at the delta every residual MAJOR is convergent and fixes in one line | fold, no further round |
| `STOP-STALL` | a round adds no BLOCKER or MAJOR and ≥80% of what came back is NIT, anchors, or restatements | fold, declare convergence |
| `STOP-CAP` | hard cap: 2 full rounds + 1 delta | residual BLOCKER/MAJOR → your human partner; MINOR/NIT → tasks in the plan |
| `STOP-CONFLICT` | two lenses contradict each other on the same finding for two rounds | the only mid-loop question to your human partner |

State the reason and the round numbers in your reply. Then `superpowers:writing-plans`.

Round 1 finds structural errors; round 2 finds the regressions your consolidation introduced; round 3 returns
anchoring nits. A spec that changes or removes existing behaviour earns the full loop; a purely additive one earns
one round. Four lenses on a full loop cost in the order of a million agent tokens and 15-30 minutes of wall clock.

## Red flags

- Reading `agents/panel-*.md` to write a dispatch.
- Passing `model` in an `Agent` call.
- Folding an anchor you did not `sed`.
- "Two lenses agree, that settles it."
- "While I'm in there…" during a consolidation.
- Archiving an Optional section unread.
- A brief, results or calibration file for this round.
- Closing because the findings got small, without naming a stop reason.
- Measuring after rev.1.

## Reference files

- `agents/panel-adversarial.md`, `agents/panel-critical.md`, `agents/panel-quality-scalability.md`,
  `agents/panel-security-tenancy.md` — the lens system prompts, at the plugin root. Edit a lens there; the change
  is live at the next dispatch.
