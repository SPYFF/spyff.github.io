---
categories: ["mptcp"]
tags: ["mptcp", "wifi"]
date: 2017-10-21T15:00:00Z
summary: Just added master branch with the actual (currently unstable but working)
  MPTCP implementation.
title: LEDE MPTCP with kernel 4.9
slug: "mptcp"
draft: true
---

![](/images/mptcp_lede/lede_mptcp_4.9.png)

# Some small update

The MPTCP LEDE project previously supported only the stable 17.01 LEDE branch and the latest stable MPTCP kernel branch. Not much after new MPTCP version released or at least started the development of that. So we have the in development [branch](https://github.com/multipath-tcp/mptcp/tree/mptcp_v0.93) of MPTCP kernel version 0.93 which is based on the 4.9 vanilla kernel version. Because long term supported kernels also supported by LEDE (4.4 and 4.9 currently) I updated my fork.

## What we have now

[Master branch](https://github.com/spyff/lede-mptcp/tree/master) with unstable MPTCP supported 4.9 kernel

[Lede-mptcp-17.01](https://github.com/spyff/lede-mptcp/tree/lede-mptcp-17.01) branch with stable MPTCP supported 4.4 kernel

## Usage

Just as before, before the start of building LEDE, its required to run `make kernel_menuconfig` then enable MPTCP support in `Networking support -> Networking options` section of kernel menuconfig. In the submenus You can select path-manager and scheduler modules to compile. 

