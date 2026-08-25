---
name: antigravity-delegation
description: Send a bounded coding, review, debugging, architecture, or second-opinion task directly from the main Claude Code conversation to Google Antigravity CLI in headless mode. Use when an independent external-model perspective would materially improve confidence.
---

# Antigravity Delegation

Execute this work order directly from the main Claude Code conversation. Do not
spawn a Claude Code subagent:

$ARGUMENTS

Antigravity output is advisory until the main conversation independently verifies
its material claims.

## Work order

Expect:

```text
mode: consult | review | implement
prompt: |
  <complete, ready-to-send Antigravity prompt>
antigravity_model: <optional base model name; default gemini-3.7-flash>
effort: <optional low | medium | high; default high>
```

If `mode` is omitted, use `consult`. Inspect the relevant repository context and
complete any missing prompt context before invoking Antigravity. Do not forward
hidden chain-of-thought, secrets, or irrelevant conversation history. Keep the
base model and reasoning effort separate. The default invocation uses model
`gemini-3.7-flash` with `effort: high`. Check `agy models` instead of guessing an
unavailable model or effort combination.

## Workflow

1. Build a self-contained prompt with the objective, scope, constraints,
   evidence, required output, and write rules.
2. Invoke `agy` once in headless JSON mode from the current workspace.
3. Validate the process result, JSON status, response, and permission diagnostics.
4. Expose only `.response` from a clean successful run.
5. Verify important claims, tests, and workspace changes before accepting them.

## Mode rules

### `consult`

- Use for architecture, debugging hypotheses, alternatives, planning, or tradeoffs.
- Tell Antigravity the task is read-only and files must not be modified.
- Require concrete conclusions, assumptions, and evidence.

### `review`

- Use for code, diff, test, security, or regression review.
- Tell Antigravity the task is read-only and files must not be modified.
- Require prioritized findings with file paths, symbols, tests, or observed behavior.

### `implement`

- Use only when the user explicitly authorizes workspace changes.
- State the exact allowed scope and prohibit unrelated changes.
- Prohibit commits, pushes, merges, publishing, deployment, branch deletion, and
  remote-state changes unless the user explicitly requests them.
- Preserve unrelated user changes.
- After the run, inspect `git status --short` and `git diff`, then run the
  narrowest relevant verification before broader checks.

## Required Antigravity prompt

Use this shape:

```text
ROLE
You are an independent coding agent providing work to another coding agent that
will verify your result.

MODE
<consult | review | implement>

OBJECTIVE
<exact task>

SCOPE
<files, modules, services, or repository areas>

CONSTRAINTS
<technical and behavioral constraints>

EVIDENCE
<errors, failing tests, current behavior, or requirements>

REQUIRED OUTPUT
- concise conclusion;
- concrete findings and evidence;
- file paths and symbols when relevant;
- proposed or actual changes, according to the mode;
- verification actually performed;
- risks and unresolved questions.

RULES
- Do not claim something was verified unless you verified it.
- Prefer repository evidence and executable tests over speculation.
- In consult/review mode, do not modify files.
- In implement mode, modify only the authorized scope.
- Do not commit, push, merge, publish, deploy, or alter remote state.
```

## Direct CLI invocation

Use one-shot JSON output:

```bash
agy -p "$PROMPT" \
  --add-dir "$PWD" \
  --model "$ANTIGRAVITY_MODEL" \
  --output-format json \
  --effort "$EFFORT" \
  --print-timeout 10m
```

Pass the prompt with safe shell quoting. Capture stdout and stderr separately so
the host tool result does not expose diagnostics or the JSON envelope on success.
Do not use `stream-json`, `--continue`, or `--conversation` for normal one-shot
work. Never use `--dangerously-skip-permissions` by default.

## Result gate

Before printing anything from a successful run, confirm all of the following:

- the `agy` process exited successfully;
- parsed JSON has `status == "SUCCESS"`;
- `.response` is a non-empty string;
- captured stderr contains no permission denial or unavailable-approval notice.

When every check passes, print only `jq -r '.response'` to the host tool result.
Do not print the JSON envelope, usage, thinking-token count, conversation ID,
progress, or stderr.

On failure, do not print `.response`:

- permission denial or unavailable approval: return a short `BLOCKED` reason;
- `CANCELED`, `INTERRUPTED`, `WAITING`, or empty response: return `BLOCKED`;
- non-zero exit, invalid JSON, `ERROR`, or `INVALID`: return a short `ERROR`
  using the exit code and `.error` when available.

Retry once only for a clearly transient failure or correctable invocation error.
Never weaken permissions to force success.

## Verification gate

The main Claude Code conversation owns the final decision:

- check material consultation/review claims against source files, tests, logs, or
  authoritative documentation;
- inspect the diff and changed-file scope after implementation;
- prefer reproducible evidence over either model's confidence;
- never report a claim as verified merely because Antigravity returned `SUCCESS`.
