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

Just run `/claude-commands` in Claude Code. It will detect whether you're a first-time user or have commands installed, and present appropriate options.

**First-time users** see:
- Install all commands
- Select specific commands
- Browse available commands

**Returning users** see:
- Add more commands
- Update all
- Remove commands
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

### Examples

```bash
# Install all available commands
/claude-commands install

# Add just the start-issue command
/claude-commands add start-issue

# Update all installed commands
/claude-commands update

# Remove a command
/claude-commands remove codex

# See what's installed
/claude-commands list
```

## How It Works

1. **Zero config** - The command discovers where it's cloned by reading its own symlink
2. **GitHub API** - Available commands are fetched live from the organization
3. **Symlink-based installation** - Commands are cloned and symlinked to `~/.claude/commands/`

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- [GitHub CLI](https://cli.github.com/) (`gh`) - authenticated
- Git with SSH access to GitHub

## Updates

```bash
cd <clone-path>/command-claude-commands && git pull
```

Or use another instance of `/claude-commands update` if you have other commands installed.
