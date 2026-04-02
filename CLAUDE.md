# CobolHelper

A cross-cutting companion for analyzing and understanding legacy COBOL code. This repo is not tied to any single application — it serves as a shared knowledge base and toolset across a large COBOL landscape.

This repo does not contain runnable software. It contains knowledge, skills, and structured context that Claude uses to provide expert COBOL assistance through natural language conversation.

## Personality

Read and follow `src/personality.md` at all times. These are non-negotiable rules governing tone, honesty, and communication style. No exceptions.

## Session Start

At the beginning of a conversation, if the user does not already specify:
1. **Ask which project** they are working on (check `project_map.json` for known projects)
2. **Ask which COBOL version/dialect** is in use (check `version_map.json` for known versions)

Load the relevant project and version files immediately once identified. If the user doesn't know the version, attempt to infer it from the source code. If it cannot be inferred, proceed without loading version files.

## Input Handling

Users will provide COBOL source in one of three ways:
- **File path(s)**: One or more paths to files on the local system
- **Pasted code**: Raw COBOL source copied into chat
- **Files placed in `input/`**: Dropped directly into the input folder

Regardless of how source is provided, always copy it into `input/` before doing any work:
- File paths: copy the file(s) into `input/`
- Pasted code: create a `.cbl` file in `input/` with a reasonable name based on the PROGRAM-ID or content
- Already in `input/`: no action needed

This ensures agents and skills can read from a consistent location without passing full source through conversation context.

If `input/` contains files that the user has not explicitly referenced in the current session, ask whether these are leftover from a previous session and whether to remove or keep them.

## Knowledge Structure

All persistent knowledge lives in `src/`. This is Claude's long-term memory for COBOL — committed to the repo and built up over time.

| Folder | Purpose |
|---|---|
| `src/cobol/` | General COBOL knowledge — patterns, common structures, dialect notes, idioms |
| `src/versions/` | One file per COBOL version/dialect with version-specific behaviors and differences |
| `src/projects/` | One file per known COBOL project/application across the landscape |
| `src/mind_maps/` | JSON index files for efficient lookup into the above folders |

## Mind Maps — Reading and Lookup

Mind maps are JSON index files in `src/mind_maps/` that allow efficient lookup into knowledge files without loading everything into context.

Each mind map is an array of entries:
```json
[
  {
    "name": "",
    "location": "",
    "description": "",
    "key_words": []
  }
]
```

**Lookup rules:**
1. Read the relevant mind map JSON first — never bulk-read all `.md` files in a folder.
2. Match the user's request or code context against `name`, `description`, and `key_words`.
3. If more than three entries match on keywords, prioritize entries with **multiple keyword matches** before loading single-match files.
4. Load only the files you need. If the first batch answers the question, stop.

**Which mind maps to consult:**

| Map | When to read |
|---|---|
| `cobol_map.json` | When Claude encounters unfamiliar COBOL syntax, patterns, or structures — check here before searching online or relying on training data |
| `version_map.json` | As soon as the COBOL version/dialect is known or inferred from the source. If the version is unknown and cannot be inferred, do not load any version files |
| `project_map.json` | When the user references a project by name, or when code can be identified as belonging to a known project |

## Knowledge Lookup Hierarchy

When Claude needs to understand something in COBOL code, follow this order:
1. **`src/versions/`** — load the relevant version file immediately when the dialect is identified
2. **`src/cobol/`** — consult via `cobol_map.json` before going to general knowledge
3. **Training knowledge / web** — last resort only

## Skills

All skills live in `.claude/skills/{skill_name}/SKILL.md`. Claude discovers and invokes them implicitly based on the user's natural language — no slash commands. The user just talks.

### Background Skills (always active)

| Skill | Purpose |
|---|---|
| `update-info` | Keeps the knowledge base current. Runs continuously — when new information surfaces (from user, code analysis, or web), updates the appropriate files in `src/`. Also updates `src/personality.md` when the user gives communication feedback. |
| `mind-map` | Governs creation and update of mind map index files. Referenced by `update-info` when knowledge files change. |

### Forked Skills (spawn subagents)

These skills use `context: fork` — they spawn isolated subagents for focused work. Claude matches user intent to the right skill(s) automatically.

| Skill | Triggers | Output |
|---|---|---|
| `code-specialist` | Bug hunting, debugging, logic tracing, "what's wrong", "trace X", abend analysis | Chat |
| `business-analyst` | "What does this do", explain purpose, business rules, process flow, non-technical explanation | Chat |
| `documentor` | **Only when user explicitly asks** for docs, reports, diagrams, "write this up", "something to share" | `output/` folder |
| `modernization-advisor` | "Can we modernize", migration difficulty, refactoring options, deprecated patterns, complexity | Chat |
| `impact-analyzer` | "What if we change X", field/copybook usage, blast radius, upstream/downstream effects | Chat |
| `comparator` | "What changed", diff two versions, compare programs, old vs new | Chat |

### Orchestration Rules

1. **Implicit routing only.** Match the user's intent to the right skill(s) — never ask the user to pick one.
2. **Parallel when possible.** If a request triggers multiple skills (e.g., "explain this program and find any bugs" = business-analyst + code-specialist), spawn them in parallel.
3. **Wait and combine.** When multiple skills are spawned, wait for all to return and deliver a single combined response — unless the documentor is one of them.
4. **Documentor exception.** If the documentor is spawned alongside other skills, deliver the chat response from the other skills immediately. Let the documentor run in the background and notify the user when docs are ready in `output/`.
5. **Documentor reuses context.** When the documentor runs after other skills have already analyzed the code, it should build on their findings rather than re-analyzing from scratch.
6. **Simple questions don't need forked skills.** If the user asks something Claude can answer directly from knowledge files or a quick code read, just answer — don't over-orchestrate.
