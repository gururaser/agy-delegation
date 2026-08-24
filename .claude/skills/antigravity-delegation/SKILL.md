---
name: antigravity-delegation
description: Delegates a bounded coding, review, debugging, architecture, or second-opinion task to Google Antigravity CLI in an isolated custom subagent. Use when an independent external-model perspective would improve confidence, catch mistakes, compare approaches, or produce a candidate implementation. Invoke with a self-contained work order as arguments.
context: fork
agent: antigravity
---

# Antigravity Delegation

Execute the following work order through Google Antigravity CLI:

$ARGUMENTS

## Work-order interpretation

Determine the mode from the work order:

- `consult`: independent reasoning, debugging hypotheses, architecture, planning, or tradeoffs.
- `review`: code, diff, test, security, or implementation review.
- `implement`: a bounded implementation explicitly allowed to modify the workspace.

If the caller does not specify a mode, use `consult`.

The parent Claude Code conversation remains the orchestrator and final decision-maker. Antigravity output is advisory until independently verified by the parent.

## Delegation requirements

1. Convert the work order into a precise Antigravity prompt.
2. Keep the delegated scope bounded.
3. Use Antigravity headless mode with machine-readable JSON output.
4. Validate both the process result and Antigravity's JSON status.
5. Return a concise evidence-focused report rather than the full raw response.
6. Never treat Antigravity's answer as ground truth.

## Mode rules

### `consult`

- Read-only.
- Antigravity may inspect relevant repository files.
- Do not intentionally modify workspace files.
- Ask for concrete conclusions, alternatives, assumptions, and evidence.

### `review`

- Read-only.
- Ask Antigravity to cite concrete file paths, symbols, tests, or behaviors.
- Prioritize correctness, security, regression risk, and missing verification over style.
- Do not intentionally modify workspace files.

### `implement`

Use only if the work order explicitly allows changes.

- Restrict changes to the stated scope.
- Do not commit, push, merge, publish, deploy, or alter remote state.
- Preserve unrelated user changes.
- After Antigravity finishes, inspect `git status --short` and `git diff`.
- Report every observed changed file to the parent.

## Antigravity CLI

Preferred single-run form:

```bash
agy -p "<PROMPT>" \
  --add-dir "$PWD" \
  --model "gemini-3.7-flash-high" \
  --output-format json \
  --effort high \
  --print-timeout 10m
```

If the work order specifies a different `antigravity_model`, replace the default slug:

```bash
agy -p "<PROMPT>" \
  --add-dir "$PWD" \
  --model "<MODEL_SLUG>" \
  --output-format json \
  --effort high \
  --print-timeout 10m
```

If the work order specifies `effort`, use `low`, `medium`, or `high` accordingly.

Default `antigravity_model` to `gemini-3.7-flash-high`. Available model slugs can be inspected with `agy models`.

Never use `--dangerously-skip-permissions` by default. A permission failure is a blocker to report, not a reason to bypass controls.

Do not use `stream-json` unless the work order genuinely requires a multi-turn Antigravity conversation.

## Antigravity prompt shape

Build a self-contained prompt with:

```text
ROLE
You are an independent coding agent providing work to another coding agent that will verify your result.

MODE
<consult | review | implement>

OBJECTIVE
<exact delegated task>

SCOPE
<relevant files, modules, services, or repository areas>

CONSTRAINTS
<technical, compatibility, safety, performance, API, or product constraints>

EVIDENCE
<errors, failing tests, observed behavior, requirements>

REQUIRED OUTPUT
- concise conclusion;
- concrete findings;
- file paths and symbols when relevant;
- proposed changes or actual changes, depending on mode;
- verification actually performed;
- risks and unresolved questions.

RULES
- Do not claim something was verified unless you actually verified it.
- Prefer repository evidence and executable tests over speculation.
- In consult/review mode, do not modify files.
- In implement mode, modify only the authorized scope.
- Do not commit, push, merge, publish, deploy, or alter remote state.
```

Send only task-relevant context. Do not forward hidden chain-of-thought or unrelated conversation history.

## Result validation

Treat the Antigravity run as successful only when:

- the command exits successfully; and
- the parsed JSON contains `"status": "SUCCESS"`; and
- `.response` is a non-empty string.

On success:

- read `.response`;
- retain `.conversation_id`;
- retain usage metadata only when useful;
- in `implement` mode, inspect workspace changes independently.

On failure:

1. inspect the process exit status;
2. inspect `stderr`;
3. inspect JSON `status` and `error` when present;
4. classify permission denials, `CANCELED`, and empty responses as `BLOCKED`;
5. retry once only for a clearly correctable invocation issue or plausible transient failure;
6. do not weaken permissions merely to force success.

## Return format

Return to the parent conversation using:

```text
Status: SUCCESS | ERROR | BLOCKED
Mode: consult | review | implement
Antigravity model: <slug if explicitly selected or observable; otherwise "default">
Conversation ID: <id if available>

Summary:
<short result>

Findings:
- <finding with concrete evidence>

Files changed:
- <paths, or "None">

Verification performed:
- <commands/checks actually performed>

Risks / unresolved questions:
- <remaining uncertainty, or "None">
```

Do not dump the full raw Antigravity response unless explicitly requested.

The parent Claude Code conversation owns final verification, acceptance, and integration.
