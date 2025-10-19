# Beads Integration for Claude Code

This project uses **Beads** for issue tracking and long-term memory. Beads is a graph-based issue tracker designed specifically for AI coding agents.

## Core Philosophy

**You (Claude) manage all issues automatically.** The user should never need to run `bd` commands directly. You handle:
- Creating issues for discovered work
- Tracking dependencies between tasks
- Finding ready work automatically
- Updating issue status as you work
- Closing completed issues

## Installation & Initialization

### Automatic Installation

**Before using any `bd` commands, check if beads is installed:**

```bash
# Check if bd command exists
if ! command -v bd &>/dev/null; then
  echo "Beads not found. Installing..."

  # Detect OS and install appropriately
  if [[ "$OSTYPE" == "darwin"* ]]; then
    # macOS - use Homebrew or direct binary install
    if command -v brew &>/dev/null; then
      brew install beads
    else
      # Direct binary install for macOS
      curl -L https://github.com/cceckman/beads/releases/latest/download/beads-darwin-amd64 -o /usr/local/bin/bd
      chmod +x /usr/local/bin/bd
    fi
  elif [[ "$OSTYPE" == "linux-gnu"* ]]; then
    # Linux - direct binary install
    curl -L https://github.com/cceckman/beads/releases/latest/download/beads-linux-amd64 -o /usr/local/bin/bd
    chmod +x /usr/local/bin/bd
  else
    echo "Unsupported OS. Please install beads manually from: https://github.com/cceckman/beads"
    exit 1
  fi

  # Verify installation
  if command -v bd &>/dev/null; then
    echo "Beads installed successfully!"
  else
    echo "Failed to install beads. Please install manually."
    exit 1
  fi
fi
```

**If installation fails:**
- Tell the user: "I need to install Beads for issue tracking. Please run: `brew install beads` (macOS) or visit https://github.com/cceckman/beads for installation instructions."
- Do not proceed with beads commands until it's installed

### Project Initialization

**After confirming beads is installed, check if the project is initialized:**

```bash
# Check if beads is initialized in this project
if ! bd ready --json &>/dev/null 2>&1; then
  echo "Initializing beads in this project..."
  bd init
fi
```

### Startup Sequence

**At the start of every work session, run this sequence:**

```bash
# 1. Ensure beads is installed
command -v bd &>/dev/null || { echo "Installing beads..."; /* run installation */ }

# 2. Ensure project is initialized
bd ready --json &>/dev/null 2>&1 || bd init

# 3. Query ready work to orient yourself
bd ready --json | jq -r '.[] | "[\(.id)] \(.title) (P\(.priority))"'
```

This orients you to what needs attention and helps you pick up where you left off.

## Issue Creation Workflow

### When to Create Issues Automatically

Create issues proactively in these situations:

1. **Discovered bugs or problems** while working on something else
   ```bash
   bd create "Fix edge case in auth validation" -t bug -p 1 --deps discovered-from:bd-42 --json
   ```

2. **Technical debt** you notice but can't fix immediately
   ```bash
   bd create "Refactor user model to use repository pattern" -t chore -p 2 --json
   ```

3. **Missing features** mentioned by the user or discovered in code
   ```bash
   bd create "Add password reset functionality" -t feature -p 1 --json
   ```

4. **Blockers** that prevent current work from completing
   ```bash
   bd create "Need API key for email service" -t task -p 0 --json
   ```

5. **Complex tasks that need breaking down** into subtasks
   ```bash
   # Create an epic
   bd create "User Authentication System" -t epic -p 1 --json
   # Create child tasks
   bd create "Implement login form" -t task -p 1 --deps parent-child:bd-15 --json
   bd create "Add password hashing" -t task -p 1 --deps parent-child:bd-15 --json
   ```

### Issue Creation Best Practices

- **Use descriptive titles**: "Fix auth bug" → "Fix race condition in JWT token validation"
- **Set accurate priorities**: 0 (critical) → 4 (nice-to-have)
- **Add context in descriptions**: Link to code locations, error messages, user requests
- **Link dependencies immediately**: Use `--deps` flag when creating related issues
- **Always use --json**: Parse output to get the new issue ID for further operations

## Dependency Management

### Dependency Types

Use the appropriate dependency type for each relationship:

