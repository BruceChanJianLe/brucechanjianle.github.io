---
title: Nix Home Manager
author: cjl
date: 2025-10-26 12:28:11 +0800
categories: [CS]
tags: [cs, linux]
---

> After almost one year from my last Nix journey post.
> Here I am with another one!
> Let's dive into using Nix Home Manager to manager all the Nix packages declaratively!

## Introduction

If you’ve ever found yourself reinstalling your system and painstakingly
re-installing every package, re-tweaking your `.bashrc`, or re-downloading
your favorite CLI tools, you’re not alone.  

Nix Home Manager is a tool that takes Nix’s declarative philosophy and brings
it to your personal environment. It allows you to define everything from your
installed packages to your shell configuration, editor settings, and dotfiles.  

With that said, I am still not full in with Nix Home Manager just yet. My
current approach is only use Home Manager to manage Nix related packages while
using a global Git method to track my dotfiles.  

## Getting Started

For a brand new machine, you can init your home-manager with the
following command, and yes, we are still on `Ubuntu`.  

```bash
nix run home-manager/master -- init --switch
```

The above command will create a `home-manager` directory under your
[`$XDG_CONFIG_HOME`](https://specifications.freedesktop.org/basedir-spec/latest/)
which is usually the `.config` directory.  

If you already have your Home Manager config, like I do! Then you can just
run the following command:  

```bash
nix run home-manager -- switch
```

## Managing Packages

Edit the `home.nix` file and update the `home.packages` session. This file can
be found at `$HOME/.config/home-manager/home.nix`.  

```nix
  home.packages = [
    # Add your nix packages here!
    pkgs.neofetch
  ];
```

Once you are done updating the packages you need, you can simply apply the
home-manager configurations by running:  

```bash
home-manager switch
```

If you are wondering where you can find the available packages, like me,
there are two ways to search.  

The first method, is to go to the official package website and use their
interactive search bar to look for the packages you need.
Visit the link below:  

```bash
https://search.nixos.org/packages
```

The second method, is to use `nix search`, you will need to provide a
`installable` source before the package name.  

```bash
nix search nixpkgs pixi
```

The nice thing about using the cli is that you can provide a few more
keywords as well as to tell it to exclue plugin, like the example below:  

```bash
nix search nixpkgs htop vim -e "plugin"
```

## Updatig Packages

Over time, you may notice that certain packages listed on 
https://search.nixos.org/packages are newer than the ones available in your
current setup. For example, the `pixi` package might be at version **0.46.0**
on the `nixpkgs` **25.05** channel, while the **25.11** channel already provides
version **0.59.0**.  

To upgrade to newer package versions, you need to update your Home Manager
config  to track the newer `nixpkgs` release. Here's how to do it.  

1. Update `home.stateVersion`

The first step is to update the `home.stateVersion` in your Home Manager config.
This setting lives in:  

```bash
cd $HOME/.config/home-manager/home.nix
```

Adjust it to point to the newer release channel:  

```nix
  home.stateVersion = "25.11";
```

2. Update Flake Inputs

Since we are using a flake-based Home Manager config, we can update the
pinned `nixpkgs` version just like any flake project. In the same directory
where the `home.nix` is found, run:

```bash
nix flake update
```

If you track your `flake.lock` file in version control, you’ll see that it has
been updated to reference the latest revision of the channel.  

3. Apply the Updated Configuration

Now that your flake inputs are updated and your `stateVersion` is aligned,
apply your new environment by running:  

```bash
home-manager switch
```

Home Manager will rebuild your environment using the updated packages and configuration.

That’s it! With those steps, your system is now using the latest package
versions from the updated `nixpkgs` channel. Using flakes makes this process
clean, reproducible, and version-controlled.  

## Managing Dotfiles

```nix
{ config, pkgs, ... }:

{
  home.username = "bruce";
  home.homeDirectory = "/home/bruce";

  # Let Home Manager manage itself
  programs.home-manager.enable = true;

  # Packages to install declaratively
  home.packages = with pkgs; [
    git
    neovim
    htop
    jq
  ];

  # Manage dotfiles declaratively
  programs.bash = {
    enable = true;
    shellAliases = {
      ll = "ls -alF";
      gs = "git status";
    };
    promptInit = ''
      export PS1="\u@\h:\w$ "
    '';
  };

  # Example: Configure git user info declaratively
  programs.git = {
    enable = true;
    userName = "Bruce Chan";
    userEmail = "bruce@example.com";
  };
}
```

That’s it — your packages, aliases, and configuration are now managed
declaratively.  

## Sharing Among Multiple Users

If share your dotfiles with others, you may need to manage the dotfiles with
a little flavor of different users.  

**Directory Structure**:  
```bash
.config/home-manager/
├── flake.nix
├── flake.lock
├── common.nix          # Shared config for all users
└── users/
    ├── user1.nix       # User1-specific overrides (optional)
    └── user2.nix       # User2-specific overrides (optional)
```
If you want to have the latest and greatest reference, simply check out the
[link](https://github.com/BruceChanJianLe/dotfiles/tree/master/.config/home-manager)!
But for the impatience, you can simply refer to the `flake.nix`, `common.nix`
and `user1.nix` files below.  

### flake.nix 

```nix
{
  description = "Home Manager configuration of dev";

  inputs = {
    # Specify the source of Home Manager and Nixpkgs.
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    home-manager = {
      url = "github:nix-community/home-manager";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

    outputs =
    { nixpkgs, home-manager, ... }:
    let
      system = "x86_64-linux";
      pkgs = nixpkgs.legacyPackages.${system};

      mkUser = username: home-manager.lib.homeManagerConfiguration {
        inherit pkgs;

        # Specify your home configuration modules here, for example,
        # the path to your home.nix.
        modules = [
          ./common.nix
          {
            home.username = username;
            home.homeDirectory = "/home/${username}";
          }
          # User-specific config if it exists
          (if builtins.pathExists ./users/${username}.nix 
           then ./users/${username}.nix 
           else {})
        ];
      };
    in
    {
      homeConfigurations = {
        "user1" = mkUser "user1";
        "user2" = mkUser "user2";
      };
    };
}
```

### common.nix

```nix
{ config, pkgs, ... }:

{
  # Shared packages across all users
  home.packages = [
    pkgs.neofetch
    pkgs.pixi
  ];

  # Let Home Manager install and manage itself.
  programs.home-manager.enable = true;
}
```

### user1.nix

```nix
{ config, pkgs, ... }:

{
  home.stateVersion = "25.05"; # Please read the comment before changing.

  # The home.packages option allows you to install Nix packages into your
  # environment.
  home.packages = [
    pkgs.htop-vim
  ];

  # Home Manager is pretty good at managing dotfiles. The primary way to manage
  # plain files is through 'home.file'.
  home.file = {
  };

  home.sessionVariables = {
    # EDITOR = "emacs";
  };
}
```

Well, that was a lot to take in! Hopefully, it's something helpful!

## Why Home Manager (and not nix-env)?

Aspect          | `nix-env` (Imperative)                                     | Home Manager (Declarative)
--------------- | ---------------------------------------------------------- | -------------------------------------
Configuration   | You install packages manually (`nix-env -iA nixpkgs.htop`) | You define packages in a `home.nix` file
State tracking  | Nix doesn’t “remember” what you installed                  | Everything is version-controlled
Reproducibility | Hard to reproduce your setup elsewhere                     | 100% reproducible and portable
Rollbacks       | Manual and limited                                         | Built-in rollbacks for config generations
Dotfiles        | Managed manually or with symlinks                          | Managed declaratively and consistently

In short: nix-env is like running apt install, while Home Manager is
like having an aptfile you can share with future-you.


## Useful Features Worth Mentioning

Home Manager is surprisingly deep. Here are some highlights of its
advanced yet practical capabilities:

**1. Version-controlled Configs**

Your entire environment is just code, well Nix code, you can commit it to Git
and reproduce it anywhere with one command.  

**2. Dotfile Management**

No need for symlink farms (stow, chezmoi, etc.). Home Manager directly manages
files in home.file.<path>:

```nix
home.file.".config/nvim/init.vim".source = ./dotfiles/init.vim;
```

Of course, I am still on the fence for this one!  

**3. Program Modules**

Home Manager has modules for many common programs: `zsh`, `starship`, `tmux`,
`alacritty`, `neovim`, `vscode`, and more, all declaratively configurable.
However, for those that are not in the list, you will have to either do a pull
request or wait for the maintainers to get there.  

**4. Per-User Profiles**

Different users on the same system can each have their own
reproducible environments.

**5. Seamless Nix Flake Integration**

For more advanced setups, integrate with flakes for fully reproducible, version-pinned configs:

```nix
homeConfigurations.bruce = home-manager.lib.homeManagerConfiguration {
  pkgs = import nixpkgs { system = "x86_64-linux"; };
  modules = [ ./home.nix ];
};
```

Then update it anywhere with:

```bash
nix run .#homeConfigurations.bruce.activationPackage
```

**6. Automatic Rollbacks**

Home Manager keeps generations, so if something breaks, just roll back:

```bash
home-manager generations
home-manager rollback
```

## Closing Thoughts

Home Manager is a perfect bridge between **Nix-the-package-manager** and
**NixOS-the-declarative-system**. It’s ideal whether you’re:
- a developer who hops between machines,
- a dotfile perfectionist, or
- just someone looking for a declarative way to manage your Nix packages

Anyways, that wraps up this article, as always, keep learning until we meet again!  
