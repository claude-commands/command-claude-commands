---
argument-hint: "[install|add|remove|update|list|move] [--user|--project]"
description: "Manage slash commands from the claude-commands GitHub organization"
allowed-tools: ["Bash", "Read", "AskUserQuestion"]
---

# Claude Commands Manager

Manage slash commands from the `claude-commands` GitHub organization.

**If `$ARGUMENTS` is empty or not provided:**

Display usage information:

**Usage:** `/claude-commands <action> [--user|--project]`

**Actions:**

| Action | Description |
|--------|-------------|
| `install` | Install all or selected commands |
| `add <name>` | Add a specific command |
| `remove <name>` | Remove a command |
| `update` | Update all installed commands |
| `list` | Show installed vs available |
| `move <name>` | Move command between user and project levels |

**Scope (optional):**

| Flag | Description |
|------|-------------|
| `--user` | Install for all projects (default) |
| `--project` | Install for current project only |

**Examples:**

```text
/claude-commands install
/claude-commands add start-issue
/claude-commands add start-issue --project
/claude-commands remove codex
/claude-commands move standup --project
/claude-commands update
/claude-commands list
```

**Workflow:**

1. Pre-flight checks (gh CLI, authentication)
2. Detect project root and clone path
3. Fetch available commands from GitHub org
4. Execute requested action with scope awareness

Then proceed to the interactive menu below.

---

**If `$ARGUMENTS` is provided:**

Route to the appropriate operation based on the first argument.

Parse scope flag if present:

- If `--project` is in arguments, set SCOPE="project"
- If `--user` is in arguments, set SCOPE="user"
- Otherwise, SCOPE will be determined interactively

---

## Pre-flight Checks

### 1. Verify gh CLI is installed

```bash
which gh > /dev/null 2>&1 && echo "GH_INSTALLED=true" || echo "GH_INSTALLED=false"
```

**If GH_INSTALLED is "false"**, stop and provide installation instructions:

> The GitHub CLI (`gh`) is required but not installed.
>
> **Install:**
>
> - macOS: `brew install gh`
> - Ubuntu/Debian: `sudo apt install gh`
> - Other: Visit <https://cli.github.com/>
>
> After installing, run `gh auth login` to authenticate.

### 2. Verify gh authentication

```bash
gh auth status > /dev/null 2>&1 && echo "GH_AUTH=true" || echo "GH_AUTH=false"
```

**If GH_AUTH is "false"**, guide user:

> GitHub CLI is not authenticated. Please run:
>
> ```bash
> gh auth login
> ```
>
> Then try `/claude-commands` again.

### 3. Ensure user commands directory exists

```bash
mkdir -p ~/.claude/commands
```

## Discover Project Root

Detect if we're inside a project:

```bash
PROJECT_ROOT=""
if git rev-parse --show-toplevel > /dev/null 2>&1; then
  PROJECT_ROOT=$(git rev-parse --show-toplevel)
  echo "PROJECT_ROOT=$PROJECT_ROOT"
else
  echo "PROJECT_ROOT=NOT_IN_PROJECT"
fi
```

**If user requests `--project` but PROJECT_ROOT is empty**, warn:

> You're not inside a project (no git repository found).
>
> Commands will be installed at user level instead.

## Discover Clone Base Path

Find where this command itself is cloned by reading the symlink:

```bash
SYMLINK_TARGET=$(readlink ~/.claude/commands/claude-commands.md 2>/dev/null)
if [ -n "$SYMLINK_TARGET" ]; then
  # Extract base path (remove /command-claude-commands/claude-commands.md)
  CLONE_BASE=$(dirname "$(dirname "$SYMLINK_TARGET")")
  echo "CLONE_BASE=$CLONE_BASE"
else
  echo "CLONE_BASE=NOT_FOUND"
fi
```

**If CLONE_BASE is "NOT_FOUND"**, this command wasn't installed via symlink. Ask user where to clone commands:

Use AskUserQuestion:

- Question: "Where should commands be cloned?"
- Options:
  - `~/code/claude-commands` - Code directory
  - `~/.local/share/claude-commands` - XDG data location
  - Other - Let user specify

## Discover Available Commands

Fetch all command repos from the organization:

```bash
gh api orgs/claude-commands/repos --paginate --jq '.[] | select(.name | startswith("command-")) | select(.name != "command-claude-commands") | "\(.name)|\(.description)"'
```

This returns lines like:

```text
command-add-feature|Implement feature from issue, add tests, push PR
command-fix-issue|Diagnose issue, create failing test, fix, push PR
```

