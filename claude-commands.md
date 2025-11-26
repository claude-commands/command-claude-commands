---
argument-hint: "[install|add|remove|update|list]"
description: "Manage slash commands from the claude-commands GitHub organization"
allowed-tools: ["Bash", "Read", "AskUserQuestion"]
---

# Claude Commands Manager

Manage slash commands from the `claude-commands` GitHub organization.

**If `$ARGUMENTS` is empty or not provided:**

Display usage information:

**Usage:** `/claude-commands <action>`

**Actions:**
| Action | Description |
|--------|-------------|
| `install` | Install all or selected commands |
| `add <name>` | Add a specific command |
| `remove <name>` | Remove a command |
| `update` | Update all installed commands |
| `list` | Show installed vs available |

**Examples:**
```
/claude-commands install
/claude-commands add start-issue
/claude-commands remove codex
/claude-commands update
/claude-commands list
```

**Workflow:**
1. Pre-flight checks (gh CLI, authentication)
2. Discover clone path from symlink
3. Fetch available commands from GitHub org
4. Execute requested action

Then proceed to the interactive menu below.

---

**If `$ARGUMENTS` is provided:**

Route to the appropriate operation based on the first argument.

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
> - macOS: `brew install gh`
> - Ubuntu/Debian: `sudo apt install gh`
> - Other: Visit https://cli.github.com/
>
> After installing, run `gh auth login` to authenticate.

### 2. Verify gh authentication

```bash
gh auth status > /dev/null 2>&1 && echo "GH_AUTH=true" || echo "GH_AUTH=false"
```

**If GH_AUTH is "false"**, guide user:

> GitHub CLI is not authenticated. Please run:
> ```
> gh auth login
> ```
> Then try `/claude-commands` again.

### 3. Ensure commands directory exists

```bash
mkdir -p ~/.claude/commands
```

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
```
command-add-feature|Implement feature from issue, add tests, push PR
command-fix-issue|Diagnose issue, create failing test, fix, push PR
```

## Discover Installed Commands

Check which commands have symlinks in ~/.claude/commands/:

```bash
for cmd_file in ~/.claude/commands/*.md; do
  if [ -L "$cmd_file" ]; then
    cmd_name=$(basename "$cmd_file" .md)
    # Skip this command itself
    if [ "$cmd_name" != "claude-commands" ]; then
      echo "INSTALLED: $cmd_name"
    fi
  fi
done
```

## Action Routing

Based on `$ARGUMENTS`, route to the appropriate action:

- **No arguments or empty**: Show smart menu (first-time vs returning user)
- **`install`**: Install selected or all commands
- **`add <name>`**: Add a specific command
- **`remove <name>`**: Remove a specific command
- **`update`**: Update all installed commands
- **`list`**: Show installed vs available commands

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
    description: "Install additional commands not yet installed"
  - label: "Update all"
    description: "Pull latest versions for all installed commands"
  - label: "Remove commands"
    description: "Uninstall commands you no longer need"
  - label: "Show status"
    description: "List all commands and their installation state"

---

## Operation: Install

Install one or more commands.

### If "Install all":

For each available command from the org:

```bash
# Clone the repo
git clone git@github.com:claude-commands/command-{name}.git $CLONE_BASE/command-{name}

# Create symlink
ln -sf $CLONE_BASE/command-{name}/{name}.md ~/.claude/commands/{name}.md
```

Show progress:
```
Installing N commands...

[1/N] add-feature
      Cloning... Done
      Symlinking... Done

[2/N] fix-issue
      Cloning... Done
      Symlinking... Done
...
```

### If "Let me choose":

Use AskUserQuestion with multiSelect: true listing all available commands with their descriptions.

Then install only selected commands.

---

## Operation: Add (add <name>)

Add a single command by name.

1. Verify the command exists in the org:
```bash
gh api repos/claude-commands/command-{name} --jq '.name' 2>/dev/null && echo "EXISTS=true" || echo "EXISTS=false"
```

2. Check if already installed:
```bash
[ -L ~/.claude/commands/{name}.md ] && echo "ALREADY_INSTALLED=true" || echo "ALREADY_INSTALLED=false"
```

3. If not installed, clone and symlink:
```bash
git clone git@github.com:claude-commands/command-{name}.git $CLONE_BASE/command-{name}
ln -sf $CLONE_BASE/command-{name}/{name}.md ~/.claude/commands/{name}.md
```

4. Report success:
```
Added! Use /{name} to run the command.
```

---

## Operation: Remove (remove <name>)

Remove an installed command.

1. Check if installed:
```bash
[ -L ~/.claude/commands/{name}.md ] && echo "IS_INSTALLED=true" || echo "IS_INSTALLED=false"
```

2. If not installed, report error and list what is installed.

3. If installed, use AskUserQuestion:
- Question: "How should '{name}' be removed?"
- Header: "Cleanup"
- Options:
  - label: "Unlink only"
    description: "Remove symlink but keep cloned repo for later"
  - label: "Full removal"
    description: "Remove symlink and delete cloned repository"

4. Execute chosen action:

**Unlink only:**
```bash
rm ~/.claude/commands/{name}.md
```

**Full removal:**
```bash
rm ~/.claude/commands/{name}.md
rm -rf $CLONE_BASE/command-{name}
```

5. Report success:
```
Removed /{name} command.
```

---

## Operation: Update

Update all installed commands by pulling latest.

1. Find all installed commands (symlinks pointing to CLONE_BASE):
```bash
for cmd_file in ~/.claude/commands/*.md; do
  if [ -L "$cmd_file" ]; then
    target=$(readlink "$cmd_file")
    if [[ "$target" == "$CLONE_BASE"* ]]; then
      cmd_name=$(basename "$cmd_file" .md)
      echo "$cmd_name"
    fi
  fi
done
```

2. For each installed command:
```bash
cd $CLONE_BASE/command-{name}
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

3. Show progress and results:
```
Checking for updates...

[1/3] add-feature: Updated (3 files changed)
[2/3] fix-issue: Up to date
[3/3] start-issue: Updated (1 file changed)

Update complete!
```

---

## Operation: List

Show all commands and their status.

1. Fetch available commands from org
2. Check installation status of each
3. Display formatted table:

```
Claude Commands Status

INSTALLED:
  /add-feature     - Implement feature from issue, add tests, push PR
  /fix-issue       - Diagnose issue, create failing test, fix, push PR

AVAILABLE (not installed):
  /start-issue     - Create a new git worktree for a GitHub issue
  /prune-worktree  - Clean up completed issue worktrees
  /codex           - Delegate a task to OpenAI Codex CLI

Clone path: <clone-path>

Run /claude-commands to install more commands.
```

---

## Error Handling

### Repository not found
```
Error: Command '{name}' not found in claude-commands organization.

Available commands:
- add-feature
- fix-issue
- start-issue
- prune-worktree
- codex
```

### Network error
```
Error: Unable to reach GitHub.

Please check your internet connection and try again.
If the problem persists, verify your gh authentication with: gh auth status
```

### Clone failed
```
Error: Failed to clone command-{name}.

This might be due to:
- Network issues
- SSH key not configured for GitHub
- Repository access permissions

Try: gh repo clone claude-commands/command-{name}
```

### Already installed
```
Command '{name}' is already installed.

To reinstall, first remove it:
  /claude-commands remove {name}
Then add it again:
  /claude-commands add {name}
```
