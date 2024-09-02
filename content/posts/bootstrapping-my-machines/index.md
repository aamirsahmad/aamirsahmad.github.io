---
title: "Bootstrapping my machines"
date: 2024-01-23
description: "How I use various tools to automate the process of bootstrapping my machines."
summary: "How I use various tools to automate the process of bootstrapping my machines."
tags: ["tech", "tools"]
---
## Preamble
Every time I get a new MacBook—usually when I switch jobs—I find myself facing the familiar challenge of configuring it just the way I like. The process is tedious, often error-prone, and, to make matters worse, I have to go through it every few years. On top of this, I manage two Linux home servers and one cloud instance for my hobby projects. The effort to "opinionate" my machines has only grown as I keep adding new tools to my daily workflow.

My journey toward automating the bootstrapping of my machines began with a simple list of software and tools, meticulously maintained in a Notion page. I'd use Homebrew for macOS and Apt for Debian to install these tools. Before long, I started writing small Bash scripts—little snippets scattered throughout my Notion page. I even jotted down pseudocode to remind myself of the order in which configurations needed to be applied. Writing a full-fledged Bash script, however, felt overwhelming, so I stuck with my semi-automated solution, hoping one day I'd find the right tool.

Early in my career, I stumbled upon Zsh and Powerlevel10k, which made my shell look incredibly cool and "hippie." But at the time, I had no idea about the underlying complexity of this highly opinionated framework. Over the years, I started craving simplicity and more control over my shell environment, so I ditched Powerlevel and began from scratch. Naturally, I kept a list of must-have Zsh plugins in my trusty Notion page.

Around this time, I also began experimenting with terminal multiplexers like Tmux and Screen on my home servers. I loved how they allowed me to switch between different windows and panes with just a few keystrokes. I was still using iTerm on my Mac, but Tmux enabled me to manage everything within a single iTerm window, making it an indispensable part of my daily routine. After jostling with many terminal applications, I ended up choosing Alacritty which can be customized using a toml file. I even customized the leader key in Tmux, among other things, to better suit my needs—yet another addition to my ever-expanding Notion page.

Then came one cold Toronto night when I decided, with some reluctance, to give Vim a try. At first, I hated everything about it. It slowed me down, made me less productive, and was sheer agony to use. But the pros on the internet assured me that this was a normal reaction during the early learning phase. They were right. Soon, I started to get the hang of Vim motions, and my productivity began to increase, almost exponentially. I found myself enjoying coding in Vim.

Enter Neovim. My world turned upside down when I discovered Neovim. I started tinkering with Neovim's configuration, which is written in Lua. I added numerous plugins to replicate the features I had grown accustomed to in other IDEs like IntelliJ and VSCode. To my surprise, I found Neovim's plugins superior to those of other IDEs. And the low memory footprint? An absolute game-changer.

Throughout this journey, I uncovered a treasure trove of other handy tools, like Lazygit and Delta (a git diff tool), and my curated list of software on Notion grew even longer.

My semi-automated solution for maintaining this list of essential tools started to feel inadequate. It was time to fully automate the process—or at least, that’s what I thought.

## Automation: Phase 1

It occurred to me that I couldn't be the only one facing this challenge, so I turned to the internet, searching for hints and insights into how others were solving it. That's when I stumbled upon the concept of "dotfiles." Intrigued, I meticulously gathered all my dotfiles and placed them in a GitHub repository, which allowed me to clone them across all my machines. However, the process of manually symlinking each one was still incredibly tedious.

Then one day, I discovered GNU Stow—a true breakthrough moment. Stow made the symlinking process a breeze, and suddenly, what once felt like a chore became a simple, almost enjoyable task.

My `dotfiles` directory started looking like this:

```
├── config
│   └── .config
│       ├── alacritty
│       │   └── alacritty.toml
│       ├── nvim
│       │   └── init.lua
│       └── tmux
│           └── tmux.conf
├── vim
│   └── .vimrc
└── zsh
    └── .zshrc
```

```sh
stow config vim zsh
```

While I was mostly satisfied with this solution—especially since my dotfiles were now automated—I still found myself manually applying many other changes that I had carefully curated in my Notion page.

## Automation: Phase 2

I wasn't fully satisfied with my solution, so I decided to write a Bash script. While it got the job done, it was messy and difficult to read. Every time I revisited the script after a few months, I found myself pulling my hair out trying to make sense of it. I wanted something better—something more readable, something declarative. And then, I found Ansible. Another game changer. I migrated my Bash script to Ansible, and everything started to fall into place.

`ansible/bootstrap_mac.yaml`

```yaml
---
- name: Bootstrap Mac
  hosts: hosts
  connection: local
  vars_files:
    - vault/secret.yml

  tasks:
    - name: Set a hostname
      become: true
      ansible.builtin.hostname:
        name: "{{ host_name }}"

    - name: Install core packages with Homebrew
      ansible.builtin.package:
        name:
          - curl
          - git
          - gnupg
        state: latest

    - name: Install packages with Homebrew
      ansible.builtin.package:
        name:
          - alacritty
          - antigen
          - dust
          - firefox
          - fzf
          - gh
          - git-delta
          - gnu-sed
          - jq
          - lazygit
          - neovim
          - notion
          - ripgrep
          - stow
          - tmux
          - zsh
          - ...
        state: latest

    - name: Stow dotfiles
      ansible.builtin.shell:
        cmd: cd $HOME/dotfiles && stow {{ item }}
      loop:
        - config
        - vim
        - zsh
```

