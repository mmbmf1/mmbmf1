<!--
#zsh #terminal #productivity #development #workflow
-->

# Terminal Setup for Flow

## Introduction

Keyboard-first terminal setup with zsh. Stay in flow, avoid mouse clicks, work faster with shortcuts and plugins. A well-configured terminal becomes a powerful productivity tool that keeps you focused on coding. These configurations help you work faster and maintain your flow state.

## The Problem

Reaching for the mouse and clicking through GUIs can break flow and add context switches. Looking for ways to stay on the keyboard. Every time you switch from keyboard to mouse, you lose focus and momentum. GUI tools require visual scanning and clicking, which interrupts your coding flow. You need everything accessible from the keyboard to maintain that deep focus state.

## The Solution

Use zsh plugins and shortcuts to keep everything on the keyboard. Oh My Zsh provides a framework for managing zsh configurations and plugins. Git shortcuts eliminate typing long commands, the `z` command learns your directory habits for instant navigation, and syntax highlighting catches errors before you run commands. Autosuggestions remember your command history and suggest completions.

**Setup:**
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

**`.zshrc` configuration:**
```bash
plugins=(git z zsh-autosuggestions zsh-syntax-highlighting)

PROMPT='%F{cyan}%*%f %F{green}%~%f %F{yellow}$(git branch --show-current 2>/dev/null)%f %# '
```

**Git shortcuts:**
```bash
gco -b new-feature  # git checkout -b
gst                 # git status
ga .                # git add .
gc "message"        # git commit -m
```

**Smart directory jumping:**
```bash
z api        # Instead of cd ~/projects/api/src/components/admin/dashboard
z projects
z dashboard
```

**Workflow example:**
```bash
z api
gst
gco -b new-feature
ga .
gc "Add feature"
```

**Reload config:**
```bash
source ~/.zshrc
```

## Benefits

- **Stay in flow** - No mouse clicks needed. Everything accessible from the keyboard keeps you in that focused coding state without interruptions.
- **Faster** - Shortcuts save keystrokes. Git shortcuts like `gst` instead of `git status` add up over time, making common operations lightning fast.
- **Visual feedback** - Syntax highlighting catches errors before you run commands. Invalid commands are highlighted immediately, preventing mistakes.
- **Smart navigation** - `z` learns your habits. Jump to frequently used directories with just a few characters instead of typing full paths.

This post is separate from the PostgreSQL/Next.js series and focuses on terminal productivity.
