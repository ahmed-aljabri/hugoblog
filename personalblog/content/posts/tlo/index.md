---
description: 'A deep-dive into the challenges with evolving the transport layer and lessons for the future.'
title: "Transport Layer Ossification"
date: 2024-12-17T17:23:15Z
draft: true
toc: false
images:
tags:
  - untagged
---

## Introduction

I was first introduced to the ’Transport Layer Ossification’ (TLO) phenomenon when I watched a recorded
presentation by ex-google software engineer Jana Iyengar. In that session, the agenda was mainly around
QUIC, a new multiplexed transport protocol that was initially designed to enhance the web’s performance and
in particular HTTP. However QUIC later evolved to be an independent transport protocol that could support
other domains not just the web. Nevertheless, Jana started by highlighting some context before talking about
QUIC and in particular the challenges faced when attempting to deploy the Stream Control Transport Protocol
(SCTP), during that segment Jana explained how middleboxes are powerful enough that they have ossified the
transport layer making it near impossible to deploy new functional transport protocols. From there I became
curious to learn about TLO in depth, understand the direct implications on modern-day networks and the
potential workarounds.

> Note:
This post is not particularly about SCTP, QUIC or HTTP’s evolution to adopt QUIC,
I discuss the latter in more detail here. Alternatively, the context as to why these
technologies had to exist is what this blog post is all about.
It is also important to note that TLO is a subset of a more generic concept, Protocol Ossification which will be
described next to build our understanding of the topic of this post.


## Background
### 2.1 Context on protocol ossification
The general term Protocol Ossification refers to the increasing complexity of protocol innovation due to various
impairments imposed mainly by middlebox policies. In simpler terms, when a protocol becomes so ubiquitous
that it is almost practically impossible to replace or modify, we say that this protocol has been ’ossified’.

>Note:
A Middlebox as Lixia Zhang coined is a device that lies between communicating
endpoints that performs operations other than standard routing and forwarding.
Examples of such devices consist of proxies that make requests on your behalf by
rewriting packet headers, likewise NATs doing something similar to prevent your
direct exposure to the public internet. Additionally other types of middleboxes
enforce internal policies on passing traffic by striping certain TCP options and the
list goes on.

To elaborate further on what protocol ossification phenomenon means or how it came to exist, it is helpful to
begin with some context.

Since the early days of IP networks, the end-to-end principle established the idea that diversity and intelligence
should lie at the network edge while the network core should simply be responsible for supporting IP routing
and forwarding operations. This entailed that when changes needed to happen whether to facilitate new
high level functionality or improve upon existing ones, only the communicating parties (end systems) need to
support this change. While on-path nodes simply provided basic networking functionality and hence don’t
need to be aware of it.

*Figure 1: End-to-End Principle*

Following this principle, it is critical to highlight that control resides with the network edge. In this sense
control refers to the ability to dictate and modify how host-to-host network exchanges should occur by
determining which protocols to use. As illustrated in figure 1, the on path systems simply operate up to the
network layer dealing with IP packets. Hence the end systems have full control of deciding which higher level
protocol to use, not to forget the ability to create new protocols as long as they are built on top of the IP protocol.
Nevertheless, today’s networks have evolved to become much more complex that what is illustrated above.
For instance if there was an update to the HTTP protocol, the support should only be needed to occur at the
Network edge (HTTP clients and servers) according to the end-to-end principle for this update to be functional.
With the existence of middleboxes however, that is no longer the case. What happens if a client decides to
use HTTP2 but their traffic is directed through a proxy that doesn’t support it? Tough luck, in that case the
client will have to do with HTTP 1.1 until the proxy supports the upgrade. Consequently, the existence of such
middleboxes introduces additional complexity when enhancing existing protocols or introducing new ones
since they also need to be aware of such changes.

