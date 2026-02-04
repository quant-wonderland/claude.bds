---
name: Skill Creator
description: Create effective Claude Code skills with best practices and templates
---

# Skill Creator

A meta-skill for creating effective Claude Code skills. This guide teaches skill design principles, provides templates, and walks through the creation workflow.

## What is a Skill?

A skill is a `SKILL.md` file containing:
1. **YAML frontmatter** with `name` and `description`
2. **Markdown content** with instructions, workflows, templates, and examples

Skills are loaded when triggered via `/skill-name` command and provide domain-specific guidance to Claude.

## Skill Design Principles

### 1. Concise Context

Only include information the agent doesn't already know. Every piece must justify its token cost.

**Avoid:**
- General programming concepts
- Standard library documentation
- Common CLI usage patterns

**Include:**
- Project-specific conventions and paths
- Domain knowledge unique to your workflow
- Configuration specifics and patterns

### 2. Appropriate Specificity Levels

Choose the right level for each part of your skill:

| Level | When to Use | Example |
|-------|-------------|---------|
| **Text** | Flexible approaches, multiple valid solutions | "Use appropriate error handling" |
| **Pseudocode** | Patterns to follow, structure matters | "Loop through files, filter by extension, process each" |
| **Specific Code** | Critical consistency required | Exact code templates, bash commands |

### 3. Progressive Disclosure

Structure content so details load only when needed:
- **YAML frontmatter**: Always visible in skill listings
- **Main SKILL.md**: Loaded when skill is triggered
- **Subdirectories**: Referenced files load on-demand (e.g., `./examples/template.py`)

## Interactive Workflow

### Step 0: Gather Skill Requirements

**Ask user for the following information using AskUserQuestion:**

1. **Skill Name**: A short, descriptive kebab-case name
   - Examples: `factor-migration`, `nix-packaging`, `api-client-generator`

2. **Skill Purpose**: What task does this skill help accomplish?
   - Be specific: "Migrate Python factors to C++" not "Help with code"

3. **Complexity Level**:
   - **Simple**: Single workflow, few steps (~50 lines)
   - **Moderate**: Multiple scenarios, some templates (~150 lines)
   - **Complex**: Many patterns, extensive examples (~300+ lines)

### Step 1: Identify Key Components

Based on the skill purpose, determine what sections to include:

| Component | Include When... |
|-----------|-----------------|
| Prerequisites | External tools, repos, or setup required |
| Files to Modify | Skill involves editing specific files |
| Code Templates | Patterns must be followed exactly |
| Validation Steps | Output needs verification |
| Common Pitfalls | Known issues users frequently encounter |
| Subdirectories | Multiple variations (languages, platforms) |

### Step 2: Choose Organization Pattern

#### Pattern A: Single File (Simple to Moderate Skills)

```
skills/
  my-skill/
    SKILL.md        # Everything in one file
```

Best for: Focused tasks, single workflow, < 300 lines

#### Pattern B: Modular with Subdirectories (Complex Skills)

```
skills/
  my-skill/
    SKILL.md        # Overview and main workflow
    python/
      python.md     # Python-specific guidance
      example/
        file.py     # Working example
    rust/
      rust.md       # Rust-specific guidance
```

Best for: Multiple language/platform variations, extensive examples

### Step 3: Draft Using Template

Select the appropriate template from the Templates section below.

### Step 4: Create the Skill File

```bash
mkdir -p /path/to/skills/[skill-name]/
# Write SKILL.md with drafted content
```

## Templates

### Minimal Template

For simple, focused skills:

```markdown
---
name: My Skill
description: Brief description of what this skill does
---

# My Skill

[One paragraph overview]

## Workflow

1. [Step 1 description]
2. [Step 2 description]
3. [Step 3 description]

## Example

[Concrete usage example]
```

### Interactive Template

For skills requiring user input:

```markdown
---
name: My Interactive Skill
description: Interactive workflow with user input gathering
---

# My Interactive Skill

## Overview

[What problem this solves and when to use it]

## Interactive Workflow

### Step 0: Gather User Input

**Ask user for the following information:**

1. **Input Name**: Description of what's needed
   - Example: `value1`, `value2`

Use `AskUserQuestion` tool if not provided in arguments.

### Step 1: [Analysis Step]

[Instructions for analyzing the situation]

### Step 2: [Implementation Step]

[Concrete actions to take]

## Verification

- [ ] Verification item 1
- [ ] Verification item 2
```

### Comprehensive Template

For complex skills with patterns and validation:

