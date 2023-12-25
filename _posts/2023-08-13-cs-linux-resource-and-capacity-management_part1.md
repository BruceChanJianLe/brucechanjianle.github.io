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

The Big 4 Performance Metrics:
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

Monitor one process, along with its other processes
```bash
top -Hp $PROCESS_ID
```

INSTALLING:
```bash
# BASH TOP
sudo apt install bashtop
# BTOP ++
sudo apt install btop
# BPYTOP
sudo apt install bpytop
```

Now, you might ask.
What about frozen process.
Because frozen process does not consume any CPU, `top` is not good at it.
We could use the `ps` command.

```bash
# To list all the processes that is run by chanjl user.
ps --user chanjl
# Another old command
ps aux | grep $PROCESS_NAME
# a for showing all, show me everything
# u for showing who the users are
# x for showing processes that does not have a tty attached,
#   for example system services, things without a terminal
```

Alright, what about seeing how one process is connected another?
Of course, we can use `pstree`.

```bash
# Run it with a pid
pstree $PROCESS_ID
# Better if we just run the command raw
pstree
```

Well, is there any other thing that a process might how itself into?
Of course, sometimes it might be a library that is locked up the thread.
Or it could be generating a lot of disk IO because it constantly has to go to the disk to find these library as it is not being cached into the RAM.
One of the solutions is to move the libraries to a less contention drives for better access.

```bash
# List the libraries that a process is hooked onto
# pmap is a process map, typically you will need sudo to view
sudo pmap $PROCESS_ID
```

For example, here is an RVIZ2 application.
```bash
sudo pmap 7616
7616:   rviz2
000000004037a000    700K rw---   [ anon ]
00000000411cf000      8K r-xs- .glXXXXXX (deleted)
000000004163a000    700K rw---   [ anon ]
0000000041d23000    700K rw---   [ anon ]
00005635523a4000      8K r---- rviz2
00005635523a6000      8K r-x-- rviz2
00007fc25b59e000   2048K ----- libnvidia-glcore.so.470.223.02
00007fc25b79e000   5488K r---- libnvidia-glcore.so.470.223.02
00007fc25bcfa000   1680K rw--- libnvidia-glcore.so.470.223.02
00007fc264000000    132K rw---   [ anon ]
00007fc264021000  65404K -----   [ anon ]
00007fc26803a000    692K r---- DejaVuSans-Bold.ttf
00007fc2680e7000      4K -----   [ anon ]
00007fc2680e8000   8192K rw---   [ anon ]
00007fc2688e8000     40K r---- libstd_msgs__rosidl_typesupport_introspection_cpp.so
00007fc2688f2000     16K r-x-- libstd_msgs__rosidl_typesupport_introspection_cpp.so
00007fc2688f6000      8K r---- libstd_msgs__rosidl_typesupport_introspection_cpp.so
00007fc2688f8000      4K ----- libstd_msgs__rosidl_typesupport_introspection_cpp.so
00007fc2688f9000      8K r---- libstd_msgs__rosidl_typesupport_introspection_cpp.so
00007fc2688fb000      8K rw--- libstd_msgs__rosidl_typesupport_introspection_cpp.so
00007fc27bf28000      8K r---- ld-linux-x86-64.so.2
...
00007fc27bf2a000      8K rw--- ld-linux-x86-64.so.2
00007fff944d1000    132K rw---   [ stack ]
00007fff945e2000     12K r----   [ anon ]
00007fff945e5000      8K r-x--   [ anon ]
ffffffffff600000      4K --x--   [ anon ]
 total           649804K
```

Well, `pmap` is good for libraries.
But it is bad for config files.
When we start up many processes, they usually read up some config files.
Some applications like mySQL might even hold a file open for read and writing.
These files are typically locked and you are not allowed to edit the file.
How can we know what application is holding this file?

```bash
# List which are the files a process are locking
sudo lsof -p $PROCESS_ID
```

