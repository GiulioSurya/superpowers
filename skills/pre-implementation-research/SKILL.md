---
name: pre-implementation-research
description: Use when specs/design are approved, before writing the implementation plan, and the implementation will rely on SDK/library/API code (types, methods, classes, response shapes) not recently verified against authoritative sources
---

# Pre-Implementation Research

## Overview

Bridge between approved specs and `superpowers:writing-plans`. Forces explicit verification of every SDK/library/API surface the plan will touch, **delegated to subagents** to keep coordinator context clean. Output is a persistent markdown artifact passed to downstream subagents.

**Core principle:** Agents bluff what they don't know. The discipline is making unknowns explicit and verifying them against authoritative sources before the plan commits to them.

## When to Use

**Trigger:** Spec exists in `docs/superpowers/specs/`, about to invoke `superpowers:writing-plans`, and the implementation will use SDK/library/API code not recently verified.

**Don't use when:**
- All tech in spec is already used in the current codebase (grep-verifiable)
- No external SDK/API involvement (pure internal refactor)
- The spec itself is a research document

## The Iron Rule

```
NO PLAN WITHOUT THE RESEARCH ARTIFACT
NO SUBAGENT DISPATCH WITHOUT USER-APPROVED TECH INVENTORY
```

`writing-plans` cannot start until the artifact at `docs/superpowers/research/<spec-name>.md` exists with every spec-named tech classified. T1/T2 subagents cannot be dispatched until the user has seen and explicitly approved the tech inventory + classification table.

**No exceptions:**
- Don't classify a tech as T0 without a verifiable source citation
- Don't skip subagent dispatch and verify directly in coordinator context
- Don't leave `Open assumptions` empty without explicit "verified all" justification per spec mention
- Don't dispatch any T1/T2 subagent before presenting the tech inventory to the user and receiving explicit approval — this is a hard gate, not a courtesy update

## Workflow

1. **Detect paths**
   - Spec: `docs/superpowers/specs/<name>.md`
   - Artifact: `docs/superpowers/research/<name>.md` (create directory if missing)
   - Spike files: `.claude/spikes/<tech-slug>.<ext>` (create directory if missing)

2. **Build tech inventory** — extract every SDK / library / API / type / method / class / response shape named in the spec.

3. **Classify each tech** into T0 / T1 / T2 (table below). Classification justification must be explicit and verifiable.

4. **Present the tech inventory to the user — MANDATORY GATE.** Output the full inventory + classification table directly in the conversation, in markdown, BEFORE dispatching any subagent. Include for every entry: tech name, exact version, tier, justification (citation for T0, reason for T1/T2). Then explicitly ask the user to confirm or correct: (a) completeness of the tech list against the spec, (b) version pins, (c) tier assignments. Wait for explicit approval. Do NOT proceed to step 5 on coordinator initiative — even if the inventory looks obvious. The user is the only reader who can catch missing tech and misclassifications before they propagate into wasted subagent runs and a bluffed plan.

5. **Dispatch subagents in parallel** for T1 and T2 — single message, multiple Task tool calls. T0 entries are inlined directly with citation, no subagent needed.
   - T1 → use `./doc-verifier-prompt.md`
   - T2 → use `./spike-runner-prompt.md`

6. **Aggregate fragments** into the artifact (template below). Coordinator never reads raw docs or spike code — only structured fragments returned by subagents.

7. **Apply spec amendments** (if any). Research findings often contradict assumptions baked into the spec. If the artifact's `## Required spec amendments` section is non-empty, patch the spec file NOW — before `writing-plans`. See "Spec Amendments After Research" section below.

8. **Hand off** — pass the artifact path in the dispatch context of `writing-plans`. The spec at this point already reflects research findings; `writing-plans` reads only the spec.

## Tier Classification

| Tier | When to assign | Action |
|---|---|---|
| **T0** | Stdlib OR already used in current codebase | Inline annotation: `verified at <file>:<line> on <YYYY-MM-DD>`. Citation REQUIRED. No subagent. |
| **T1** | Stable API, official docs clear, no edge-case behavior needed | Dispatch doc-verifier subagent. Doc reading only, NO code execution. |
| **T2** | Recent version / known breaking changes / undocumented behavior / composite signature / agent uncertainty about return shape | Dispatch spike-runner subagent. Persistent spike file required. |

