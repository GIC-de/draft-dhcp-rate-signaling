---
title: "DHCP Explicit Rate Signaling"
docname: draft-ietf-intarea-dhcp-rate-signaling-latest
submissiontype: IETF
ipr: trust200902
category: std
v: 3
area: Internet Area
workgroup: Internet Area Working Group
keyword:
  - DHCP
  - DHCPv6
  - broadband
  - BNG
  - CPE
  - AQM
  - L4S

venue:
  group: "Internet Area"
  type: "Working Group"
  mail: "int-area@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/int-area/"
  github: "GIC-de/draft-dhcp-rate-signaling"
  latest: "https://GIC-de.github.io/draft-dhcp-rate-signaling/draft-ietf-intarea-dhcp-rate-signaling.html"
  text: >-
    Broadband-specific aspects of this document can also be discussed on the
    non-WG broadband mailing list ([broadband@ietf.org](mailto:broadband@ietf.org)),
    which is archived at <https://mailarchive.ietf.org/arch/browse/broadband/>.
    The Internet Area Working Group mailing list remains the primary venue for
    this document.

author:
- fullname: "Christian Giese"
  organization: "RtBrick"
  country: "Germany"
  email: "christian@rtbrick.com"
- fullname: "Richard Patterson"
  organization: "Sky UK"
  country: "United Kingdom"
  email: "Richard.Patterson@sky.uk"

informative:
   TR101:
      title: "Migration to Ethernet-Based Broadband Aggregation"
      target: https://www.broadband-forum.org/pdfs/tr-101-2-0-0.pdf
      author:
      - org: Broadband Forum
      date: 2011-07
      seriesinfo:
         TR: 101 Issue 2
   TR069:
      title: "CPE WAN Management Protocol v1.1"
      target: https://www.broadband-forum.org/pdfs/tr-069-1-1-0.pdf
      author:
      - org: Broadband Forum
      date: 2006-11
      seriesinfo:
         TR: 069 Issue 1 Amendment 1
   G.984.1:
      title: "Gigabit-capable passive optical networks (GPON): General characteristics"
      target: https://www.itu.int/rec/T-REC-G.984.1
      author:
      - org: ITU-T
      date: 2008-03
      seriesinfo:
         ITU-T Recommendation: G.984.1

--- abstract

This document defines Dynamic Host Configuration Protocol (DHCP) options for DHCPv4 and DHCPv6 that
signal the intended upstream and downstream rates of a network connection to clients and
intermediate nodes. In many broadband access networks, the provisioned upstream rate is enforced at
the service edge by ingress policers, while the Customer Premises Equipment (CPE) sends at the speed
of its physical link. The resulting policer drops cause unstable and reduced throughput and render
upstream prioritization at the CPE ineffective. By learning the intended upstream rate via DHCP, the
CPE can shape its upstream traffic, moving the queue to the CPE, where it can be managed by
prioritization, Active Queue Management (AQM), and the Low Latency, Low Loss, and Scalable
Throughput (L4S) architecture. Intermediate nodes acting as DHCP relay agents or performing DHCP
snooping can use the same information to instantiate per-subscriber shapers and policers. All rates
are Layer 3 rates, which match the speed perceived by users and can be applied directly to common
traffic shapers without encapsulation-specific calculations.

--- middle

# Introduction

## Problem Statement

In broadband access networks, the rate available to a subscriber is typically lower than the speed
of the physical link that connects the Customer Premises Equipment (CPE) to the network. The
provisioned rate is commonly enforced at the Broadband Network Gateway (BNG) {{TR101}}, which limits
the traffic of each subscriber to the purchased service tier. In addition, the upstream rate can be
limited by an access device connected to the CPE, such as an Optical Network Terminal (ONT)
{{G.984.1}}, a Digital Subscriber Line (DSL) modem, or a cable modem, or within the access network,
for example by the upstream bandwidth allocation of a passive optical network, by the
synchronization rate of a DSL line, or by the upstream rate limit enforced by a cable modem. The CPE
is typically unaware of these rates and sends at the speed of its physical link.

In the upstream direction, the provisioned rate is typically enforced at the BNG on traffic
received from the subscriber. Per-subscriber queuing in the upstream direction is not supported
by all BNG platforms. Where it is supported, it is often restricted to specific hardware, subject
to additional constraints, or available only at reduced scale, as it competes for the same
queuing and scheduling resources as downstream hierarchical QoS. Upstream enforcement is
therefore commonly implemented using ingress policers. A policer does not queue packets; it
discards all packets that exceed the configured rate and burst size. As the CPE sends at the
speed of its physical link, which can exceed the provisioned rate by an order of magnitude,
traffic bursts are discarded at the policer. Congestion control algorithms react to these losses
by repeatedly reducing their sending rate, resulting in an unstable sawtooth throughput pattern,
and the achieved throughput often remains well below the provisioned rate. Users experience this
as poor and fluctuating results, for example in speed tests, even though no part of the network
is congested.

Where the upstream bottleneck is located at the access device or within the access network, the
resulting queue is typically managed by neither Active Queue Management (AQM) {{?RFC7567}} nor
prioritization of the subscriber's traffic. Depending on the buffering of the access device, this
results in packet loss, with effects similar to those of a policer, or in increased latency under
load. In both cases, all traffic of the subscriber is affected alike, including latency-sensitive
applications.

Many CPE implementations support upstream prioritization, for example of DNS, Voice over IP
(VoIP), online gaming, or video conferencing traffic, of TCP acknowledgments, or of selected
clients or applications. Such prioritization is only effective if the CPE is the bottleneck of
the upstream path. As long as the CPE sends at the speed of its physical link, its queues remain
empty, and packets are discarded or delayed at the actual bottleneck, such as the policer of the
BNG or the access device, regardless of their priority.

In the downstream direction, the provisioned rate is typically enforced at the BNG using egress
shapers, where per-subscriber queuing is widely available. The downstream rate is therefore of lower
relevance for traffic management at the CPE, although a CPE can use it to control downstream queuing
delay where the shaper at the BNG does not apply AQM. It is, however, relevant for intermediate
nodes that apply per-subscriber traffic management in both directions, and it is useful for
informational purposes, for example to display the provisioned service tier in the user interface of
the CPE.

A similar lack of visibility exists within local area networks (LANs), such as home, enterprise,
or public Wi-Fi hotspot networks, where end devices are typically connected at link speeds that
exceed the capacity of the upstream connection (see {{lan}}).

## Solution Overview {#solution-overview}

This document defines a Rate Option for each of DHCPv4 and DHCPv6. The option has a fixed format
and carries exactly two values: the intended upstream rate and the intended downstream rate, each as
a 64-bit unsigned integer in bits per second. The rates are always Layer 3 rates: they refer to the
IP header and payload of each packet as transmitted on the link of the receiver (see {{layer3}}).

If the CPE knows the upstream rate, it can shape its upstream traffic to this rate, regardless of
where the bottleneck is located. The queue then builds at the CPE, where it can be managed using
prioritization, AQM, and flow queuing {{?RFC8290}}. Under
normal operation, the policer at the BNG is not triggered, and the access device is not congested
(see {{fig-policer}}). The resulting upstream throughput is stable and close to the signaled rate,
and latency under load remains low. When combined with the Low Latency, Low Loss, and Scalable
Throughput (L4S) architecture {{?RFC9330}}, the queue at the CPE can additionally provide low
latency for L4S traffic.

~~~ aasvg
                        Without rate signaling

