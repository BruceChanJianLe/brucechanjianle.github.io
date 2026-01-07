---
title: Dotfiles
author: cjl
date: 2024-11-04 18:57:26 +0800
categories: [CS]
tags: [cs, linux]
---

> Setup your dotfiles blazingly fast with ease!

Dotfiles are configuration files for various applications and system settings,
often stored in your home directory and named with a preceding dot (.),
which makes them hidden by default in Unix-based systems.
They are essential for customizing your development environment and workflow!


For a cleaner home directory, it is encouraged that dotfiles are stored into
the `XDG_CONFIG_HOME` directory. There are also recommended practices for user-
specific data files and caches.


VARIABLE NAME | DEFAULT PATH | REMARKS
--- | --- | ---
XDG_CONFIG_HOME | `~/.config` | Stores configuration files.
XDG_DATA_HOME | `~/.local/share` | Stores user-specific data files, like app data that isn’t configuration (e.g., bookmarks or sessions).
XDG_CACHE_HOME | `~/.cache` | Stores cache files, which can be deleted without losing important data.

Now that we understand what dotfiles are, we also want to know why they’re important.
Dotfiles are crucial because they define your personal development environment,
including everything from terminal color themes, shell aliases,
and keybindings to complex tool configurations and plugins.
These settings shape your workflow, tailoring it to your preferences and optimizing productivity.
Having these dotfiles properly configured and tracked enables you
to recreate your environment on a new laptop or system effortlessly,
saving significant setup time and ensuring continuity across devices.

## Quick Demo

Let’s dive straight into a quick demo to see why managing your dotfiles is
absolutely worth the effort. You can get started right away by setting up
my personal dotfiles on your own machine — it works seamlessly on both
`Ubuntu` and `Arch Linux`!  

Here’s how you can do it on `Ubuntu`:


```bash
# Install dependencies
sudo apt install ansible git -y
# Run this script!
ansible-pull -U https://github.com/brucechanjianle/ansible --ask-become-pass
```

That’s it!  

With just two commands, you’ll have `ansible` take care of installing essential
software and setting up all your configuration files.

I use `ansible` to automate the entire environment setup process, it ensures
that every tool, dependency, and configuration lands exactly where it should.
However, for this post, our main focus isn’t on `ansible` itself, but on how
to track and version-control your configuration files - your dotfiles.  

We’ll save the detailed on `ansible` for another time.  

For more information on setting up my dotfiles, please check out my
repo https://github.com/brucechanjianle/ansible.  

## Tracking Your Dotfiles

When it comes to managing your dotfiles, there are several excellent approaches
available, including [Nix Home Manager](https://brucechanjianle.github.io/posts/cs-linux-nix-home-manager)
, GNU Stow, and the classic bare Git repository method.

Personally, I prefer the bare Git approach. It’s simple, intuitive, and gives me
complete control without any extra abstraction layers. All my dotfiles are
tracked in a bare Git repository located at `~/.dotfiles`. If you’re familiar
with Git worktrees, this will feel right at home.  

Here’s how you can set it up yourself:  

```bash
# Initialize a bare Git repository
git init --bare $HOME/.dotfiles

# Hide untracked files from showing up in status
/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME config --local status.showUntrackedFiles no

#Create a convenient alias for managing your dotfiles, do put this in your .bash_aliases file
alias config='/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'
```

Now you can use `config` just like you would use git, but specifically
for your dotfiles.  

Let’s make your first commit!  

```bash
# Track your .bashrc
config add .bashrc

# Commit it to your repository
config commit -m "Batman! Because he has no parents!"

# Push it to the cloud (after adding your remote)
config push
```

And just like that, your configuration is safely version-controlled.  

Next time you set up a new system, you can simply clone your repository and
restore your entire environment in minutes!  

**Pro Tips:**  
If you’re planning to use this method across multiple machines, make sure to
exclude sensitive credentials (like SSH keys or tokens) from your tracked files.
Consider using `.gitignore`, [Git Hooks](https://git-scm.com/book/ms/v2/Customizing-Git-Git-Hooks),
or encryption tools like `ansible-vault` or `git-crypt` for added safety.

## Updating Your Dotfiles

If you have made changes on another computer and decided to push it, you just
need to do a git pull! And probably the git resolve conflicts if there are any.  

```bash
config pull --recurse-submodules
```

## Closing Remarks

Managing your dotfiles may seem like a small thing, but it’s one of the most
impactful ways to take control of your development environment.
Once your setup is tracked and reproducible, switching machines or starting
fresh becomes effortless, just do an `ansible-pull`, and you’re home again.  

Whether you use Nix Home Manager, GNU Stow, or a bare Git repo, the key idea
is the same: make your environment portable, consistent, and yours.
So take the time to version your dotfiles, in turn, your future self
(and every new laptop) will thank you.  

Until then, keep learning and stay curious!  
