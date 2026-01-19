# Local Development Environment

On this page, you will:

- [x] Set up iTerm2 terminal
- [x] Configure shell (Zsh with Oh My Zsh)
- [x] Install Homebrew package manager
- [x] Install and configure VS Code
- [x] Install Git and configure SSH keys
- [x] Install Terraform and related tools

## Overview

Setting up a consistent development environment ensures you have all the necessary tools and can follow along with the rest of the documentation smoothly. This is a highly opinionated set of tools that you could make use of.

!!! info "macOS Focus"
    These instructions are specific to macOS. If you're using Linux or Windows, you'll need to adapt the installation commands, though the concepts remain the same. 
    
    If you are using Windows, it's strongly recommended to make use of the [Windows Subsystem for Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/about). This is because Unix based Operating Systems like Linux and macOS are much better adapted for the development we'll talk about here. Trying to build on Windows can be painful at best.

## Install VS Code

[VS Code](https://code.visualstudio.com/download) is a powerful, free code editor with excellent support for Terraform, Python, and more. It has a number of extensions which really elevate the development experience.

### Install VS Code Command Line Tool

Open VS Code, then:

1. Press **⌘⇧P** to open command palette
2. Type "shell command"
3. Select **"Shell Command: Install 'VS Code' command in PATH"**

Now you can open files and folders from terminal:

```sh
code .  # Opens current directory in VS Code
```

### Recommended VS Code Extensions

These are some excellent extensions to improve your experience. You can download them by clicking on the Extensions button in the side bar on the left. 

#### Language Support

1. [HashiCorp Terraform](https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform) - Terraform support
2. [Red Hat's YAML](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml)
3. [Markdown All in One ](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one)
4. [Microsoft's Python ](https://marketplace.visualstudio.com/items?itemName=ms-python.python)
5. [Microsoft's Jupyter](https://marketplace.visualstudio.com/items?itemName=ms-toolsai.jupyter)
6. [Prettier - Code formatter](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode) - Formatting
7. [Ruff](https://marketplace.visualstudio.com/items?itemName=charliermarsh.ruff) - Formatting

#### Functional

1. [Gitlens - Git supercharged](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens)
2.  [Batch Rename](https://marketplace.visualstudio.com/items?itemName=JannisX11.batch-rename-extension)
3.  [Copy file name](https://marketplace.visualstudio.com/items?itemName=nemesv.copy-file-name)
4. [EditorConfig for VS Code](https://marketplace.visualstudio.com/items?itemName=EditorConfig.EditorConfig)
5. [Todo Tree](https://marketplace.visualstudio.com/items?itemName=Gruntfuggly.todo-tree) - Creates todo lists from comments in code
6. [Claude Code for VS Code](https://marketplace.visualstudio.com/items?itemName=anthropic.claude-code) - optional AI support with [Claude](https://claude.com/product/claude-code)

#### Stylistic, and helpful

1. [vscode-icons](https://marketplace.visualstudio.com/items?itemName=vscode-icons-team.vscode-icons) - make the file icons relate to the type of file it is.
2. [Rainbow CSV](https://marketplace.visualstudio.com/items?itemName=mechatroner.rainbow-csv) - makes it easier to read csv files.

### Configure VS Code Settings

Create a settings file for consistent formatting across projects. In VS Code, press Cmd + Shift + P to open the command palette. Type "Settings" and choose "Preferences: Open User Settings (JSON)"

Add these recommended settings:

```json
{
  // General editor setting
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.rulers": [80, 120],
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "workbench.editor.titleScrollbarSizing": "large",

  // Terraform settings
  "terraform.languageServer.enable": true,
  "[terraform]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true
  },

  // Python settings
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.organizeImports.ruff": "always"
    }
  },
  "ruff.configurationPreference": "filesystemFirst",

  // JSON settings
  "[jsonc]": {
    "editor.defaultFormatter": "vscode.json-language-features"
  },

  // Markdown settings
  "[markdown]": {
    "editor.wordWrap": "on",
    "editor.formatOnSave": false
  },

  // Yaml settings
  "yaml.schemas": {
    "https://squidfunk.github.io/mkdocs-material/schema.json": "mkdocs.yml"
  },
  "yaml.format.enable": true,
  "[yaml]": {
    "editor.insertSpaces": true,
    "editor.tabSize": 2,
    "editor.defaultFormatter": "redhat.vscode-yaml"
  }
}
```

### Optional - Setup Claude Code

Claude is an AI agent that can help you with coding. You can set it up using the docs [here](https://code.claude.com/docs/en/vs-code). It is optional to use, but can be very helpful when programming.

## Install iTerm2

[iTerm2](https://iterm2.com/) is a powerful terminal emulator for macOS with better features than the default Terminal app. A terminal (or shell) is a way of interacting with your computer programmatically. 

You can get the installation instructions [here](https://iterm2.com/).

### Configure iTerm2 (Optional)

1. Open iTerm2
2. Go to **Preferences** (⌘,)
3. Under **Profiles** → **Terminal**, set scrollback lines to **Unlimited**

## Install XCode Command Line Tools

If you haven't done development before, you'll need to install Xcode Command Line Tools. Run:

```sh
xcode-select --install
```

## Configure Shell (Oh My Zsh)

macOS uses Zsh by default. Let's enhance it with [Oh My Zsh](https://ohmyz.sh/) for better productivity.

### Install Oh My Zsh

In iTerm2, type the following and hit enter. You should see Oh-My-Zsh being installed.

```sh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### Configure Oh-My-Zsh settings

Your settings for your shell are all stored in a `~/.zshrc` file. You can open this by typing the following in iTerm:

```sh
code ~/.zshrc
```

Take a look at the [themes](https://github.com/ohmyzsh/ohmyzsh/wiki/Themes) available and pick one you like the look of. You can use it by including the following in your `~/.zshrc`:

```sh
ZSH_THEME="themename"
```

!!! tip "git branch"
  I strongly recommend choosing a theme that includes your git branch in the prompt. This is incredibly useful for development. You can see the themes that include it in the docs, as they will contain `master` or `main`. If you get stuck with the wide range of choice, the default `robbyrussell` is a good one.

Additionally, oh-my-zsh has a number of [plugins](https://github.com/ohmyzsh/ohmyzsh/wiki/Plugins) available to improve the functionality. To use a plugin, often you first have to install it, then you have to add it to the plugins list in your `~/zshrc` file.

Here are some recommended ones:

- [git](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/git)
- [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions)
- [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting)
- [you-should-use](https://github.com/MichaelAquilina/zsh-you-should-use)

We'll install them in the next step.

## Install Homebrew

[Homebrew](https://brew.sh/) is the package manager for macOS, making it easy to install and manage development tools.

### Install Homebrew

```sh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Follow the on-screen instructions. After installation, you may need to add Homebrew to your [PATH](https://www.howtouselinux.com/post/what-is-path-variable-in-linux-and-how-it-works):

```sh
# For Apple Silicon Macs (M1/M2/M3)
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zshrc
eval "$(/opt/homebrew/bin/brew shellenv)"

# For Intel Macs
echo 'eval "$(/usr/local/bin/brew shellenv)"' >> ~/.zshrc
eval "$(/usr/local/bin/brew shellenv)"
```

Verify installation by typing the following - if there are no errors, you have installed it correctly.

```sh
brew --version
```

### Update Homebrew

Keep Homebrew up to date:

```sh
brew update
brew upgrade
```

## Install Git

Git should already be install on macOS, but let's install the latest version via Homebrew:

```sh
brew install git
```

## Configure Git

Set up your Git identity:

```sh
git config --global user.name "Your Name"
git config --global user.email "your.email@company.com"
```

Set default branch name to `main`:

```sh
git config --global init.defaultBranch main
```

Configure line endings (prevents cross-platform issues):

```sh
git config --global core.autocrlf input
```

Set up automatic remote branch creation:

```sh
git config --global push.autoSetupRemote true
```

Set up squash on pull by default

```sh
git config --global pull.rebase false
git config --global pull.ff only
git config --global pull.squash true
```

## Set Up a Global .gitignore

A global `.gitignore` ensures that sensitive or unnecessary files are never committed to any repository on your machine. This is especially important for files like `.env` and `.envrc` that may contain secrets or local configuration.

### Create and Configure Global .gitignore

```sh
# Create a global .gitignore file in your home directory
take ~/.config/git
touch ~/.config/git/.gitignore_global

# Add common patterns
echo -e ".env\n.envrc\n.DS_Store\n.python-version\n__pycache__/\n*.pyc\nnode_modules/\n.vscode/" >> ~/.config/git/.gitignore_global

# Set this file as the global gitignore
git config --global core.excludesfile ~/.config/git/.gitignore_global
```

You can edit `~/.config/git/.gitignore_global` at any time to add more patterns. Make sure `.env` and `.envrc` are included to avoid accidentally committing secrets.

### Set Up SSH Keys for GitHub

Generate an SSH key to securely authenticate with GitHub:

```sh
# Generate SSH key (replace email with your GitHub email)
ssh-keygen -t ed25519 -C "your.email@company.com"

# When prompted, press Enter to accept default location
# Set a passphrase for extra security if required - just hit enter to ignore
```

Start the SSH agent and add your key:

```sh
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

Copy your public key to clipboard:

```sh
pbcopy < ~/.ssh/id_ed25519.pub
```

Add the key to GitHub:

1. Go to [github.com/settings/keys](https://github.com/settings/keys)
2. Click **New SSH key**
3. Title: "MacBook Pro" (or your device name)
4. Paste your key and click **Add SSH key**

Test the connection:

```sh
ssh -T git@github.com
```

You should see: "Hi username! You've successfully authenticated..."

## Install oh-my-zsh plugins

Now you have `git` installed, it's simple to install your oh-my-zsh plugins.

```sh
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
git clone https://github.com/MichaelAquilina/zsh-you-should-use.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/you-should-use
```

Finally, add them to your `~/.zshrc` file - remember you can open it using `code ~/.zshrc`:

```sh
# ~/.zshrc
plugins=(git zsh-autosuggestions zsh-syntax-highlighting you-should-use)
```

To test them, restart your shell by typing:

```sh
exec $SHELL 
```

## Install uv (Python package manager)

[uv](https://github.com/astral-sh/uv) is a fast Python package manager and virtual environment tool, designed as a modern alternative to pip and venv.

Install uv using Homebrew:

```sh
brew install uv
```

Verify installation:

```sh
uv --version
```

You can now use uv to create virtual environments and manage Python dependencies efficiently.

## Install Terraform

[Terraform](https://www.terraform.io/) is the infrastructure-as-code tool we'll use to manage Snowflake, AWS, and other cloud resources.

### Install tfenv (Terraform Version Manager)

Using `tfenv` allows you to easily switch between Terraform versions:

```sh
brew install tfenv
```

### Install Terraform

```sh
# Install the required version (1.14.x)
tfenv install 1.14.3
tfenv use 1.14.3

# Verify installation
terraform version
```

You should see: `Terraform v1.14.3`

### Install Terraform Documentation Tool

```sh
brew install terraform-docs
```

This tool auto-generates documentation for Terraform modules.

## Install AWS CLI

The AWS Command Line Interface is needed for managing Terraform state and AWS resources:

```sh
brew install awscli
```

Verify installation:

```sh
aws --version
```

We'll configure AWS credentials later when we set up the infrastructure.

## Install Pre-commit

[Pre-commit](https://pre-commit.com/) runs automated checks before each commit to catch issues early:

```sh
brew install pre-commit
```

We'll configure pre-commit hooks when we set up the Terraform project.

## Install direnv

[direnv](https://direnv.net/) is a very useful tool to allow you to automatically set up a project environment when you `cd` into the project folder. For example, set environment variables.

```sh
brew install direnv
```

Add direnv hook to your shell profile:

```sh
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
source ~/.zshrc
```

## Verify Your Setup

Run these commands to ensure everything is installed correctly:

```sh
# Package manager
brew --version

# Version control
git --version

# Infrastructure as code
terraform version
terraform-docs --version

# Cloud tools
aws --version

# Code quality
pre-commit --version

# Editor
code --version
```

All commands should return version numbers without errors.

## Optional: Install Additional Tools

### jq (JSON processor)

Useful for parsing API responses and Terraform outputs:

```sh
brew install jq
```

### Tree (Directory visualisation)

Displays directory structures:

```sh
brew install tree
```

### GitHub CLI

Interact with GitHub from the command line. At the prompt, login to GitHub to authenticate.

```sh
brew install gh
gh auth login
```

## Create a Projects Directory

Organise your work in a dedicated directory:

```sh
# Create a directory for all your data projects
mkdir -p ~/projects/data
cd ~/projects/data

# Clone your infrastructure repository (update with your org name)
git clone git@github.com:YOUR-ORG/data-stack-infrastructure.git
cd data-stack-infrastructure
```

!!! success
    Your local development environment is now fully configured and ready for building infrastructure!

## Next Steps

Now that your development environment is ready, learn about the development workflow and best practices.

Continue to [Development Workflow](3-development-workflow.md) →