+-------------+  1 Gbit/s bursts  +--------+    +-------------------+
|     CPE     |------------------>| Access |--->|        BNG        |
|             |                   +--------+    | ingress policer   |
|             |                                 | 100 Mbit/s: drops |
+-------------+                                 +-------------------+

                         With rate signaling

+-------------+ shaped 100 Mbit/s +--------+    +-------------------+
|     CPE     |------------------>| Access |--->|        BNG        |
| shaper, AQM |                   +--------+    | ingress policer   |
| 100 Mbit/s  |                                 | 100 Mbit/s: idle  |
+-------------+                                 +-------------------+
~~~
{: #fig-policer title="Upstream Policing at the BNG with and without CPE Shaping"}

The enforcement at the BNG remains in place. The shaper at the CPE does not replace this
enforcement but prevents it from being triggered. CPE that do not support the options defined in
this document continue to be subject to the enforcement at the BNG, which can be configured
slightly above the signaled rate for this purpose (see {{processing}}).

Signaling Layer 3 rates keeps the mechanism simple. A Layer 3 rate corresponds to the speed
perceived by users and measured by speed tests, and it is independent of the encapsulation used on
any segment of the path. The same value is therefore valid at the CPE, at an access node, and at the
BNG, irrespective of Virtual LAN (VLAN) tags or Point-to-Point Protocol over Ethernet (PPPoE)
{{?RFC2516}} headers added or removed along the path, and it can be configured directly into common
CPE shapers, such as the Common Applications Kept Enhanced (CAKE) queue discipline, without
encapsulation-specific calculations (see {{application}}).

The same rate information can be used by intermediate nodes. In many IP over Ethernet (IPoE)
{{TR101}} architectures, the BNG operates as a DHCP relay agent. While per-subscriber traffic
management policies are typically provisioned via Authentication, Authorization, and Accounting
(AAA) protocols such as Remote Authentication Dial-In User Service (RADIUS) {{?RFC2865}},
deployments lacking AAA integration at the service edge require an in-band alternative. The BNG can
use the signaled rates to instantiate downstream shapers and upstream policers, and access nodes
performing DHCP snooping can use them to configure their local queues and shapers. DHCPv6
additionally allows the server to signal different rates to the client and to each relay agent in
the path (see {{dhcpv6-server}}).

## Applicability

The mechanism defined in this document is primarily designed for broadband access networks using
IPoE or PPPoE. In PPPoE deployments, DHCPv6 is commonly used over the Point-to-Point Protocol (PPP)
{{?RFC1661}} session for IPv6 Prefix Delegation and conveys the options in the same way (see
{{pppoe}}). The options are defined for both DHCPv4 and DHCPv6 to support IPv4-only, IPv6-only, and
dual-stack deployments.

Unlike auto-configuration protocols such as TR-069 {{TR069}}, which are not deployed by all
operators and are typically not available for customer-owned CPE, DHCP is supported by virtually
all CPE, including retail routers and devices running open-source firmware (e.g., OpenWrt), and
it is visible to intermediate nodes along the path. Rates conveyed via DHCP are refreshed with
every lease renewal and can be updated dynamically. The options are also applicable within LANs
(see {{lan}}).

{{rationale}} discusses the reasoning behind this mechanism and addresses frequently raised
questions, such as why rate enforcement at the BNG alone is not sufficient, why upstream shaping is
not performed at the BNG, and why only Layer 3 rates are signaled.

# Conventions and Definitions {#definitions}

{::boilerplate bcp14-tagged}

This document makes use of the terms defined in {{!RFC2131}} and {{!RFC9915}}.
The following additional terms are used:

Binding
: The state that an intermediate node maintains for a client, associating the client with its
  client-facing port or session and with the addresses or prefixes leased to it, for example the
  subscriber session of a BNG or the binding table of a snooping device.

DHCP
: The abbreviation DHCP is used throughout this document to refer to both DHCPv4 and DHCPv6
  protocols.

DHCPv4
: The Dynamic Host Configuration Protocol {{!RFC2131}}.

DHCPv6
: The Dynamic Host Configuration Protocol for IPv6 {{!RFC9915}}.

Default Configuration
: The configuration of the traffic management functions of a receiver for a given direction that
  applies when no signaled rate is in effect for that direction. It consists of any rate or traffic
  management policy statically configured by the administrator of the receiver for that direction
  or, if no such configuration exists, of no rate limitation by the receiver, i.e., traffic is
  forwarded up to the capacity of the physical link. The default configuration does not depend on
  rates previously learned from a Rate Option.

Downstream
: The direction of traffic from the network towards the client. For a client, this corresponds to
  traffic received over the link on which the Rate Option was received. For a relay agent or
  snooping device, this corresponds to traffic sent on the client-facing interface or port.

Enforcement Point
: The node or link at which the provisioned rates are enforced or which constitutes the bottleneck
  of the path, for example a policer at the BNG or an access link.

Intended Rate
: The maximum rate, in a given direction, at which the network intends to forward the traffic of a
  client. Traffic exceeding the intended rate is subject to being dropped or delayed at the
  enforcement point. The intended rate is not a guarantee: the achievable rate can be lower, for
  example due to congestion elsewhere in the network. The Rate Option signals the intended rates;
  this document also refers to them as the signaled rates.

Intermediate Node
: A relay agent or snooping device on the path between a client and a server that processes the Rate
  Option, for example a BNG acting as a DHCP relay agent or an access node performing DHCP snooping.

Layer 3 Rate
: A rate calculated over the outermost IP header of each packet and its payload, as transmitted on
  the link of the receiver (see {{layer3}}).

Provisioned Rate
: The rate configured in the network for a subscriber according to the subscribed service, typically
  enforced at the enforcement point (e.g., the BNG). The intended rates are typically derived from
  the provisioned rates.

Rate Option
: The DHCPv4 Rate Option (OPTION_V4_RATE) or the DHCPv6 Rate Option (OPTION_V6_RATE) defined in this
  document. Where the distinction is not relevant, this document refers to both as the Rate Option.

Receiver
: Any node that processes a received Rate Option, including DHCP clients, relay agents (including
  Lightweight DHCPv6 Relay Agents {{!RFC6221}} and Layer 2 DHCPv4 Relay Agents {{TR101}}), and DHCP
  snooping devices.

Subscriber
: The customer of a network access service. Within the network, a subscriber is typically
  represented by a subscriber session at the BNG (e.g., an IPoE or PPPoE session), to which the
  provisioned rates and other per-subscriber policies are applied.

Upstream
: The direction of traffic from the client towards the network. For a client, this corresponds to
  traffic sent over the link on which the Rate Option was received. For a relay agent or
  snooping device, this corresponds to traffic received on the client-facing interface or port.

# Rate Option {#rate-options}

This document defines one Rate Option for each of DHCPv4 and DHCPv6. The encoding of the option
follows the conventions of the respective base protocol (see {{dhcpv4}} and {{dhcpv6}}). The value
of the option has the same fixed format in both protocols, and its semantics are identical and are
specified in this section.

The value of the Rate Option consists of two fields, in the following order:

Upstream Rate:
: A 64-bit unsigned integer in network byte order, indicating the intended rate for the upstream
  direction in bits per second (bps).

Downstream Rate:
: A 64-bit unsigned integer in network byte order, indicating the intended rate for the downstream
  direction in bits per second (bps).

The 64-bit format allows rates beyond the approximately 4.29 Gbit/s that can be represented by
32-bit rate attributes in bits per second, such as those defined in {{?RFC4679}}.

A value of 0 in either field indicates that no rate is signaled for the respective direction. A
receiver that has previously applied a rate learned from a Rate Option for that direction MUST
remove that rate and revert to its default configuration for that direction (see {{processing}}).

## Layer 3 Rates {#layer3}

All rates signaled by the Rate Option are Layer 3 rates. A Layer 3 rate is calculated over the
outermost IP header of each packet (including any IPv6 extension headers) and its payload, as
transmitted or received on the link of the receiver. For a client, this is the link on which the
Rate Option was received; for a relay agent or snooping device, it is the client-facing link. The
Layer 3 rate excludes all headers and trailers below the outermost IP header, such as Ethernet
headers, VLAN tags, PPPoE and PPP headers, and the Ethernet Frame Check Sequence
(FCS), as well as all physical-layer overhead.

Consequently, any tunnel or translation applied by the receiver itself is part of the Layer 3 rate.
For example:

- For a client acting as a Dual-Stack Lite (DS-Lite) Basic Bridging BroadBand (B4) element
  {{?RFC6333}} or as a Mapping of Address and Port with Encapsulation (MAP-E) Customer Edge
  {{?RFC7597}}, the Layer 3 rate covers the outer IPv6 header and the encapsulated IPv4 packet.
- For a client using translation, such as Mapping of Address and Port using Translation (MAP-T)
  {{?RFC7599}} or the customer-side translator of 464XLAT {{?RFC6877}}, the Layer 3 rate covers the
  translated packet as transmitted on the link.
- For a client that encapsulates traffic in an IP-based tunnel on the link of the receiver, such as
  an IPsec, Generic Routing Encapsulation (GRE), or Layer 2 Tunneling Protocol (L2TP) tunnel, the
  Layer 3 rate covers the outer IP header and all headers and payload carried within it.

A receiver that applies its traffic management functions to the inner traffic of such a tunnel or
translation, rather than to the packets on the link, accounts for the resulting difference locally,
as the encapsulation or translation is performed by the receiver itself. Encapsulations added or
removed by other nodes between the receiver and the enforcement point, such as an L2TP tunnel
between an L2TP Access Concentrator (LAC) and an L2TP Network Server (LNS) {{?RFC2661}}, are
not part of the Layer 3 rate of the receiver; they are treated like any other difference in
accounting at the enforcement point (see {{operations}}).

Layer 3 rates have the following properties:

- They correspond to the speed perceived by users and measured by common speed test applications,
  and to the rate in which service tiers are typically described and marketed.
- They are independent of the link-layer encapsulation used on the link of the receiver and on any
  other segment of the path. Unless an IP-based tunnel is added or removed between the receiver and
  the enforcement point, a single value applies at every node, regardless of VLAN tags or PPPoE
  headers that are added or removed along the path.
- They can be applied directly by traffic management functions that account traffic at the IP
  layer, such as shapers attached to a PPP or IP interface, or shapers that can be configured to
  account the network-layer packet size (see {{application}}).

The Rate Option does not describe the per-packet accounting of the enforcement point. Operators
account for differences between the Layer 3 rate and the accounting of the enforcement point when
configuring the signaled rates or the enforcement point (see {{operations}}).

## Traffic Management Using Signaled Rates {#application}

The Rate Option conveys rates that are intended to be applied to local traffic management functions,
such as traffic shapers, policers, or AQM parameters. A client that supports such traffic management
functions SHOULD apply the upstream rate and MAY apply the downstream rate. Relay agents and
snooping devices MAY apply the rates in either direction (see {{dhcpv4}}, {{dhcpv6}}, and
{{snooping}}).

From the perspective of a client, the upstream rate is the relevant one: downstream traffic is
typically already shaped by the network before it reaches the client, whereas upstream traffic is
enforced by the network only after it has left the client. Where the downstream shaper at the BNG
does not apply AQM, a client can additionally shape received traffic slightly below the downstream
rate to control queuing delay. Intermediate nodes typically apply the rates in both directions.

A receiver whose traffic management functions account traffic at the IP layer applies the signaled
rate directly. A receiver whose traffic management functions account link-layer frames SHOULD, where
supported, compensate for the link-layer headers that are present on its own interface, such as the
Ethernet, VLAN, PPPoE, and PPP headers, all of which are known locally, so that the effective rate
corresponds to the signaled Layer 3 rate. A receiver without such support MAY apply the signaled
rate to its link-layer accounting directly. As the accounted frame is larger than the IP packet, the
resulting effective Layer 3 rate is slightly lower than the signaled rate, which does not trigger
the enforcement at the enforcement point.

A receiver that does not apply a signaled rate, for example because it does not support the required
traffic management functions, because its processing capacity is not sufficient to apply traffic
management at the signaled rate, or because applying the downstream rate is optional for clients,
treats the rate as information only. In particular, such a receiver can expose the rate in its
management interfaces (see {{operations}}).

Explicit configuration by the administrator of the receiver, such as disabling the application of
signaled rates or configuring fixed rates or per-packet overhead values, takes precedence over the
signaled rates. In the absence of such configuration, the signaled rates take precedence over the
default configuration of the receiver.

## Rate Processing {#processing}

The processing of the Rate Option is stateless with respect to the DHCP protocol version and the
DHCP lease, such that it can be implemented by simple event-driven scripts on the client. A receiver
only needs to retain the rate currently in effect for each direction.

Rates are maintained per link and per direction. DHCPv4 and DHCPv6 messages received over the same
link share the rates in effect, even if they are processed on different logical interfaces of the
receiver.

Each valid Rate Option received in a DHCPv4 DHCPACK message or a DHCPv6 Reply message (or, for
DHCPv6 relay agents, in the corresponding Relay-reply message) replaces the rates in effect for both
directions, regardless of whether the previous rates were learned via DHCPv4 or DHCPv6. For each
direction:

- If the value is non-zero, the receiver applies the signaled rate as specified in {{application}}.
- If the value is 0, the receiver MUST remove any rate previously applied for that direction and
  revert to its default configuration for that direction (see {{definitions}}), i.e., to any
  statically configured local policy or, if none exists, to no rate limitation by the receiver.

If a DHCPACK or Reply message does not contain a Rate Option, the receiver MUST retain the rates in
effect unchanged.

In some deployments, DHCP renewals occur frequently, and most of them carry unchanged rates. A
receiver therefore compares the signaled rates with the rates currently in effect and treats the
processing as idempotent: if a signaled rate is identical to the rate currently in effect, the
receiver SHOULD NOT reconfigure its traffic management functions for that direction. A stateless
implementation can achieve this by storing the rates currently in effect and comparing them before
any reconfiguration.

For clients, the rates in effect are not bound to the lifetime of the DHCP lease from which they
were learned. Relay agents and snooping devices remove the rates applied for a client together with
the corresponding binding (see {{dhcpv4}}, {{dhcpv6-relay}}, and {{snooping}}). An intermediate node
that does not maintain bindings removes the rates applied for a client when the lease or the
longest valid lifetime conveyed in the message that carried the Rate Option expires without being
extended, or when the client-facing port or session is terminated. A receiver MUST
revert to its default configuration when the underlying link or session (e.g., the PPPoE session) is
terminated.

Relay agents and snooping devices apply signaled rates to the traffic of an individual client.
This document assumes that each such client is associated with a dedicated physical or logical
client-facing port, such as an access line or a per-client VLAN (e.g., the 1:1 VLAN model
described in {{TR101}}). If multiple clients share the same client-facing port, and the device is
not able to apply rates to the traffic of each client individually, the device SHOULD NOT apply
the signaled rates to that port.

Intermediate nodes that enforce the signaled rates MAY apply an implementation-specific
correction factor, for example by enforcing a rate slightly above the signaled rate (e.g., by
1%). This allows a client that supports this option to shape its traffic to the signaled rate
without triggering the enforcement at the intermediate node, while the intermediate node still
enforces the provisioned rate for clients that do not support or ignore this option.
Alternatively, in DHCPv6, the server can signal node-specific rates (see {{dhcpv6-server}}).

A client can receive a rate that exceeds the maximum physical link speed of its interfaces (e.g.,
2 Gbit/s signaled to a CPE with a 1 Gbit/s Wide Area Network (WAN) port). In such scenarios, the
client SHOULD cap the applied traffic shaping, policing, or AQM parameters to the capacity of the
physical link.

In dual-stack deployments, DHCPv4 and DHCPv6 are frequently served by independent servers, for
example a centralized DHCPv4 server and a DHCPv6 server co-located with the BNG, which cannot
inspect each other's state. Network operators MUST ensure that the DHCPv4 and DHCPv6 services for
the same client are configured with identical intended rates, or that only one of the two protocols,
which is expected to be negotiated by every client, conveys the Rate Option. In the latter case,
messages of the other protocol do not carry a Rate Option and therefore leave the rates in effect
unchanged.

Even with identical configuration, the two protocols can transiently convey different rates, for
example while a rate change is being provisioned on both servers, or because the renewal times of
DHCPv4 and DHCPv6 are not aligned. As the most recently received valid Rate Option takes precedence,
a receiver can then alternate between the old and the new rates until both protocols convey the new
rates. Operators can minimize this period by updating the rates on both servers together and, where
supported, by triggering an immediate update via DHCPFORCERENEW or Reconfigure (see
{{dhcpv4-server}} and {{dhcpv6-server}}).

The processing specified in this section is stateless by design, so that it can be implemented by
simple scripts. Implementations MAY maintain additional state to improve robustness, provided that
the rates eventually applied correspond to the most recently received valid Rate Option. For
example, an implementation can:

- apply a hold-down or hysteresis timer, delaying the reconfiguration of its traffic management
  functions after a change until the signaled rates have been stable for a short period, to avoid
  repeated reconfiguration when DHCPv4 and DHCPv6 transiently convey different rates;
- track the protocol from which the rates in effect were learned, in order to detect and log
  conflicting rates conveyed via DHCPv4 and DHCPv6; or
- revert to its default configuration on reconnect events, for example when a new DHCP exchange is
  initiated after the lease has expired or when the DHCP server changes, until a Rate Option is
  received again.

Such behavior is desirable but not required for interoperability.

## Error Handling {#errors}

A Rate Option whose length is not 16 octets is malformed. A receiver MUST discard a malformed Rate
Option, which is then treated as if it were not present (see {{processing}}).

A sender MUST NOT include more than one instance of the Rate Option in a DHCPv4 message, or at the
same encapsulation level of a DHCPv6 message (i.e., within the message exchanged with the client or
within a given Relay-reply message). A receiver that receives more than one instance MUST discard
all instances. For DHCPv4, this is consistent with {{?RFC3396}}: the concatenation of multiple
instances is not 16 octets long and is therefore malformed.

A Rate Option contained in a message other than those listed in {{dhcpv4}} and {{dhcpv6}} MUST be
ignored.

# DHCPv4 {#dhcpv4}

## DHCPv4 Rate Option

The DHCPv4 Rate Option (OPTION_V4_RATE) has the following format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Code (TBD1)  |   Length=16   |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                     Upstream Rate (64 bits)                   |
+                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                    Downstream Rate (64 bits)                  |
+                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-v4-format title="DHCPv4 Rate Option Format"}

