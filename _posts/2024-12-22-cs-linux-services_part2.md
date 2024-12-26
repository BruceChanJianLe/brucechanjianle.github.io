---
title: Linux Services (Part 2)
author: cjl
date: 2024-12-22 21:02:07 +0800
categories: [CS]
tags: [cs, linux]
---

> In this post we shall talk more about config files for services
> and also write a few simple services of our own as a practice!

## Service File

A service file is divided into three primary sections - Unit, Service and Install.
It is important that we don't change the syntax of a service file, it is case
sensitive! 


Let's understand more what each section does.

### Unit Secion

Directive | Description | Example | Notes
--- | --- | --- | ---
`Description=` | A brief explanation of what the service does. | `Description=Hello World Service` | This text is visible in systemctl status.
`After=` | Specifies dependencies for starting the service. The service starts only after these units. | `After=network.target` | Ensures that the specified units are started before this service.
`Requires=` | Lists units that are essential for the service. If a required unit fails, the service stops. | `Requires=basic.target` | Implies a hard dependency between the service and the listed units.
`Wants=` | Specifies units that should start alongside the service but are not mandatory. | `Wants=network-online.target` | A softer dependency; failure of these units won’t stop the service.
`Conflicts=` | Specifies units that should not run simultaneously with this service. | `Conflicts=foo.service` | Stops the conflicting service if it’s running.
`Before=` | Specifies that this unit must start before the listed units. | `Before=multi-user.target` | Used to control the startup order.
`OnFailure=` | Lists units to trigger if this service fails. | `OnFailure=alert-email.service` | Typically used to send alerts or take corrective actions.

### Service Secion

Directive | Description | Example | Notes
--- | --- | --- | ---
`ExecStart=` | The command that starts the service. | `ExecStart=/usr/bin/python3 /usr/local/bin/hello_service.py` | Runs the specified command or script to start the service.
`Restart=` | Determines the behavior when the service exits or fails. | `Restart=always` | Options: always, on-failure, no. always ensures the service is restarted in all cases.
`User=` and Group= | Specifies the user and group under which the service runs. | `User=nobody Group=nogroup` | Running as a non-root user improves security.
`WorkingDirectory=` | Sets the directory from which the service starts. | `WorkingDirectory=/var/www` | Helps ensure the service runs from a specific location.
`Environment=` | Sets environment variables for the service. | `Environment="MY_ENV_VAR=value"` | Can set multiple variables for configuration or context.

### Install Secion

Directive | Description | Example | Notes
--- | --- | --- | ---
`WantedBy=` | Specifies the target under which the service will be started or enabled. | `WantedBy=multi-user.target` | Ensures the service starts in a specific target like multi-user.target or graphical.target.
`Alias=` | Specifies alternative names for the service. | `Alias=my-service-alias.service` | Allows the service to be referenced using different names.

## The Basics

Before going into writing our own service, let's have a look at some basic 
commands for Linux services.

Action | Command
--- | ---
Start a service | `sudo systemctl start <service_name>`
Stop a service | `sudo systemctl stop <service_name>`
Restart a service | `sudo systemctl restart <service_name>`
Check the status of a service | `systemctl status <service_name>`
Enable a service to start at boot | `sudo systemctl enable <service_name>`
Disable a service from starting at boot | `sudo systemctl disable <service_name>`
Reload a service, some service provides this as a way to read updated config files | `sudo systemctl reload <service_name>`

## Example Service (SSH)

A practical service to have on your system would be SSH. It is a secure shell
that allows you to access remotely to another system in with a text interface.
If you haven't have OpenSSH installed on your Ubuntu, you could easily install
it by running the command below:

```bash
sudo apt install ssh
```

The config file for ssh is kept in `/etc/ssh/sshd_config` file.
```bash
sudoedit /etc/ssh/sshd_config
```

This is where you would want to update your ssh config, for example, by default
ssh uses port 22, which you can change it to another port. Of course, only if
you have the actual reason to do so.

Once something is updated in the config file, you will need to
restart the ssh server for the changes to take effect.
```bash
sudo systemctl restart sshd
```

Great! Now you know how to configure a service! Let's try to write your own
service file.

## Example of Your Own Service (SYSTEM-WIDE)

Well, hold up! Before we can get there, we first need to address the difference
between the service files in the `etc` directory and `~/.config`.

