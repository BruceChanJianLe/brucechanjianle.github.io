---
title: Linux Resource and Capacity Management (Part 4)
author: cjl
date: 2023-12-29 23:34:52 +0800
categories: [CS]
tags: [cs, linux]
---

> Of the big four metrics of performance.
> Network is the least likely to be a problem,
> but we do need to know how to monitor it.

Welcome to part 4 of the series where we look at
the ways to monitor our network IO.

# Network

## Real-time Network Monitoring

We will look at how our network adapter perform, 
how much data it is moving through and who it is 
that is comsuming the data, in order for us to figure out
whether we have enough network bandwidth or troubleshoot and isolating
things that are using more bandwidth than they should be.

Now, there are many tools out there which can use.
But I generally prefer something that is in the command line and
built into Linux. However, if you have enough space and you are not
monitoring something remotely, perhaps, you can install `ntopng`.

```bash
# install network-top-next-generation
sudo apt install ntopng 
```

More about `ntopng` at ntop.org. Now, it is very fancy as it has a lot of
features, like graphical representations of your network bandwidth, you just
need to point and click. But it has a ton of dependencies!

With that said, we have anothe cli tool which can pretty friendly
where it has the information we need. It is called `iftop`, which stands
for interface top. It can show the network interface on my system.

Now, like storage, many of the things we would not have visibility if
we are regular users, hence, remember to run with elevated privilege.

```bash
# Install iftop
sudo apt install iftop
# Run it with privilege
sudo iftop
```

Well, as you can see `iftop` will what application is using up
my network bandwidth, who is the primary active connection.
What we are concerned with is the bottom of the tui.
Where the cummulative transmission, cummulative received,
peak transmission and peak received is showed.

For example, if I am on a gigabit network, and what I see is 9 megabits
received, then I am well within range. However, if I observed 100 megabits
received, then I maxed out my network adapter.

```bash
# Bottom of iftop
...
TX:             cum:   44.8KB   peak:    608b                                                                                                                             rates:      0b      0b     45b
RX:                     573KB           23.1Kb                                                                                                                                        0b    205b   1.41Kb
TOTAL:                  618KB           23.1Kb                                                                                                                                        0b    205b   1.45Kb
```

Well, `iftop` actually shows us all the traffic and all of our
network interfaces. So if you have multiple interfaces, everything
will look kind of jumbled up together. In this case, you might want to
do filter and restrict what to display.

```bash
# With -i we can specify only to display one interface
sudo iftop -i $NAME_OF_INTERFACE
```

Alright, that's great! So the next question would be can `iftop` logs
it data or show historical data? And the answer is no...it is very much 
like `top` utility, it is really good at showing what is running right now.
But not to worry, as we already have a tool which can assist use to look
at historical data. Please check out the next section on how to monitor
historical network data.

Let's move on and discuss about identifying network performance. Well,
I must say troubleshooting network can be tricky.
This is because there are so many components that are involved.
When we started the section, we know that typically network is not your
bottleneck, at least not the network in your computer. As we are passing
through 5 to 10 different routers from other different systems. So the
issue could be anywhere between the path.

In this case, we need a visibility into our network to really get an
understanding where our bottleneck are really at. There is a tool that can
do this which is `mtr`. It stands for Mike's traceroute, perhaps a guy name
Mike wrote this, but it is an advanced traceroute.

Therefore, when we discover the internet is really slow, the issue could be
the DNS server, it could be our firewall is taking too long, or our ISP
is having an outage, like having to route through some weird places. With `mtr`

```bash
# install mtr
sudo apt install mtr
# Run it to trace google
# -t flag to use tui
mtr -t www.google.com
```

Don't worry if you see some `???` usually it means it hit a firewall and it
does not let me report beyong the spot.

You might have learned about BGP (Border Gateway Protocol), where interent is
made up of all these different networks or autonomous system that are connected
together. With `mtr`, you will be able to see the the networks you pass through
on the way to a destination. You don't just see generic IP addresses for routers
anymore, but you can actually see the AS numbers.

```bash
# -z is how to review the Autonomous System
mtr -t -z www.google.com
```

In addtion to the the data that we saw a moment ago, it is able to show us
autonomous numbers. So this lets us see the pathway and evaluate performance
along the way.

Well, you might ask, is there any other tools for networking?
And the answer is, of course, let's go through a few before going on to the
next section. Here we have `ip`, which I believe is a newer version of
`ifconfig`.

```bash
# To show your interface
ip addr
```

There is also a tool called `ss`, which stands for Socket Statistics. It
is a newer version of `netstat` and it works pretty much the same.

```bash
# To be able to view all the sockets that
# are listening on my system and who they
# are talking to.
ss -natp
```

Moving on, we also have `lsof`, the list open file. It surely does not sound
like a network command, but if you sudo it, it can actually show you who is
using what port.

```bash
# This shows us all the TCP ports that are open
# and that have established connections on them right now.
sudo lsof -iTCP -sTCP:ESTABLISHED
```

Welll, you might think that list open files util might not be able to do this.
But it actually can, I trust that this is because of the nature of Linux, how
it treats many of the things as files and directories.

## Over Time (Historical) Network Monitoring

That's correct! It's our good old `sar` command!
As mentioned before, `sar` can log data of the big four performance metrics.
However, networking is a little bit trickier. You need to provide additional
details for `sar` to give us what we want.

It expects a keyword, and keyword could be a number of different things.
Surely, read the good old friendly manual / documentation.

```bash
# Looking into the man page of sar
-n { keyword [,...] | ALL }
      Report network statistics.

      Possible keywords are DEV, EDEV, FC, ICMP, EICMP, ICMP6, EICMP6, IP, EIP, IP6, EIP6, NFS, NFSD, SOCK, SOCK6, SOFT, TCP, ETCP, UDP and UDP6.

# N for network
sar -n DEV

# If looking for ipv4
sar -n IP

# If looking for ipv6
sar -n IP6

# If looking for tcp
sar -n TCP
```

Like we previously discuss, `sar` can show us filtered down time frame of
the day.

```bash
# -n for network
# -f for the logfile
# -s start of the time
# -e end of the time
sar -n DEV -f /var/log/sysstat/sa17 -s 08:00:00 -e 09:00:00
```

With this we could do comparison, look at tranding of the data
and figure out what is going to look like in the future.

Well, before we close this, I just wanted to point out that,
you should use any tools that is at your disposal. Try new things,
do not be afraid of the command lines, hard things, or things that
seems hard at first glance can be very benefitial for the long run.