Code:
: TBD1.

Length:
: 16.

Upstream Rate, Downstream Rate:
: The Layer 3 rates in bits per second, each as a 64-bit unsigned integer in network byte order
  (see {{rate-options}}).

The DHCPv4 Rate Option can appear in the following messages:

| Message                                  | Use                                        |
| :--------------------------------------- | :----------------------------------------- |
| DHCPOFFER                                | Server selection (see {{dhcpv4-client}})   |
| DHCPACK                                  | Rate information (see {{processing}})      |
{: #tab-appearance-v4 title="Appearance of the DHCPv4 Rate Option"}

## DHCPv4 Client Behavior {#dhcpv4-client}

DHCPv4 clients that support the Rate Option SHOULD include its code in the Parameter Request List
(PRL). This signals client support, enabling the DHCPv4 server to determine whether to include the
Rate Option in its response. Furthermore, it provides network
operators with visibility into client capabilities, which aids in troubleshooting and resolving
customer service quality complaints.

A client MUST NOT apply rates received in any DHCPv4 message other than DHCPACK. A client MAY use
the rates contained in a DHCPOFFER message as a criterion for selecting among multiple offers.
Rates received in a DHCPACK message are processed as specified in {{processing}}.

## DHCPv4 Server Behavior {#dhcpv4-server}

When a DHCPv4 server that supports the Rate Option receives a client message whose PRL includes the
code of the Rate Option, and a rate is configured for the client for at least one direction, the
server MUST include the Rate Option in the resulting DHCPOFFER and DHCPACK messages, with a value
of 0 for a direction for which no rate is configured. Because a missing Rate Option leaves the rates
in effect unchanged, a server that needs to remove previously signaled rates MUST include the Rate
Option with a value of 0 for the respective directions.

The server MAY derive the rates from local configuration profiles, centralized AAA systems such as
RADIUS, or external policy servers.

A server MAY include the Rate Option in its responses even if the client did not request it via the
PRL, provided the operator has explicitly configured the server to do so, for example to
provision Layer 2 DHCPv4 Relay Agents or snooping devices.

To update the rates of a client before its next scheduled renewal, a server MAY send a
DHCPFORCERENEW message {{?RFC3203}}, causing the client to initiate a renewal transaction.
DHCPFORCERENEW messages are subject to the authentication requirements of {{?RFC3203}}, which
can be satisfied using the mechanism specified in {{?RFC6704}}. For clients that do not support
DHCPFORCERENEW, updated rates take effect with the next renewal, the timing of which is
controlled by the renewal (T1) time value.

## DHCPv4 Relay Agent Behavior

DHCPv4 relay agents, including Layer 2 DHCPv4 Relay Agents {{TR101}}, MAY extract the Rate Option
from DHCPACK messages traversing the network and use them to instantiate shapers and policers on
their client-facing interfaces. Relay agents SHOULD remove rates applied for a client when the
corresponding binding is removed. Layer 2 DHCPv4 Relay Agents MUST only process a Rate Option
contained in DHCPACK messages received on trusted, network-facing ports.

Modifying the message destined for the client is outside the relay agent behavior specified in
{{!RFC2131}} and {{?RFC3046}}. A device that populates, modifies, or removes the Rate Option
destined for the client based on subscriber attributes (e.g., retrieved from an AAA server such as
RADIUS) therefore acts as a DHCPv4 server or DHCPv4 proxy towards the client, and the server
behavior specified in this document applies to it.

# DHCPv6 {#dhcpv6}

## DHCPv6 Rate Option

The DHCPv6 Rate Option (OPTION_V6_RATE) has the following format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      option-code (TBD2)       |         option-len=16         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                    Upstream Rate (64 bits)                    +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                   Downstream Rate (64 bits)                   +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-v6-format title="DHCPv6 Rate Option Format"}

