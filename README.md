# Claude Code Custom Commands

A collection of custom command templates for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that enhance development workflows with AI-powered automation.

## What is this?

This repository contains custom commands that extend Claude Code's capabilities. These commands automate common development tasks like code reviews, analysis, and quality checks using AI assistance.

## Setup Instructions

### 1. Install Prerequisites

- **Claude Code**: Install from [Anthropic's official documentation](https://docs.anthropic.com/en/docs/claude-code)
- **GitHub CLI**: Install `gh` for GitHub integration
  ```bash
  # macOS
  brew install gh
  
  # Linux
  curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
  sudo apt update
  sudo apt install gh
  ```

### 2. Clone Repository

Clone this repository to your Claude Code commands directory:

```bash
# Create the commands directory if it doesn't exist
mkdir -p ~/.claude/commands

# Clone the repository
git clone https://github.com/rajatsrma/claude-commands ~/.claude/commands

# Or if you prefer to clone to a different location and symlink:
git clone https://github.com/rajatsrma/claude-commands ~/dev/claude-commands
ln -s ~/dev/claude-commands ~/.claude/commands
```

### 3. Authenticate GitHub CLI

```bash
gh auth login
```

Follow the prompts to authenticate with your GitHub account.

### 4. Verify Installation

Start a Claude Code session in any Git repository and you should see the custom commands available:

```bash
claude-code
# In the Claude session, try:
/help
```

## Available Commands

### 📋 Code Review Commands (`/pr/`)
- **`/pr-review`** - Comprehensive AI-powered PR review
- **`/pr`** - Standard PR analysis with categorized findings  
- **`/pr-review-multi-line`** - Advanced hybrid review with programmatic analysis

[See pr/README.md for detailed documentation](pr/README.md)

### 🚀 More Commands Coming Soon
This repository will expand to include commands for:
- Code quality analysis
- Documentation generation
- Security scanning
- Performance optimization
- Testing automation

## How Commands Work

Claude Code custom commands are Markdown files with:
1. **YAML frontmatter** - Command metadata and configuration
2. **Command logic** - Bash scripts and analysis frameworks
3. **AI prompts** - Instructions for Claude's analysis

## Command Structure Example

```markdown
---
description: "Command description"
argument-hint: "<REQUIRED_ARG> e.g. /command 123" 
allowed-tools: ["Bash(*)"]
---

# Command implementation here
```

## Contributing

1. Fork this repository
2. Create commands following the existing patterns
3. Test thoroughly with various scenarios
4. Submit a pull request with:
   - Command documentation
   - Usage examples
   - Any new prerequisites

## Security Considerations

- All commands follow security-first principles
- Commands are read-only by default (no destructive operations)
- Sensitive data handling follows best practices
- Repository access permissions are respected

## Troubleshooting

### Commands Not Showing Up
- Ensure files are in `~/.claude/commands/` directory
- Check file permissions are readable
- Verify YAML frontmatter is valid
- Restart Claude Code session

### GitHub API Issues  
- Confirm `gh auth status` shows authenticated
- Check repository permissions for PR access
- Verify network connectivity to GitHub

### Tool Execution Errors
- Ensure all prerequisites are installed
- Check command argument format matches `argument-hint`
- Review command output for specific error messages

## Support

- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code)
- [GitHub CLI Documentation](https://cli.github.com/manual/)
- [File Issues](https://github.com/rajatsrma/claude-commands/issues)

## License

MIT License - see LICENSE file for details.