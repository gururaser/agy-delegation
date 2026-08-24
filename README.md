# Antigravity Delegation for Codex and Claude Code

**English** | [Türkçe](README.tr.md)

Use Google Antigravity CLI as an external coding/review agent from **OpenAI Codex** or **Claude Code**, while keeping the host coding agent responsible for final verification.

The package implements the same workflow on both hosts:

```text
User
  │
  ▼
Codex / Claude Code main agent
  │
  ▼
Antigravity delegation skill
  │
  ▼
Platform-specific Antigravity subagent
  │
  ▼
agy --output-format json
  │
  ▼
Antigravity-hosted model
  │
  ▼
Concise structured findings
  │
  ▼
Main agent verifies repository state, diff, tests, and evidence
```

Antigravity is deliberately treated as an **independent external opinion**, not as a source of truth.

## Why there are two SKILL.md files

Codex and Claude Code both support reusable skills, and Claude Code follows the Agent Skills open standard. Their host-specific execution features are not identical, however.

The Claude Code version uses native skill frontmatter such as:

```yaml
context: fork
agent: antigravity
```

This causes the skill to execute in an isolated custom Claude Code subagent.

The Codex version instead instructs the main Codex agent to delegate the bounded work order to the custom `antigravity` Codex subagent.

The workflow and safety policy are intentionally equivalent, but the wrappers remain separate so the package does not depend on either host ignoring unknown configuration fields.

## Repository layout

```text
.
├── README.md
├── README.tr.md
├── .agents/
│   └── skills/
│       └── antigravity-delegation/
│           └── SKILL.md
├── .codex/
│   └── agents/
│       └── antigravity.toml
└── .claude/
    ├── skills/
    │   └── antigravity-delegation/
    │       └── SKILL.md
    └── agents/
        └── antigravity.md
```

## Requirements

You need:

- Google Antigravity CLI installed and authenticated;
- `agy` available on `PATH`;
- Codex, Claude Code, or both;
- Git for implementation/diff verification.

Check Antigravity availability:

```bash
agy models
```

The integration uses Antigravity headless mode and expects JSON output with fields such as `status`, `response`, and `conversation_id`.

For Claude Code, use a current release. The included skill uses `context: fork` with `agent: antigravity`, which is the documented mechanism for running a skill in an isolated subagent context.

### Workspace and headless permissions

Every adapter invocation passes the host's current project directory to Antigravity:

```bash
agy -p "<PROMPT>" --add-dir "$PWD" ...
```

This prevents an Antigravity project retained from an earlier session from making the current repository appear to be outside the active workspace. File access inside the added workspace follows Antigravity's workspace policy.

Shell commands still default to `Ask` in headless mode. If a delegated review must run Git or project checks, add only the required commands to `~/.gemini/antigravity-cli/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "command(git (status|diff|rev-parse))",
      "command(npm run (build|lint|test))"
    ]
  }
}
```

Replace the test-command rule with the commands used by your project. A matching `deny` or `ask` rule takes precedence over `allow`, so keep broader conflicting rules out of the policy.

