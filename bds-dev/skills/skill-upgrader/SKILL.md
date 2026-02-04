---
name: Skill Upgrader
description: Analyze skill usage sessions and upgrade skills based on real-world patterns and improvements
---

# Skill Upgrader

A meta-skill for upgrading existing skills by analyzing their usage sessions. This skill extracts patterns, identifies improvements, and updates skill files based on real-world usage data.

## Overview

When you use a skill repeatedly, valuable patterns emerge:
- Common workflows that should be documented
- Edge cases and pitfalls discovered during usage
- Code templates that work well in practice
- Missing steps that were added ad-hoc

This skill systematically extracts these insights from exported sessions and incorporates them into the skill definition.

## Prerequisites

- Exported sessions in `/var/lib/wonder/warehouse/database/lxb/claude-sessions/lxb/`
- Target skill must exist in `bds-dev/skills/` directory
- Sessions should be tagged with skill name or contain skill invocation

## Interactive Workflow

### Step 0: Gather User Input

**Ask user for the following information:**

1. **Target Skill Name**: The skill to upgrade (kebab-case)
   - Examples: `factor-migration`, `nix-packaging`, `export-sessions`

2. **Session Filter** (optional): How to find relevant sessions
   - By tag: `auto/skill-name` or `custom/tag`
   - By date range: last N days
   - By keyword: search in session content

Use `AskUserQuestion` tool if not provided in arguments.

### Step 1: Locate Target Skill

Find the skill file to upgrade:

```bash
# Search for skill in bds-dev
ls -la /home/lxb/github/claude.bds/bds-dev/skills/[skill-name]/SKILL.md
```

Read and analyze the current skill content:
- Current sections and structure
- Existing templates and examples
- Documented pitfalls and validation steps

### Step 2: Find Related Sessions

Search for sessions that used this skill:

**Method A: Search by tag**
```bash
grep -l "tags:.*[skill-name]" /var/lib/wonder/warehouse/database/lxb/claude-sessions/lxb/*.md
```

**Method B: Search by skill invocation**
```bash
grep -l "/bds-dev:[skill-name]" /var/lib/wonder/warehouse/database/lxb/claude-sessions/lxb/*.md
```

**Method C: Search by content keywords**
```bash
grep -l "[relevant-keywords]" /var/lib/wonder/warehouse/database/lxb/claude-sessions/lxb/*.md
```

List found sessions with dates and titles for user review.

### Step 3: Analyze Sessions

For each relevant session, extract:

| Category | What to Look For |
|----------|------------------|
| **Workflow Patterns** | Steps that were repeated across sessions |
| **Code Templates** | Exact code that worked well |
| **Edge Cases** | Situations not covered by original skill |
| **Pitfalls** | Errors encountered and how they were resolved |
| **Missing Steps** | Actions added that weren't in the skill |
| **User Questions** | Clarifications needed during execution |

**Analysis Approach:**

1. **Read each session chronologically** - understand the flow
2. **Note deviations from skill** - where did execution differ from documented workflow?
3. **Capture successful solutions** - what code/commands solved problems?
4. **Identify pain points** - where did the user need to intervene or clarify?

### Step 4: Generate Upgrade Recommendations

Compile findings into actionable improvements:

```markdown
## Upgrade Recommendations for [skill-name]

### New Patterns Discovered
1. [Pattern]: [Description]
   - Found in: session-file-1.md, session-file-2.md
   - Suggested addition: [Code or text]

### Missing Steps
1. [Step]: Should be added after Step X
   - Reason: [Why this is needed]

### New Pitfalls
1. **[Pitfall Name]**: [Description and solution]
   - Source: [session-file.md]

### Template Updates
1. [Template Section]: Needs update
   - Current: [current code]
   - Suggested: [improved code]

### Structural Changes
1. [Change]: [Description]
   - Rationale: [Why]
```

### Step 5: Present Changes to User

Before modifying, show the user:
1. Summary of sessions analyzed
2. Key findings by category
3. Proposed changes to SKILL.md

Use `AskUserQuestion` to confirm:
- Which improvements to include
- Any modifications to suggestions
- Whether to proceed with updates

### Step 6: Update Skill File

Apply approved changes to the skill:

1. **Add new workflow steps** - insert at appropriate positions
2. **Update code templates** - replace or add new examples
3. **Expand pitfalls section** - add newly discovered issues
4. **Update verification checklist** - add new checkpoints
5. **Add session references** - document where patterns came from

### Step 7: Validate Updated Skill

After updating, verify:

- [ ] YAML frontmatter is valid
- [ ] All sections properly formatted
- [ ] Code templates are copy-paste ready
- [ ] No duplicate content
- [ ] References are accurate

## Files to Modify

| File | Operation | Purpose |
|------|-----------|---------|
| `bds-dev/skills/[name]/SKILL.md` | Modify | Update skill with improvements |

## Session Analysis Patterns

### Pattern: Repeated Commands

Look for commands executed multiple times across sessions:
```
# If same command appears in 3+ sessions, consider adding to skill
grep -h "Command:" sessions/*.md | sort | uniq -c | sort -rn | head -20
```

### Pattern: Error-Solution Pairs

Find errors and their resolutions:
```
# Look for error messages followed by fixes
grep -B2 -A5 "error\|Error\|ERROR" session.md
```

### Pattern: User Clarifications

Identify where users needed to provide input:
```
# Find AskUserQuestion responses
grep -A3 "User has answered" session.md
```

## Output Format

After upgrading, report:

```
Skill Upgrade Complete: [skill-name]

Sessions Analyzed: N
- session-1.md (YYYY-MM-DD)
- session-2.md (YYYY-MM-DD)

Changes Applied:
1. Added Step 2.5: [Description]
2. New Pitfall: [Name]
3. Updated Template: [Section]
4. Added to Checklist: [Item]

Files Modified:
- bds-dev/skills/[skill-name]/SKILL.md

Backup saved to: /tmp/[skill-name]-SKILL.md.backup
```

## Common Pitfalls

1. **Noise in Sessions**: Sessions may contain unrelated conversation. Focus on sections where the skill was actively used.

2. **Contradictory Patterns**: Different sessions might show different approaches. Prefer the most recent successful pattern.

3. **Over-generalization**: Don't add every one-off solution to the skill. Look for patterns that appear in 2+ sessions.

4. **Breaking Existing Users**: When updating templates, ensure backward compatibility or clearly document breaking changes.

## Verification Checklist

- [ ] Target skill identified and read
- [ ] Relevant sessions found (at least 2)
- [ ] Sessions analyzed for patterns
- [ ] Recommendations reviewed with user
- [ ] Changes applied to skill file
- [ ] Updated skill validates correctly
- [ ] Backup created before modification