```sh
ansible-playbook --become-password-file become.txt --vault-password-file vault.txt -i hosts -l personal bootstrap_mac.yml
```

I decided to create a separate file for Linux as well. 

Moreover, for MacOS, I used the defaults tool to set some OS level configurations which are also belong to the Ansible playbook.

```sh
 defaults write com.apple.dock "tilesize" -int "26" && killall Dock
 defaults write com.apple.dock "autohide" -bool "true" && killall Dock
 defaults write com.apple.dock "orientation" -string "left" && killall Dock
 defaults write com.apple.dock "autohide-time-modifier" -float "0" && killall Dock
 defaults write com.apple.finder "ShowPathbar" -bool "true" && killall Finder
 defaults write com.apple.finder "FXPreferredViewStyle" -string "clmv" && killall Finder
 defaults write com.apple.finder "FXDefaultSearchScope" -string "SCcf" && killall Finder
 defaults write com.apple.finder "FXRemoveOldTrashItems" -bool "true" && killall Finder
 defaults write com.apple.menuextra.clock "DateFormat" -string "\"EEE d MMM h:mm\""
```

To make things easier, I wrapped everything in a bash script:

```sh
#!/bin/bash

if [ "$#" -ne 2 ]; then
    echo "Usage: $0 {mac|linux} {machine_name}"
    exit 1
fi

TYPE=$1
MACHINE_NAME=$2

echo "Bootstrapping $TYPE machine: $MACHINE_NAME..."

PYTHON_VERSION="3.12.5"

BECOME_FILE="ansible/become.txt"
VAULT_FILE="ansible/vault.txt"
INVENTORY="ansible/inventory.ini"

# Function to install build dependencies on Linux
install_linux_dependencies() {
    echo "Installing build dependencies on Linux..."
    sudo apt-get update
    sudo apt-get install -y build-essential python-tk python3-tk tk-dev zlib1g-dev libffi-dev libssl-dev libbz2-dev libreadline-dev libsqlite3-dev liblzma-dev libncurses-dev
}

# Function to install pyenv on all platforms
install_pyenv() {
    echo "Installing pyenv..."
    if ! command -v pyenv &> /dev/null; then
        curl https://pyenv.run | bash
        export PATH="$HOME/.pyenv/bin:$PATH"
        eval "$(pyenv init --path)"
        eval "$(pyenv init -)"
    else
        echo "pyenv is already installed."
    fi

    echo "Installing Python $PYTHON_VERSION..."
    pyenv install $PYTHON_VERSION --skip-existing
    pyenv global $PYTHON_VERSION
}

# Function to install Ansible via pip
install_ansible() {
    echo "Checking if pip is installed..."
    if python3 -m pip -V &> /dev/null; then
        echo "pip is installed."
    else
        echo "pip is not installed. Please install pip before proceeding."
        exit 1
    fi

    echo "Installing Ansible..."
    python3 -m pip install --user ansible
}

# Function to install or update Homebrew on macOS
install_homebrew() {
    echo "Checking if Homebrew is installed..."
    if ! command -v brew &> /dev/null; then
        echo "Installing Homebrew..."
        /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
        eval "$(/opt/homebrew/bin/brew shellenv)"
    else
        echo "Homebrew is already installed. Updating Homebrew..."
        brew update
    fi
}

case $TYPE in
    mac)
        install_homebrew
        install_pyenv
        install_ansible
        ansible-playbook --become-password-file "$BECOME_FILE" --vault-password-file "$VAULT_FILE" -i "$INVENTORY" -l "$MACHINE_NAME" ansible/bootstrap_mac.yml
        echo "Bootstrapping complete ✅"
        ;;
    linux)
        install_linux_dependencies
        install_pyenv
        install_ansible
        ansible-playbook --become-password-file "$BECOME_FILE" --vault-password-file "$VAULT_FILE" -i "$INVENTORY" -l "$MACHINE_NAME" ansible/bootstrap_linux.yml
        echo "Bootstrapping complete ✅"
        ;;
    *)
        echo "Invalid type: $TYPE. Use 'mac' or 'linux'."
        exit 1
        ;;
esac
```

```sh
./bootstrap.sh mac personal
./bootstrap.sh linux homeserver
```

Looking back, it's been quite the journey—from the frustration of manual configurations to discovering tools that transformed the way I manage my machines. Each step brought me closer to the streamlined, automated process I had always envisioned. With Ansible in place, I finally feel like I've cracked the code. But in the ever-evolving world of tech, I'm sure there are more discoveries to be made, more tools to uncover, and more ways to refine my setup. And that's what keeps this journey exciting—there's always something new around the corner.
