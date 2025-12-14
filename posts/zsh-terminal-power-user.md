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
# Open Finder → Navigate folders → Open Git GUI → Click menus
```

## The Solution

Use zsh plugins and shortcuts to keep everything on the keyboard.

**Plugins:**
- `git` - Git shortcuts
- `z` - Smart directory jumping
- `zsh-autosuggestions` - Command suggestions
- `zsh-syntax-highlighting` - Visual validation

**Setup:**
```bash
# Install Oh My Zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Install plugins
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

**`.zshrc` configuration:**
```bash
# ~/.zshrc
plugins=(
  git
  z
  zsh-autosuggestions
  zsh-syntax-highlighting
)

# Custom prompt
PROMPT='%F{cyan}%*%f %F{green}%~%f %F{yellow}$(git branch --show-current 2>/dev/null)%f %# '
```

**Git shortcuts:**
```bash
# Instead of
git checkout -b new-feature
git status
git add .
git commit -m "message"

# Use shortcuts
gco -b new-feature
gst
ga .
gc "message"
```

**Smart directory jumping with `z`:**
```bash
# Instead of
cd ~/projects/api/src/components/admin/dashboard

# Use z (learns your habits)
z api
z projects
z dashboard
```

**Autosuggestions:**
- Gray suggestions appear as you type
- Press right arrow to accept
- Based on command history

**Syntax highlighting:**
- Valid commands turn green
- Invalid commands turn red
- Instant feedback before hitting enter

**Complete workflow example:**
```bash
z api              # Jump to project
gst                # Check status
gco -b new-feature # Create branch
ga .               # Stage all
gc "Add feature"   # Commit
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
- Set once - Configure and forget

This post is separate from the PostgreSQL/Next.js series and focuses on terminal productivity.
