---
categories: ["net"]
date: 2025-05-04T10:00:00Z
summary: How to set priority metadata for individual packets.
title: Per-packet SO_PRIORITY socket API
# slug:
canonical: "https://fejes.dev/posts/net/so-priority-cmsg/"
---

# Intro

The Qdisc subsystem of the Linux kernel allows us to configure network QoS-related mechanisms.
There are various Qdiscs that implement simple tri-band priority queueing, and more advanced ones such as HTB trees.
Many of these Qdiscs (e.g.: `fq`, `ets`, `mqprio`, `fq_codel`, etc.) rely on packet priority in their decision making.
Prior to Linux 6.14, there was no socket API for setting this priority for individual packets.
With Linux 6.14, an API was introduced to allow the application to set or receive this value with an ancillary message.

# Brief history

The central unit of packet processing in the Linux network stack is the `struct sk_buff` (`skb` for short).
In addition to the sent/received data, it contains many internal metadata fields.
One such [metadata](https://elixir.bootlin.com/linux/v6.14.5/source/include/linux/skbuff.h#L1036) is `skb->priority`, which was introduced with Linux 0.98 in 1992.
In addition to the field, it's userspace API was also introduced, the `SO_PRIORITY` option to `setsockopt`.
This option allows the application to set the priority of the socket.
Then every packet sent on the socket inherited this value, which was used by the IP layer.

Originally, the sole purpose of this metadata was to set the _Type of Service_ field of the IPv4 header.
Later, it was used extensively by many Qdiscs to implement packet prioritization.
In addition, NIC drivers used it for VLAN priority tagging and it's offload.


# Limitation of the per-socket approach

As an endpoint application, the priority to socket mapping makes sense.
However, Linux can be used as a VPN gateway, proxy, or software switch (among many other things).
In such use cases, we might have a userspace application that multiplexes packets into a tunnel.

For example, a VPN gateway opens a socket that represents the tunneling device.
The packets are received with different priorities, set by their originating socket or some packet filtering mechanism.
The VPN application also has a socket representing the tunnel interface.
But what would be the priority set for this tunnel socket?
Whatever the application sets, it will mask the original priority of the packet.
A more flexible approach is required: set priority on a per-packet basis on the same tunnel socket.

# The per-packet CMSG API

Per-packet metadata is not new to Linux.
The API for it is called ancillary or aux message, sometimes referred to as out-of-band control message.
Let's stick with the control message, or CMSG for short.
It is used to set fragmentation, traffic class, hop limit, and other options.
For a comprehensive example, see the `cmsg_sender.c` [selftest in the source tree](https://elixir.bootlin.com/linux/v6.14.5/source/tools/testing/selftests/net/cmsg_sender.c).

The new CMSG options [introduced](https://lore.kernel.org/netdev/20241213084457.45120-1-annaemesenyiri@gmail.com/T/) in Linux 6.14 are `SO_PRIORITY` and `SO_RCVPRIORITY`.
The priority metadata is 32 bits long and unsigned.
Without `CAP_NET_ADMIN` only values from 0 to 6 are accepted.
The following C code snippet gives an example of how to use this.
Unfortunately, due to the nature of the C network API, much of the code is boilerplate.
To reduce the length, the error handling is omitted.
The important part is where we set the `SO_PRIORITY` type and it's value.

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <string.h>

int main() {
    char *data = "Data to send";
    const size_t clen = sizeof(unsigned int);
    char cbuf[CMSG_SPACE(clen)];
    memset(cbuf, 0, clen);

    struct iovec iov = {
        .iov_base = data,
        .iov_len = strlen(data),
    };

    struct sockaddr_in destionation = {
        AF_INET, htons(5555),
        .sin_addr.s_addr = inet_addr("127.0.0.1")
    };

    struct msghdr msg = {
        .msg_name = &destionation,
        .msg_namelen = sizeof(destionation),
        .msg_iov = &iov,
        .msg_iovlen = 1,
        .msg_control = cbuf,
        .msg_controllen = CMSG_SPACE(clen)
    };

    struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
    *cmsg = (struct cmsghdr) {
        .cmsg_level = SOL_SOCKET,
        .cmsg_type = SO_PRIORITY, // !
        .cmsg_len = CMSG_LEN(clen)
    };

    unsigned int *priority = (unsigned int *) CMSG_DATA(cmsg);
    *priority = 987; // !

    int sk = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    sendmsg(sk, &msg, 0);

    close(sk);
}
```

With Linux 6.14 and above this code send a UDP packet on loopback and instruct the kernel to set `skb->priority` to 987.
__Important__ to execute it as root or with `CAP_NET_ADMIN` capability.

# Workarounds for older kernels

Unfortunately there are no clean way to get this behavior on older kernels.
Some workarounds what I used before:

1. Set `SO_MARK` per-packet and copy the value with an eBPF tc program from `skb->mark` to `skb->priority` before Qdisc processing.
2. Open a new socket for each priority for the same destionation.
3. Very fragile: use `setsockopt` to change the priority before each send.