**Disambiguation:** "I know this SDK from training" is NEVER a T0 justification. T0 requires `git grep` evidence in the current repo with file path and line number.

**Version requirement:** Every entry in the inventory MUST specify an exact version, version range, or pinned commit (e.g., `0.45+`, `>=1.2,<2.0`, `pinned at 1.18.3`). "Latest" is forbidden — different agents at different times resolve it differently, breaking reproducibility and obscuring breaking-change risk.

## Artifact Structure

```markdown
# Pre-implementation research: <topic>
Spec: docs/superpowers/specs/<name>.md
Date: YYYY-MM-DD

## Tech inventory & classification
| Technology | Version | Tier | Justification |
|---|---|---|---|
| <name> | <version> | T0/T1/T2 | <citation or reason> |

## Per-tech findings
[T0 entries: inline one-liner with citation]
[T1/T2 entries: fragment returned by subagent — paste verbatim]

## Open assumptions
Things not verified, fragile in the downstream plan. Explicit list — never empty unless every spec mention is covered above.

## Out of scope
What was deliberately not researched, with reason.

## Required spec amendments
Concrete edits to apply to the spec BEFORE writing-plans is invoked.
If the section is empty: explicit statement `_no amendments — spec assumptions held against research_`.
Otherwise every amendment MUST use this exact structure (no shortcuts, no free-form prose):

### Amendment <N> — <file/area touched + gist of the change in one line>

**Spec section:** <breadcrumbs inside the spec, e.g. "Componenti → Nuovi → upload.py → Logica → punto 6">

**Current text (in spec):**
> <verbatim quote from the current spec — multi-line allowed; preserve formatting>

**Issue:** <what's wrong, anchored to the Verified facts above; cite the per-tech finding line when possible>

**Replacement text:**
> <verbatim replacement, complete enough to be applied without rewriting; code fences allowed inside the blockquote>

**Status:** Applied at <YYYY-MM-DD HH:MM> — <commit SHA or "spec patch in commit (next)">

## Subagent dispatch log
- <YYYY-MM-DD HH:MM> <tech>@<version> (Tier) → result/duration
```

The sections passed to downstream subagents (in `writing-plans` and `executing-plans` dispatch context) are: **Verified facts**, **Constraints**, **Open assumptions**.

## Spec Amendments After Research

The research artifact is **evidence**. The spec is the **single source of truth** that downstream skills (`writing-plans`, `executing-plans`) read. When research findings contradict spec assumptions, the spec must be patched — keeping two contradictory documents alive forces every downstream reader to reconcile, and most readers will follow the spec (broken plan) or ignore it (broken document chain).

**Required workflow (step 7):**

1. The artifact's `## Required spec amendments` section MUST list every place the spec contradicts research findings, structured per the template in "Artifact Structure" above (numbered amendments with breadcrumbs, current/replacement blockquotes, issue anchored to verified facts, status). Free-form prose is not acceptable.
2. The coordinator (or a dedicated subagent dispatched for this purpose) applies those amendments directly to the spec file. **Every patched section in the spec MUST carry an HTML comment back-reference, inserted immediately above the patched block:**
   `<!-- amended per docs/superpowers/research/<name>.md amendment <N> on <YYYY-MM-DD> -->`
   This makes the chain auditable in both directions: a reader of the spec sees why a section changed and where the evidence lives; a reader of the artifact can grep the spec for `research/` and confirm every amendment landed. Without the back-reference, the spec patch looks like a free rewrite — exactly the bluff signature this skill exists to prevent.
3. Each applied amendment is marked in the artifact: `Applied at <YYYY-MM-DD HH:MM>` (or commit SHA if version controlled).
4. If the section is genuinely empty, the artifact must say so explicitly: `_no amendments — spec assumptions held against research_`. Empty without a statement is a workflow gap.