```markdown
---
name: My Comprehensive Skill
description: Full-featured skill with patterns, templates, and validation
---

# My Comprehensive Skill

[Brief overview paragraph]

## Overview

[Detailed description: problem solved, when to use, key concepts]

## Interactive Workflow

### Step 0: Gather User Input

**Ask user for the following information:**

1. **Required Input 1**: Description
   - Example: `example_value`

2. **Required Input 2**: Description

Use `AskUserQuestion` tool if inputs are not provided.

### Step 1: [Analysis Phase]

[How to analyze the current state]

Search locations:
- `/path/to/search1`
- `/path/to/search2`

### Step 2: [Implementation Phase]

[Step-by-step implementation guidance]

### Step 3: [Validation Phase]

[How to verify success]

## Prerequisites

- [Prerequisite 1 with path/version]
- [Prerequisite 2]

## Files to Modify

| File | Operation | Purpose |
|------|-----------|---------|
| `path/to/file1` | Create | Description |
| `path/to/file2` | Modify | Description |

## Implementation Patterns

### [Pattern 1 Name]

```[language]
// Exact code template
// With comments explaining non-obvious parts
```

### [Pattern 2 Name]

```[language]
// Another pattern template
```

## Validation Workflow

### Step 1: Build/Test

```bash
# Commands to run
cd /path/to/project
./build.sh
./test.sh
```

### Step 2: Verify Output

[How to check results match expectations]

## Verification Checklist

- [ ] Checkpoint 1
- [ ] Checkpoint 2
- [ ] Checkpoint 3

## Common Pitfalls

1. **[Pitfall Name]**: Description of the issue and how to avoid it
2. **[Another Pitfall]**: Description and solution
```

## Best Practices

### Content Guidelines

1. **Be Specific About Paths**
   - Good: `/home/user/github/project/src/`
   - Bad: "the source directory"

2. **Use Tables for Structured Data**
   ```markdown
   | File | Operation | Purpose |
   |------|-----------|---------|
   | `path/file.cc` | Create | New implementation |
   ```

3. **Include Working Examples**: Copy-paste ready, not just descriptions

4. **Document Edge Cases**: Invalid input, missing files, error scenarios

### Structure Guidelines

1. **Lead with Workflow**: Users want to know *what to do* first
2. **Put Reference Material Last**: Patterns, templates, checklists at the end
3. **Use Tables Over Lists**: For structured information with multiple attributes

### Interactive Workflow Guidelines

1. **Batch Related Questions**: Use single `AskUserQuestion` call with multiple questions
2. **Provide Defaults**: Make common choices easy to accept
3. **Show Context First**: Display relevant info before prompting for decisions

### Code Template Guidelines

1. **Mark Placeholders Clearly**: Use `[PLACEHOLDER]` or `XXX` for user-specific values
2. **Include Comments**: Explain non-obvious logic
3. **Show Full Context**: Include surrounding code, not just the changed line

## Example Skills Analysis

### factor-migration (Comprehensive Single-File)

**Location**: `bds-dev/skills/factor-migration/SKILL.md`

**Strengths:**
- Clear step-by-step workflow starting with user input gathering
- Extensive code templates at specific-script level
- File modification table with exact paths
- Verification checklist ensures completeness
- Common pitfalls section prevents known issues

**Key Patterns:**
- Interactive workflow with `AskUserQuestion`
- Dependency resolution table
- Multiple code template sections
- Validation workflow with bash commands

### nix-packaging (Modular Multi-File)

**Location**: `bds-dev/skills/nix-packaging/`

**Structure:**
```
nix-packaging/
  SKILL.md          # Overview + workflow
  python/
    python.md       # Python packaging guide
    tyro/package.nix
    darts/package.nix
  rust/
    rust.md
  js/
    js.md
```

**Strengths:**
- Main SKILL.md provides overview and workflow choice
- Language-specific guides in subdirectories
- Real working examples alongside documentation
- Cross-references between files

**Key Patterns:**
- Progressive disclosure via linked files
- Examples as real package files
- Workflow branching (new vs update)

## Skill Creation Checklist

### Structure
- [ ] YAML frontmatter with `name` and `description`
- [ ] Overview section explains purpose
- [ ] Workflow section with numbered steps

### Content
- [ ] Prerequisites listed (if applicable)
- [ ] Files to modify documented (if applicable)
- [ ] Code templates use appropriate specificity level
- [ ] Examples are copy-paste ready
- [ ] Edge cases addressed

### Usability
- [ ] Interactive steps use `AskUserQuestion` appropriately
- [ ] Verification steps included
- [ ] Common pitfalls documented (if known)

### Registration
- [ ] Skill file at `skills/[name]/SKILL.md`
- [ ] Directory structure matches chosen pattern

## Registration

Skills in this project are automatically discovered from the `skills/` directory.

### Directory Structure

```
bds-dev/
  .claude-plugin/
    plugin.json       # Contains: "skills": "./skills/"
  skills/
    your-skill/
      SKILL.md        # Your skill file
```

### Testing Your Skill

1. Save your `SKILL.md` file
2. In Claude Code, invoke `/bds-dev:your-skill` (or with arguments)
3. Verify the skill content loads and guides the workflow correctly

### Plugin Configuration

The `plugin.json` already registers the skills directory:

```json
{
  "name": "bds-dev",
  "skills": "./skills/"
}
```

No changes needed unless using a different directory structure.
