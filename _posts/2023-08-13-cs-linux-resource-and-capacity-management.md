---
title: Linux Resource and Capacity Management
author: cjl
date: 2023-08-13 08:56:55 +0800
categories: [CS]
tags: [cs, linux]
---

> Everytime when your computer becomes slow, I guess the first thing we do 
> is to blame it on the computer for being not POWERFUL enough. But in reality,
> maybe it just us not managing the resource properly.
> Let's explore some meaningful ways to debugging our computers before we decide to get
> a faster and stronger one.

The Big 4 of Performance:
- CPU
- RAM
- Disk IO
- Network IO

# CPU

## Real-Time CPU Monitoring

For dynamic and real time monitoring CPU resource use `top` or `htop`.
`htop` is typically not installed by default, and it brings in a lot of dependencies with it.
Recommend to use just use `top`.
If you like fancy application, you may want to check out [bashtop](https://github.com/aristocratos/bashtop), [btop](https://github.com/aristocratos/btop), [bpytop](https://github.com/aristocratos/bpytop).
Where `bashtop` was the original project.
`btop` is the c++ version of the project.
Where `bpytop` is the python version of the project.
All of them are available on Ubuntu 22.04.

USAGE: (top)

Shortcut | Description
--- | ---
F | re-arrange the monitoring columns
M | monitor by memory
m | toggle visualization of memory
z | toggle color mode

INSTALLING:
```bash
# BASH TOP
sudo apt install bashtop
# BTOP ++
sudo apt install btop
# BPYTOP
sudo apt install bpytop
```

## Over Time CPU Monitoring

For monitoring CPU resource over a period of time.
We have a tool called `sar` which stands for System Activity Report.
This tool can monitor more than just CPU resources which we will see soon!

USAGE: (sar)

Enable performance logging
By default, after installing sysstat, you will have the file below:
```bash
sudoedit /etc/default/sysstat
# Change false to true
# ENABLED="true"
```
Then edit cron job
```bash
sudoedit /etc/cron.d/sysstat
```

Looking for log files
```bash
ls /var/log/sysstat/
# sa17 sa18 # if enabled
```
Reading log files
```bash
# URB(A)N, this is just a mnemonic
# U - CPU
# R - RAM
# B - BYTE
# N - NETWORK
sar -u
sar -r
sar -b
sar -n # This needs to be configured
```

USAGE: (uptime)

To check how long your system has been alive
```bash
uptime
```

USAGE: (w)

To verify if there is any other users
```bash
w
```
INSTALLING:
```bash
# System activity report
sudo apt install sysstat
```