For example, here is the files an RVIZ2 application is locking.
```bash
sudo lsof -p 9283
COMMAND  PID      USER   FD      TYPE             DEVICE SIZE/OFF     NODE NAME
rviz2   9283 developer  cwd       DIR               0,66     4096 28434892 /home/developer
rviz2   9283 developer  rtd       DIR               0,66     4096 28074818 /
rviz2   9283 developer  txt       REG               0,66    23048 31397568 /opt/ros/humble/bin/rviz2
rviz2   9283 developer  DEL       REG                0,1            170731 /memfd:/.glXXXXXX
rviz2   9283 developer  mem       REG               0,66    80264 29653946 /opt/ros/humble/lib/libstd_msgs__rosidl_typesupport_introspection_cpp.so
rviz2   9283 developer  mem       REG               0,66    14560 29649382 /opt/ros/humble/lib/libunique_identifier_msgs__rosidl_typesupport_introspection_cpp.so
rviz2   9283 developer  mem       REG               0,66    66936 29664822 /opt/ros/humble/lib/libgeometry_msgs__rosidl_typesupport_introspection_cpp.so
rviz2   9283 developer  mem       REG               0,66    96616 29653944 /opt/ros/humble/lib/libstd_msgs__rosidl_typesupport_fastrtps_cpp.so
rviz2   9283 developer  mem       REG               0,66    73264 31383140 /opt/ros/humble/lib/librcl_action.so
rviz2   9283 developer  mem       REG               0,66    77200 31380191 /var/cache/fontconfig/f682ffa3-8fac-4980-8c6d-49ccdffc3673-le64.cache-7
rviz2   9283 developer  mem       REG               0,66    43488 28645646 /usr/lib/x86_64-linux-gnu/libXcursor.so.1.0.2
rviz2   9283 developer  mem       REG               0,66   149760 27927898 /usr/lib/x86_64-linux-gnu/libgpg-error.so.0.32.1
rviz2   9283 developer  mem       REG               0,66  1296312 27927892 /usr/lib/x86_64-linux-gnu/libgcrypt.so.20.3.4
rviz2   9283 developer  mem       REG               0,66    39024 28583600 /usr/lib/x86_64-linux-gnu/libcap.so.2.44
rviz2   9283 developer  mem       REG               0,66   125152 27927914 /usr/lib/x86_64-linux-gnu/liblz4.so.1.9.3
rviz2   9283 developer  mem       REG               0,66   170456 27927916 /usr/lib/x86_64-linux-gnu/liblzma.so.5.2.5
rviz2   9283 developer  mem       REG               0,66   807936 27947104 /usr/lib/x86_64-linux-gnu/libsystemd.so.0.32.0
rviz2   9283 developer  mem       REG               0,66    31096 28589241 /usr/lib/x86_64-linux-gnu/libxcb-util.so.1.0.0
rviz2   9283 developer   29u      CHR              195,0      0t0      532 /dev/nvidia0
...
rviz2   9283 developer   30uw     REG               0,66      872 31468592 /home/developer/.cache/nvidia/GLCache/66dbeada1994f20a809e9e1189583fb5/83fad13e2a2fd784/2af83171a1bb0740.toc
rviz2   9283 developer   31uw     REG               0,66     5413 31468596 /home/developer/.cache/nvidia/GLCache/66dbeada1994f20a809e9e1189583fb5/83fad13e2a2fd784/2af83171a1bb0740.bin
rviz2   9283 developer   32u      CHR              195,0      0t0      532 /dev/nvidia0
rviz2   9283 developer   33u     unix 0xffff896e2e30f000      0t0   170732 type=STREAM
```

Hmm...is there any other tools we could use to monitor our CPU?
Oh yes, there is!
It's `mpstat`, it is very much like `sar` but it is much more real-time.
It can get pretty granular, meaning you could even tell it to monitor which CPU core.

```bash
# Monitor CPU for every two seconds
mpstat 2
# And when you ^C it will give you an average
Linux 5.4.0-150-generic (chanjl)        12/25/2023      _x86_64_        (16 CPU)

12:29:04 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
12:29:06 PM  all    0.53    0.00    0.28    0.00    0.00    0.00    0.00    0.00    0.00   99.18
12:29:08 PM  all    0.47    0.00    0.32    0.00    0.00    0.06    0.00    0.00    0.00   99.15
12:29:10 PM  all    0.53    0.00    0.31    0.00    0.00    0.00    0.00    0.00    0.00   99.15
Average:     all    0.51    0.00    0.30    0.00    0.00    0.02    0.00    0.00    0.00   99.16
```

So for testing, you might want to run the critical application.
And then start `mpstat` at 1 second intervals.

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
sar -u
# R - RAM
sar -r
# B - BYTE
sar -b
# N - NETWORK
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

That's all for now, let's continue exploring RAM in the next article!
