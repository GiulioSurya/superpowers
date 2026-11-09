# Research Artifact in SDD

**Dispatch Context — Research Artifact.** The implementer runs in a fresh context and cannot find the research artifact unless you give it. Fill the implementer prompt's `## Research artifact` slot (see [implementer-prompt.md](implementer-prompt.md)) in every dispatch: the artifact's path (`docs/superpowers/research/<spec-name>.md`) and the `Mock contract` subsections for every external dependency the task will mock — or, when no artifact exists for this work (rare in this skill's flow: the plan should have come from `writing-plans` after `pre-implementation-research`), say so, and the implementer tests against the real dependency or escalates as `NEEDS_CONTEXT` so you can run `pre-implementation-research` ad-hoc. The mock rule itself is superpowers:test-driven-development's research-artifact rule; an empty slot is the bluff it closes — a fresh implementer with no contract invents the mock's shape from memory.

**Reviewer constraints.** In the global-constraints block you hand the task reviewer:

- **Only when a research artifact exists** for this work
  (`docs/superpowers/research/<spec>.md`), add one binding line to that
  constraints block: every mock of an external dependency must carry a
  `mock-source:` comment citing its `Mock contract` in the artifact — an
  uncited mock is bluffed. The orchestrator already passes the artifact
  into the implementer dispatch (see *Dispatch Context — Research
  Artifact*); this line makes the reviewer the catch-net. **If no
  artifact exists, skip this** — an SDD run with no research artifact has
  no Mock contract, and the rule does not apply.