## Discover Installed Commands

Check both user-level and project-level locations:

```bash
echo "=== User-Level Commands ==="
for cmd_file in ~/.claude/commands/*.md; do
  if [ -L "$cmd_file" ]; then
    cmd_name=$(basename "$cmd_file" .md)
    if [ "$cmd_name" != "claude-commands" ]; then
      echo "USER: $cmd_name"
    fi
  fi
done

echo "=== Project-Level Commands ==="
if [ -n "$PROJECT_ROOT" ] && [ -d "$PROJECT_ROOT/.claude/commands" ]; then
  for cmd_file in "$PROJECT_ROOT/.claude/commands"/*.md; do
    if [ -L "$cmd_file" ]; then
      cmd_name=$(basename "$cmd_file" .md)
      echo "PROJECT: $cmd_name"
    fi
  done
else
  echo "PROJECT: (none - not in a project or no .claude/commands/ directory)"
fi
```

## Action Routing

Based on `$ARGUMENTS`, route to the appropriate action:

- **No arguments or empty**: Show smart menu (first-time vs returning user)
- **`install`**: Install selected or all commands
- **`add <name>`**: Add a specific command
- **`remove <name>`**: Remove a specific command
- **`update`**: Update all installed commands
- **`list`**: Show installed vs available commands
- **`move <name>`**: Move command between user and project scopes

---

## Operation: Default Menu (no arguments)

Determine if this is a first-time user (no other commands installed) or returning user.

**If no commands installed (first-time)**:

Use AskUserQuestion:

- Question: "Welcome to Claude Commands! How would you like to get started?"
- Header: "Setup"
- multiSelect: false
- Options:
  - label: "Install all commands"
    description: "Install all available commands from the organization"
  - label: "Let me choose"
    description: "Select specific commands to install"
  - label: "Just show what's available"
    description: "List commands without installing"

**If commands are installed (returning user)**:

Use AskUserQuestion:

- Question: "What would you like to do?"
- Header: "Action"
- multiSelect: false
- Options:
  - label: "Add more commands"
    description: "Install additional commands to user or project level"
  - label: "Update all"
    description: "Pull latest versions for all installed commands"
  - label: "Remove commands"
    description: "Uninstall commands from user or project level"
  - label: "Move command"
    description: "Move a command between user and project levels"
  - label: "Show status"
    description: "List all commands and their installation scope"

---

## Scope Selection

When installing/adding without an explicit `--user` or `--project` flag, and if inside a project, ask:

Use AskUserQuestion:

- Question: "Where should this command be installed?"
- Header: "Scope"
- multiSelect: false
- Options:
  - label: "User (all projects)"
    description: "Available everywhere via ~/.claude/commands/"
  - label: "Project (this project only)"
    description: "Only in this project via ./.claude/commands/"

**If not in a project**, skip this question and default to user scope.

## Handle Project Directory Creation

If user selects "Project" scope and `$PROJECT_ROOT/.claude/commands/` doesn't exist:

Use AskUserQuestion:

- Question: "Create .claude/commands/ in this project?"
- Header: "Setup"
- multiSelect: false
- Options:
  - label: "Yes, create it"
    description: "Creates $PROJECT_ROOT/.claude/commands/"
  - label: "No, use user-level instead"
    description: "Install to ~/.claude/commands/ instead"

If "Yes", create the directory:

```bash
mkdir -p "$PROJECT_ROOT/.claude/commands"
```

---

## Operation: Install

Install one or more commands.

### If "Install all"

First, ask for scope (see Scope Selection above).

For each available command from the org:

```bash
# Determine target directory
if [ "$SCOPE" = "project" ]; then
  TARGET_DIR="$PROJECT_ROOT/.claude/commands"
else
  TARGET_DIR="$HOME/.claude/commands"
fi

# Clone the repo (if not already cloned)
if [ ! -d "$CLONE_BASE/command-{name}" ]; then
  git clone git@github.com:claude-commands/command-{name}.git "$CLONE_BASE/command-{name}"
fi

# Create symlink
ln -sf "$CLONE_BASE/command-{name}/{name}.md" "$TARGET_DIR/{name}.md"
```

Show progress:

```text
Installing N commands to [user/project] level...

[1/N] add-feature
      Cloning... Done
      Symlinking to ~/.claude/commands/... Done

[2/N] fix-issue
      Already cloned
      Symlinking to ~/.claude/commands/... Done
...
```

### If "Let me choose"