In the example above with regards to the introduced complexity of enhancing HTTP, this is considered to be of
the lower end when it comes to the severity of this phenomenon. The severity of protocol ossification increases
as we traverse down the protocols stack towards the IP protocol. Based on this we can begin to understand
why for instance enhancing HTTP is easier than TCP, HTTP is mostly used for web exchanges while TCP is
a general protocol for reliable host-to-host communication. Hence due to TCP being a more fundamental
protocol, more systems will have to be involved in deploying a major change to TCP than HTTP echoing this
varying degree of ossification.

*Figure 2: Increasing level of protocol ubiquity as you move down the protocol stack mirrors the degree of ossification*

>Note:
Although it is true that application layer protocols are more flexible to developments
than lower layers mainly due to their existence in user-space rather than kernal
space, the increasing sophistication of middleboxes that tamper with higher layers
such as Layer-7 loadbalancers is spreading this phenomenon across the stack.

Moving on from the general concept of protocol ossification, next we transition into the main focus of this post
which is the protocol ossification phenomenon specifically occurring at the transport layer ie Transport Layer
Ossification.

### 2.2 Transport Layer Ossification
As mentioned briefly above, TCP is a fundamental transport protocol for host to host communication. Evidently,
one of the most common questions that emerge when we think about system design is the choice between
using TCP and UDP as transport. More so the TCP/IP stack is nowadays synonymous with the internet because
almost every kernel in the world supports it hence if a given system wishes to communicate at global scale in
today’s networks it has to understand this stack.

With such enormous ubiquity at the transport layer, transport protocols have become foundational to network
communications and so changing them or replacing them is almost practically impossible leading to the
existence of Transport Layer Ossification which is what this blog post is all about. [link to paper (How practical
is it to make changes to TCP)]

## What causes such a phenomenon to exist?
With the above in mind, how do middleboxes actually cause such a phenomenon ?
The way in which networks are governed is in some ways similar to how different states around the world
govern themselves, in different networks different policies are enforced where some are more restricted than
others. In such networks, strict policies are usually enforced for security and control reasons which often
translates to the deployment of highly sophisticated middleboxes (Proxies, Firewalls, IDS, IPS etc) that inspect
passing traffic and takes action based on the implemented polices.
So in order for middleboxes to be able to deeply inspect traffic to perform various impairments reflecting their
policy, such devices need to be sophisticated enough that they operate at layers further than the Network layer
as illustrated in figure X. This way, the described end-to-end approach discussed earlier is broken due to the
spread of sophisticated systems performing functions other what a standard IP router does across networks as
shown below and not just at the edge.

*Figure 3: Distributed share of sophisticated systems across networks*

We can elaborate further on how middleboxes result in such a protocol using the figure above. We observer
that the end system to the right has it’s traffic directed through a firewall that operates at the transport layer.
In other words the firewall inspects the visible headers at the transport layer and makes a decision either to
allow or deny passing traffic. On the other side, assuming that the traffic is coming from the left, the layer 7
loadbalancer could be put in place to distribute the traffic across the end systems behind it, thereby spreading
the traffic load. In the case of a HTTP exchange, the loadbalancer will inspect the incoming HTTP packets and
base its balancing decision on the HTTP headers such as the resource requested, request type, etc.

With such sophistication at the network core, protocol development have to take into account the support
of those systems to be functional today. Say that the end system to the right in the example above wishes to
establish a SCTP connection in the example above, but the firewall on path has based its rules on the most
commonly used transport protocols UDP and TCP. In this case the firewall is likely to block this traffic as it is
unable to understand it, this will likely be the case in more restricted networks.

Another example from the above would be regarding the layer 7 loadbalancer distributing HTTP traffic among
web servers. If a major change to the HTTP protocol is made that changes the header structure or adds
additional headers, without the support of the loadbalancer it will not be able to perform its loadbalancing
due to the unfamiliar HTTP header structure.

Consequently, the spread of intelligence across the core of networks makes it extremely hard to innovate
protocols and it only gets worse as more and more devices support these legacy protocols.
As a result of the situation described above, it natural to ask the simple question, why should we care? In the
following section I attempt to address a few of those concerns.

