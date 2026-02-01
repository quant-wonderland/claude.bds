---
name: Export Sessions
description: Export Claude Code conversation sessions to Markdown format with auto-generated and custom tags
---

# Export Sessions Command

Export Claude Code conversation history to well-organized Markdown files with metadata tags.

## Data Sources

Claude Code stores conversation data in two locations:

1. **Global history index**: `~/.claude/history.jsonl`
   - Each line: `{"display": "...", "timestamp": 1234567890, "project": "/path/to/project", "sessionId": "uuid"}`
   - Used for listing sessions and getting metadata

2. **Project-level sessions**: `~/.claude/projects/{encoded-path}/{sessionId}.jsonl`
   - Directory name encoding: `/home/user/project` -> `-home-user-project`
   - Contains full conversation records with message types: `user`, `assistant`, `summary`, `file-history-snapshot`

## Workflow

### Step 1: Scan Available Projects

Read the `~/.claude/projects/` directory to list all projects with recorded sessions.

```bash
ls ~/.claude/projects/
```

For each project directory:
1. Decode the directory name by replacing leading `-` with `/` and internal `-` with `/` (note: this is a heuristic; verify by checking if the path exists)
2. Count `.jsonl` files (excluding `agent-*.jsonl` subagent files if desired, or include them)
3. Get the most recent modification time

Present to the user:
```
Available projects with Claude Code sessions:

1. /home/user/github/reporting (15 sessions)
2. /home/user/github/claude.bds (8 sessions)
3. /home/user/github/my-app (12 sessions)
...

Enter project number(s) to export (comma-separated, e.g., "1,3", or "all"):
```

### Step 2: Select Sessions

For each selected project, read `~/.claude/history.jsonl` and filter by project path.

Read the history file and parse each line as JSON. Filter entries where `project` matches the selected project path.

Present sessions to the user with:
- Date/time (convert timestamp from milliseconds)
- First user message preview (the `display` field, truncated to ~60 chars)

Example:
```
Sessions for /home/user/github/reporting:

1. [2026-01-31 20:11] "Please refactor the reporting code..."
2. [2026-01-30 15:42] "Change the table background color..."
3. [2026-01-29 10:15] "Read the pnl.py file and explain..."
...

Select sessions (comma-separated, range "1-5", or "all"):
```

### Step 3: Configure Export Options

Ask the user for export configuration:

```
Export Configuration:

1. Export path (default: ./claude-exports/):
   > [user input or press Enter for default]

2. Custom tags (comma-separated, optional):
   > [e.g., "work, finance, q4-project"]

3. Include full tool outputs? (y/n, default: n)
   Large tool outputs will be truncated. Choose 'y' to include complete outputs.
   > [user input]

4. Include AI thinking blocks? (y/n, default: n)
   > [user input]
```

### Step 4: Process Each Session

For each selected session:

#### 4.1 Read the Session File

Read the JSONL file at `~/.claude/projects/{encoded-project-path}/{sessionId}.jsonl`

Parse each line and filter by type:
- **Include**: `user`, `assistant` (these form the conversation)
- **Include for metadata**: `summary` (use as title if available)
- **Skip**: `file-history-snapshot`, `system`

#### 4.2 Build Conversation Chain

Messages are linked via `parentUuid` -> `uuid`. Build the conversation in chronological order using timestamps.

#### 4.3 Auto-detect Tags

Analyze the conversation content to generate tags:

**Language Detection** - scan file paths and commands:
| Pattern | Tag |
|---------|-----|
| `.py`, `python`, `pip` | `auto/python` |
| `.ts`, `.tsx` | `auto/typescript` |
| `.js`, `.jsx` | `auto/javascript` |
| `.nix`, `nix build`, `nix develop` | `auto/nix` |
| `.rs`, `cargo` | `auto/rust` |
| `.go` | `auto/go` |
| `.md` | `auto/markdown` |

**Task Type Detection** - scan user messages for keywords:
| Keywords | Tag |
|----------|-----|
| fix, bug, error, issue, broken | `auto/bug-fix` |
| add, implement, create, new, build | `auto/new-feature` |
| refactor, clean, reorganize, restructure | `auto/refactoring` |
| test, unittest, pytest, spec | `auto/testing` |
| doc, readme, comment, explain | `auto/documentation` |
| review, analyze, understand | `auto/code-review` |

**Tech Stack Detection** - scan for framework/library patterns:
| Pattern | Tag |
|---------|-----|
| streamlit | `auto/streamlit` |
| react, nextjs | `auto/react` |
| pytorch, torch | `auto/pytorch` |
| pandas, numpy | `auto/data-science` |
| fastapi, flask, django | `auto/web-backend` |
| docker, kubernetes | `auto/devops` |

#### 4.4 Generate Markdown Content

