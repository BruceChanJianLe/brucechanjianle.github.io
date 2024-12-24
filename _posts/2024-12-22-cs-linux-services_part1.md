---
title: Linux Services Part 1
author: cjl
date: 2024-12-22 12:38:58 +0800
categories: [CS]
tags: [cs, linux]
---

> Ahh...Linux services, cannot talk about them without mentioning systemd!
> Let's go over some theoretical things of Linux services in the first part,
> and we shall have a more practical example on the second part!

Linux services are vital components of the system,
responsible for ensuring various functionalities run smoothly.
They are managed through init systems like `systemd` and can be tailored
to meet the specific needs of the system or user.


Let's have a look at some importances of linux services!

## Linux Services

<details>
  <summary>
    <strong>
      1. Background Processes
    </strong>
  </summary>
<ul>
  <li>
    Services typically run in the background, meaning they do not
    interact with the user directly through the terminal of GUI.
  </li>
  <li>
    They are often referred to as <em>daemons</em> and typically have names
    ending with a <code>d</code>, e.g. <code>sshd</code>,
    <code>systemd</code>, <code>httpd</code>.
  </li>
</ul>
</details>

<details>
  <summary>
    <strong>
      2. Managed by Init System
    </strong>
  </summary>
<ul>
  <li>
    Services are started, stopped and managed by an <em>init</em> system
    The most commmon init systems are:
    <ul>
      <li><strong>systemd</strong>: The most widely used on modern Linux distributions.</li>
      <li><strong>SysVinit</strong>: Used in older Linux systems</li>
      <li><strong>Upstart</strong>: An older init system, primarily used by Ubuntu in the past.</li>
    </ul>
  </li>
  <li>
    The <code>systemctl</code> command is used to manage services in <code>systemd</code>
  </li>
</ul>
</details>

<details>
  <summary>
    <strong>
      3. Configuration
    </strong>
  </summary>
<ul>
  <li>
    Services are configured through files stored in locations such as
    <ul>
      <li><code>/etc/systemd/system/</code>: highest priority.</li>
      <li><code>/run/systemd/system/</code></li>
      <li><code>/lib/systemd/system/</code>: lowest priority.</li>
    </ul>
  </li>
</ul>
</details>

<details>
  <summary>
    <strong>
      4. Starts at Boot
    </strong>
  </summary>
<ul>
  <li>
    Some services are enabled to start automatically when the system boots.
    For example, a web server like `nginx` can be configured to start
    every time the system is restarted.
  </li>
</ul>
</details>

<details>
  <summary>
    <strong>
      5. Common Linux Services
    </strong>
  </summary>
<ul>
  <li>
    <strong>
      System Services:
    </strong>
    <ul>
      <li><strong>networkd</strong>: manages network configurations and connectivity.</li>
      <li><strong>systemd-journald</strong>: handles logging and journal services, capturing system logs and events.</li>
      <li><strong>cron</strong>: schedules and executes recurring tasks or scripts at specified times.</li>
      <li><strong>dbus</strong>: facilitates communication between system processes and application through a messages bus.</li>
    </ul>
  </li>
  <li>
    <strong>
      Network Services:
    </strong>
    <ul>
      <li><strong>sshd</strong>: SSH server.</li>
      <li><strong>nginx</strong>: web server.</li>
      <li><strong>dnsmasq</strong>: DNS.</li>
    </ul>
  </li>
  <li>
    <strong>
      Application Services:
    </strong>
    <ul>
      <li><strong>mysql</strong>: database server.</li>
      <li><strong>mariadb</strong>: database server.</li>
      <li><strong>docker</strong>: virtual environment.</li>
    </ul>
  </li>
  <li>
    <strong>
      Hardware Services:
    </strong>
    <ul>
      <li><strong>cups</strong>: for printing.</li>
      <li><strong>bluetooth</strong>: for bluetooth devices.</li>
    </ul>
  </li>
</ul>
</details>


Now that we know what services do in Linux, let's address the elephant in the
room. What is `systemd`? And why it is relatd to Linux services.

## Systemd

