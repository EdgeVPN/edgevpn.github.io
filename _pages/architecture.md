---
permalink: /architecture/
title: "EdgeVPN architecture"
header:
  overlay_color: "#303030"
  overlay_image: /assets/images/texture.png
---

# Introduction

This document describes the architecture of EdgeVPN, at a high level - not diving deep into the code, but overviewing the major modules and their interactions. The target audience is developers who are interested in contributing to the code base.

# Background

EdgeVPN is the evolution of IP-over-P2P (IPOP) codebase. IPOP has gone through significant architecture and implementation changes over time - for reference, an overview of these changes appears in the [2019 IEICE paper](https://search.ieice.org/bin/pdf_link.php?category=B&lang=E&year=2020&fname=e103-b_1_2&abst=) This architecture overview reflects the 4th generation of the architecture, which is the basis for EdgeVPN.

# Overview and terminology

EdgeVPN integrates several technologies and standards to deliver a flexible virtual network. These include NAT traversal, signaling, tunneling, network virtualization, overlay networking, and software-defined networking. To get started, the following diagram provides an overview of how various EdgeVPN modules come together, and basic terminology used throughout this document:

![EdgeVPN architecture](/assets/images/edgevpn_architecture.png)

* NAT: A [Network Address Translator](https://en.wikipedia.org/wiki/Network_address_translation)

* EdgeVPN node: a (virtual) machine that runs the EdgeVPN software. An EdgeVPN deployment consists of a set of EdgeVPN nodes, whereas nodes may have private addresses and subject to NAT(s).

* XMPP server: a (virtual) machine that runs a server compliant to the [XMPP messaging protocol](https://xmpp.org/). Typically, an EdgeVPN deployment uses a single XMPP server with a public Internet address

* STUN server: a (virtual) machine that runs a server compliant to the [STUN protocol](https://en.wikipedia.org/wiki/STUN). An EdgeVPN deployment requires at least one STUN server (possibly more, for fault-tolerance) with a public Internet address

* TURN server: a (virtual) machine that runs a server compliant to the [TURN protocol](https://en.wikipedia.org/wiki/Traversal_Using_Relays_around_NAT). An EdgeVPN deployment may use one or more TURN servers (for fault tolerance and load balancing) with a public Internet address

* WebRTC: [an open framework for the web that enables Real-Time Communications (RTC) capabilities](https://webrtc.org/). WebRTC is implemented in C++

* TinCan link: an abstraction of a virtual link that carries encapsulated, encrypted Ethernet frames between two EdgeVPN nodes. A TinCan link builds upon a WebRTC connection; the EdgeVPN tincan process implements this abstraction. The tincan code is implemented in C++

* SDN: [Software-Defined Networking](https://www.opennetworking.org/sdn-definition/). The SDN architecture decouples the network control and forwarding functions, enabling the network control to become directly programmable and the underlying infrastructure to be abstracted for applications and network services

* OpenFlow: [a standard protocol for SDN solutions](https://en.wikipedia.org/wiki/OpenFlow)

* OVS: short for [Open vSwitch](https://www.openvswitch.org/); an open-source software implementation of an SDN-programmable Ethernet switch

* tap: a [virtual network interface](https://en.wikipedia.org/wiki/TUN/TAP) that allows a user-level process (tincan) to read/write packets from the kernel. In EdgeVPN, one tap device is created for each TinCan link, and each tap device is bound to an OVS port

* EdgeVPN controller: a Python-based program responsible for the control of various aspects of EdgeVPN, including: tincan link management, topology management, and signaling through XMPP

* Ryu SDN controller: [Ryu](https://github.com/faucetsdn/ryu) is a Python-based framework for SDN controllers. EdgeVPN implements a Ryu-based controller that is responsible for programming flow rules that govern packet switching in an EdgeVPN OVS switch

* Topology: EdgeVPN nodes belonging to the same overlay are logically organized according to a graph topology, where each node is a vertex, and each TinCan link is an edge

* node ID: a unique 128-bit ID associated with each EdgeVPN node

* Symphony: the current topology implemented in EdgeVPN is a structured peer-to-peer overlay topology where nodes self-organize into a ring ordered by unique node IDs, and with randomly-assigned “long-distance” links, based on the approach described in [Symphony](http://infolab.stanford.edu/~bawa/Pub/symphony.pdf). This topology is scalable: the average distance between two nodes can scale as a log(N) function, where N is the number of EdgeVPN nodes, while each node has log(N) TinCan links

* Bounded flood: the protocol used among EdgeVPN to support broadcast (e.g. ARP) and form a basis for multicast. Bounded flood refers to the fact that, in a broadcast, nodes forward messages to "flood" their topology neighbors, but the flooding is bound by a range of node IDs to prevent loops

# EdgeVPN from the ground up

We now overview the EdgeVPN architecture, starting from the basics (an individual virtual link with NAT-constrained nodes) and climbing up in complexity to encompass the entire architecture.

## NAT traversal

One of the key features that set EdgeVPN apart from most VPNs is the fact that VPN links can connect devices that are behind different NATs. The most common VPN types typically connect a device behind a NAT (e.g. your laptop at home) to a public server (e.g. a commercial or enterprise VPN server). When you have a VPN client behind a NAT connecting to a public VPN server, NAT traversal is not necessary - the client knows how to access VPN server(s) because their IP address and port (an endpoint) are well-known and stable. In contrast, when two nodes are behind NATs, there is no well-known stable IP address and port for either - NATs map IP and port addresses dynamically. Because EdgeVPN targets deployments that may cross multiple edge providers (each of which with its own NAT) it is necessary to support links between NATed devices.

At a very high level, peer-to-peer NAT traversal requires that two nodes with private IP addresses and behind NATs: 1) discover their public IP:port endpoints mapped at their respective NATs, 2) exchange this information with each other, and 3) establish and maintain an open communication channel by periodically sending messages to each other. Occasionally, some NATs will block peer-to-peer communication as described, and nodes need to find an intermediary on the public Internet to relay messages to each other.

While this sounds relatively simple, the mechanics of managing NAT traversal across the Internet get complicated. There are standards that have been built to wrangle this complexity: STUN allows nodes to discover their NATed IP:port endpoints; XMPP allows messaging through a public service to exchange endpoints; TURN supports relays. The ICE protocol leverages STUN and TURN to generate NAT traversal candidates, and which can be successfully sent and received through NATs via XMPP. EdgeVPN builds on these standards, and leverages an open-source project (WebRTC) which implements this functionality. [This document provides a useful introduction on NAT traversal and WebRTC](https://temasys.io/webrtc-ice-sorcery/).

With WebRTC then, EdgeVPN can build the basic primitive upon which the whole systems is built - a peer-to-peer tunnel. This tunnel is private - data is encrypted before being sent over the tunnel, and decrypted on the other side. While in typical WebRTC applications, the data that goes over the tunnel is media (e.g. audio, video), in EdgeVPN we tunnel *Ethernet frames*. That way we can run any application that uses TCP/IP over Ethernet - including, but not restricted to, media. We refer to the WebRTC tunnels that are used to carry EdgeVPN Ethernet traffic as *TinCan links* - drawing an analogy with tin can telephone toys. 

In order to carry Ethernet traffic across TinCan links, we need to be able to pick frames from a computer, and inject frames into another. For this, we use network virtualization.

## Network virtualization

While a WebRTC application might pick input data from, say, a microphone, and inject data output into a speaker, EdgeVPN deals with Ethernet frames. This requires it to interface with a low-level operating system kernel, where the networking system resides. Fortunately, operating sytems feature a virtual network device - a "tap" device - which provides this binding. A tap device allows an application (in EdgeVPN, the tincan process) to read and write frames using the system call interface. So, if a client application in the computer on one side of a TinCan link sends an Ethernet frame over the virtual network interface (e.g. say the client is trying to open an HTTP session), the tincan process is able to read that frame from the tap device. 

Once a packet has been captures, it is then written to the TinCan/WebRTC tunnel. The message then is encrypted on the sender's side, and sent along the NAT-traversed path across the Internet. At the destination, the message is read from the tunnel, decrypted, and then injected into the tap interface at the destination, from which an application (e.g. a Web server) is able to read.

Through virtualizartion, the whole process is essentially hidden from the applications - the client and server do not see a difference between sending messages over EdgeVPN compared to sending messages over a local-area network. The important thing is to configure the network interfaces properly, so messages are carried over this link.

## From one link to many



## Overlay networking

## Software-defined networking

## Putting it all together