Create the markdown file with this structure:

```markdown
---
title: "[Session title from summary or first user message truncated to 50 chars]"
date: YYYY-MM-DD HH:MM
end_date: YYYY-MM-DD HH:MM
project: /path/to/project
session_id: uuid-here
git_branch: main
tags:
  - auto/python
  - auto/refactoring
  - custom/user-provided-tag
---

# Session: [Title]

**Project**: `/path/to/project`
**Date**: YYYY-MM-DD HH:MM - HH:MM
**Git Branch**: main

---

## Conversation

### User (HH:MM)

[User message content - preserve markdown formatting]

---

### Assistant (HH:MM)

[Assistant text response]

#### Tool: [ToolName]

**[Tool-specific details, e.g., File: /path/to/file, or Command: git status]**

<details>
<summary>Output (click to expand)</summary>

```
[Tool output, truncated if too long]
```

</details>

---

### User (HH:MM)

[Next user message...]

---

[Continue for all messages...]
```

### Step 5: Handle Message Content

#### User Messages

If `message.content` is a string, output it directly.

If `message.content` is an array with `tool_result` items, format as:

```markdown
### User (HH:MM) - Tool Result

**Tool**: [tool name from context]

<details>
<summary>Result</summary>

```
[tool result content]
```

</details>
```

#### Assistant Messages

Process `message.content` array:

- **`text` blocks**: Output the text directly
- **`tool_use` blocks**: Format based on tool type:

  For **Bash**:
  ```markdown
  #### Tool: Bash
  **Command**: `[command]`
  ```

  For **Read**:
  ```markdown
  #### Tool: Read
  **File**: `[file_path]`
  ```

  For **Edit**:
  ```markdown
  #### Tool: Edit
  **File**: `[file_path]`
  **Change**: Replace `[old_string snippet]` with `[new_string snippet]`
  ```

  For **Grep**:
  ```markdown
  #### Tool: Grep
  **Pattern**: `[pattern]`
  **Path**: `[path]`
  ```

  For **Write**:
  ```markdown
  #### Tool: Write
  **File**: `[file_path]`
  ```

- **`thinking` blocks**: If user chose to include them:
  ```markdown
  <details>
  <summary>Thinking</summary>

  [thinking content]

  </details>
  ```

### Step 6: Truncate Large Outputs

For tool outputs longer than 100 lines:

```markdown
<details>
<summary>Output (truncated, showing first 80 and last 20 of 500 lines)</summary>

```
[first 80 lines]

... [340 lines omitted] ...

[last 20 lines]
```

</details>
```

### Step 7: Write Files

1. Create the export directory if it doesn't exist
2. For each session, write the markdown file with naming: `{YYYY-MM-DD}_{project-name}_{session-id-first-6-chars}.md`
3. Create an `index.md` file in the export directory

### Step 8: Generate Index File

Create `index.md` with:

```markdown
# Claude Code Session Exports

**Exported on**: YYYY-MM-DD HH:MM
**Total sessions**: N
**Projects**: M

---

## Sessions by Project

### project-name

| Date | Title | Tags | File |
|------|-------|------|------|
| 2026-01-31 | Refactor reporting code | python, refactoring | [Link](./2026-01-31_reporting_c74f37.md) |
| 2026-01-30 | Fix table styling | python, bug-fix | [Link](./2026-01-30_reporting_f7ef9f.md) |

### another-project

| Date | Title | Tags | File |
|------|-------|------|------|
| ... | ... | ... | ... |

---

## Tags Summary

| Tag | Count |
|-----|-------|
| auto/python | 5 |
| auto/refactoring | 3 |
| custom/work | 4 |
```

### Step 9: Report Results

Provide a summary:

```
Export Complete!

Location: /path/to/exports/
Sessions exported: 5
Files created:
  - 2026-01-31_reporting_c74f37.md (python, refactoring)
  - 2026-01-30_reporting_f7ef9f.md (python, bug-fix)
  - 2026-01-29_reporting_a446f7.md (python, data-science)
  - index.md

Note: Please review exported files for any sensitive information before sharing.
```

## Error Handling

- **Corrupted JSONL lines**: Skip and log a warning, continue with next line
- **Missing session files**: Report which sessions couldn't be found
- **Permission errors**: Report and suggest checking file permissions
- **Invalid JSON**: Log the line number and skip

At the end, report any errors encountered:

```
Warnings:
- Skipped 2 corrupted lines in session c74f37
- Session f7ef9f.jsonl not found (may have been deleted)
```

## Notes

- The `~/.claude/` directory contains sensitive conversation data
- Always remind users to review exports before sharing
- Session files can be large (several MB for long conversations)
- Timestamps in history.jsonl are Unix milliseconds; in session files they are ISO 8601
