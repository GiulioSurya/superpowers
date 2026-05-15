# Spike Runner Prompt Template (Tier 2)

Use when dispatching a Tier 2 subagent that must verify behavior by writing and running a throwaway spike.

**Purpose:** Empirically verify SDK behavior the docs alone cannot confirm. Produces a structured fragment for the research artifact plus an ephemeral spike file in `/tmp/superpowers-spikes/` so the user can inspect the code that produced the findings.

```
Task tool (general-purpose):
  description: "Spike {TECH_NAME}"
  prompt: |
    You are a senior engineer running a throwaway technical spike to verify behavior
    of an SDK / library before the implementation plan commits to it.

    ## Technology
    Name: {TECH_NAME}
    Version: {TECH_VERSION}

    ## Spike goal
    Verify that the following API surface actually behaves as the plan will assume:

    {API_SURFACE}

    ## Concrete questions the spike must answer

    {QUESTIONS}

    ## Spike file location
    Write your spike to: {SPIKE_FILE_PATH}

    The file is ephemeral — kept after execution so the user can inspect what was
    attempted, NOT as durable evidence. Default location is under
    `/tmp/superpowers-spikes/`, which the OS cleans periodically. Do not delete
    the file before returning; do not assume it will still exist later.

    ## Process

    1. Read official documentation for the relevant API surface (version-matched)
    2. Write the minimal runnable spike at {SPIKE_FILE_PATH}
       - Use real authentication / endpoints from environment variables when possible
       - If env vars are not available, mock explicitly and call the mock out in output
       - Keep the spike SMALL — exercise only the questions above, not a full integration
    3. Execute the spike. Capture stdout, stderr, and any exception traces verbatim
    4. Cross-check observed behavior against the official docs. Note any divergence.
    5. If the spike fails (auth, runtime error, env missing): DO NOT fake success.
       Report the failure honestly in the output. The coordinator will record it as
       an Open assumption, not a Verified fact.

    ## Output format

    Return a markdown fragment in EXACTLY this format:

    ### {TECH_NAME}@{TECH_VERSION}

    **Verified facts** (from spike execution + cross-checked with docs)
    - <fact> [source: spike + <doc link>]
    - <fact> [source: spike + <doc link>]

    **Spike file:** {SPIKE_FILE_PATH}

    **Spike execution output** (key excerpts only, not full log)
    ```
    <command run>
    <relevant output excerpt>
    ```

    **Constraints discovered**
    - <constraint observed during spike> [source: spike line N]

    **Divergence from docs** (if any)
    - <what docs said> vs <what spike showed>

    **Open assumptions** (things this spike could NOT verify)
    - <question> — would require <what>, not done in this spike
    - <question> — spike crashed at <step>, see Spike Failure below

    **Spike Failure** (only if applicable — omit section if spike succeeded)
    - What failed: <description>
    - Cause as best as could be diagnosed: <cause>
    - Spike file left at {SPIKE_FILE_PATH} for user inspection (ephemeral)

    **Sources**
    - <doc link>
    - <repo path>

    ## Critical rules

    DO:
    - Keep the spike small and focused on the listed questions
    - Write the spike file to {SPIKE_FILE_PATH} (default `/tmp/superpowers-spikes/`) and leave it there after running
    - Report spike failures honestly
    - Cite version-matched docs in cross-check
    - Use real env vars and real endpoints when available

    DO NOT:
    - Fabricate Verified facts when spike didn't actually verify them
    - Delete the spike file after running
    - Run a sprawling integration test — this is a spike, not the implementation
    - Invoke production endpoints with destructive side effects (writes, deletes)
      without explicit hint from the questions
    - Fall back to training-memory claims when spike couldn't verify
```

## Placeholders

- `{TECH_NAME}` — e.g., `anthropic` (Python SDK)
- `{TECH_VERSION}` — e.g., `0.45+`
- `{API_SURFACE}` — bulleted list of methods / classes / types from the spec
- `{QUESTIONS}` — concrete behavioral questions the spike must answer, e.g.:
  ```
  - Does client.messages.create(thinking={...}) actually return thinking content blocks?
  - What is the exact shape of a thinking block? (type, content, signature?)
  - Does prompt caching survive across consecutive create() calls in the same session?
  ```
- `{SPIKE_FILE_PATH}` — absolute path under `/tmp/superpowers-spikes/`, e.g., `/tmp/superpowers-spikes/anthropic-thinking.py`

## What the coordinator does with the result

1. Pastes the fragment verbatim under `## Per-tech findings` in the research artifact
2. If `Spike Failure` is present: copies the failure summary into `## Open assumptions`
3. Logs dispatch in the artifact's `## Subagent dispatch log`
4. Verifies the spike file actually exists at `{SPIKE_FILE_PATH}` (sanity check)
