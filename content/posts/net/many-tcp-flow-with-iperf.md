---
categories: ["net"]
tags: ["tcp", "measurement", "iperf"]
date: 2019-07-14T15:00:00Z
summary: How to do correct iperf measurements with many TCP flows on GNU/Linux test environment
title: Thousand TCP flows with iperf
slug: scaling-iperf
---

__Update 2025:__ Contrary to the original post, iperf3 gained multi-threading support (see [PR](https://github.com/esnet/iperf/pull/1591)) in
version 3.16 (released at 2023. 11. 30.)

# Abstract

The classic **iperf** and lately the **iperf3** widely used for various network performance measurements.
The original iperf written in C++ while the iperf3 written in C, but thats not the only difference,
from performance measurement standpoint there is a big one:
iperf3 is single threaded and iperf is multi-threaded.
This is quite important when it comes to multiple flow TCP measurements.
In many cases, iperf3 fails to utilize the whole network capacity due to CPU bottleneck:
each TCP flow share one thread and therefore one CPU core.
In contrast, iperf create a new thread for every TCP flow.
Nevertheless handling many TCP flows is still tricky even with the multi-threaded iperf.
In this post, we will investigate the pitfalls of many flow measurements.


# The naive approach

First, we have to install iperf. In my Ubuntu 18.04 environment, this looks like the following.

```
$ sudo apt install iperf
```

To verify the operation, we can test the TCP performance on loopback.
We need to start an iperf server and client in separated terminal sessions:

```
$ iperf -s #server
```

Then the client:

```
$ iperf -c 0
```

I got the following output from the client in my notebook:

```
spyff@hp:~$ iperf -c 0
------------------------------------------------------------
Client connecting to 0, TCP port 5001
TCP window size: 2.50 MByte (default)
------------------------------------------------------------
[  3] local 127.0.0.1 port 40232 connected with 127.0.0.1 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.0 sec  45.1 GBytes  38.8 Gbits/sec
```

That was a single TCP flow measurement.
We can repeat the test with 2 flows, just pass the `-P 2` parameter to the client iperf:

```
spyff@hp:~$ iperf -c 0 -P 2
------------------------------------------------------------
Client connecting to 0, TCP port 5001
TCP window size: 2.50 MByte (default)
------------------------------------------------------------
[  4] local 127.0.0.1 port 40258 connected with 127.0.0.1 port 5001
[  3] local 127.0.0.1 port 40256 connected with 127.0.0.1 port 5001
[ ID] Interval       Transfer     Bandwidth
[  4]  0.0-10.0 sec  33.6 GBytes  28.8 Gbits/sec
[  3]  0.0-10.0 sec  34.1 GBytes  29.3 Gbits/sec
[SUM]  0.0-10.0 sec  67.7 GBytes  58.1 Gbits/sec
```

# The problem of small backlog

Unfortunately, this is not a very scalable approach, if we try it with 1000 flows it will fail.
In my case, the iperf trying to spawn threads but that happens very slowly and some flow even
fails with the following error: `write failed: Connection reset by peer`.
Unfortunately that's not a very helpful error message, but it means that the remote peer got
too many incoming connections which fills his listener TCP socket's backlog queue full.
When the backlog is full, the kernel refuses to accept new incoming TCP connections so send
TCP reset back to the iperf client.
But at the same time, the backlog also processed by the kernel, so some new TCP flow will
succeed to connect because of the liberated places in the backlog queue.
To check how big the listener's backlog, we can use the `perf trace` utility.
The `perf trace` can monitor the system calls used by iperf, so we have to
monitor the `listen` system call of the iperf server, and check the length of the backlog:


```
spyff@hp:~$ sudo perf trace -e listen
     0.000 ( 0.025 ms): iperf/13104 listen(fd: 3<socket:[379613]>, backlog: 2147483647)  = 0
```

In my case the backlog is big enough (`MAX_INT`), because I use the iperf version `2.0.10`.
But in older version, the backlog was set to `5` which is too small for many connections.
So there is another limit somewhere which prevent us from our successful measurements:
the `SOMAXCONN` sysctl value.
This is set to `128` by default. We can increase it to `65536`:

```
spyff@hp:~$ sysctl net.core.somaxconn
net.core.somaxconn = 128
spyff@hp:~$ 
spyff@hp:~$ sudo sysctl -w net.core.somaxconn=65536
net.core.somaxconn = 65536
```

Now the `write failed: Connection reset by peer` error should be gone.

# iperf fails to report the aggregated throughput

So far so good, but there is a small annoying bug still there.
With `-P 100` we can see the aggregated throughput in the last row `[SUM]` at the end of the measurement:

```
...
[ 44]  0.0-10.2 sec   726 MBytes   595 Mbits/sec
[ 43]  0.0-10.1 sec   621 MBytes   516 Mbits/sec
[ 47]  0.0-10.1 sec   597 MBytes   495 Mbits/sec
[ 32]  0.0-10.1 sec   767 MBytes   636 Mbits/sec
[ 58]  0.0-10.2 sec   333 MBytes   273 Mbits/sec
[SUM]  0.0-10.2 sec  49.2 GBytes  41.3 Gbits/sec

```

However, this row disappearing at `-P 1000` or to be more precise, at `-P 128`.
This is really suspicious for anyone who coded in C/C++ and experienced integer overflow with signed `char` type.
I investigated the place of the problem in the iperf source code, and it turned out somewhere
in the aggregating code it compares a `char` with an `int` and increases both at every iteration (for every thread).
Then the `char` overflows, and the condition `if(char == int)` will obviously evaluate as `false`.
I submitted the bug to the maintainer of `iperf`, you can find the commit [here](https://sourceforge.net/p/iperf2/code/ci/be417e3be47fa4929c06cf4080b756fbc270d0f1/).
I recommend using the latest version of iperf to avoid that bug.


# Summary

* ~iperf is better for multi TCP flow measurement than iperf3, because multi-threaded.~ iperf3 also multi-threaded see the beginning of the post
* Older versions of iperf explicitly set backlog length to `5` which is insufficient, so use iperf version `>= 2.0.10` or modify the value in the iperf source code manually to a greater value
* `sudo sysctl -w net.core.somaxconn=65536`: in linux, set `SOMAXCONN` value from the default `128` to `65536`
* If the aggregated throughput `[SUM]` disappears with big parallel-flow values (`-P > 127`) use the latest version of iperf: [iperf master branch at Sourceforge](https://sourceforge.net/p/iperf2/code/ci/master/tree/)
