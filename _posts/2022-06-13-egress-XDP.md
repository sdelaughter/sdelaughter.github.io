---
layout: post
title: "Egress XDP with Linux VETH Pairs"
date: 2022-06-13
tags: networks XDP eBPF Linux DoS
---

# Introduction
[XDP (eXpress Data Path)](https://www.iovisor.org/technology/xdp) enables high performance analysis, modification, and redirection of Linux network traffic.  XDP Programs can be written and launched from user space at run-time, but are executed in the kernel, providing a terrific combination of speed and usability. They act on packets almost immediately upon receipt, before a socket buffer is allocated, which can enable extremely efficient DoS mitigation, protocol translation, overlay network routing, etc.  The two downsides are that programs a) must pass a fairly strict verifier, and b) can only operate on inbound traffic.  There are plenty of other posts describing how to write programs that will pass the XDP verifier.  The inbound-only restriction seems to be generally accepted as unavoidable, but with a layer of indirection egress XDP is possible.

Outbound traffic *can* be manipulated in a similar way using eBPF, of which XDP is an extension, but there are important differences.  Most notable is that XDP programs operate on raw ethernet frames immediately after a packet is received, while regular eBPF programs operate on sk_buff (socket buffer) structs.  In some cases sk_buffs are easier to work with, as they expose additional metadata and helper functions that facilitate packet parsing and modification.  Yet this metadata can also be burdensome when performing certain modifications, as it requires additional bookkeeping.  For example, say you want to modify the initial sequence number of a TCP SYN -- it's not sufficient to modify the bits of the sequence number field in the first packet you send, you must also update metadata that tracks the socket's ISN.  XDP programs have no such metadata and therefore no such restrictions.

In most cases eBPF is probably the better choice, but if you do want egress XDP, it can be implemented in a roundabout way by making use of Linux VETH (virtual ethernet) pairs.  The basic concept is to first send traffic to yourself from inside a virtual namespace, handle it with an XDP program when "receiving" it on a virtual interface, and then forward it out the regular interface.  Step-by-step instructions are provided below, with the VETH configuration steps adapted in large part from [this](https://superuser.com/a/765078) Stack Exchange answer.  I've structured them as a single bash script so you should be able to copy and paste after configuring (at minimum) the device name of the network interface you want to send from.

![Virtual Namespace and VETH Pair Configuration](/assets/egress_xdp_namespaces.png)

# Code

```
#Set the name of the device you want to send from
send_dev=eth0

#Set addresses for the outer and inner halves of the VETH pair
outer_addr=10.0.0.1
inner_addr=10.0.0.2

# Create a new VETH pair, named veth_1a and veth_1b
ip link add dev veth_1a type veth peer name veth_1b

# Bring up the outer half of the pair
ip link set dev veth_1a up

# Create and bring up a tap device
ip tuntap add tap_1 mode tap
ip link set dev tap_1 up

# Create a bridge device
ip link add br_1 type bridge

# Attach the tap to the bridge
ip link set tap_1 master br_1

# Attach the outer half of the VETH pair to the bridge
ip link set veth_1a master br_1

# Assign the bridge an IP address and bring it up
ip addr add $outer_addr/24 dev br_1
ip link set br_1 up

# Create a new network namespace, let's call it eXDP
ip netns add eXDP

# Transfer the inner half of the VETH pair to the new namespace
ip link set veth_1b netns eXDP

# Create a loopback interface in the new namespace
ip netns exec eXDP ip link set dev lo up

# Enable Network Address Translation
iptables -t nat -A POSTROUTING -o br_1 -j MASQUERADE
iptables -t nat -A POSTROUTING -o $send_dev -j MASQUERADE

# Assign an IP to the inner half of the VETH pair and bring it up
ip -netns eXDP addr add $inner_addr/24 dev veth_1b
ip -netns eXDP link set dev veth_1b up

# Set the bridge as the default route for the network namespace
ip -netns eXDP route add default via $outer_addr
```

Now attach your XDP program to the veth_1a device.  Traffic sent from within the virtual namespace will be received and handled by the XDP program, then forwarded out $send_dev.  

To generate this traffic, we can start a new terminal session in the virtual namespace with:
```
ip netns exec eXDP bash &
```

Any traffic you generate from this session will be processed by XDP before being sent.

# Performance Analysis

Naturally there is some amount of overhead to this approach since we're adding a layer of NATing to the system, but it's not much.  In measuring 100,000 pings, containerization adds an average of ~0.015ms of latency to each request.

![](/assets/egress_xdp_benchmark.png)