**Why this matters:** if the spec disagrees with verified reality at hand-off time, the plan author has two contradictory inputs. They will either (a) follow the spec and write a plan based on bluffs, (b) ignore the spec and break the document chain, or (c) burn time reconciling. Patching the spec eliminates the divergence at its source. Single source of truth survives.

**This is not optional.** A research run that exposes spec contradictions and stops at "wrote them down in the artifact" has only done half the job. The skill is incomplete until the spec reflects what was learned.

## Failed Spike Handling

If a T2 spike fails (auth missing, runtime error, env not available):
- The tech entry's `Verified facts` stays empty
- The failure (with cause) goes into `Open assumptions`
- Workflow does NOT block — the plan downstream is informed of the fragility
- The spike file is kept for re-run by future agents

## Red Flags

| Symptom | Reality |
|---|---|
| "I'll just read the docs myself, it's faster" | Coordinator-side research = context pollution. Dispatch the subagent. |
| "This SDK is well-known, T0" | T0 requires a `<file>:<line>` citation in this codebase. Otherwise T1. |
| "Open assumptions is empty because I verified everything" | Then explicitly map every spec mention to a Verified fact entry. |
| "Spike failed but I'll figure it out during planning" | Record it as an Open assumption now. The plan must commit to fragility deliberately, not by accident. |
| "I'll skip this and let writing-plans handle research" | writing-plans does not do research. Without the artifact, the plan is grounded in agent guesses. |
| Plan lists 10+ specific signatures with no `[source: ...]` markers | This is the bluff signature. Verified facts MUST cite source. Mixed verified/unverified content with uniform confidence is the exact failure mode this skill exists to prevent. |
| Tech inventory says "latest" or omits version | "Latest" is a moving target. Breaking changes between versions are the #1 cause of drift. Every entry needs an exact version, range, or pinned commit. |
| Artifact has `Required spec amendments` items but the spec file wasn't patched | Workflow step 6 was skipped. `writing-plans` downstream will read a spec that contradicts research → broken plan from bluffed assumptions. Apply the amendments to the spec NOW, then mark them `Applied at <timestamp>` in the artifact. |
| "I'll let writing-plans figure out the contradictions between spec and artifact" | `writing-plans` reads the spec, not the artifact. It cannot reconcile silently. The contradiction must be resolved at the spec, not deferred. |
| "The inventory is obvious, I'll dispatch subagents directly" | The inventory MUST be shown to the user before any T1/T2 dispatch (workflow step 4). Misclassifications and missing tech are the most common failure mode and only the user can catch them — the agent cannot self-audit its own blind spots. Skipping the gate burns subagent runs on the wrong scope and lets bluffs survive into the artifact. |
| "I already mentioned the techs in passing while planning" | A passing mention is not the inventory gate. Step 4 requires the structured table (tech / version / tier / justification) presented as a discrete checkpoint with an explicit ask for approval. |
| "I patched the spec, the amendment work is done" | Patching without the `<!-- amended per docs/superpowers/research/<name>.md amendment <N> on <YYYY-MM-DD> -->` back-reference is half the job. Future readers can't audit the spec change against the artifact, and `writing-plans` reads a spec that looks freely rewritten. The back-reference is mandatory, not stylistic. |
| "Free-form prose for the amendment is fine, the gist is clear" | The structured template (numbered amendment, breadcrumbs, current/replacement blockquotes, issue, status) exists because free-form amendments routinely lose the verbatim "current text" or skip the spec section reference, and downstream readers can't apply them mechanically. Use the template verbatim. |

## Integration

- **Precedes:** `superpowers:writing-plans` (the plan reads the artifact)
- **Follows:** spec approval (manual or via `superpowers:brainstorming`)
- **Consumed by:** `superpowers:executing-plans` and `superpowers:subagent-driven-development` — implementer subagents receive `Verified facts` + `Constraints` + `Open assumptions` in their dispatch context

## Templates

- `./doc-verifier-prompt.md` — T1 subagent (doc-only verification)
- `./spike-runner-prompt.md` — T2 subagent (runnable spike with persistent file)
