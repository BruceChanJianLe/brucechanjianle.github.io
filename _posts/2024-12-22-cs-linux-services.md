---
title: Linux Services
author: cjl
date: 2024-12-22 12:38:58 +0800
categories: [CS]
tags: [cs, linux]
---

> Ahh...Linux services, cannot talk about them without mentioning systemd!

## Introduction

Linux services are vital components of the system,
responsible for ensuring various functionalities run smoothly.
They are managed through init systems like `systemd` and can be tailored
to meet the specific needs of the system or user.

### Linux Services

<details>
  <summary>**Background Processes**</summary>

- Services typically run in the background, meaning they do not
  interact with the user directly through the terminal of GUI.
- They are often referred to as _daemons_ and typically have names
  ending with a `d`, e.g. `sshd`, `systemd`, `httpd`.

</details>


1. **Managed by Init System**:
  - Services are started, stopped and managed by an _init_ system
    The most commmon init systems are:
        + **systemd**: The most widely used on modern Linux distributions.
        + **SysVinit**: Used in older Linux systems.
        + **Upstart**: An older init system, primarily used by Ubuntu in the past.
  - The `systemctl` command is used to manage services in `systemd`

1. **Configuration**:
  - Services are configured through files stored in locations such as
    + `/etc/systemd/system/`: highest priority
    + `/run/systemd/system/`
    + `/lib/systemd/system/`: lowest priority

1. **Starts at Boot**:
  - Some services are enabled to start automatically when the system boots.
    For example, a web server like `nginx` can be configured to start
    every time the system is restarted.

1. **Common Linux Services**:
  - System Services:
    + networkd: manages network configurations and connectivity
    + systemd-journald: handles logging and journal services,
                        capturing system logs and events
    + cron: schedules and executes recurring tasks or scripts at specified times
    + dbus: facilitates communication between system processes and
            application through a messages bus
  - Network Services:
    + sshd: SSH server
    + nginx: web server
    + dnsmasq: DNS
  - Application Services:
    + mysql: database server
    + mariadb: database server
    + docker: virtual environment
  - Hardware Services:
    + cups: for printing
    + bluetooth: for bluetooth devices

### Systemd

`systemd` is an init system and service manager for Linux operating systems,
designed to bootstrap the system and manage services and processes during runtime.
It is an init system that is the first process to runs, hence, with PID1.
It manages all the units on the system, and service is just on of the units.

## Units

Below are some common types of units:

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
- Service: nginx.service runs 
- Timer: backup.timer triggers a backup service at specific intervals.
- Target: graphical.target represents the system state for a graphical desktop environment.

## The Basics

Action | Command
--- | ---
Check status of a service | `sudo systemctl status <service_name>`
Start a service | `sudo systemctl start <service_name>`
Stop a service | `sudo systemctl stop <service_name>`
Restart a service | `sudo systemctl restart <service_name>`
Check the status of a service | `systemctl status <service_name>`
Enable a service to start at boot | `sudo systemctl enable <service_name>`
Disable a service from starting at boot | `sudo systemctl disable <service_name>`


## Reference

- https://www.youtube.com/watch?v=Kzpm-rGAXos
