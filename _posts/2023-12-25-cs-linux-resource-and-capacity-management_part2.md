---
title: Linux Resource and Capacity Management (Part 2)
author: cjl
date: 2023-12-25 13:08:26 +0800
categories: [CS]
tags: [cs, linux]
---

> Memory is an essential part of your performance.
> And swap is the enemy of performance,
> it will slow down your computer extensively.

This is part 2 of the series where we look at
the ways to monitor RAM.

# RAM

## Physical Memory

There is a tool called `free`.
Not how much money, but how much memory is free.
Typicall, you want to run it with -h to for human readable output.

```bash
free -h
# output
               total        used        free      shared  buff/cache   available
Mem:            30Gi       4.3Gi        22Gi       127Mi       4.4Gi        26Gi
Swap:          7.4Gi          0B       7.4Gi
```

With a closer look, you might wonder.
The numbers do not really add up do they?
Well, this is because some memory are used for caching.
Caching is when CPU guesses you might need this information 
and it decides to keep it in the memory.
It is trying to get ahead of the game, 
but it also knows that this information is completely dispensable.

If you are doing some testing, you can keep the `free` tool running.
Like so:

```bash
# Run free for every 2 seconds
free -hs 2
```

Surely, `top` could also show the memory usage.
But if you would like to store the data for processing later... 
Perhaps, `free` and `top` are not the best tool.

We can consider using `sar` which we talked about before when monitoring CPU.

```bash
# Remember, r for RAM
sar -r
```

Column | Description
--- | ---
kbmemfree | kilo-bytes of memory free
memused | how much memory is actually being used.
memused | how much memory is actually being used.
kbdirty | dirty memory are stuff that can be deleted / overwritten. Or sometimes even cannot be trusted. So it tries not to leverage on these memory.

## Virtual Memory

Oh well, we have been talking about physical memory.
What about virtual memory?
Like swap space.
We measure hard drive speed in milliseconds.
We mesuare RAM speed in nanoseconds.
So virtual memory is at least 10 times slower.
It does not mean we should never used virtual memory, 
but we should try to avoid it in most scenarios.

Before we dive into another tool, let's use `sar` 
because it can also show some of swap memory too.

```bash
# Well S for swap!
sar -S
# Or use W to show how much page swap in and out
sar -W
```

There is another tool called `vmstat` which can show swap memory.

```bash
# unit in Mega byte for prettier output
vmstat --unit=M
# vmstat can also do intervals, here we do it every 2 seconds
vmstat --unit=M 2
```

Looking under the `io` column, there is `bi` and `bo`.
`bi` stands for block in, and `bo` block out.
Block in means there something has been read into the memory.
But block out we need to pay attention to.

Block out could mean two things, old memory got cleaned up.
Or memory is full and we it got read out to swap space.

To determine whether it is one or the other.
We need to pay attention to the `swap` column.
If you see a 20 in `bo` and a 20 in `si`,
this means that those block got out of memory
and got into swap space.

We could also use `sar`, again, to check for input and output memory write
```bash
# For physical memory
sar -B
```

Column | Description
--- | ---
pgpgin | page in per second. Similar to block in, but in a different unit.
pgpgout | page out per second. Similar to block out, but in a different unit.
fault | faults per second. They are not generally a bad thing, it means cache missed, if an index is out of the RAM. A fault occurs and we will need to obtain the data from disk to RAM.

Phew! Here another note on managing resources on Linux.
In part 3, we will look at measuring disk activities!