## 4 Why Protocol Ossification is a problem today?
Over time, middleboxes develop assumptions and patterns on network traffic based on its observations at the
various protocol layers they inspect which is then classified to usual or unusual categories. Take for example
the TCP protocol, middleboxes have become accustomed to the frequent types of options that is typically seen
and as a result, in strict domains if two parties attempt to establish a TCP connection with a relatively new
or unusual option, it is likely that some middleboxes will intervene either to completely block or strip the
TCP option causing at least some added latency or more seriously a complete hault to the communication.
Examples of this has been studied by Korian Edeline and Benoit Donnet in their respective paper that looked
at the Multi-Path TCP (MPTCP) and the Selective Acknowledgements (SACK) TCP options that showed to
often experience impairments by middleboxes. After all, this makes it extremely hard to enhance such legacy
protocols.

Given that middleboxes develop such assumptions and enforce strict policies on passing traffic, this forces
applications to build on top of such ossified protocols just so that they can coexist with middleboxes. With the
distribution of control across the edge and centre of networksThis is an issue because such legacy protocols
were created when the internet was way less complex than it is today and therefore the abilities of such legacy
protocols fall short to fulfil the evolving requirements of modern networks.

A modern example of the above can be seen with the evolution of HTTP which I discuss in some detail here.
However in short, The ability of HTTP to enhance its performance to fit the needs of the modern web was
limited by it’s dependence on TCP at the transport layer. In the end the decision was to instead use UDP, a
lightweight transport layer protocol and implement the needed transport functions on top of that at a higher
level with QUIC (see figure x) freeing them from the performance latency’s of TCP’s legacy design.

## 5 Workarounds
• End-to-End Encryption
exposure of packet headers is what enables middleboxes to ossify protocols and build assumptions, they
cant do that when unnecessary headers are encrypted.
As we know from foundational protocol studies, the lower we move down the protocol stack the closer we are
to the actual hardware. In this case if we observe the separation between the user and kernal space domains,
we see that it it traditionally lies between the transport and application layers.

*Figure 4: Traditional protocol scope*

From that we can make sense of transport layer protocols ubiquity and why they have been ossified over time.
The kernel space is typically where minimal core functions are implemented unlike user space where a much
larger variety of other operations reside. Therefore, protocols like TCP and UDP are implemented in kernel
space due to them being fundamental to network devices. And so if any developments are made to TCP they
have to be pushed to system kernels world-wide for the change to be operational globally.

This also explains why we tend to see new modifications of TCP implemented and developed for certain
domains such as Data Center TCP (DCTCP) which is a TCP version that is tailored for Data Center operations.
We dont see those versions deployed world wide for the reasons mentioned above and also for the simple fact
that sometimes it doesnt provide much benifit to the normal end user.

That being said, regardless of the fact that transport protocols have traditionally resided in kernel space, that
does not mean that certain transport functions cant be moved to the user space scope. And you might wonder
why would we be interested in that?

As we learn from QUIC and SCTP, the attempt to enhance TCP or develop a protocol from scratch (in kernel
space) that better fits the needs of today’s networks is almost impossible, thanks to middleboxes and the
complexity involved with making world-wide system kernels support this development. In QUIC’s case, in
order to bypass these limitations the solution involved another approach which was to build on top of an
already supported or for a better term ossified protocol in this case UDP.

*Figure 5: Transport layer breakthrough from it’s traditional scope within the kernel space*

What this does is it eliminates the challenge of attempting to coexist with middleboxes and moves the needed transport functions (congestion control, packet reordering etc) to user space where there is much more flexibility of deployment and support. With HTTP3, this approach is implemented where QUIC contains the
needed transport functions and assumes that the very lightweight transport protocol UDP is beneath it.

*Figure 6: Difference between TCP and UDP Header Formats from RFCs 761 and 768*

With that being said, it leads us to question the effectiveness of this workaround which I intend to explore in
the following.
- How effective is this workaround?
- How will middleboxes act with more frequent UDP traffic (HTTP3, DNS-over-QUIC ...)?
- how easy is it to block QUIC?
- best solution we have or are there others?

*Figure 7: Transport layer breakthrough from it’s traditional scope within the kernel space*

## Closing remarks