Sources: [Antigravity headless mode](https://antigravity.google/docs/cli/headless/#permissions-in-headless-mode) and [fine-grained permissions](https://antigravity.google/docs/cli/permissions/).

## Installation

### Option A — Project-scoped installation

Copy this repository's hidden directories into the root of your project.

For **Codex**, keep:

```text
.agents/skills/antigravity-delegation/SKILL.md
.codex/agents/antigravity.toml
```

For **Claude Code**, keep:

```text
.claude/skills/antigravity-delegation/SKILL.md
.claude/agents/antigravity.md
```

If you use both tools on the same repository, keep all four files.

Start a new Codex/Claude Code session after installing or changing custom agent definitions.

### Option B — Global installation

#### Codex

```bash
mkdir -p ~/.agents/skills/antigravity-delegation
mkdir -p ~/.codex/agents

cp .agents/skills/antigravity-delegation/SKILL.md \
  ~/.agents/skills/antigravity-delegation/SKILL.md

cp .codex/agents/antigravity.toml \
  ~/.codex/agents/antigravity.toml
```

#### Claude Code

```bash
mkdir -p ~/.claude/skills/antigravity-delegation
mkdir -p ~/.claude/agents

cp .claude/skills/antigravity-delegation/SKILL.md \
  ~/.claude/skills/antigravity-delegation/SKILL.md

cp .claude/agents/antigravity.md \
  ~/.claude/agents/antigravity.md
```

Restart the coding-agent session after installing the custom agent.

## Delegation modes

The integration defines three modes.

### `consult`

Read-only external reasoning.

Use it for:

- architecture;
- debugging hypotheses;
- design tradeoffs;
- planning;
- alternative approaches;
- second opinions.

### `review`

Read-only external review.

Use it for:

- code review;
- diff review;
- security review;
- regression-risk analysis;
- test-coverage review;
- implementation critique.

### `implement`

Allows Antigravity to produce a bounded candidate implementation.

Use it only when external implementation is actually useful.

The host agent must subsequently inspect:

```bash
git status --short
git diff
```

and run relevant tests before accepting the changes.

## Usage

### Codex

You can ask Codex naturally:

```text
Use the antigravity-delegation skill in review mode.
Ask Antigravity to independently review the authentication changes for
correctness, race conditions, and missing tests. Then verify its findings yourself.
```

A more structured work order:

```text
Use antigravity-delegation.

mode: review
objective: Review the new token refresh implementation for concurrency bugs.
scope: src/auth/, tests/auth/
constraints: Do not modify files.
expected_output: Findings ranked by severity with concrete file references.
effort: high
```

The expected flow is:

```text
Codex main
  -> antigravity-delegation skill
  -> Codex antigravity subagent
  -> agy
  -> structured report
  -> Codex verification
```

### Claude Code

Invoke the skill directly:

```text
/antigravity-delegation mode: review
objective: Review the new token refresh implementation for concurrency bugs.
scope: src/auth/, tests/auth/
constraints: Do not modify files.
expected_output: Findings ranked by severity with concrete file references.
effort: high
```

Because the Claude Code skill uses `context: fork` and `agent: antigravity`, the work order runs in the isolated custom Antigravity adapter subagent and its concise result is returned to the main conversation.

You can also ask Claude Code to use the skill when appropriate; the skill description allows model invocation.

## Selecting an Antigravity model

The adapters default to `gemini-3.7-flash-high`. This keeps substantive implementation work on the fast external worker while the host's larger model remains responsible for planning and verification.

List available model slugs:

```bash
agy models
```

Include `antigravity_model` only when you want to override the default:

```text
antigravity_model: gemini-3.7-flash-medium
```

The adapter will invoke:

```bash
agy -p "<PROMPT>" \
  --add-dir "$PWD" \
  --model "gemini-3.7-flash-medium" \
  --output-format json \
  --effort high \
  --print-timeout 10m
```

If an unknown model is pinned, Antigravity headless mode fails instead of silently selecting another model.

## Result contract

The platform-specific adapter returns a compact report:

```text
Status: SUCCESS | ERROR | BLOCKED
Mode: consult | review | implement
Antigravity model: ...
Conversation ID: ...

Summary:
...

Findings:
- ...

Files changed:
- ...

Verification performed:
- ...

Risks / unresolved questions:
- ...
```

The raw external-model response is intentionally not dumped into the main context unless requested.

## Verification model

The host coding agent remains responsible for the final answer.

A successful Antigravity result requires exit code `0`, JSON status `SUCCESS`, and a non-empty `response`. It means only that the external run completed successfully; it does **not** prove that the result is correct.

For important claims, the host should verify against:

- repository code;
- tests;
- build/lint/type-check results;
- logs;
- specifications;
- authoritative documentation.

For implementation tasks, the host should inspect the diff before accepting any change.

## Permissions and safety

The package intentionally does **not** enable:

```bash
--dangerously-skip-permissions
```

by default.

If Antigravity requires an operation that its headless permission policy blocks, configure the narrowest appropriate Antigravity permission instead of globally bypassing checks.

The adapters also instruct Antigravity not to:

- commit;
- push;
- merge;
- deploy;
- publish;
- delete branches;
- alter remote state;

unless the parent task explicitly requires such an action.

In `consult` and `review` modes, file modification is prohibited by instruction.

## Cost and context strategy

The extra subagent layer is intentional.

Without isolation:

```text
main agent -> agy -> long external response -> main context
```

With this package:

```text
main agent
   -> small adapter subagent
      -> agy
      -> potentially verbose external reasoning
      -> compressed evidence report
   -> main context
```

This reduces context pollution and lets the main coding agent spend its context on the repository and final verification.

The adapter models are deliberately lightweight:

- Codex adapter: `gpt-5.6-luna`, low reasoning;
- Claude Code adapter: `haiku`, low effort.

The external worker defaults to `gemini-3.7-flash-high` and performs the substantive delegated reasoning.

### Codex model override compatibility

If your Codex runtime rejects the custom Luna override, edit:

```text
.codex/agents/antigravity.toml
```

and remove:

```toml
model = "gpt-5.6-luna"
model_reasoning_effort = "low"
```

The subagent will then inherit the parent Codex configuration.

## Troubleshooting

### `agy: command not found`

Confirm that Antigravity CLI is installed and visible in the environment from which Codex or Claude Code is launched:

```bash
which agy
agy models
```

### Antigravity returns `ERROR`

Inspect:

- process exit code;
- stderr;
- JSON `error`;
- the requested model slug;
- Antigravity permission policy.

Do not solve permission errors by automatically enabling `--dangerously-skip-permissions`.

### Claude Code does not see the subagent

Check:

```text
.claude/agents/antigravity.md
```

or:

```text
~/.claude/agents/antigravity.md
```

Then restart the Claude Code session and inspect `/agents`.

### Claude Code does not see the skill

Check:

```text
.claude/skills/antigravity-delegation/SKILL.md
```

or:

```text
~/.claude/skills/antigravity-delegation/SKILL.md
```

Then use `/skills` or invoke `/antigravity-delegation`.

### Codex cannot spawn `antigravity`

Confirm:

```text
.codex/agents/antigravity.toml
```

or:

```text
~/.codex/agents/antigravity.toml
```

is present. Start a new Codex session after adding the custom agent.

If the error is specifically related to the custom adapter model, remove the model override as described above.

## Design principles

This package follows five rules:

1. **External models are reviewers/workers, not authorities.**
2. **The main coding agent owns the final decision.**
3. **Consult and review are read-only by default.**
4. **Implementation is bounded and diff-verified.**
5. **Verbose external reasoning stays outside the main context whenever possible.**