1. **blocks** - Hard blocker (affects ready work)
   ```bash
   bd dep add bd-10 bd-5 --type blocks --json
   # bd-10 cannot start until bd-5 is closed
   ```

2. **parent-child** - Hierarchical (epic → task)
   ```bash
   bd dep add bd-11 bd-8 --type parent-child --json
   # bd-11 is a subtask of epic bd-8
   ```

3. **discovered-from** - Tracks discovered work
   ```bash
   bd dep add bd-12 bd-9 --type discovered-from --json
   # bd-12 was discovered while working on bd-9
   ```

4. **related** - Soft relationship (context only)
   ```bash
   bd dep add bd-13 bd-7 --type related --json
   # bd-13 and bd-7 are related but not blocking
   ```

### Automatic Dependency Tracking

**When working on an issue, automatically link any new issues you create:**

```bash
# Working on bd-42
CURRENT_ISSUE="bd-42"

# Discovered a bug
NEW_ISSUE=$(bd create "Fix memory leak in event handler" -t bug -p 1 --deps discovered-from:$CURRENT_ISSUE --json | jq -r '.id')

# Found that it blocks current work
bd dep add $CURRENT_ISSUE $NEW_ISSUE --type blocks --json
```

## Status Updates

**Update issue status as you work:**

```bash
# Before starting work
bd update bd-42 --status in_progress --json

# During work (add notes)
bd update bd-42 --notes "Implemented OAuth flow, testing edge cases" --json

# After completing
bd close bd-42 --reason "Implemented and tested" --json
```

## Ready Work Detection

**Always check for ready work** when:
- Starting a new conversation
- Completing an issue
- The user asks "what's next?" or "what should we work on?"
- You're unsure what to prioritize

```bash
# Get ready work with full details
bd ready --json | jq -r '.[] | "\n[\(.id)] \(.title)\nPriority: \(.priority) | Type: \(.issue_type) | Assignee: \(.assignee // "unassigned")\n"'

# Get just high-priority ready work
bd ready --priority 0 --json
bd ready --priority 1 --json
```

**Tell the user about ready work proactively:**
"I see there are 3 issues ready to work on. The highest priority is [bd-15] Fix authentication timeout (P0). Would you like me to tackle that first?"

## Responding to User Queries

**Handle common questions about issue status:**

### "What's outstanding?" / "What's open?"

```bash
# Show all open issues with priority
bd list --status open --json | jq -r '.[] | "[\(.id)] \(.title) (P\(.priority), \(.issue_type))"'

# Group by priority
bd list --status open --json | jq -r 'group_by(.priority) | .[] | "\nPriority \(.[0].priority):", (.[] | "  [\(.id)] \(.title)")'
```

Response format: "You have 5 open issues. Here are the high-priority ones: [bd-10] Fix auth bug (P1), [bd-12] Add user settings (P1). Would you like me to work on any of these?"

### "What's completed recently?" / "What did we finish?"

```bash
# Show recently closed issues
bd list --status closed --json | jq -r 'sort_by(.updated_at) | reverse | limit(10; .[]) | "[\(.id)] \(.title) - closed \(.updated_at)"'

# Show issues closed today/this week (requires date filtering in your response)
bd list --status closed --json
```

Response format: "Recently completed: [bd-8] User authentication (closed 2 hours ago), [bd-15] Fix navbar bug (closed yesterday). Great progress!"

### "What am I working on?" / "What's in progress?"

```bash
# Show in-progress issues
bd list --status in_progress --json | jq -r '.[] | "[\(.id)] \(.title) - started \(.updated_at)"'
```

Response format: "You're currently working on [bd-42] Implement dark mode, which was started 3 hours ago."

### "What's blocked?" / "What can't we work on?"

```bash
# Show all blocked issues with their blockers
bd blocked --json | jq -r '.[] | "[\(.id)] \(.title) - blocked by: \(.blocked_by | join(", "))"'
```

Response format: "[bd-20] Add payment integration is blocked by [bd-18] Set up Stripe API keys. We need to complete the blocker first."

### "What's high priority?" / "What's urgent?"

```bash
# Show P0 and P1 issues
bd ready --priority 0 --json
bd ready --priority 1 --json
bd list --status open --json | jq '[.[] | select(.priority <= 1)]'
```

