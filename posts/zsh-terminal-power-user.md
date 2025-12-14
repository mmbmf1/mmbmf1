<!--
#zsh #terminal #productivity #development #workflow
-->

# Terminal Setup for Flow

## Introduction

Keyboard-first terminal setup with zsh. Stay in flow, avoid mouse clicks, work faster with shortcuts and plugins.

## The Problem

Reaching for the mouse and clicking through GUIs breaks flow and adds context switches.

```bash
# Slow - GUI navigation
```

## The Solution

Use zsh plugins and shortcuts to keep everything on the keyboard.

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

- Stay in flow - No mouse clicks
- Faster - Shortcuts save keystrokes
- Visual feedback - Syntax highlighting catches errors
- Smart navigation - `z` learns your habits

This post is separate from the PostgreSQL/Next.js series and focuses on terminal productivity.