`systemd` is an init system and service manager for Linux operating systems,
designed to bootstrap the system and manage services and processes during runtime.
It is an init system that is the first process to runs, hence, with PID1.
It manages all the units on the system, and service is just one of the units.

> Below are some common types of units:

Unit Type | File Suffix | Description
--- | --- | ---
**Service** | `.service` | Manages system services (e.g., daemons like `nginx.service` or `ssh.service`).
**Socket** | `.socket` | Manages IPC or network sockets, often used to start services on demand (e.g., `cups.socket`).
**Target** | `.target` | Groups other units to represent a system state or boot stage (e.g., `multi-user.target`).
**Device** | `.device` | Tracks and manages kernel devices (e.g., USB devices).
**Mount** | `.mount` | Controls filesystem mount points (e.g., `home.mount` for `/home`).
**Automount** | `.automount` | Sets up automount points that trigger mounting on access (e.g., `home.automount`).
**Swap** | `.swap` | Manages swap space activation and deactivation (e.g., `swapfile.swap`).
**Path** | `.path` | Monitors filesystem paths and triggers services based on changes (e.g., `myapp.path`).
**Timer** | `.timer` | Provides timer-based activation for services, replacing traditional cron jobs (e.g., `backup.timer`).
**Slice** | `.slice` | Organizes and limits resources for a group of processes (e.g., `user.slice`).
**Scope** | `.scope` | Tracks and manages externally created processes (e.g., started by non-systemd tools).
**Snapshot** | `.snapshot` | Represents a saved state of the systemd manager for rollback purposes.

**Examples**:
- Service: `nginx.service` runs 
- Timer: `backup.timer` triggers a backup service at specific intervals.
- Target: `graphical.target` represents the system state for a graphical desktop environment.

`systemd` is used by Ubuntu, and a few Linux distros,
Fedora, CentOS, RedHat, and Arch, just to name a few.
It starts a target which can start various other targets and services.
These are what we called units.
And for Mac user, it would be `launchd` that is managing your daemon and services.
You can read more about it [here](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html).


To see which target is the default for this system:
```bash
systemctl get-default
# graphical.target
```

An Ubuntu system, generally runs at the graphical target level.
Which itself requires the multi-user.targets, which requires various targets,
such as basic.target, sockets.target, network.target, etc.

To list the dependencies of a target, you can run:
```bash
systemctl list-dependencies graphical.target
```

To list the currectly active targets, we can run:
```bash
systemctl list-units --type target
```

Let's take a look at a simple example of service file, `rsyslog`.
It resides in the `/etc/systemd/system/syslog.service`.

```systemd
[Unit]
Description=System Logging Service
Requires=syslog.socket
Documentation=man:rsyslogd(8)
Documentation=man:rsyslog.conf(5)
Documentation=https://www.rsyslog.com/doc/

[Service]
Type=notify
ExecStartPre=/usr/lib/rsyslog/reload-apparmor-profile
ExecStart=/usr/sbin/rsyslogd -n -iNONE
StandardOutput=null
StandardError=journal
Restart=on-failure

# Increase the default a bit in order to allow many simultaneous
# files to be monitored, we might need a lot of fds.
LimitNOFILE=16384

CapabilityBoundingSet=CAP_BLOCK_SUSPEND CAP_CHOWN CAP_DAC_OVERRIDE CAP_LEASE CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_SYS_ADMIN CAP_SYS_RESOURCE CAP_SYSLOG CAP_MAC_ADMIN CAP_SETGID CAP_SETUID
SystemCallFilter=@system-service
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
NoNewPrivileges=yes
ProtectHome=no
ProtectClock=yes
ProtectHostname=yes

[Install]
WantedBy=multi-user.target
Alias=syslog.service
```

We will go over more in detail about a service file.
For now, let's look at the bottom, you can see that it is wanted
by the `multi-user.target`. This is automatically done, when you
installed a software. Therefore, if it needs a service file,
the installer will ensure creating it and making `systemd` aware.

Let's learn more about service files and writing your own in the next
[post](https://brucechanjianle.github.io/posts/cs-linux-services_part2/).

## Reference

- https://www.youtube.com/watch?v=Kzpm-rGAXos