Use AskUserQuestion with multiSelect: true listing all available commands with their descriptions.

Then ask for scope, then install only selected commands.

---

## Operation: Add (add <name>)

Add a single command by name.

**Step 1:** Verify the command exists in the org:

```bash
gh api repos/claude-commands/command-{name} --jq '.name' 2>/dev/null && echo "EXISTS=true" || echo "EXISTS=false"
```

**Step 2:** Determine scope (from flag or ask):

If `--project` flag: SCOPE="project"
If `--user` flag: SCOPE="user"
Otherwise: Ask using Scope Selection (if in a project)

**Step 3:** Set target directory:

```bash
if [ "$SCOPE" = "project" ]; then
  TARGET_DIR="$PROJECT_ROOT/.claude/commands"
else
  TARGET_DIR="$HOME/.claude/commands"
fi
```

**Step 4:** Check if already installed at this scope:

```bash
[ -L "$TARGET_DIR/{name}.md" ] && echo "ALREADY_INSTALLED=true" || echo "ALREADY_INSTALLED=false"
```

**Step 5:** If not installed, clone (if needed) and symlink:

```bash
# Clone if not already present
if [ ! -d "$CLONE_BASE/command-{name}" ]; then
  git clone git@github.com:claude-commands/command-{name}.git "$CLONE_BASE/command-{name}"
fi

# Ensure target directory exists (for project scope)
mkdir -p "$TARGET_DIR"

# Create symlink
ln -sf "$CLONE_BASE/command-{name}/{name}.md" "$TARGET_DIR/{name}.md"
```

**Step 6:** Report success:

```text
Added /{name} at [user/project] level!

Use /{name} to run the command.
```

---

## Operation: Remove (remove <name>)

Remove an installed command.

**Step 1:** Check where the command is installed:

```bash
USER_INSTALLED=false
PROJECT_INSTALLED=false

[ -L ~/.claude/commands/{name}.md ] && USER_INSTALLED=true
[ -n "$PROJECT_ROOT" ] && [ -L "$PROJECT_ROOT/.claude/commands/{name}.md" ] && PROJECT_INSTALLED=true
```

**Step 2:** If not installed anywhere, report error and list what is installed.

**Step 3:** If installed at only one level, or if `--user`/`--project` flag specified:

Use AskUserQuestion:

- Question: "How should '{name}' be removed?"
- Header: "Cleanup"
- Options:
  - label: "Unlink only"
    description: "Remove symlink but keep cloned repo for later"
  - label: "Full removal"
    description: "Remove symlink and delete cloned repository"

**Step 4:** If installed at BOTH levels (and no flag specified):

Use AskUserQuestion:

- Question: "'{name}' is installed at both user and project levels. Which to remove?"
- Header: "Scope"
- multiSelect: false
- Options:
  - label: "User-level only"
    description: "Remove from ~/.claude/commands/"
  - label: "Project-level only"
    description: "Remove from ./.claude/commands/"
  - label: "Both"
    description: "Remove from all locations"

**Step 5:** Execute chosen action:

**Unlink only:**

```bash
rm "$TARGET_DIR/{name}.md"
```

**Full removal:**

```bash
rm "$TARGET_DIR/{name}.md"
# Only remove repo if no other symlinks point to it
rm -rf "$CLONE_BASE/command-{name}"
```

**Step 6:** Report success:

```text
Removed /{name} from [user/project/both] level(s).
```

---

## Operation: Update

Update all installed commands by pulling latest.

**Step 1:** Find all installed commands (symlinks from both locations pointing to CLONE_BASE):

```bash
COMMANDS_TO_UPDATE=""

# User-level
for cmd_file in ~/.claude/commands/*.md; do
  if [ -L "$cmd_file" ]; then
    target=$(readlink "$cmd_file")
    if [[ "$target" == "$CLONE_BASE"* ]]; then
      cmd_name=$(basename "$cmd_file" .md)
      COMMANDS_TO_UPDATE="$COMMANDS_TO_UPDATE $cmd_name"
    fi
  fi
done

# Project-level (if in a project)
if [ -n "$PROJECT_ROOT" ] && [ -d "$PROJECT_ROOT/.claude/commands" ]; then
  for cmd_file in "$PROJECT_ROOT/.claude/commands"/*.md; do
    if [ -L "$cmd_file" ]; then
      target=$(readlink "$cmd_file")
      if [[ "$target" == "$CLONE_BASE"* ]]; then
        cmd_name=$(basename "$cmd_file" .md)
        # Avoid duplicates
        if [[ ! "$COMMANDS_TO_UPDATE" =~ "$cmd_name" ]]; then
          COMMANDS_TO_UPDATE="$COMMANDS_TO_UPDATE $cmd_name"
        fi
      fi
    fi
  done
fi
```