option-code:
: TBD2.

option-len:
: 16.

Upstream Rate, Downstream Rate:
: The Layer 3 rates in bits per second, each as a 64-bit unsigned integer in network byte order
  (see {{rate-options}}).

The DHCPv6 Rate Option can appear in the following messages:

| Message                                  | Use                                        |
| :--------------------------------------- | :----------------------------------------- |
| Advertise                                | Server selection (see {{dhcpv6-client}})   |
| Reply to Solicit, Request, Renew, Rebind, or Information-request | Rate information (see {{processing}}) |
| Relay-reply                              | Rate information for the relay agent (see {{dhcpv6-relay}}) |
{: #tab-appearance-v6 title="Appearance of the DHCPv6 Rate Option"}

The Rate Option is not included in Reconfigure messages, as specified in
{{Section 18.3.11 of !RFC9915}}.

## DHCPv6 Client Behavior {#dhcpv6-client}

DHCPv6 clients that support the Rate Option SHOULD include its code in the Option Request option
(ORO) {{!RFC9915}}, for the same reasons as specified for
DHCPv4 in {{dhcpv4-client}}.

A client MUST NOT apply rates received in any DHCPv6 message other than Reply. A client MAY use
the rates contained in an Advertise message as a criterion for selecting among multiple Advertise
messages. Rates received in a Reply message are processed as specified in {{processing}}.

## DHCPv6 Server Behavior {#dhcpv6-server}

When a DHCPv6 server that supports the Rate Option receives a Solicit, Request, Renew, Rebind, or
Information-request message whose ORO includes the code of the Rate Option, and a rate is configured
for the client for at least one direction, the server MUST include the Rate Option in the resulting
Advertise and Reply messages, with a value of 0 for a direction for which no rate is configured. A
server that needs to remove previously signaled rates MUST include the Rate Option with a value of 0
for the respective directions.

A server MAY include the Rate Option in its responses even if the client did not request it via the
ORO, provided the operator has explicitly configured the server to do so.

In relayed exchanges, the Rate Option for the client is included in the Reply message, which is
encapsulated in one or more Relay-reply messages. The server MAY additionally include the Rate
Option in a Relay-reply message to provision the corresponding relay agent. The nested Relay-reply
structure of DHCPv6 allows the server to address each relay agent in the path individually, in
addition to the client. As DHCPv6 does not define a mechanism for relay agents to request options
from the server, the server includes the Rate Option in a Relay-reply message based on local
configuration, for example by identifying the relay agent by the link-address or peer-address field
of the Relay-forward message, or by relay agent options such as Interface-Id {{!RFC9915}} or
Remote-Id {{?RFC4649}}.

Including the Rate Option in a Relay-reply message is OPTIONAL for servers, and processing it is
OPTIONAL for relay agents. Relay agents that do not support this document are expected to ignore the
option, as they do not recognize it. Relay agents that do not receive rates in this way can instead
obtain them by inspecting the Rate Option destined for the client, as specified for snooping devices
in {{snooping}}, or through other means, such as AAA.

When provisioning multiple encapsulation levels, the server MAY supply different rates to each
node. For example, an operator might configure the upstream policer of a relay agent with a
slightly higher rate than the upstream shaper of the CPE, to accommodate minor burstiness of the
CPE traffic. DHCPv4 does not provide a comparable mechanism for addressing multiple relay agents
individually.

To update the rates of a client before its next scheduled renewal, a server MAY send a
Reconfigure message as specified in {{Section 18.3.11 of !RFC9915}}, causing the client to
initiate a Renew, Rebind, or Information-request transaction. As specified in {{!RFC9915}},
a server can only send Reconfigure messages to clients that have included the Reconfigure Accept
option in their messages, and Reconfigure messages are authenticated using the Reconfiguration
Key Authentication Protocol (RKAP; see {{Section 20.4 of !RFC9915}}).

For clients that do not support Reconfigure, updated rates take effect with the next Renew or
Rebind transaction or, for stateless clients, with the next Information-request transaction.
Operators can control the timing of such updates through the T1 value and the Information
Refresh Time option ({{Section 21.23 of !RFC9915}}).

## DHCPv6 Relay Agent Behavior {#dhcpv6-relay}

DHCPv6 relay agents that support the Rate Option, including Lightweight DHCPv6 Relay Agents (LDRA)
{{!RFC6221}}, process a Rate Option contained in a Relay-reply message addressed to them as
specified in {{processing}}. Such relay agents MAY use the signaled rates to instantiate shapers,
policers, or AQM disciplines on their client-facing interfaces. Relay agents SHOULD remove rates
applied for a client when the corresponding binding is removed. Relay agents MUST only process
Relay-reply messages received from the network side; in particular, an LDRA discards Relay-reply
messages received on client-facing interfaces as specified in {{Section 6.1.1 of !RFC6221}}.

A Rate Option contained in the encapsulated message destined for the client targets the client only.
Relay agents MUST NOT use such an option to configure their own traffic management, since the DHCPv6
relay architecture provides the Relay-reply message as a dedicated channel for signaling rates to
relay agents.

Modifying the message destined for the client is outside the relay agent behavior specified in
{{!RFC9915}}. A device that populates, modifies, or removes the Rate Option destined for the client
based on subscriber attributes (e.g., retrieved from an AAA server such as RADIUS) therefore acts as
a DHCPv6 server or DHCPv6 proxy towards the client, and the server behavior specified in this
document applies to it.

# DHCP Snooping {#snooping}

DHCP snooping devices are typically deployed as intermediate Layer 2 devices that passively monitor
DHCP message exchanges to enforce security policies and build binding databases, without modifying
the DHCP payloads. These devices MAY inspect the Rate Option in messages destined for clients and
use them to provision traffic shapers, policers, or hardware queues on the corresponding
client-facing ports. Snooping devices SHOULD remove rates applied for a client when the
corresponding binding is removed.

Snooping devices MUST only process a Rate Option contained in DHCPACK (DHCPv4) and Reply (DHCPv6)
messages received on trusted, network-facing ports. A Rate Option contained in a message received on
a client-facing port MUST NOT be used to configure traffic management.

# Deployment Scenarios

## PPPoE {#pppoe}

In PPPoE {{?RFC2516}} architectures, the CPE typically employs DHCPv6 over the PPP link
to obtain an IPv6 prefix via Prefix Delegation (IA_PD) {{!RFC9915}}. This DHCPv6 exchange conveys
the Rate Option in the same way as in IPoE deployments. DHCPv4 transactions within a PPPoE session
can similarly convey the DHCPv4 Rate Option.

The processing rules for a Rate Option received over PPPoE are identical to those defined for IPoE,
as specified in {{processing}}. In particular, the rates in effect revert to the default
configuration of the device when the PPPoE session is terminated.

As the signaled rates are Layer 3 rates, a CPE that attaches its shaper to the PPP interface applies
them directly, without accounting for the PPPoE, PPP, Ethernet, or VLAN headers.

If a client receives rate information from other sources, for example within PPP authentication
messages, and also receives a Rate Option, the client SHOULD give precedence to the Rate Option.
This ensures that rate information that is refreshed with every DHCP renewal supersedes rate
information that is conveyed only once per session.

## Local Area Networks {#lan}

The Rate Option can also be used by DHCP servers in local area networks, for example a DHCP server
co-located with a CPE or another gateway, to inform clients about the rates available to them. In
this case, the signaled rates indicate the maximum rate a client can expect through the network it
is attached to. Examples include:

- Virtual Private Network (VPN) gateways or tunnel endpoints that shape traffic towards their peers
  and whose processing capacity exceeds the capacity of the upstream connection.
- Public Wi-Fi hotspots that enforce per-user rate limits, where clients can align their sending
  rate with the enforced limit to avoid policer drops.
- Applications on end devices that use the signaled rates as an initial estimate, e.g., to
  estimate transfer times, to select an initial video quality, or to choose between available
  network interfaces.

A device that acts as a DHCP client on its network-facing interface and as a DHCP server on its
LAN interfaces MAY derive the rates it signals to LAN clients from the rates learned on its
network-facing interface, from local configuration, or both. In this case, it SHOULD signal the
lower of the learned rate and any locally configured limit for the client. When the learned rates
change, the device SHOULD signal the updated rates to its LAN clients with their next renewal and
MAY trigger an earlier update as specified in {{dhcpv4-server}} and {{dhcpv6-server}}. As Layer 3
rates are independent of the link-layer encapsulation, the learned rates can typically be passed on
unchanged.

Unless the signaled rates are enforced for each client individually, they describe a capacity shared
by all clients of the LAN. A client that shapes its traffic to such a rate does not exceed the
shared capacity on its own, but it does not obtain a guaranteed share of it either.

Clients cap the effective rates to the capacity of their physical link (see {{processing}}). For
links with variable capacity, such as wireless links, the effective rate is the lower of the
signaled rate and the current link capacity.

The signaled rates do not account for other bottlenecks along the path to a remote peer. Clients
MUST NOT use the signaled rates as a replacement for congestion control or other adaptive rate
mechanisms, and MUST NOT use them to increase the sending rate of a congestion control algorithm,
such as its initial sending rate, beyond the rate that the algorithm would otherwise use. Where rate
limits need to be enforced, enforcement remains the responsibility of the network; the Rate Option
only informs clients about such limits.

# Operational Considerations {#operations}

Deploying explicit rate signaling via DHCP concentrates upstream congestion at the CPE, where it can
be managed by AQM and, where deployed, L4S-compatible queue management, and thereby minimizes
queuing delay in the access segment.

All signaled rates are Layer 3 rates. Operators typically signal the IP-layer rate the subscriber is
intended to achieve, which is commonly the rate of the subscribed service as described to the
subscriber and as measured by speed tests. Because the same Layer 3 rate applies at every node, the
same value can be signaled to the CPE and to intermediate nodes without node-specific conversion,
regardless of VLAN tags or PPPoE encapsulation added or removed along the path.

On the CPE, the signaled rate can be fed directly into the shaper. A shaper attached to a PPP or IP
interface accounts IP packets and applies the rate as is. Shapers that account Ethernet frames can
typically be configured to account the network-layer packet size instead, or to subtract the locally
known link-layer headers, and thereby enforce exactly the signaled Layer 3 rate (see
{{impl-status}} for an example). Where no such
configuration is possible, applying the rate to the frame size results in a slightly lower effective
Layer 3 rate and is therefore safe (see {{application}}).

Enforcement points frequently account traffic at Layer 2, and physical bottlenecks additionally
consume per-packet overhead, such as encapsulation headers, the Ethernet FCS, preamble, and
inter-packet gap. The relative overhead depends on the packet size. For example, the Ethernet, VLAN,
PPPoE, and PPP headers add 26 octets to each IP packet, which corresponds to approximately 1.7% for
1500-octet packets but to a considerably larger share for small packets. Operators account for this
difference when configuring the enforcement point or the signaled rate, for example by configuring
the enforcement point with a margin above the signaled rate (see {{processing}}) or by signaling a
correspondingly lower Layer 3 rate. Traffic consisting predominantly of small packets can still
exceed a Layer 2 enforcement point that is configured close to the signaled rate; such residual
inaccuracy is accepted in exchange for the simplicity of a single, encapsulation-independent
parameter (see {{rationale}}).

Where the actual rate of the access line is known and lower than the provisioned rate, for example
due to the synchronization rate of a DSL line, operators are advised to signal the lower of the two
rates, so that the client shapes its traffic to the actual bottleneck. The actual rate is typically
known at the BNG, for example from the Actual-Data-Rate-Upstream and Actual-Data-Rate-Downstream
attributes {{?RFC4679}} or the corresponding Access Node Control Protocol (ANCP) Type-Length-Value
(TLV) elements {{?RFC6320}}. As these attributes describe the net data rate of the access line, the
server converts them to a Layer 3 rate, for example by deducting the encapsulation overhead for
typical packet sizes and, for lines using Packet Transfer Mode (PTM) with 64/65-octet encapsulation,
the corresponding 1/65 of the rate. Access lines with cell-based link layers, such as Asynchronous
Transfer Mode (ATM) on Asymmetric Digital Subscriber Line (ADSL), have a packet-size-dependent
overhead that a single rate cannot describe precisely; operators of such lines can signal a
conservatively reduced rate, and precise modeling remains a matter of local configuration of the
CPE.

Without the use of DHCPFORCERENEW or Reconfigure, changes of the provisioned rates take effect with
the next renewal of the client. Operators are advised to consider this delay when choosing lease and
renewal times, for example when rates are changed due to service upgrades or fair-use policies.

The inclusion of the code of the Rate Option in the Parameter Request List or Option Request option
allows operators to determine the share of clients supporting these options, for example to plan the
introduction of correction factors or to identify clients that require enforcement at the network.

Implementations are encouraged to expose the signaled rates within CPE management interfaces, such
as the local Web User Interface (UI) or remote management protocols. As the signaled rates are
Layer 3 rates, the displayed values are directly comparable with the results of speed tests, which
can reduce support inquiries related to perceived bandwidth issues.

As noted in {{processing}}, a client can receive a rate that exceeds the maximum physical link speed
of its interfaces. In such scenarios, management interfaces can expose both the signaled rate and
the effective, capped rate applied by the device, and implementations can log this discrepancy
if logging facilities are enabled. Such events can indicate a mismatch between the subscriber's
provisioned service tier and the installed equipment.

# Implementation Status {#impl-status}
{:removeinrfc="true"}

{::boilerplate rfc7942}

At the time of writing, no implementations of this document are known to the authors.

## Implementation Feasibility on Open-Source CPE

The authors analyzed the feasibility of implementing the client behavior on OpenWrt without
modifying the DHCP client software. The DHCPv4 client (BusyBox udhcpc) and DHCPv6 client (odhcp6c)
used by OpenWrt can request the Rate Option and expose received options of unknown type as
hexadecimal environment variables to user-defined hook scripts. All client behavior specified in
this document can therefore be implemented in such scripts, which split the 16-octet value into two
integers, apply the processing rules of {{processing}}, and configure local traffic management, for
example via Smart Queue Management (SQM) or the CAKE queue discipline. As the signaled rates are
Layer 3 rates, they can be passed to CAKE directly as its bandwidth parameter, together with an
overhead of 0, without any encapsulation-specific calculation. The rates learned on the WAN
interface can be propagated to LAN clients using the existing custom-option support of the OpenWrt
DHCPv4 server (dnsmasq) and DHCPv6 server (odhcpd).

# Security Considerations {#security}

DHCP messages are typically transmitted as plaintext and are unauthenticated. Consequently, the Rate
Option defined in this document is vulnerable to interception, modification, or spoofing by on-path
attackers. This applies to any path the options traverse, including the path between servers and
clients as well as the path between servers and relay agents, where a Rate Option carried in a
Relay-reply message configures the relay agent itself (see {{dhcpv6-relay}}). A malicious actor or a
rogue DHCP server could inject artificially low rates to throttle a client's connection, resulting
in a localized Denial of Service (DoS).

The severity of this risk is generally no greater than that of standard DHCP threat vectors, such
as rogue default gateway assignment, DNS hijacking, or IP pool exhaustion, which typically yield a
much more critical and immediate loss of service.

Operators can mitigate rogue DHCP servers by deploying DHCP snooping for DHCPv4 and DHCPv6-Shield
{{?RFC7610}} for DHCPv6 on access nodes, which discard server messages received on client-facing
ports. These mechanisms protect both the clients and the intermediate nodes that apply signaled
rates.

To mitigate the impact of malicious, spoofed, or malformed options, clients SHOULD apply sanity
checks before applying signaled rates:

- Clients SHOULD ignore a non-zero rate below a minimum below which basic connectivity is no longer
  ensured, and retain the rate currently in effect for that direction. The appropriate minimum
  depends on the deployment and the use case of the device and is therefore left to the
  implementation, which can allow the administrator or operator to configure it. For example, a
  broadband CPE might ignore rates below 1 Mbit/s.
- Clients SHOULD NOT apply a rate above the capacity of the physical link of the respective
  interface, but cap the applied rate to that capacity (see {{processing}}). An excessively high
  rate therefore has the same effect as the absence of a signaled rate.

Conversely, an attacker could inject an excessively high rate or a rate of 0, thereby removing any
effective rate limit. Because clients are expected to cap the effective rate to the capacity of the
physical link (see {{processing}}), the impact is limited to the loss of the benefits of rate
signaling, such as increased queuing delay at the bottleneck, which corresponds to the behavior of a
device without support for this option. Where an intermediate node derives the enforcement of the
provisioned rate from the Rate Option (see {{solution-overview}}), however, an attacker on the path
between the server and that node that raises or removes the signaled rates also raises or removes
the enforcement, allowing a client to exceed its provisioned rate. Mechanisms for securing the
messages exchanged between servers and relay agents are described in {{?RFC8213}}.

In local area networks, DHCP snooping is often not deployed, making rogue DHCP servers more
likely. A rogue server signaling artificially low rates could cause applications on end devices
to reduce their quality or to prefer another network interface, which can incur additional cost
(e.g., cellular data). Clients SHOULD NOT use the signaled rates as the sole criterion for
selecting a network interface, and the sanity checks described above apply.

A rogue server can also advertise excessively high rates in DHCPOFFER or Advertise messages to be
selected by clients that use rates as a criterion for selecting among servers (see {{dhcpv4-client}}
and {{dhcpv6-client}}). Clients SHOULD NOT use the rates as the sole criterion for server selection,
and the mitigations against rogue DHCP servers described above apply.

Relay agents and snooping devices that instantiate traffic management functions based on the Rate
Option consume finite resources, such as hardware queues and shapers. Because only server messages
received on trusted ports are processed and rates are applied per client-facing port (see
{{processing}}), the amount of such resources is bounded by the number of client-facing ports.
However, a client can trigger frequent DHCP exchanges, potentially causing repeated reconfiguration
of these functions. In addition to not reconfiguring unchanged rates (see {{processing}}),
implementations MAY rate-limit DHCP messages received on client-facing ports or the reconfiguration
of traffic management functions.

# Privacy Considerations {#privacy}

The general privacy considerations for DHCP specified in {{?RFC7844}} apply to this document.

The Rate Option conveys the rates provisioned for a client, which typically reflect the service tier
of a subscriber. Because DHCP messages are typically transmitted as plaintext, this information is
exposed to on-path observers, as well as to relay agents and snooping devices that process the
options. The rate information itself does not identify a subscriber, but in combination with other
information it can reveal commercial details of the service contract.

Clients that include the code of the Rate Option in the Parameter Request List or Option Request
option disclose additional information that can contribute to fingerprinting the device. Clients
that implement the anonymity profiles specified in {{?RFC7844}} MAY omit the code of the Rate Option
from the Parameter Request List or Option Request option.

The operational practice of exposing and logging signaled and effective rates (see {{operations}})
can provide operators with information about the equipment installed by a subscriber. Such data
is subject to the operator's applicable privacy policies.

# IANA Considerations {#iana}

RFC Editor: please replace TBD1 and TBD2 throughout this document with the values assigned by IANA,
replace "RFC XXXX" with the RFC number assigned to this document, and remove this note.

## DHCPv4 Option

IANA is requested to assign a new option code in the "BOOTP Vendor Extensions and DHCP Options"
registry within the "Dynamic Host Configuration Protocol (DHCP) and Bootstrap Protocol (BOOTP)
Parameters" registry group (<https://www.iana.org/assignments/bootp-dhcp-parameters>) as follows:

| Tag  | Name           | Data Length | Meaning                                   | Reference |
| :--- | :------------- | ----------: | :---------------------------------------- | :-------- |
| TBD1 | OPTION_V4_RATE |          16 | Intended upstream and downstream rates    | RFC XXXX  |
{: #tab-iana-v4 title="DHCPv4 Option Code"}

## DHCPv6 Option

IANA is requested to assign a new option code in the "Option Codes" registry within the
"Dynamic Host Configuration Protocol for IPv6 (DHCPv6)" registry group
(<https://www.iana.org/assignments/dhcpv6-parameters>) as follows:

| Value | Description    | Client ORO | Singleton Option | Reference |
| :---- | :------------- | :--------- | :--------------- | :-------- |
| TBD2  | OPTION_V6_RATE | Yes        | Yes              | RFC XXXX  |
{: #tab-iana-v6 title="DHCPv6 Option Code"}

This document does not create any new registries.

--- back

# Design Rationale {#rationale}

This appendix summarizes the reasoning behind the mechanism defined in this document and
addresses questions that are frequently raised.

## Why Is Rate Enforcement at the BNG Not Sufficient?

Rate enforcement at the BNG ensures that a subscriber does not exceed the provisioned rate, but
it does not ensure that the subscriber can use this rate efficiently. In the upstream direction,
enforcement is typically implemented with ingress policers that discard excess traffic without
queuing it. Senders that are unaware of the policed rate experience bursty losses, which leads
to unstable throughput below the provisioned rate (see {{fig-policer}}). Furthermore, upstream
prioritization at the CPE, for example of DNS, VoIP, or selected clients and applications, is
ineffective as long as packets are discarded at the BNG regardless of their priority.

Moreover, the upstream bottleneck is not always located at the BNG. Where it is located at an
access device, such as an ONT or a DSL modem, or within the access network, neither the BNG nor
the access device typically manages the resulting queue. Shaping at the CPE is then the only
practical means to control upstream latency and to make prioritization effective.

Shaping at the CPE complements the enforcement at the BNG: the BNG continues to enforce the
provisioned rate, for example with a small correction factor (see {{processing}}), while the CPE
ensures that this enforcement is not triggered under normal operation. CPE that do not support this
option continue to be subject to the enforcement at the BNG, so that the provisioned service tier is
maintained in all cases.

## Why Not Shape Upstream Traffic at the BNG?

Shaping requires queuing. Some forwarding hardware used for BNGs does not support per-subscriber
queuing in the upstream direction at all. On forwarding hardware based on virtual output queues,
for example, queues are associated with a destination port and traffic class rather than with
the subscriber from which the traffic is received. Where per-subscriber upstream queuing is
supported, it is commonly restricted: depending on the platform, it is available only on specific
line cards or ports, requires the forwarding hardware to operate in a dedicated mode, or consumes
the same queuing, buffer, and scheduling resources that are required for downstream hierarchical
QoS, thereby reducing the overall subscriber scale. Alternatives such as the recirculation of
upstream traffic through the egress pipeline reduce the forwarding capacity. For these reasons,
upstream enforcement is commonly implemented using policers, even on platforms that support
ingress queuing.

Furthermore, even where upstream shaping at the BNG is available, the BNG has limited knowledge
of the traffic it shapes. Traffic from multiple devices behind the CPE typically shares a single
address due to Network Address Translation (NAT), so that prioritization of individual clients or
applications in the home network can only be performed at the CPE.

Shaping at the CPE, in contrast, scales naturally: each CPE shapes only its own traffic using a
single shaper, and the required resources are distributed across all subscribers. In addition,
the queue is located at the device that has the most information about its traffic, allowing
prioritization, flow queuing, and AQM per subscriber without any additional resources in the
network.

## Why Not Configure the Rate Manually or via TR-069?

The provisioned rate is typically unknown to the user, may change over time (e.g., due to
service upgrades or fair-use policies), and differs from the physical link speed of the CPE.
Manual configuration is therefore error-prone and rarely performed. Auto-configuration protocols
such as TR-069 {{TR069}} are not deployed by all operators and are typically not available for
customer-owned CPE. In addition, they only target the CPE and cannot provision intermediate
nodes.

## Why DHCP?

DHCP is already used by virtually all broadband CPE, is processed by the BNG acting as relay
agent or server, and is visible to intermediate access nodes. Rates conveyed via DHCP are
refreshed with every lease renewal and can be updated dynamically. The options are simple enough
to be processed by event-driven scripts in existing DHCP clients, without changes to the DHCP
client software (see {{impl-status}}).

## Why Only Layer 3 Rates? {#why-layer3}

Earlier versions of this document allowed the server to select the layer to which the rates refer
and to describe the per-packet accounting of the enforcement point, including overhead, minimum
packet size, and cell-based link layers, as well as separate informational rates. This allowed a
precise model of any bottleneck, but required every receiver to implement the corresponding
calculations, to translate parameters between the link of the receiver and the enforcement point,
and to handle the interactions between parameters for each direction.

Restricting the signaled rates to Layer 3 removes this complexity:

- A Layer 3 rate matches what users perceive and what speed tests measure, so a single value can
  serve both traffic management and display, without separate informational rates.
- A Layer 3 rate is independent of the encapsulation. The same value is valid at the CPE, at an
  access node, and at the BNG, even if VLAN tags or other headers are added or removed between them.
- Common CPE shapers, such as CAKE, can be configured with a Layer 3 rate directly. Receivers that
  cannot account at Layer 3 err on the safe side by applying the rate to the larger link-layer
  frame.
- The remaining inaccuracy, which arises when the enforcement point accounts link-layer overhead,
  is small for typical packet sizes and can be absorbed by the operator through the configuration
  of the enforcement point or of the signaled rate (see {{operations}}). Precise modeling of legacy
  cell-based link layers remains possible through local configuration of the CPE.

## Why a Single Fixed-Format Option?

The Rate Option carries exactly two fixed-size values. Compared with a container option with
sub-options, this avoids sub-option parsing, a dedicated sub-option registry, and the corner cases
of empty or partially valid containers. Compared with separate options per direction, it consumes
only one code from the limited DHCPv4 option code space, and a single option conveys the complete
rate information, so that receivers never need to combine rates from different options or messages.
Receivers that only apply the upstream rate simply ignore the Downstream Rate field. Should
additional parameters be required in the future, they can be defined as separate options.

## Relation to Access Line Information

Standardized mechanisms exist to convey the characteristics of an access line from the access node
to the network: access loop characteristics are carried in DHCP relay agent options and PPPoE tags
{{TR101}}, mapped to RADIUS attributes such as Actual-Data-Rate-Upstream and
Actual-Data-Rate-Downstream {{?RFC4679}}, and reported via ANCP {{?RFC6320}}. These mechanisms
inform the BNG and the AAA infrastructure about what the access line can carry. No standardized
mechanism, however, conveys the resulting rates to the CPE and to intermediate nodes. This document
closes this gap, using the access line information as input for the signaled rates (see
{{operations}}).

## Relation to Existing Proprietary Solutions

Several operators have successfully deployed proprietary mechanisms to provide the CPE with its
provisioned rate, for example by embedding rate information in PPP authentication messages or by
vendor-specific management interfaces. The deployment of such mechanisms demonstrates the need for
this information at the CPE, and CPE implementations already provide the corresponding traffic
management functions. However, these mechanisms have significant limitations: they are typically
supported only by operator-provided CPE, so that customer-owned CPE fall back to the policing
behavior described above; rate changes often take effect only after the session is re-established;
and the information is not available to intermediate nodes. This document defines a standardized
and interoperable alternative that addresses these limitations.

# Change Log
{:removeinrfc="true"}

## Since draft-ietf-intarea-dhcp-rate-signaling-00

The following changes are based on the feedback received during the presentation at the last IETF
meeting, the subsequent mailing list discussions, and the GitHub issues of this document.

Motivation and scope:

- Restructured the Introduction into Problem Statement, Solution Overview, and Applicability,
  explaining upstream policing at the BNG, other upstream bottlenecks, and the limitations of
  prioritization at the CPE; added a figure.
- Added the Design Rationale appendix addressing frequently raised questions, including the
  restriction to Layer 3 rates, the use of a single fixed-format option, and the relation to access
  line information and to existing proprietary solutions.
- Added applicability to local area networks.

Protocol changes (not backward compatible with -00):

- Replaced the sub-option structure of OPTION_V4_RATE and OPTION_V6_RATE by a fixed 16-octet
  format carrying the upstream and downstream rates as 64-bit values in bits per second.
- All rates are Layer 3 rates, calculated over the outermost IP header on the link of the receiver,
  including tunnels and translations performed by the receiver. Removed the rate type and layer
  selection, the informational rates, and the DHCP Rate Sub-Options registry.

Processing and behavior:

- Added the sections Traffic Management Using Signaled Rates, Rate Processing, and Error Handling,
  and specified the messages in which the options can appear.
- Clarified the requirement levels for applying rates, the precedence of administrator
  configuration, and the behavior of servers, relay agents, and snooping devices.
- Defined the terms Intended Rate and Binding, including the removal of rates by intermediate nodes
  without per-client state.
- Operators ensure that DHCPv4 and DHCPv6 convey identical rates for the same client; added
  guidance on transient conflicts between DHCPv4 and DHCPv6, optional hold-down timers, and
  idempotent processing of unchanged rates.
- Defined the term Default Configuration.
- Changed the client sanity checks against implausibly low or high rates from MAY to SHOULD.
- Made the provisioning of relay agents via Relay-reply messages explicitly optional.

Other:

- Changed the intended status to Standards Track; replaced RFC 8415 by RFC 9915.
- Moved TR-101 to the informative references and RFC 6221 to the normative references.
- Editorial updates following IANA and RFC Editor style review, including the IANA registry
  group names and the expansion of abbreviations on first use.
- Extended the Security Considerations; added Privacy Considerations, Operational Considerations,
  and Implementation Status.
- Explained the use of 64-bit rates and the derivation of rates from access line information
  {{?RFC4679}} {{?RFC6320}}.

# Acknowledgments
{:numbered="false"}

The authors would like to thank Glenn Deen, Jason Livingood, and
Sebastian Moeller for their valuable review comments and discussion,
which helped to significantly improve the clarity,
applicability, and operational guidance of this document.
