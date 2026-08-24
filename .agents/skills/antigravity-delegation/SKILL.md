---
name: antigravity-delegation
description: Delegate a bounded coding, code-review, debugging, architecture, or second-opinion task to Google Antigravity CLI through the dedicated Codex antigravity subagent. Use when an independent external-model perspective would materially improve confidence, catch mistakes, compare approaches, or perform an isolated implementation. Do not use for trivial tasks that Codex can complete and verify directly.
---

# Antigravity Delegation

Use the dedicated `antigravity` Codex subagent as the adapter between the main Codex thread and Google Antigravity CLI.

The main Codex agent remains the orchestrator and final decision-maker. Antigravity output is advisory until independently verified.

## Goals

Use this skill to:

- obtain an independent second opinion from an Antigravity-hosted model;
- review code, diffs, architecture, tests, or debugging hypotheses;
- compare an alternative implementation or design;
- delegate a bounded implementation when isolation is useful;
- keep verbose Antigravity interaction out of the main Codex context.

Do not delegate merely to create more agent activity. Delegation must have a concrete verification or independence benefit.

## Preferred Architecture

Use this flow:

1. Main Codex agent identifies a bounded task.
2. Main Codex agent delegates that task to the custom subagent named `antigravity`.
3. The `antigravity` subagent invokes `agy` in headless mode.
4. The subagent validates the Antigravity run and returns a concise evidence-focused report.
5. Main Codex independently inspects the relevant repository state, tests, logs, or diff.
6. Main Codex accepts, rejects, or modifies the recommendation.

Do not treat Antigravity as ground truth.

### Codex spawn contract

When spawning the custom adapter:

- use `agent_type: antigravity`;
- use `fork_turns: none` so the named custom agent can start with its own configuration;
- do not set the spawn tool's `model` field;
- pass `antigravity_model` only inside the work-order message because it selects the external `agy` model, not the Codex subagent model.

## Delegation Modes

Choose exactly one mode for each delegation.

### `consult`

Use for:

- architecture questions;
- debugging hypotheses;
- alternative approaches;
- design tradeoffs;
- planning;
- independent reasoning.

Default behavior: read-only. Do not ask Antigravity to modify files.

### `review`

Use for:

- code review;
- diff review;
- security review;
- test coverage review;
- bug-risk analysis;
- implementation critique.

Default behavior: read-only. The subagent may inspect repository files and git state but must not intentionally modify the workspace.

### `implement`

Use only when an external implementation is useful.

Examples:

- a bounded alternative implementation;
- an isolated fix to compare against Codex's implementation;
- a narrowly scoped refactor;
- generating a candidate patch for later review.

The parent task must explicitly state that file modifications are allowed.

After the subagent returns, the main Codex agent must inspect the resulting changes with `git status` and `git diff` before accepting them.

## What to Send to the Subagent

Give the `antigravity` subagent a self-contained work order containing:

- `mode`: `consult`, `review`, or `implement`;
- `objective`: the exact question or task;
- `scope`: relevant files, modules, services, or repository areas;
- `constraints`: compatibility, style, performance, API, security, or product constraints;
- `evidence`: relevant error messages, test failures, user requirements, or current implementation facts;
- `expected_output`: what the main agent needs back;
- `antigravity_model`: optional model slug; defaults to `gemini-3.7-flash-high`;
- `effort`: optional `low`, `medium`, or `high`; default to `high` for substantive technical work.

Do not forward hidden chain-of-thought or irrelevant conversation history. Send only the task context necessary for the external model to perform the work.

## Expected Subagent Return

The `antigravity` subagent should return a concise report with:

- `Status`
- `Mode`
- `Antigravity model` when known
- `Summary`
- `Findings`
- `Evidence`
- `Files changed` when applicable
- `Verification performed`
- `Risks / unresolved questions`
- `Conversation ID` when available

Prefer concrete file paths, symbols, commands, test names, and observed behavior over generic prose.

## Verification Gate

The main Codex agent owns final verification.

For review or consultation:

1. Check material claims against the repository or authoritative documentation.
2. Reproduce important bugs or behavior when practical.
3. Resolve disagreements using code, tests, logs, specifications, or other primary evidence.

For implementation:

1. Run `git status --short`.
2. Inspect `git diff`.
3. Confirm that changes remain inside the delegated scope.
4. Run the narrowest relevant tests first.
5. Run broader tests, linting, type checking, or build checks when justified.
6. Fix or reject changes that fail verification.

Never report an Antigravity claim as verified merely because the Antigravity run returned `SUCCESS`.

## Conflict Handling

If Codex and Antigravity disagree:

1. identify the exact disputed claim;
2. inspect primary evidence;
3. run a focused test or experiment when possible;
4. prefer reproducible evidence over either model's confidence;
5. state any remaining uncertainty.

For high-risk changes, prefer independent deterministic verification rather than asking the same model to validate its own work.

## Safety and Permissions

- Never use `--dangerously-skip-permissions` by default.
- Do not broaden Antigravity permissions merely to avoid a failed tool call.
- In `consult` and `review` modes, instruct Antigravity not to modify files.
- In `implement` mode, allow only the minimum workspace changes required by the task.
- Do not commit, push, merge, publish, deploy, delete branches, or alter remote state unless the parent task explicitly requires it.
- Preserve existing user changes. Do not reset or overwrite unrelated work.
- If authentication or permissions block Antigravity, report the blocker rather than bypassing controls.

## CLI Requirements

The `antigravity` subagent should use Antigravity headless mode.

Preferred single-run form:

```bash
agy -p "<TASK>" \
  --add-dir "$PWD" \
  --model "gemini-3.7-flash-high" \
  --output-format json \
  --effort high \
  --print-timeout 10m
```

When the parent specifies a different Antigravity model, replace the default slug:

```bash
agy -p "<TASK>" \
  --add-dir "$PWD" \
  --model "<MODEL_SLUG>" \
  --output-format json \
  --effort high \
  --print-timeout 10m
```

Interpret the run as successful only when:

- the process exits successfully; and
- the JSON envelope contains `"status": "SUCCESS"`; and
- `.response` is a non-empty string.

Use `.response` as the model response and retain `.conversation_id` for traceability when available.

Do not use streaming mode unless the task genuinely requires a multi-turn Antigravity conversation.

## Failure Handling

If the Antigravity run fails:

1. inspect the process exit status;
2. inspect `stderr`;
3. inspect the JSON `status` and `error` fields when present;
4. classify permission denials, `CANCELED`, and empty responses as `BLOCKED`;
5. retry only when the failure is plausibly transient or caused by a correctable invocation issue;
6. do not silently switch to `--dangerously-skip-permissions`;
7. return the failure clearly to the parent Codex agent.

If the custom `antigravity` subagent cannot be spawned but `agy` is available, the main Codex agent may use the same headless workflow directly as a fallback. Preserve the same verification rules.

## Efficiency Rules

- Keep each delegation bounded.
- Prefer one strong external review over several redundant calls.
- Do not delegate information the main agent can verify cheaply with a deterministic command.
- Use Antigravity for independence, alternative reasoning, or isolated execution—not as a substitute for tests.
- Summarize verbose external output before returning it to the main context.