Response format: "There are 2 high-priority issues ready: [bd-10] Fix authentication timeout (P0), [bd-15] Security patch for XSS (P1). The P0 should be addressed immediately."

### "Tell me about [bd-N]" / "What's the status of [bd-N]?"

```bash
# Show full issue details
bd show bd-42 --json | jq -r '"[\(.id)] \(.title)\nStatus: \(.status) | Priority: \(.priority) | Type: \(.issue_type)\nCreated: \(.created_at)\nUpdated: \(.updated_at)\nNotes: \(.notes // "none")"'

# Show dependencies
bd dep tree bd-42
```

Response format: "[bd-42] Implement dark mode is in progress (P2, feature). It has 2 subtasks: [bd-43] Add theme toggle (completed), [bd-44] Update component styles (in progress)."

### "What should I work on next?" / "What's next?"

```bash
# Get highest priority ready work
bd ready --json | jq -r 'sort_by(.priority) | .[0] | "[\(.id)] \(.title) (P\(.priority))"'
```

Response format: "The highest priority ready work is [bd-15] Fix authentication timeout (P0). I recommend starting with that."

### Query Response Best Practices

1. **Always use formatted output** - Parse JSON and present cleanly to the user
2. **Provide context** - Include priority, type, and status in summaries
3. **Offer next steps** - "Would you like me to work on [bd-X]?"
4. **Be concise** - Don't dump all issue details unless asked
5. **Group intelligently** - Group by priority, type, or status when showing multiple issues
6. **Show relationships** - Mention blockers, dependencies, or parent/child relationships when relevant

## Work Session Pattern

Follow this pattern for every work session:

```bash
# 1. Orient yourself - what's ready?
bd ready --json

# 2. Pick the highest priority item (or what user requests)
ISSUE_ID="bd-42"

# 3. Show full details
bd show $ISSUE_ID --json

# 4. Mark as in progress
bd update $ISSUE_ID --status in_progress --json

# 5. Do the work...

# 6. Create issues for any discovered work
bd create "..." --deps discovered-from:$ISSUE_ID --json

# 7. Update with progress notes
bd update $ISSUE_ID --notes "Completed X, Y pending code review" --json

# 8. Close when done
bd close $ISSUE_ID --reason "Implemented successfully" --json

# 9. Check what's ready next
bd ready --json
```

## Multi-Issue Tasks

When the user requests multiple things, create issues for each:

```bash
# User: "Add dark mode, fix the navbar bug, and refactor the API client"

# Create issues
bd create "Add dark mode toggle and theme switching" -t feature -p 2 --json
bd create "Fix navbar z-index causing dropdown overlap" -t bug -p 1 --json
bd create "Refactor API client to use axios interceptors" -t chore -p 2 --json

# Show what was created
bd list --status open --json | jq -r '.[] | "[\(.id)] \(.title)"'
```

Tell the user: "I've created 3 issues for these tasks. Let me start with the highest priority bug [bd-15]."

## Blocked Issues

**Check for blocked issues** and communicate blockers:

```bash
# See what's blocked
bd blocked --json

# If user asks about an issue that's blocked, explain why:
bd show bd-20 --json | jq '.dependencies[] | select(.type=="blocks")'
```

Response: "Issue [bd-20] is blocked by [bd-18] which needs to be completed first. Would you like me to work on the blocker instead?"

## Dependency Visualization

**For complex issues, show the dependency tree:**

```bash
bd dep tree bd-25
```

Use this when:
- User asks about task relationships
- An issue has multiple blockers
- You want to explain why something isn't ready

## JSON Parsing Patterns

**Always parse JSON output for programmatic use:**

```bash
# Get issue ID from creation
ISSUE_ID=$(bd create "Task" --json | jq -r '.id')

# Get all ready work IDs
READY_IDS=$(bd ready --json | jq -r '.[].id')

# Check if specific issue is ready
IS_READY=$(bd show bd-42 --json | jq -r '.status == "open" and (.dependencies | map(select(.type=="blocks")) | length == 0)')

# Get highest priority ready issue
NEXT_ISSUE=$(bd ready --json | jq -r 'sort_by(.priority) | .[0].id')
```

## Error Handling

**Handle errors gracefully:**