**Step 2:** For each unique installed command:

```bash
cd "$CLONE_BASE/command-{name}"
git fetch origin
LOCAL=$(git rev-parse HEAD)
REMOTE=$(git rev-parse @{u})
if [ "$LOCAL" != "$REMOTE" ]; then
  git pull
  echo "UPDATED: {name}"
else
  echo "UP_TO_DATE: {name}"
fi
```

**Step 3:** Show progress and results:

```text
Checking for updates...

[1/3] add-feature: Updated (3 files changed)
[2/3] fix-issue: Up to date
[3/3] start-issue: Updated (1 file changed)

Update complete! Changes apply to all scopes (user and project).
```

---

## Operation: List

Show all commands and their status, grouped by scope.

**Steps:**

- Fetch available commands from org
- Check installation status at both levels
- Display formatted output:

```text
Claude Commands Status

USER-LEVEL (~/.claude/commands/):
  /add-feature     - Implement feature from issue, add tests, push PR
  /fix-issue       - Diagnose issue, create failing test, fix, push PR

PROJECT-LEVEL (./.claude/commands/):
  /standup         - Generate standup notes from git commits
  /start-issue     - Create git worktree for GitHub issue

AVAILABLE (not installed):
  /prune-worktree  - Clean up completed issue worktrees
  /codex           - Delegate a task to OpenAI Codex CLI

Clone path: /path/to/clone/base
Project root: /path/to/current/project (or "Not in a project")

Run /claude-commands to install more commands.
```

---

## Operation: Move (move <name>)

Move a command between user and project scopes without re-cloning.

**Step 1:** Check current installation:

```bash
USER_INSTALLED=false
PROJECT_INSTALLED=false

[ -L ~/.claude/commands/{name}.md ] && USER_INSTALLED=true
[ -n "$PROJECT_ROOT" ] && [ -L "$PROJECT_ROOT/.claude/commands/{name}.md" ] && PROJECT_INSTALLED=true
```

**Step 2:** If not installed anywhere, report error.

**Step 3:** If `--project` flag specified (move to project):

```bash
# Ensure project directory exists
mkdir -p "$PROJECT_ROOT/.claude/commands"

# Remove from user if present
[ -L ~/.claude/commands/{name}.md ] && rm ~/.claude/commands/{name}.md

# Create project symlink
ln -sf "$CLONE_BASE/command-{name}/{name}.md" "$PROJECT_ROOT/.claude/commands/{name}.md"
```

**Step 4:** If `--user` flag specified (move to user):

```bash
# Remove from project if present
[ -L "$PROJECT_ROOT/.claude/commands/{name}.md" ] && rm "$PROJECT_ROOT/.claude/commands/{name}.md"

# Create user symlink
ln -sf "$CLONE_BASE/command-{name}/{name}.md" ~/.claude/commands/{name}.md
```

**Step 5:** If no flag, ask:

Use AskUserQuestion:

- Question: "Move '{name}' to which scope?"
- Header: "Scope"
- Options:
  - label: "User (all projects)"
    description: "Move to ~/.claude/commands/"
  - label: "Project (this project only)"
    description: "Move to ./.claude/commands/"

**Step 6:** Report success:

```text
Moved /{name} to [user/project] level.
```

---

## Error Handling

### Repository not found

```text
Error: Command '{name}' not found in claude-commands organization.

Available commands:
- add-feature
- fix-issue
- start-issue
- prune-worktree
- codex
```

### Network error

```text
Error: Unable to reach GitHub.

Please check your internet connection and try again.
If the problem persists, verify your gh authentication with: gh auth status
```

### Clone failed

```text
Error: Failed to clone command-{name}.

This might be due to:
- Network issues
- SSH key not configured for GitHub
- Repository access permissions

Try: gh repo clone claude-commands/command-{name}
```

### Already installed

```text
Command '{name}' is already installed at [user/project] level.

To reinstall, first remove it:
  /claude-commands remove {name}
Then add it again:
  /claude-commands add {name}

Or move it to a different scope:
  /claude-commands move {name} --[user|project]
```

### Not in a project

```text
You're not inside a project (no git repository found).

Project-level installation requires being inside a git repository.
Using user-level installation instead.
```