Directory | Scope | Purpose | Access Control | Precedence
--- | --- | --- | --- | ---
`/etc/systemd/system` | System-wide | Contains system services that can be enabled/disabled and started/stopped by administrators for all users. | Requires root privileges for modifications. | Overrides `/usr/lib/systemd/system` files.
`/etc/systemd/user` | User service (global) | Contains user services that apply to all users on the system. | Requires root privileges for modifications. | Overrides `/usr/lib/systemd/user` files.
`/lib/systemd/system` | System-wide (default) | Contains default service files provided by the OS or software packages. | Typically read-only; requires root privileges for modifications. | Lowest precedence; overridden by `/etc/systemd/system`.
`~/.config/systemd/user` | User service (personal) | Contains user-defined services specific to a single user, useful for managing personal applications and tasks. | Can be modified by the owning user. | Overrides both `/usr/lib/systemd/user` and `/etc/systemd/user`.

Alrighty! Now that we know the difference, let's do a hello.service! We will
need to create the file first in `/etc/systemd/system`.


```bash
sudoedit /etc/systemd/system/hello.service
```


Now, add these lines into the file and save it! We are using `sudoedit` here
because the files in `/etc/systemd/system` require root privileges.


```systemd
[Unit]
Description=Hello World Service
After=network.target

[Service]
ExecStart=/usr/bin/python3 /usr/local/bin/hello_service.py
# StandardOutput=append:/var/log/hello_service.log
# StandardError=append:/var/log/hello_service_error.log

[Install]
WantedBy=multi-user.target
```


This service file will start run a python script that will say "Hello, World!"
every 5 seconds. But since it is a daemon, we won't be seeing the printout and
will just have to believe that it is doing the thing.


For the python script, let do:
```bash
sudoedit /usr/local/bin/hello_service.py
```


Now populate the script with the following content:
```python
#!/usr/bin/env python3

# This is just a dummy service
# Echo out "Hello, World!" every 5 seconds
import time

while True:
    print("Hello, World!")
    time.sleep(5)
```


Once you are done, we will need to tell `systemd` to read in changes with:
```bash
sudo systemctl daemon-reload
```


To run the service that we have just created:
```bash
sudo systemctl start hello
```


To check the status of our service
```bash
sudo systemctl status hello
```


To stop our service
```bash
sudo systemctl stop hello
```


Finally, if you want this service to start by default when you boot up your
system. You can:
```bash
sudo systemctl enable hello
```


When you enable a service, `systemd` will symlink your service into 
the wants folder for the target specified in wanted by. And you can manage
it with `systemctl`.


## Example of Your Own Service (USER-SERVICE)

Well, sometimes we don't always need a system wide service, for example,
you just need to start a tmux script when the user has started
his or her session! I chose tmux because it is particularly challenging
for beginners and also it is something very practical. Say you have a robot,
and you have a tmux script that can startup the entire robot system,
this would be a great way to do so.


Firstly, let's create the required directories for `systemd`.
```bash
mkdir ~/.config/systemd/user
touch ~/.config/systemd/user/my_tmux.service
```


Now grab a template tmux script from [here](https://github.com/BruceChanJianLe/tmux/blob/master/tmux_template.bash).
And place it in the `tmux_conf` directory inside of you home directory.
Or you could simply run the commands below:
```bash
mkdir ~/tmux_conf
cd ~/tmux_conf
curl -O https://raw.githubusercontent.com/BruceChanJianLe/tmux/refs/heads/master/tmux_template.bash
```


Now edit `my_tmux.service` file and populate the content:
```systemd
[Unit]
Description=tmux session for user %u
After=default.target

[Service]
Type=forking
ExecStart=%h/tmux_conf/tmux_template.bash
ExecStop=/usr/local/bin/tmux kill-session -t tmux_template

[Install]
# Start when the user session starts
WantedBy=default.target
```


> Note that you can only have one tmux service, otherwise, only the first service will work.
> And the consecutive ones will not work unless with some minor tweak on the service file by
> setting `RemainAfterExit=true`, but the service will not be tracked correctly after so.

For user `systemctl`, you will need to add a flag. Let's have a look at reloading
the service files.

```bash
systemctl --user daemon-reload
```


Once our service file is ready to use, let's enable it and reboot our computer.
```bash
systemctl --user enable my_tmux
reboot
```


And voilà! If you check `my_tmux` service, it should be up and running!
```bash
systemctl --user status my_tmux
```


In summary, we have learned about service files, managing a common service
(secure shell), look at how to write our own service file for system wide and
user. Until then, keep learning!

## Reference

- https://bbs.archlinux.org/viewtopic.php?id=247292
- https://linuxhandbook.com/create-systemd-services/
- https://wiki.archlinux.org/title/Tmux#Autostart_with_systemd
- https://naftuli.wtf/2017/12/28/systemd-user-environment/
