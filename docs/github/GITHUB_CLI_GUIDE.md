# GitHub CLI (gh) Complete Guide

GitHub CLI brings GitHub to your terminal. This comprehensive guide covers installation, authentication, and all major use cases.

## Table of Contents
- [Installation](#installation)
- [Authentication](#authentication)
- [Repository Management](#repository-management)
- [Issues and Pull Requests](#issues-and-pull-requests)
- [GitHub Actions](#github-actions)
- [Releases](#releases)
- [Gists](#gists)
- [Advanced Features](#advanced-features)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)

## Installation

### Ubuntu/Debian
```bash
# Method 1: Using snap (recommended)
sudo snap install gh

# Method 2: Using apt
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install gh
```

### macOS
```bash
# Using Homebrew
brew install gh

# Using MacPorts
sudo port install gh
```

### Windows
```powershell
# Using Chocolatey
choco install gh

# Using Scoop
scoop install gh

# Using winget
winget install --id GitHub.cli
```

### Arch Linux
```bash
sudo pacman -S github-cli
```

### CentOS/RHEL/Fedora
```bash
# CentOS/RHEL
sudo dnf install 'dnf-command(config-manager)'
sudo dnf config-manager --add-repo https://cli.github.com/packages/rpm/gh-cli.repo
sudo dnf install gh

# Fedora
sudo dnf install gh
```

### From Source
```bash
git clone https://github.com/cli/cli.git gh-cli
cd gh-cli
make install
```

### Verify Installation
```bash
gh --version
gh --help
```

## Authentication

### Interactive Login
```bash
# Login with web browser
gh auth login

# Login with specific hostname
gh auth login --hostname github.com

# Login with token from stdin
echo $GITHUB_TOKEN | gh auth login --with-token
```

### Authentication Status
```bash
# Check authentication status
gh auth status

# List authentication details
gh auth status --show-token

# Check specific hostname
gh auth status --hostname github.com
```

### Logout
```bash
# Logout from GitHub
gh auth logout

# Logout from specific hostname
gh auth logout --hostname github.com
```

### Token Management
```bash
# Refresh authentication token
gh auth refresh

# Setup git with GitHub credentials
gh auth setup-git
```

## Repository Management

### Creating Repositories
```bash
# Create a new repository
gh repo create my-project

# Create with description
gh repo create my-project --description "My awesome project"

# Create public repository
gh repo create my-project --public

# Create private repository
gh repo create my-project --private

# Create from template
gh repo create my-project --template owner/template-repo

# Create and clone locally
gh repo create my-project --clone

# Create with .gitignore
gh repo create my-project --gitignore Node

# Create with license
gh repo create my-project --license MIT
```

### Repository Operations
```bash
# Clone a repository
gh repo clone owner/repo-name

# Fork a repository
gh repo fork owner/repo-name

# Fork and clone
gh repo fork owner/repo-name --clone

# View repository
gh repo view owner/repo-name

# View in browser
gh repo view owner/repo-name --web

# List repositories
gh repo list

# List user's repositories
gh repo list username

# List organization repositories
gh repo list orgname

# Search repositories
gh repo list --topic javascript
gh repo list --language python
```

### Repository Settings
```bash
# Archive repository
gh repo archive owner/repo-name

# Delete repository (requires confirmation)
gh repo delete owner/repo-name

# Rename repository
gh repo rename new-name

# Edit repository details
gh repo edit --description "New description"
gh repo edit --homepage "https://example.com"
gh repo edit --add-topic topic1,topic2
gh repo edit --remove-topic topic1
gh repo edit --visibility public
```

## Issues and Pull Requests

### Issues
```bash
# List issues
gh issue list

# List with filters
gh issue list --state open
gh issue list --state closed
gh issue list --assignee @me
gh issue list --author username
gh issue list --label bug
gh issue list --milestone v1.0

# View issue
gh issue view 123

# View issue in browser
gh issue view 123 --web

# Create issue
gh issue create --title "Bug report" --body "Description of the bug"

# Create issue interactively
gh issue create

# Create from template
gh issue create --template bug_report.md

# Edit issue
gh issue edit 123 --title "New title"
gh issue edit 123 --body "New body"
gh issue edit 123 --add-assignee username
gh issue edit 123 --add-label bug,urgent

# Close issue
gh issue close 123

# Reopen issue
gh issue reopen 123

# Pin issue
gh issue pin 123

# Unpin issue
gh issue unpin 123

# Comment on issue
gh issue comment 123 --body "This is a comment"
```

### Pull Requests
```bash
# List pull requests
gh pr list

# List with filters
gh pr list --state open
gh pr list --author @me
gh pr list --assignee username
gh pr list --label enhancement

# View pull request
gh pr view 456

# View PR in browser
gh pr view 456 --web

# Create pull request
gh pr create --title "New feature" --body "Description"

# Create PR interactively
gh pr create

# Create draft PR
gh pr create --draft

# Create PR with reviewers
gh pr create --reviewer username1,username2

# Create PR with assignees
gh pr create --assignee username

# Edit pull request
gh pr edit 456 --title "Updated title"
gh pr edit 456 --add-reviewer username

# Review pull request
gh pr review 456 --approve
gh pr review 456 --request-changes --body "Please fix..."
gh pr review 456 --comment --body "Looks good!"

# Check out PR locally
gh pr checkout 456

# View PR diff
gh pr diff 456

# Check PR status
gh pr status

# Merge pull request
gh pr merge 456

# Merge with squash
gh pr merge 456 --squash

# Merge with rebase
gh pr merge 456 --rebase

# Close PR
gh pr close 456

# Reopen PR
gh pr reopen 456

# Comment on PR
gh pr comment 456 --body "Great work!"
```

## GitHub Actions

### Workflow Management
```bash
# List workflows
gh workflow list

# View workflow
gh workflow view workflow-name

# Enable workflow
gh workflow enable workflow-name

# Disable workflow
gh workflow disable workflow-name

# Run workflow
gh workflow run workflow-name

# Run workflow with inputs
gh workflow run workflow-name --field key=value
```

### Workflow Runs
```bash
# List workflow runs
gh run list

# List runs for specific workflow
gh run list --workflow=ci.yml

# View run details
gh run view 123456789

# View run in browser
gh run view 123456789 --web

# Download run artifacts
gh run download 123456789

# Cancel run
gh run cancel 123456789

# Rerun failed jobs
gh run rerun 123456789 --failed

# Watch run progress
gh run watch 123456789
```

## Releases

### Release Management
```bash
# List releases
gh release list

# View release
gh release view v1.0.0

# View latest release
gh release view --latest

# Create release
gh release create v1.0.0 --title "Version 1.0.0" --notes "Release notes"

# Create release with assets
gh release create v1.0.0 ./dist/*

# Create draft release
gh release create v1.0.0 --draft

# Create prerelease
gh release create v1.0.0 --prerelease

# Upload assets to existing release
gh release upload v1.0.0 ./dist/app.zip

# Download release assets
gh release download v1.0.0

# Delete release
gh release delete v1.0.0
```

## Gists

### Gist Operations
```bash
# List gists
gh gist list

# View gist
gh gist view gist-id

# Create gist from file
gh gist create file.txt

# Create gist from stdin
echo "Hello World" | gh gist create --filename hello.txt

# Create private gist
gh gist create file.txt --secret

# Edit gist
gh gist edit gist-id

# Clone gist
gh gist clone gist-id

# Delete gist
gh gist delete gist-id
```

## Advanced Features

### GitHub API Access
```bash
# Make API requests
gh api repos/owner/repo

# POST request
gh api repos/owner/repo/issues --field title="New issue"

# GET with parameters
gh api repos/owner/repo/issues --field state=closed

# Paginate through results
gh api --paginate repos/owner/repo/issues
```

### Aliases
```bash
# List aliases
gh alias list

# Create alias
gh alias set prs 'pr list --author @me'

# Use alias
gh prs

# Delete alias
gh alias delete prs
```

### Extensions
```bash
# List extensions
gh extension list

# Install extension
gh extension install owner/gh-extension-name

# Upgrade extension
gh extension upgrade owner/gh-extension-name

# Remove extension
gh extension remove owner/gh-extension-name

# Create extension
gh extension create my-extension
```

### SSH Keys
```bash
# List SSH keys
gh ssh-key list

# Add SSH key
gh ssh-key add ~/.ssh/id_rsa.pub --title "My key"

# Delete SSH key
gh ssh-key delete key-id
```

### GPG Keys
```bash
# List GPG keys
gh gpg-key list

# Add GPG key
gh gpg-key add key.gpg

# Delete GPG key
gh gpg-key delete key-id
```

## Configuration

### Global Configuration
```bash
# View configuration
gh config list

# Set configuration
gh config set editor vim
gh config set git_protocol ssh
gh config set prompt enabled

# Get specific config
gh config get editor

# Available configs:
# - editor: preferred text editor
# - git_protocol: https or ssh
# - prompt: enabled or disabled
# - pager: preferred pager
```

### Environment Variables
```bash
# Authentication
export GITHUB_TOKEN=your_token
export GH_TOKEN=your_token

# Configuration
export GH_EDITOR=vim
export GH_BROWSER=firefox
export GH_PAGER=less

# Debugging
export GH_DEBUG=api
```

## Troubleshooting

### Common Issues

#### Authentication Problems
```bash
# Clear cached credentials
gh auth logout
gh auth login

# Check token permissions
gh auth status --show-token

# Refresh token
gh auth refresh
```

#### Network Issues
```bash
# Use different protocol
gh config set git_protocol https
# or
gh config set git_protocol ssh

# Debug API calls
GH_DEBUG=api gh repo list
```

#### Permission Errors
```bash
# Check repository permissions
gh repo view owner/repo

# Verify organization membership
gh api user/memberships/orgs
```

### Debugging
```bash
# Enable debug mode
export GH_DEBUG=api

# View detailed logs
gh --debug command

# Check version and environment
gh --version
gh auth status
gh config list
```

### Getting Help
```bash
# General help
gh help

# Command-specific help
gh repo help
gh pr help create

# Online documentation
gh help --web
```

## Best Practices

### Security
- Use tokens with minimal required permissions
- Regularly rotate authentication tokens
- Use SSH keys for Git operations
- Keep GitHub CLI updated

### Workflow Optimization
- Create aliases for frequently used commands
- Use templates for consistent issue/PR creation
- Leverage GitHub Actions for automation
- Use draft PRs for work in progress

### Team Collaboration
- Establish consistent labeling conventions
- Use issue templates for bug reports
- Set up required reviewers for critical paths
- Document workflows in repository README

## Useful Scripts

### Bulk Operations
```bash
#!/bin/bash
# Clone all repositories from an organization
gh repo list myorg --limit 1000 --json name --jq '.[].name' | \
xargs -I {} gh repo clone myorg/{}
```

```bash
#!/bin/bash
# Close all issues with specific label
gh issue list --label "wontfix" --json number --jq '.[].number' | \
xargs -I {} gh issue close {}
```

### Automation Examples
```bash
#!/bin/bash
# Auto-merge approved PRs
gh pr list --search "is:pr is:open review:approved" --json number --jq '.[].number' | \
xargs -I {} gh pr merge {} --squash
```

## Resources

- [Official Documentation](https://cli.github.com/manual/)
- [GitHub CLI Repository](https://github.com/cli/cli)
- [API Documentation](https://docs.github.com/en/rest)
- [Community Discussions](https://github.com/cli/cli/discussions)

---

*This guide covers GitHub CLI version 2.x. Commands and features may vary between versions.*
