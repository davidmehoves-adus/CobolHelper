# Claude Code Skills Specification Reference

Saved from Anthropic documentation for future reference. Not linked to CLAUDE.md or mind maps.

## Folder Structure

Each skill is a directory containing a required `SKILL.md` file:

```
.claude/skills/{skill_name}/
├── SKILL.md                  # Required: YAML frontmatter + markdown instructions
├── scripts/                  # Optional: executable code
├── references/               # Optional: detailed documentation
├── assets/                   # Optional: templates, resources
└── examples/                 # Optional: working code samples
```

## SKILL.md Frontmatter Fields

All optional except `description` (effectively required for discovery).

| Field | Type | Description |
|---|---|---|
| `name` | string (max 64) | Display name and `/slash-command` identifier. Lowercase + hyphens only. Defaults to directory name. |
| `description` | string (max 1024) | What the skill does and when to use it. Front-load key use case. >250 chars truncated in listings. |
| `license` | string | License name or reference. |
| `compatibility` | string (max 500) | Environment/product requirements. |
| `metadata` | YAML map | Arbitrary key-value data (author, version, etc.). |
| `allowed-tools` | string/list | Tools pre-approved when skill is active (e.g., `Read Grep Bash(git:*)`). |
| `argument-hint` | string | Autocomplete hint (e.g., `[issue-number]`). |
| `disable-model-invocation` | boolean | `true` = Claude cannot auto-load. Manual only. Default: `false`. |
| `user-invocable` | boolean | `false` = hidden from `/` menu. Claude still auto-invokes. Default: `true`. |
| `model` | string | Model override when skill is active. |
| `effort` | string | Effort level: `low`, `medium`, `high`, `max`. |
| `context` | string | `fork` = run in isolated subagent context. Skill content becomes subagent prompt. |
| `agent` | string | Subagent type with `context: fork`. Options: `Explore`, `Plan`, `general-purpose`, or custom agent name. |
| `paths` | string/list | Glob patterns limiting when skill auto-activates. |
| `shell` | string | `bash` (default) or `powershell`. |
| `hooks` | YAML object | Hooks scoped to this skill's lifecycle. |

## Dynamic String Substitutions

| Variable | Description |
|---|---|
| `$ARGUMENTS` | All arguments passed to the skill. |
| `$ARGUMENTS[N]` / `$N` | Specific argument by 0-based index. |
| `${CLAUDE_SESSION_ID}` | Current session ID. |
| `${CLAUDE_SKILL_DIR}` | Directory containing the skill's SKILL.md. |

## Inline Command Injection

Use `` !`<command>` `` to run shell commands before Claude sees the skill. Output replaces the placeholder.

## Invocation Control

| Setting | User Can Invoke | Claude Can Invoke |
|---|---|---|
| (default) | Yes | Yes |
| `disable-model-invocation: true` | Yes | No |
| `user-invocable: false` | No | Yes |

## Key Constraints

- Name: max 64 chars, lowercase + hyphens only
- Description: max 1024 chars, front-load key use case
- SKILL.md: keep under 500 lines, move detail to references/
- File references: use relative paths from SKILL.md
- `context: fork` skills have no access to main conversation history
