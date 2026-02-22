# CLAUDE.md

This file provides guidance for AI assistants (Claude and others) working in this repository.

---

## Project Overview

**Repository:** `kwaktai/claude_teslakey`
**Branch convention:** `claude/<description>-<session-id>`

This repository is currently in its initial state. As the codebase grows, update this document to reflect the actual structure, dependencies, and conventions.

---

## Repository Structure

```
claude_teslakey/
├── CLAUDE.md          # This file — AI assistant guidelines
├── README.md          # Human-facing project documentation (to be created)
└── src/               # Source code (to be created)
```

As the project evolves, maintain this tree to reflect the real layout.

---

## Development Workflow

### Branch Strategy

- Work exclusively on the branch specified at task start (format: `claude/<description>-<session-id>`).
- Never push to `main` or another branch without explicit permission.
- Create the branch locally if it does not yet exist:

```bash
git checkout -b claude/<description>-<session-id>
```

### Commit Convention

Use clear, imperative commit messages:

```
Add prompt-caching middleware for system prompts
Fix cache invalidation when tools array changes
Update CLAUDE.md with compaction strategy
```

### Push Protocol

```bash
git push -u origin <branch-name>
```

- If push fails due to a network error, retry up to 4 times with exponential backoff: 2 s → 4 s → 8 s → 16 s.
- A 403 usually means the branch name does not match the required `claude/` prefix pattern — verify the branch name first.

### Fetch / Pull

```bash
git fetch origin <branch-name>
git pull origin <branch-name>
```

Apply the same exponential-backoff retry on network failures.

---

## AI Assistant Conventions

### File Editing

- Read every file before modifying it.
- Prefer editing existing files over creating new ones.
- Never add comments, docstrings, or type annotations to code you did not change.
- Delete unused code completely; do not leave it commented out.

### Task Management

- Use the `TodoWrite` tool to plan multi-step tasks and track progress.
- Mark a todo `in_progress` before starting it, `completed` immediately after finishing.
- Only one todo should be `in_progress` at a time.

### Asking Questions

- Use `AskUserQuestion` when requirements are ambiguous or a decision is needed.
- In plan mode, clarify unknowns before calling `ExitPlanMode`.

### Security

- Never introduce command injection, XSS, SQL injection, or other OWASP Top 10 vulnerabilities.
- Validate only at system boundaries (user input, external APIs); trust internal framework guarantees.

---

## Claude API / Anthropic SDK Patterns

The following patterns apply when this project uses the Claude API or Anthropic SDK.

### Prompt Caching

Claude caches the **longest matching prefix** of a conversation. To maximise cache hits:

| Position | Content | Rationale |
|----------|---------|-----------|
| 1st | Static system instructions | Never changes → always cached |
| 2nd | Large static context (docs, schemas) | Changes rarely |
| 3rd | Conversation history | Changes per turn |
| Last | Current user message + dynamic data | Always new |

**Key rules:**
- Place the `cache_control: {type: "ephemeral"}` breakpoint on the last *stable* block.
- Adding a `cache_control` marker to a block that changes every turn (e.g. timestamp) **breaks** the cache for everything after it — avoid this.
- Cache entries survive ~5 minutes (standard) or longer with prompt caching enabled on the API.

### System Prompt Updates Without Cache Invalidation

To inject dynamic context into a cached system prompt, append a `<system-reminder>` block **after** the cached content rather than modifying the cached prefix:

```python
messages = [
    {
        "role": "user",
        "content": [
            {"type": "text", "text": system_prompt, "cache_control": {"type": "ephemeral"}},
            {"type": "text", "text": f"<system-reminder>{dynamic_context}</system-reminder>"}
        ]
    },
    ...
]
```

This keeps the expensive cached prefix intact while still surfacing fresh information.

### Model and Tool Stability

Changing the model ID or the tools array **invalidates the cache** for all subsequent content. Avoid switching models mid-session unless strictly necessary. For tasks that require a different model, use a sub-agent rather than switching the parent session's model.

### Plan Mode Design

When building an agent that supports a planning phase:

1. Expose an `EnterPlanMode` tool that sets a `planning` flag and returns a plan scaffold.
2. Expose an `ExitPlanMode` tool that clears the flag and signals readiness to execute.
3. Only execute side-effecting tools (file writes, API calls) after `ExitPlanMode` is called.

```python
tools = [
    {"name": "EnterPlanMode", "description": "Switch to planning mode. No side effects allowed."},
    {"name": "ExitPlanMode",  "description": "Approve the plan and begin execution."},
]
```

### Deferred Tool Loading

If your agent has a large tool catalogue, avoid sending all tool definitions on every turn (cost and token overhead). Use a **stub pattern**:

1. Send a lightweight `search_tools(query)` stub initially.
2. On invocation, resolve the real tool schema and inject it into the next turn's `tools` array.
3. This keeps initial context small and caches the growing tool list as it stabilises.

### Cache-Safe Compaction (Long Sessions)

When a conversation grows beyond the context window:

1. Summarise **only** the portion of history that falls outside the cached prefix.
2. Prepend the summary to the existing cached prefix block; do not alter the prefix itself.
3. The parent process and sub-agents should share the same system-prompt prefix so their caches are compatible.

```
[cached system prompt prefix]  ← never touch
[summary of old turns]         ← replace old turns with this
[recent turns]                 ← keep verbatim
```

---

## Key Principles at a Glance

| Principle | Rule |
|-----------|------|
| Cache ordering | Static content first, dynamic content last |
| Cache invalidation | Do not change model, tools, or cached blocks mid-session |
| Dynamic context | Use `<system-reminder>` appended after the cached block |
| Model switching | Use sub-agents; never switch the parent session's model |
| Plan Mode | Gate side effects behind `ExitPlanMode` approval |
| Tool loading | Defer large tool catalogues; load on demand |
| Compaction | Summarise only content outside the cached prefix |
| Commits | Imperative, descriptive; never skip pre-commit hooks |
| Branches | Always `claude/<description>-<session-id>`; never push to main |
| File edits | Read first, minimise scope, delete unused code |

---

## Updating This File

Keep CLAUDE.md current as the project evolves:

- Add the real directory structure once source files exist.
- Document the chosen language/framework, test runner, and lint commands.
- Record any project-specific naming conventions or architectural decisions.
- Update the API patterns section when new integrations are introduced.
