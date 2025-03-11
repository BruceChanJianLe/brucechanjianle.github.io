---
title: Nix Journey
author: cjl
date: 2024-10-27 15:22:21 +0800
categories: [CS]
tags: [cs, linux]
---

> After hearing about Nix OS, it seems like an interesting thing to look into.
> However, I am not currently ready to leave Ubuntu at the moment.
> Hence, here are my notes on using Nix on Ubuntu.


## Installation of Nix

Using official installation steps: https://nixos.org/download/#nix-install-linux


> What the script would do:
> 
> - make sure your computer doesn't already have Nix files
>  (if it does, I will tell you how to clean them up.)
> - create local users (see the list above for the users I'll make)
> - create a local group (nixbld)
> - install Nix in /nix
> - create a configuration file in /etc/nix
> - set up the "default profile" by creating some Nix-related files in
>  /root
> - back up /etc/bash.bashrc to /etc/bash.bashrc.backup-before-nix
> - update /etc/bash.bashrc to include some Nix configuration
> - back up /etc/zsh/zshrc to /etc/zsh/zshrc.backup-before-nix
> - update /etc/zsh/zshrc to include some Nix configuration
> - load and start a service (at /etc/systemd/system/nix-daemon.service
>  and /etc/systemd/system/nix-daemon.socket) for nix-daemon


The script is quite intuitive, just read through the prompt carefully,
and response as required.


If you want a quick way, you can use my ansible script with the command below:

```bash
ansible-pull https://github.com/BruceChanJianLe/ansible-nix.git -K
```

## Unistalling Nix

This is for multi-user nix, which is the recommended one, likely yours would
be this as well.  
```bash
# Remove Nix daemon service
sudo systemctl stop nix-daemon.service
sudo systemctl disable nix-daemon.socket nix-daemon.service
sudo systemctl daemon-reload

# Remove files created by Nix:
sudo rm -rf /etc/nix /etc/profile.d/nix.sh /etc/tmpfiles.d/nix-daemon.conf /nix ~root/.nix-channels ~root/.nix-defexpr ~root/.nix-profile

# Remove build users and their groups
for i in $(seq 1 32); do
  sudo userdel nixbld$i
done
sudo groupdel nixbld
```

Other places where you may want to clean up:
- `/etc/bash.bashrc`
- `/etc/bashrc`
- `/etc/profile`
- `/etc/zsh/zshrc`
- `/etc/zshrc`


link: https://nix.dev/manual/nix/2.18/installation/uninstall


## Setting Up Experimental Feature

Well, nix commands and flakes belongs to the experimental features, which
is what the people are using nix for, so let's learn how we can enable them.


Basically, there are two ways to enable it. System-wide method, which all users
will have the experimental features. Or you could do it per-user based.
Let's explore the method for system-wide first.

```bash
sudoedit /etc/nix/nix.conf
```

Add this line to it..
```conf
experimental-features = nix-command flakes
```


For user only method, which is what I would recommend:

```bash
mkdir ~/.config/nix/ -p
vim ~/.config/nix/nix.conf
```

Add this line to it..
```conf
experimental-features = nix-command flakes
```

## Installing Packages

You can use `nix-env` as a package manager to install
some unavailable packages on your distro, for example,
`htop-vim` which is `htop` but with `vim` key-bindings.
Let's see that in action.

```bash
nix-env -Ai nixpkgs.htop-vim
```

To see what packages you have installed, you can query it like so:

```bash
nix-env --query
```

To uninstall a package, you can run the following command:

```bash
nix-env --uninstall htop-vim-unstable
```

You may notice that the *unstable* keyword, it means that the package
is coming from the unstable channel or repository in the Nix ecosystem.
In the Nix package management system, there are multiple channels (repositories)
that provide packages. The most commonly used ones are:

- **Stable**: This channel provides well-tested, stable packages
              that are considered to be production-ready.
- **Unstable**: This channel includes newer packages and versions that are not yet
                fully tested or guaranteed to be stable. It can contain packages that
                are still under development, may have unresolved bugs, or may change frequently.

## Summary

In Summary, there is really more to learn than what we see here!
I will share more along the way as we continue this journey together.
Until then, keep learning!

## Reference

- https://tech.aufomm.com/my-nix-journey-use-nix-with-ubuntu/
- https://www.youtube.com/watch?v=BwEIXIjLTNs
