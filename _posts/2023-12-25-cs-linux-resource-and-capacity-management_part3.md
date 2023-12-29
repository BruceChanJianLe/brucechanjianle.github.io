---
title: Linux Resource and Capacity Management (Part 3)
author: cjl
date: 2023-12-25 16:12:51 +0800
categories: [CS]
tags: [cs, linux]
---

> Ah disk! They are typically the slower component in the computer.

This is part 3 of the series where we look at
the ways to measure disk input and output.

# DISK

## Real-time Disk Monitoring

From the first couple of article, we go to `top` for a quick glance.
Do we do the same here...
It turns out that `top` is only good for memory and CPU.
Maybe some disk io information, but it's really not the best place to look.
There is actually something called `iotop` which helps us to see from the
perspective of input and output operations when you are writing to disk.

```bash
# install iotop
sudo apt install iotop
# run it! typically with sudo 
sudo iotop
```

Below is a snapshot of `iotop`.
There is task ID, disk read, disk write, swaps that are happening, IO percentage of the system, 
and the command that is generating the IO.

```bash
Total DISK READ :       0.00 B/s | Total DISK WRITE :      27.09 K/s
Actual DISK READ:       0.00 B/s | Actual DISK WRITE:      19.35 K/s
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 9141 be/4 chanjl      0.00 B/s   19.35 K/s  0.00 %  0.00 % nvim --embed -S
    1 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % init splash
...
```

Well, as you can see not a lot is happening.
But we can generate some activity with disk checksum.

```bash
# Find out what drives are mounted
# and filter the result with fzf.
# Use grep if you don't have fzf installed
mount | fzf
# Run a checksum on your disk
sudo md5sum /dev/sda1
```

Now, if you run `iotop` command, it will show you what it is happening at the moment.
However, typically, we want to monitor it over time.
Hence, you can start `iotop` with `-a`, where a stands for accumulate.
Which is what we usually should do :D

```bash
# Start iotop with accumulation
# Do not forget to give privilege the tool
sudo iotop -a
```

Hmm...are we able to see what the process is doing with the IO?
From `iotop` we won't be able to see much.
For some cases, we may be able to tell from the command.
For example, `md5sum`. We know it is reading across the entire disk.
But for some commands like, `nvim`, you would not be able to tell
what it is doing.

For these type of process, you will need another command
which we have introduced before, it is the `lsof` - list open files.

```bash
# If you know the process id
sudo lsof -p $PROCESS_ID
# If you don't know, you can try matching
# But it is not accurate
sudo lsof -c $PROCESS_NAME
```

However, for some cases, like `md5sum` it is a command and not a process.
You won't be able to find it with `lsof -c md5sum`.
Therefore, it is always better of to know the process id and search based of it.

This is the part where we combine what we have learned.
Use `top` to obtain the process id, and then use `lsof` to look at what it is up to.

With this, we can know what files a process is touching.
However, we are not so concerned about what the individual files are
but where these files are stored.
For example, `/usr` will be on the main drive.
And if many files are on the main drive, we might want to move some files
over to another drive to reduce some of the contentions,
so that it has dedicated IO open to them.
Or maybe move over to a raid zero.

raid 0 - a stripe, potential performance improvemnt.
raid 1 - a mirror, no performance improve out of that.
raid 5 - a strip with parity, potentially a little performance improvemnt.
raid 10 - a strip mirro, potentially a lot of performance improvemnt.

Other, potential methods are using SSDs or array of SSDs...
Well it is limited to how much money you have at the end of the day :'D
Which is true for most of the things in this world.


Ahem! Ahem!

Well, the above explanation is very processes focus.
Could we see IO on a per disc basis?
Of course, maybe we do not just have one process to blame,
perhaps we are thinking about the disc in general.
So, if you suspect something is not right going on that disc.
We could potentially look at just that one disc with `iostat`.

Well, `iostat` can show us IO statistic, it allows us to pick the individual
disk that we are interested in. Surely, we can see all the discs!

```bash
iostat
# Result are as below
Linux 5.4.0-150-generic (chanjl)        12/29/2023      _x86_64_        (16 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           2.83    0.00    0.95    0.01    0.00   96.21

Device             tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
loop0             0.00         0.01         0.00        110          0
loop1             0.00         0.05         0.00       1064          0
loop2             0.00         0.05         0.00       1061          0
loop3             0.00         0.05         0.00       1074          0
loop4             0.00         0.05         0.00       1068          0
loop5             0.00         0.05         0.00       1044          0
loop6             0.00         0.05         0.00       1063          0
loop7             0.00         0.00         0.00         45          0
nvme0n1           6.66       116.67        59.67    2440243    1248085
```

From the information, we are able to deduce some useful insights about the disk.
For instance, a SATA drive can typically transfer up to 6GB/s.
We can verify from `iostat` whether the drive is getting 6GB/s.
And if we are not getting that performance and we should, then
maybe we are using the wrong driver for the SATA controller, maybe the disk is failing,
maybe the disc is not a 6GB/s drive? Maybe we are using a wrong cable which can cost a protocal
miss match and it stepped down to a more compatible protocol that is slower.
Those are some reason for a slow drive which we need to consider when looking at the IO.

```bash
# the flag -x will show more detail stats
iostat -x
```

With the `-x` flag we are able to to see the utilization of the disk (how full is it?).
We can monitor that which did not show up before.
We could also see `r_await` (read waiting)  or `w_await` (write waiting), these can be potential
issues. It means some applications tried to read or write and they had to queue up.
That indicates slow down or that we are at the bottle-neck.
So we want to keep an eye out for such things.

```bash
# the flag -x will show more detail stats every 1 second
# this is the recommended command
iostat -xt 1
```

## Over Time (Historical) Disk Monitoring

What about looking at historical data from the past?
Then we are back to `sar`!!!
Surely, we can use `sar` to look at our storage data.

```bash
# B for byte or block
sar -b
```

With `sar` you can see the block read (bread/s), write (bwrtn/s),
read request (rtps) and write request (wtps) per second based on time.
Now, a request for one file, but it can mean asking for many blocks since a file
can be made out many blocks.
And block size varies, it is something that is decided when you format the disk.

And we can use `sar` to do some filtering.
For example, looking at a specific time frame or date.
Something to note, is that `sar` keep the data of one month by default.
So the log files are name like so as well, if 17th of August, you will
look for `sa17`.

```bash
# -b for block
# -f for the logfile
# -s start of the time
# -e end of the time
sar -b -f /var/log/sysstat/sa17 -s 00:00:00 -e 09:00:00
```

We could check whether something occurs the same time everyday.
It could just be an outlier, or is it a steady increase in the IO
which could pontentially be a problem.
And we could extrapolate from there to know when we will get into trouble.

Well, there are a lot of metrics there (in `iostat -x`).
But those that we want to keep an eye for are blocks read,
blocks written. As that is the amount of data that you are moving throught
the disc, and it can only do so much.

Know your throughput, and compare that to the numbers you are seeing.
If you are within range, then all is fine.
But if you are hitting the maximum number, then it means you are falling behind.
And also the queue sizes, if they are consistently going up,
then that is a bottle-neck.

Well, like always, keep marching on! See you at the last article for Network IO :D
