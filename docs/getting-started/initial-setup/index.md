# Getting Started

Welcome! This guide will help you set up everything you need to build a production-grade modern data stack from scratch. We'll start with the fundamentals: setting up your development environment and establishing good workflows.

## What You'll Set Up

By the end of this section, you'll have:

- [x] A GitHub organisation with proper repository structure
- [x] A fully configured local development environment (macOS)
- [x] Understanding of the development workflow and best practices
- [x] A secrets management strategy using 1Password and AWS Secrets Manager
- [x] Claude Code configured with CLAUDE.md and skills for AI-assisted development
- [x] All necessary tools installed and configured

## Prerequisites

Before starting, you should have:

- A GitHub account (free tier is fine to start)
- A Mac running macOS (these guides are macOS-focused)
- Basic familiarity with the command line
- Willingness to learn Git and infrastructure-as-code concepts

## Why This Matters

Building a data stack isn't just about writing code - it's about creating a **sustainable, collaborative, production-ready system**. This requires:

- **Version control**: Track every change, enable collaboration, enable rollback
- **Consistent environments**: Everyone on your team uses the same tools and configurations
- **Code review**: Catch issues before they reach production
- **Automation**: Reduce manual errors and increase deployment speed

Starting with proper foundations saves countless hours of debugging and refactoring later.

## Git vs GitHub

Git is a tool that allows version control of text-based documents. GitHub remotely hosts git based projects and allows collaboration with other team members. Git is ubiquitous and there are very few alternatives. GitHub however, whilst being the largest git host, has several competitors including GitLab, BitBucket, Azure DevOps and many many others.

I will use a number of terms that relate to git management. If git is not something you are familiar with, I thoroughly recommend taking a pause and familiarising yourself with it. There is a good interactive tutorial at [learn git branching](https://learngitbranching.js.org/), or many other blog posts and other teaching aids to learn git with. It's the building block of any data project.

## Approach

These guides are opinionated and prescriptive, recommending specific tools and workflows that work well together:

- **GitHub** for version control and collaboration
- **VS Code** as the primary IDE
- **iTerm2** for terminal work
- **Homebrew** for package management
- **Terraform** for infrastructure as code
- **Pre-commit hooks** for code quality

You can adapt these to your preferences, but following the recommendations ensures a smooth experience with the rest of the documentation.

## What's Next

Follow these guides in order:

1. **[GitHub Organisation Setup](1-github-setup.md)** - Create repositories, configure teams, set branch protection
2. **[Local Development Environment](2-local-environment.md)** - Install and configure all necessary tools
3. **[Development Workflow](3-development-workflow.md)** - Learn the branching strategy, PR process, and best practices
4. **[Secrets Management](4-secrets-management.md)** - Set up 1Password and AWS Secrets Manager for secure credential handling
5. **[Claude Code Setup](5-claude-code-setup.md)** - Configure CLAUDE.md and skills so Claude Code can work with your repositories

Let's get started!