```bash
# Check if beads is initialized
if ! bd ready --json &>/dev/null; then
  echo "Beads not initialized. Initializing..."
  bd init
fi

# Check if issue exists before updating
if bd show bd-42 --json &>/dev/null; then
  bd update bd-42 --status in_progress --json
else
  echo "Issue bd-42 not found. Creating new issue..."
fi
```

## Communication Guidelines

### What to Tell the User

**DO tell the user:**
- "I've created issue [bd-42] to track this bug"
- "This is blocked by [bd-15], so I'll work on that first"
- "I found 3 ready issues. The highest priority is [bd-10]"
- "I've completed [bd-42] and moved on to [bd-43]"
- "While working on this, I discovered [bd-50] which I've filed for later"

**DON'T tell the user:**
- Raw JSON output from bd commands
- Internal beads database details
- Every single bd command you're running
- Technical details about SQLite or JSONL storage

### Natural Integration

Make beads invisible to the user. Instead of:
> "I'm going to run `bd create 'Fix bug' --json` to create an issue..."

Say:
> "I've noted this bug for tracking. Let me fix it now..."

The user should feel like you have perfect memory, not like you're managing a database.

## Quick Reference

### Essential Commands

```bash
# Orientation
bd ready --json                                    # What can I work on?
bd blocked --json                                  # What's blocked?
bd list --status open --json                       # All open issues

# Issue lifecycle
bd create "Title" -t TYPE -p PRIORITY --json      # Create
bd show bd-N --json                                # View details
bd update bd-N --status STATUS --json              # Update
bd close bd-N --reason "..." --json                # Complete

# Dependencies
bd dep add bd-N bd-M --type TYPE --json            # Add dependency
bd dep tree bd-N                                   # Visualize tree

# Discovery pattern
bd create "Task" --deps discovered-from:bd-N --json
```

### Types
- `bug` - Something broken
- `feature` - New functionality
- `task` - Work item
- `epic` - Large feature (parent of tasks)
- `chore` - Maintenance/refactoring

### Priorities
- `0` - Critical (do immediately)
- `1` - High (do soon)
- `2` - Medium (normal)
- `3` - Low (when time permits)
- `4` - Nice-to-have (someday/maybe)

### Statuses
- `open` - Not started
- `in_progress` - Currently working
- `closed` - Completed

## Advanced: Git Sync

Beads automatically syncs via git. You don't need to manage this, but be aware:

- `.beads/issues.jsonl` is committed to git
- Changes auto-export after 5 seconds
- `git pull` auto-imports updates
- Conflicts are rare and AI-resolvable

If the user reports beads issues after a git pull:
```bash
bd sync  # Manually sync if needed
```

## Troubleshooting

### Common Issues

1. **"bd: command not found"**
   - Beads is not installed
   - Run the installation sequence from the Installation & Initialization section
   - If automatic installation fails, ask user to install manually:
     - macOS: `brew install beads`
     - Linux: Download from https://github.com/cceckman/beads/releases
   - Verify with: `command -v bd`

2. **"Database not found"**
   ```bash
   bd init  # Initialize in project root
   ```

3. **"Permission denied" during installation**
   - Installation requires sudo for `/usr/local/bin`
   - Tell user: "I need sudo access to install beads. Please run: `sudo curl -L https://github.com/cceckman/beads/releases/latest/download/beads-darwin-amd64 -o /usr/local/bin/bd && sudo chmod +x /usr/local/bin/bd`"

4. **"Cycle detected in dependencies"**
   - Dependencies must form a DAG (no cycles)
   - Remove the problematic dependency
   ```bash
   bd dep remove bd-N bd-M --json
   ```

5. **"Issue already exists"**
   - Check if issue was already created
   ```bash
   bd list --json | jq '.[] | select(.title | contains("search term"))'
   ```

## Remember

1. **Ensure beads is installed** - Check for `bd` command and install if needed
2. **Be proactive** - Create issues as you discover work
3. **Be organized** - Use proper types, priorities, and dependencies
4. **Be transparent** - Tell the user what you're tracking
5. **Be efficient** - Always use `--json` and parse programmatically
6. **Be natural** - Make beads feel invisible to the user

The goal is perfect memory and organization without the user ever thinking about issue tracking.
