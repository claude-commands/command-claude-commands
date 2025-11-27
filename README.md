# /claude-commands

Manage slash commands from the [claude-commands](https://github.com/claude-commands) GitHub organization.

This is the **meta-command** - install it once, then use it to discover, install, update, and remove all other commands.

## Installation

```bash
# Clone to your preferred location
git clone git@github.com:claude-commands/command-claude-commands.git <clone-path>/command-claude-commands

# Symlink (use full path to cloned repo)
ln -s <clone-path>/command-claude-commands/claude-commands.md ~/.claude/commands/claude-commands.md
```

## Usage

### Interactive Mode

Just run `/claude-commands` in Claude Code. It will detect whether you're a first-time user or have
commands installed, and present appropriate options.

**First-time users** see:

- Install all commands
- Select specific commands
- Browse available commands

**Returning users** see:

- Add more commands
- Update all
- Remove commands
- Move command (between scopes)
- Show status

### Direct Commands

| Command | Description |
|---------|-------------|
| `/claude-commands` | Interactive menu |
| `/claude-commands install` | Install commands (interactive selection) |
| `/claude-commands add <name>` | Add a specific command |
| `/claude-commands remove <name>` | Remove a specific command |
| `/claude-commands update` | Update all installed commands |
| `/claude-commands list` | Show installed vs available commands |
| `/claude-commands move <name>` | Move command between user and project levels |

### Installation Scopes

Commands can be installed at two levels:

| Scope | Location | Availability |
|-------|----------|--------------|
| **User** | `~/.claude/commands/` | All projects (default) |
| **Project** | `./.claude/commands/` | Current project only |

**Single source of truth:** Command repos are always cloned to one global location. Only the symlinks
differ based on scope. This means updates apply to all installations automatically.

### Scope Flags

Add `--user` or `--project` to any command:

```bash
# Install at user level (available everywhere)
/claude-commands add start-issue --user

# Install at project level (only this project)
/claude-commands add start-issue --project

# Move from user to project level
/claude-commands move standup --project

# Move from project to user level
/claude-commands move standup --user
```

If no flag is provided and you're inside a project, you'll be asked which scope to use.

### Examples

```bash
# Install all available commands (asks for scope)
/claude-commands install

# Add just the start-issue command at user level
/claude-commands add start-issue

# Add standup for this project only
/claude-commands add standup --project

# Update all installed commands (both scopes)
/claude-commands update

# Remove a command
/claude-commands remove codex

# Move a command from user to project
/claude-commands move explain --project

# See what's installed at each level
/claude-commands list
```

## How It Works

1. **Zero config** - The command discovers where it's cloned by reading its own symlink
2. **GitHub API** - Available commands are fetched live from the organization
3. **Symlink-based installation** - Commands are cloned once and symlinked to user or project level
4. **Scope detection** - Automatically detects if you're in a project (git repository)

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- [GitHub CLI](https://cli.github.com/) (`gh`) - authenticated
- Git with SSH access to GitHub

## Updates

```bash
cd <clone-path>/command-claude-commands && git pull
```

Or use another instance of `/claude-commands update` if you have other commands installed.
