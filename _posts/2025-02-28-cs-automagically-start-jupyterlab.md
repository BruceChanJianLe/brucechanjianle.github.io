---
title: Automagically Start Jupyterlab
author: cjl
date: 2025-02-28 14:02:49  +0800
categories: [CS]
tags: [cs, linux, python]
---

> This is for all you data scientists and ai lovers out there!
> What if your Jupyter Lab can quietly start behind the scene
> everytime you start your computer, won't that be great?


Yes, you can automatically start your jupyterlab everytime you boot into
your system. It's not a dream! With what we learned from Linux service,
it is quite easy to do that. But first you will need to install jupyterlab
on your system.

## Installation

If you already have jupyterlab installed on your machine, feel free to skip
this section. Otherwise, I would recommend using nix to install micromamba,
and use micromamba to install jupyterlab.  

Let's install nix and micormamba.  

```bash
# Installing nix
ansible-pull -U https://github.com/brucechanjianle/ansible --ask-become-pass --tags nix
# Installing latest micromamba from nix store (for user)
nix-env -Ai nixpkgs.micromamba
```

And now, you can install jupyterlab with the following command:  

```bash
micromamba create -n jupyterlab jupyterlab -c conda-forge
```

## Setup Linux Service

You will need to create the respective directories for user level service,
meaning these services will only start when user is ready.  

```bash
# Create directories
mkdir ~/.config/systemd/user/ -p
```

Now, add a file and poputate the content!  

```bash
# You can use nano if vim is not something you are familiar with.
vim ~/.config/systemd/user/jupyter_notebook.service 
```

Paste the following content:
```bash
[Unit]
Description=Jupyter Lab Server
After=network.target

[Service]
ExecStart=/home/developer/.nix-profile/bin/micromamba run -n jupyterlab jupyter lab --port 7777 --no-browser
WorkingDirectory=%h/Documents
# Restart=on-failure
Environment=RORT=7777
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
```


Here I set jupyterlan to start from the documents directory,
if you want it to start from somewhere else, please change it by all means.
I used port 7777 so that if you start your own jupyterlab it will be able
to use the default 8888 port. Anyways, we still got a few steps left, let's
continue on.  


Reload user level service.  
```bash
systemctl --user daemon-reload
```

Enable jupyterlan service to start on boot.  

```bash
systemctl --user enable jupyter_notebook.service
```

And now, let start it! Of course, you want to use it right away!  

```bash
systemctl --user start jupyter_notebook.service
```

To validate whether the service has been run succesfully, you can run:  

```bash
systemctl --user status jupyter_notebook.service
```


Well, congratulations! You made it to the end of the article, hopefully it
was a helpful journey. And this is a super fast way to start your service
instead of doing it at the sudo level like some other articles suggested.
Until next time, keep growing!
