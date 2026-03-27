# Initial Setup

This section sets up the foundations before you touch any cloud infrastructure: your GitHub organisation, local development environment, and the workflows and conventions used throughout the rest of this guide.

## On This Page, You Will:

- [x] Create a GitHub organisation with proper repository structure
- [x] Configure a local development environment
- [x] Establish a development workflow and branching strategy
- [x] Set up secrets management using 1Password and AWS Secrets Manager
- [x] Configure Claude Code with CLAUDE.md and skills for AI-assisted development

## Prerequisites

Before starting, you should have:

- [A GitHub account](https://github.com/signup) (free tier is fine to start)
- A Mac running macOS (these guides are macOS-focused, but you should still be able to follow along on Windows - some local development steps will differ)
- Basic familiarity with the command line

## Git vs GitHub

Git is a version control tool for tracking changes to text-based files. GitHub remotely hosts git projects and enables collaboration. Git is near-universal; GitHub is the largest host but has competitors including GitLab, Bitbucket, and Azure DevOps.

If git is new to you, take a moment to get familiar with it before continuing. [Learn Git Branching](https://learngitbranching.js.org/) is a good interactive starting point.

## Local Tool Choices

These guides are opinionated about local tooling. The recommended setup is:

- **GitHub** for version control and collaboration
- **VS Code** as the primary IDE
- **iTerm2** for terminal work
- **Homebrew** for package management
- **Pre-commit hooks** for code quality enforcement

You can adapt these to your preferences, but following the recommendations ensures a smooth experience with the rest of the documentation.

## What's Next

Follow these guides in order:

1. **[GitHub Organisation Setup](1-github-setup.md)** - Create repositories, configure teams, set branch protection
2. **[Local Development Environment](2-local-environment.md)** - Install and configure all necessary tools
3. **[Development Workflow](3-development-workflow.md)** - Learn the branching strategy, PR process, and best practices
4. **[Secrets Management](4-secrets-management.md)** - Set up 1Password and AWS Secrets Manager for secure credential handling
5. **[Claude Code Setup](5-claude-code-setup.md)** - Configure CLAUDE.md and skills so Claude Code can work with your repositories
