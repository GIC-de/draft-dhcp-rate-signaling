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

normative:
   TR101:
      title: "Migration to Ethernet-Based Broadband Aggregation"
      target: https://www.broadband-forum.org/pdfs/tr-101-2-0-0.pdf
      author:
      - org: Broadband Forum
      date: 2011-07
      seriesinfo:
         TR: 101 Issue 2

informative:
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
upstream prioritization at the CPE ineffective. By learning the available rate via DHCP, the CPE can
shape its upstream traffic, moving the queue to the CPE, where it can be managed by prioritization,
Active Queue Management (AQM), and the Low Latency, Low Loss, and Scalable Throughput (L4S)
architecture. Intermediate nodes acting as DHCP relay agents or performing DHCP snooping can use the
same information to instantiate per-subscriber shapers and policers. These options are also
applicable within local area networks, for example to inform end devices about the capacity of the
upstream connection.

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
resulting queue is typically managed by neither AQM nor prioritization of the subscriber's
traffic. Depending on the buffering of the access device, this results in packet loss, with
effects similar to those of a policer, or in increased latency under load. In both cases, all
traffic of the subscriber is affected alike, including latency-sensitive applications.

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

This document defines four DHCP options: an upstream rate option and a downstream rate option for
each of DHCPv4 and DHCPv6. Each option carries exactly one value, the intended rate for one
direction as a 64-bit unsigned integer in bits per second. The rate is always a Layer 3 rate: it
refers to the IP header and payload of each packet (see {{layer3}}). No other parameters are
defined.

If the CPE knows the upstream rate, it can shape its upstream traffic to this rate, regardless of
where the bottleneck is located. The queue then builds at the CPE, where it can be managed using
prioritization, Active Queue Management (AQM) {{?RFC7567}}, and flow queuing {{?RFC8290}}. Under
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
any segment of the path. The same value is therefore valid at the CPE, at an access node, and at
the BNG, irrespective of VLAN tags or PPPoE headers added or removed along the path, and it can be
configured directly into common CPE shapers, such as the Common Applications Kept Enhanced (CAKE)
queue discipline, without encapsulation-specific calculations (see {{application}}).

The same rate information can be used by intermediate nodes. In many IP over Ethernet (IPoE)
{{TR101}} architectures, the BNG operates as a DHCP relay agent. While per-subscriber traffic
management policies are typically provisioned via Authentication, Authorization, and Accounting
(AAA) protocols such as RADIUS {{?RFC2865}}, deployments lacking AAA integration at the service
edge require an in-band alternative. The BNG can use the signaled rates to instantiate downstream
shapers and upstream policers, and access nodes performing DHCP snooping can use them to
configure their local queues and shapers. DHCPv6 additionally allows the server to signal
different rates to the client and to each relay agent in the path (see {{dhcpv6-server}}).

## Applicability

The mechanism defined in this document is primarily designed for broadband access networks using
IPoE or Point-to-Point Protocol over Ethernet (PPPoE) {{?RFC2516}}. In PPPoE deployments, DHCPv6 is
commonly used over the PPP session for IPv6 Prefix Delegation and conveys the options in the same
way (see {{pppoe}}). The options are defined for both DHCPv4 and DHCPv6 to support IPv4-only,
IPv6-only, and dual-stack deployments.

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

DHCP
: The abbreviation DHCP is used throughout this document to refer to both DHCPv4 and DHCPv6
  protocols.

DHCPv4
: The Dynamic Host Configuration Protocol {{!RFC2131}}.

DHCPv6
: The Dynamic Host Configuration Protocol for IPv6 {{!RFC9915}}.

Upstream
: The direction of traffic from the client towards the network. For a client, this corresponds to
  traffic sent over the link on which the rate options were received. For a relay agent or
  snooping device, this corresponds to traffic received on the client-facing interface or port.

Downstream
: The direction of traffic from the network towards the client. For a client, this corresponds to
  traffic received over the link on which the rate options were received. For a relay agent or
  snooping device, this corresponds to traffic sent on the client-facing interface or port.

Enforcement Point
: The node or link at which the provisioned rates are enforced or which constitutes the bottleneck
  of the path, for example a policer at the BNG or an access link.

Intermediate Node
: A relay agent or snooping device on the path between a client and a server that processes the
  rate options, for example a BNG acting as a DHCP relay agent or an access node performing DHCP
  snooping.

Layer 3 Rate
: A rate calculated over the IP header and IP payload of each packet (see {{layer3}}).

Provisioned Rate
: The rate configured for a subscriber according to the subscribed service, typically enforced
  at the enforcement point (e.g., the BNG). The signaled rates are typically derived from the
  provisioned rates.

Rate Options
: Collectively, the options defined in this document: OPTION_V4_UPSTREAM_RATE,
  OPTION_V4_DOWNSTREAM_RATE, OPTION_V6_UPSTREAM_RATE, and OPTION_V6_DOWNSTREAM_RATE. The upstream
  rate option refers to OPTION_V4_UPSTREAM_RATE or OPTION_V6_UPSTREAM_RATE, and the downstream rate
  option to OPTION_V4_DOWNSTREAM_RATE or OPTION_V6_DOWNSTREAM_RATE.

Receiver
: Any node that processes a received rate option, including DHCP clients, relay agents (including
  Lightweight DHCPv6 Relay Agents {{?RFC6221}} and Layer 2 DHCPv4 Relay Agents {{TR101}}), and DHCP
  snooping devices.

Subscriber
: The customer of a network access service. Within the network, a subscriber is typically
  represented by a subscriber session at the BNG (e.g., an IPoE or PPPoE session), to which the
  provisioned rates and other per-subscriber policies are applied.

# Rate Options {#rate-options}

This document defines one upstream rate option and one downstream rate option for each of DHCPv4
and DHCPv6. The encoding of each option follows the conventions of the respective base protocol
(see {{dhcpv4}} and {{dhcpv6}}). The semantics of the four options are identical and are specified
in this section.

The value of each rate option is a 64-bit unsigned integer in network byte order, indicating the
intended rate for the respective direction in bits per second (bps). The 64-bit format allows rates
beyond the approximately 4.29 Gbit/s that can be represented by 32-bit rate attributes in bits per
second, such as those defined in {{?RFC4679}}.

A value of 0 indicates that no rate is signaled for the respective direction. A receiver that has
previously applied a rate learned from a rate option for that direction MUST remove that rate and
revert to its default configuration for that direction (see {{processing}}).

## Layer 3 Rates {#layer3}

All rates signaled by the rate options are Layer 3 rates. A Layer 3 rate is calculated over the IP
header (including any IPv6 extension headers) and the IP payload of each packet. It excludes all
headers and trailers below the IP layer, such as Ethernet headers, Virtual LAN (VLAN) tags, PPPoE
and PPP headers, and the Ethernet Frame Check Sequence (FCS), as well as all physical-layer
overhead.

Layer 3 rates have the following properties:

- They correspond to the speed perceived by users and measured by common speed test applications,
  and to the rate in which service tiers are typically described and marketed.
- They are independent of the encapsulation used on the link of the receiver and on any other
  segment of the path. A single value applies at every node, regardless of VLAN tags or tunnel
  headers that are added or removed between the receiver and the enforcement point.
- They can be applied directly by traffic management functions that account traffic at the IP
  layer, such as shapers attached to a PPP or IP interface, or shapers that can be configured to
  account the network-layer packet size (see {{application}}).

The rate options do not describe the per-packet accounting of the enforcement point. Operators
account for differences between the Layer 3 rate and the accounting of the enforcement point when
configuring the signaled rate or the enforcement point (see {{operations}}).

## Traffic Management Using Signaled Rates {#application}

The rate options convey rates that are intended to be applied to local traffic management functions,
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

The processing of the rate options is stateless with respect to the DHCP protocol version and the
DHCP lease, such that it can be implemented by simple event-driven scripts on the client. A receiver
only needs to retain the rate currently in effect for each direction.

Rates are maintained per link and per direction. DHCPv4 and DHCPv6 messages received over the same
link share the rates in effect, even if they are processed on different logical interfaces of the
receiver.

Each valid rate option received in a DHCPv4 DHCPACK message or a DHCPv6 Reply message (or, for
DHCPv6 relay agents, in the corresponding Relay-reply message) replaces the rate in effect for the
corresponding direction, regardless of whether the previous rate was learned via DHCPv4 or DHCPv6:

- If the value is non-zero, the receiver applies the signaled rate as specified in {{application}}.
- If the value is 0, the receiver MUST remove any rate previously applied for that direction and
  revert to its default configuration for that direction.
- If the rate option for a direction is absent, the receiver MUST retain the rate in effect for that
  direction unchanged.

If a signaled rate is identical to the rate currently in effect, the receiver SHOULD NOT reconfigure
its traffic management functions.

For clients, the rates in effect are not bound to the lifetime of the DHCP lease from which they
were learned. Relay agents and snooping devices remove the rates applied for a client together with
the corresponding binding (see {{dhcpv4}}, {{dhcpv6-relay}}, and {{snooping}}). A receiver MUST
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
1%). This allows a client that supports these options to shape its traffic to the signaled rate
without triggering the enforcement at the intermediate node, while the intermediate node still
enforces the provisioned rate for clients that do not support or ignore these options.
Alternatively, in DHCPv6, the server can signal node-specific rates (see {{dhcpv6-server}}).

A client can receive a rate that exceeds the maximum physical link speed of its interfaces (e.g.,
2 Gbit/s signaled to a CPE with a 1 Gbit/s Wide Area Network (WAN) port). In such scenarios, the
client SHOULD cap the applied traffic shaping, policing, or AQM parameters to the capacity of the
physical link.

In dual-stack deployments, servers SHOULD either signal identical rates via both DHCPv4 and DHCPv6,
or signal rates via only one of the two protocols that is expected to be negotiated by every client.
In the latter case, messages of the other protocol do not carry rate options and therefore leave the
rates in effect unchanged. If neither approach is followed, the rate in effect for each direction
depends on which message was received last.

## Error Handling {#errors}

A rate option whose length is not 8 octets is malformed. A receiver MUST discard a malformed rate
option, which is then treated as if it were not present (see {{processing}}). A malformed rate
option for one direction does not affect the processing of the rate option for the other direction.

A sender MUST NOT include more than one instance of each rate option in a DHCPv4 message, or at the
same encapsulation level of a DHCPv6 message (i.e., within the message exchanged with the client or
within a given Relay-reply message). If a DHCPv4 receiver nevertheless receives multiple instances
of a rate option, it concatenates them as specified in {{!RFC3396}}; the result is malformed unless
its length is 8 octets. If a DHCPv6 receiver receives more than one instance of a rate option at the
same encapsulation level, it MUST discard all instances of that option.

Rate options contained in messages other than those listed in {{dhcpv4}} and {{dhcpv6}} MUST be
ignored.

# DHCPv4 {#dhcpv4}

## DHCPv4 Rate Options

The DHCPv4 Upstream Rate option (OPTION_V4_UPSTREAM_RATE) uses option code TBD1, and the DHCPv4
Downstream Rate option (OPTION_V4_DOWNSTREAM_RATE) uses option code TBD2. Both options have the
following format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Code      |   Length=8    |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                         Rate (64 bits)                        |
+                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-v4-format title="DHCPv4 Rate Option Format"}

Code:
: TBD1 for OPTION_V4_UPSTREAM_RATE, TBD2 for OPTION_V4_DOWNSTREAM_RATE.

Length:
: 8.

Rate:
: The Layer 3 rate in bits per second, as a 64-bit unsigned integer in network byte order (see
  {{rate-options}}).

The DHCPv4 rate options can appear in the following messages:

| Message                                  | Use                                        |
| :--------------------------------------- | :----------------------------------------- |
| DHCPOFFER                                | Server selection (see {{dhcpv4-client}})   |
| DHCPACK                                  | Rate information (see {{processing}})      |
{: #tab-appearance-v4 title="Appearance of DHCPv4 Rate Options"}

## DHCPv4 Client Behavior {#dhcpv4-client}

DHCPv4 clients that support the rate options SHOULD include the codes of the rate options they
support in the Parameter Request List (PRL). This signals client support, enabling the DHCPv4 server
to determine whether to include the rate options in its response. Furthermore, it provides network
operators with visibility into client capabilities, which aids in troubleshooting and resolving
customer service quality complaints.

A client MUST NOT apply rates received in any DHCPv4 message other than DHCPACK. A client MAY use
the rates contained in a DHCPOFFER message as a criterion for selecting among multiple offers.
Rates received in a DHCPACK message are processed as specified in {{processing}}.

## DHCPv4 Server Behavior {#dhcpv4-server}

When a DHCPv4 server that supports the rate options receives a client message whose PRL includes a
rate option code, and a rate is configured for the client for the corresponding direction, the
server MUST include that rate option in the resulting DHCPOFFER and DHCPACK messages. Because a
missing rate option leaves the rate in effect unchanged, a server that needs to remove a previously
signaled rate MUST include the corresponding rate option with a value of 0.

The server MAY derive the rates from local configuration profiles, centralized AAA systems such as
RADIUS, or external policy servers.

A server MAY include the rate options in its responses even if the client did not request them via
the PRL, provided the operator has explicitly configured the server to do so, for example to
provision Layer 2 DHCPv4 Relay Agents or snooping devices.

To update the rates of a client before its next scheduled renewal, a server MAY send a
DHCPFORCERENEW message {{?RFC3203}}, causing the client to initiate a renewal transaction.
DHCPFORCERENEW messages are subject to the authentication requirements of {{?RFC3203}}, which
can be satisfied using the mechanism specified in {{?RFC6704}}. For clients that do not support
DHCPFORCERENEW, updated rates take effect with the next renewal, the timing of which is
controlled by the renewal (T1) time value.

## DHCPv4 Relay Agent Behavior

DHCPv4 relay agents, including Layer 2 DHCPv4 Relay Agents {{TR101}}, MAY extract the rate options
from DHCPACK messages traversing the network and use them to instantiate shapers and policers on
their client-facing interfaces. Relay agents SHOULD remove rates applied for a client when the
corresponding binding is removed. Layer 2 DHCPv4 Relay Agents MUST only process rate options
contained in DHCPACK messages received on trusted, network-facing ports.

Modifying the message destined for the client is outside the relay agent behavior specified in
{{!RFC2131}} and {{?RFC3046}}. A device that populates, modifies, or removes the rate options
destined for the client based on subscriber attributes (e.g., retrieved from an AAA server such as
RADIUS) therefore acts as a DHCPv4 server or DHCPv4 proxy towards the client, and the server
behavior specified in this document applies to it.

# DHCPv6 {#dhcpv6}

## DHCPv6 Rate Options

The DHCPv6 Upstream Rate option (OPTION_V6_UPSTREAM_RATE) uses option code TBD3, and the DHCPv6
Downstream Rate option (OPTION_V6_DOWNSTREAM_RATE) uses option code TBD4. Both options have the
following format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          option-code          |         option-len=8          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                         Rate (64 bits)                        +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-v6-format title="DHCPv6 Rate Option Format"}

option-code:
: TBD3 for OPTION_V6_UPSTREAM_RATE, TBD4 for OPTION_V6_DOWNSTREAM_RATE.

option-len:
: 8.

Rate:
: The Layer 3 rate in bits per second, as a 64-bit unsigned integer in network byte order (see
  {{rate-options}}).

The DHCPv6 rate options can appear in the following messages:

| Message                                  | Use                                        |
| :--------------------------------------- | :----------------------------------------- |
| Advertise                                | Server selection (see {{dhcpv6-client}})   |
| Reply to Solicit, Request, Renew, Rebind, or Information-request | Rate information (see {{processing}}) |
| Relay-reply                              | Rate information for the relay agent (see {{dhcpv6-relay}}) |
{: #tab-appearance-v6 title="Appearance of DHCPv6 Rate Options"}

The rate options are not included in Reconfigure messages, as specified in
{{Section 18.3.11 of !RFC9915}}.

## DHCPv6 Client Behavior {#dhcpv6-client}

DHCPv6 clients that support the rate options SHOULD include the codes of the rate options they
support in the Option Request option (ORO) {{!RFC9915}}, for the same reasons as specified for
DHCPv4 in {{dhcpv4-client}}.

A client MUST NOT apply rates received in any DHCPv6 message other than Reply. A client MAY use
the rates contained in an Advertise message as a criterion for selecting among multiple Advertise
messages. Rates received in a Reply message are processed as specified in {{processing}}.

## DHCPv6 Server Behavior {#dhcpv6-server}

When a DHCPv6 server that supports the rate options receives a Solicit, Request, Renew, Rebind, or
Information-request message whose ORO includes a rate option code, and a rate is configured for the
client for the corresponding direction, the server MUST include that rate option in the resulting
Advertise and Reply messages. A server that needs to remove a previously signaled rate MUST include
the corresponding rate option with a value of 0.

A server MAY include the rate options in its responses even if the client did not request them via
the ORO, provided the operator has explicitly configured the server to do so.

In relayed exchanges, the rate options for the client are included in the Reply message, which is
encapsulated in one or more Relay-reply messages. The server MAY additionally include the rate
options in a Relay-reply message to provision the corresponding relay agent. The nested Relay-reply
structure of DHCPv6 allows the server to address each relay agent in the path individually, in
addition to the client. As DHCPv6 does not define a mechanism for relay agents to request options
from the server, the server includes the rate options in a Relay-reply message based on local
configuration, for example by identifying the relay agent by the link-address or peer-address field
of the Relay-forward message, or by relay agent options such as Interface-Id {{!RFC9915}} or
Remote-Id {{?RFC4649}}.

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

DHCPv6 relay agents that support the rate options, including Lightweight DHCPv6 Relay Agents (LDRA)
{{?RFC6221}}, process rate options contained in a Relay-reply message addressed to them as specified
in {{processing}}. Such relay agents MAY use the signaled rates to instantiate shapers, policers, or
AQM disciplines on their client-facing interfaces. Relay agents SHOULD remove rates applied for a
client when the corresponding binding is removed. Relay agents MUST only process Relay-reply
messages received from the network side; in particular, an LDRA discards Relay-reply messages
received on client-facing interfaces as specified in {{Section 6.1.1 of ?RFC6221}}.

Rate options contained in the encapsulated message destined for the client target the client only.
Relay agents MUST NOT use such options to configure their own traffic management, since the DHCPv6
relay architecture provides the Relay-reply message as a dedicated channel for signaling rates to
relay agents.

Modifying the message destined for the client is outside the relay agent behavior specified in
{{!RFC9915}}. A device that populates, modifies, or removes the rate options destined for the client
based on subscriber attributes (e.g., retrieved from an AAA server such as RADIUS) therefore acts as
a DHCPv6 server or DHCPv6 proxy towards the client, and the server behavior specified in this
document applies to it.

# DHCP Snooping {#snooping}

DHCP snooping devices are typically deployed as intermediate Layer 2 devices that passively monitor
DHCP message exchanges to enforce security policies and build binding databases, without modifying
the DHCP payloads. These devices MAY inspect the rate options in messages destined for clients and
use them to provision traffic shapers, policers, or hardware queues on the corresponding
client-facing ports. Snooping devices SHOULD remove rates applied for a client when the
corresponding binding is removed.

Snooping devices MUST only process rate options contained in DHCPACK (DHCPv4) and Reply (DHCPv6)
messages received on trusted, network-facing ports. Rate options contained in messages received on
client-facing ports MUST NOT be used to configure traffic management.

# Deployment Scenarios

## PPPoE {#pppoe}

In PPPoE {{?RFC2516}} architectures, the CPE typically employs DHCPv6 over the PPP {{?RFC1661}} link
to obtain an IPv6 prefix via Prefix Delegation (IA_PD) {{!RFC9915}}. This DHCPv6 exchange conveys
the rate options in the same way as in IPoE deployments. DHCPv4 transactions within a PPPoE session
can similarly convey the DHCPv4 rate options.

The processing rules for rate options received over PPPoE are identical to those defined for IPoE,
as specified in {{processing}}. In particular, the rates in effect revert to the default
configuration of the device when the PPPoE session is terminated.

As the signaled rates are Layer 3 rates, a CPE that attaches its shaper to the PPP interface applies
them directly, without accounting for the PPPoE, PPP, Ethernet, or VLAN headers.

If a client receives rate information embedded within a PPP authentication message and also receives
rate options, the client SHOULD give precedence to the rate options. This ensures that the
standardized, dynamic DHCP signaling supersedes proprietary rate information conveyed during the PPP
authentication phase.

## Local Area Networks {#lan}

The rate options can also be used by DHCP servers in local area networks, for example a DHCP server
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
rates are independent of the encapsulation, the learned rates can be passed on unchanged.

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
limits need to be enforced, enforcement remains the responsibility of the network; the rate options
only inform clients about such limits.

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
known link-layer headers; for example, CAKE with an explicitly configured overhead of 0 accounts the
network-layer packet size and thereby enforces exactly the signaled Layer 3 rate. Where no such
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
due to the synchronization rate of a DSL line, operators should signal the lower of the two rates,
so that the client shapes its traffic to the actual bottleneck. The actual rate is typically known
at the BNG, for example from the Actual-Data-Rate-Upstream and Actual-Data-Rate-Downstream
attributes {{?RFC4679}} or the corresponding Access Node Control Protocol (ANCP) TLVs {{?RFC6320}}.
As these attributes describe the net data rate of the access line, the server converts them to a
Layer 3 rate, for example by deducting the encapsulation overhead for typical packet sizes and, for
lines using Packet Transfer Mode (PTM) with 64/65-octet encapsulation, the corresponding 1/65 of the
rate. Access lines with cell-based link layers, such as Asynchronous Transfer Mode (ATM) on
Asymmetric Digital Subscriber Line (ADSL), have a packet-size-dependent overhead that a single rate
cannot describe precisely; operators of such lines can signal a conservatively reduced rate, and
precise modeling remains a matter of local configuration of the CPE.

Without the use of DHCPFORCERENEW or Reconfigure, changes of the provisioned rates take effect
with the next renewal of the client. Operators should consider this delay when choosing lease
and renewal times, for example when rates are changed due to service upgrades or fair-use
policies.

The inclusion of the rate option codes in the Parameter Request List or Option Request option allows
operators to determine the share of clients supporting these options, for example to plan the
introduction of correction factors or to identify clients that require enforcement at the
network.

Implementations are encouraged to expose the signaled rates within CPE management interfaces, such
as the local Web User Interface (UI) or remote management protocols. As the signaled rates are
Layer 3 rates, the displayed values are directly comparable with the results of speed tests, which
can reduce support inquiries related to perceived bandwidth issues.

As noted in {{processing}}, a client can receive a rate that exceeds the maximum physical link speed
of its interfaces. In such scenarios, management interfaces should expose both the signaled rate and
the effective, capped rate applied by the device, and implementations should log this discrepancy
if logging facilities are enabled. Such events can indicate a mismatch between the subscriber's
provisioned service tier and the installed equipment.

# Implementation Status {#impl-status}
{:removeinrfc="true"}

{::boilerplate rfc7942}

At the time of writing, no implementations of this document are known to the authors.

## Implementation Feasibility on Open-Source CPE

The authors analyzed the feasibility of implementing the client behavior on OpenWrt without
modifying the DHCP client software. The DHCPv4 client (BusyBox udhcpc) and DHCPv6 client (odhcp6c)
used by OpenWrt can request the rate options and expose received options of unknown type as
hexadecimal environment variables to user-defined hook scripts. All client behavior specified in
this document can therefore be implemented in such scripts, which convert the 8-octet value to an
integer, apply the processing rules of {{processing}}, and configure local traffic management, for
example via Smart Queue Management (SQM) or the CAKE queue discipline. As the signaled rates are
Layer 3 rates, they can be passed to CAKE directly as its bandwidth parameter, together with an
overhead of 0, without any encapsulation-specific calculation. The rates learned on the WAN
interface can be propagated to LAN clients using the existing custom-option support of the OpenWrt
DHCPv4 server (dnsmasq) and DHCPv6 server (odhcpd).

# Security Considerations {#security}

DHCP messages are typically transmitted as plaintext and are unauthenticated. Consequently, the rate
options defined in this document are vulnerable to interception, modification, or spoofing by
on-path attackers. This applies to any path the options traverse, including the path between
servers and clients as well as the path between servers and relay agents, where rate options carried
in a Relay-reply message configure the relay agent itself (see {{dhcpv6-relay}}). A malicious actor
or a rogue DHCP server could inject artificially low rates to throttle a client's connection,
resulting in a localized Denial of Service (DoS).

The severity of this risk is generally no greater than that of standard DHCP threat vectors, such
as rogue default gateway assignment, DNS hijacking, or IP pool exhaustion, which typically yield a
much more critical and immediate loss of service.

Operators can mitigate rogue DHCP servers by deploying DHCP snooping for DHCPv4 and DHCPv6-Shield
{{?RFC7610}} for DHCPv6 on access nodes, which discard server messages received on client-facing
ports. These mechanisms protect both the clients and the intermediate nodes that apply signaled
rates.

To mitigate the impact of malicious or malformed options, client implementations MAY apply sanity
checks before applying signaled rates. In particular, a client implementation MAY assume a minimum
rate below which basic connectivity is no longer ensured, and treat a rate option signaling a
non-zero rate below this minimum as if it were not present (see {{processing}}). The appropriate
minimum depends on the deployment and the use case of the device and is therefore left to the client
implementation. For example, a broadband CPE might reject rates below 1 Mbit/s.

Conversely, an attacker could inject an excessively high rate or a rate of 0, thereby removing any
effective rate limit. Because clients are expected to cap the effective rate to the capacity of the
physical link (see {{processing}}), the impact is limited to the loss of the benefits of rate
signaling, such as increased queuing delay at the bottleneck, which corresponds to the behavior of a
device without support for these options. Where an intermediate node derives the enforcement of the
provisioned rate from the rate options (see {{solution-overview}}), however, an attacker on the path
between the server and that node that raises or removes the signaled rates also raises or removes
the enforcement, allowing a client to exceed its provisioned rate. Mechanisms for securing the
messages exchanged between servers and relay agents are described in {{?RFC8213}}.

In local area networks, DHCP snooping is often not deployed, making rogue DHCP servers more
likely. A rogue server signaling artificially low rates could cause applications on end devices
to reduce their quality or to prefer another network interface, which can incur additional cost
(e.g., cellular data). Clients SHOULD NOT use the signaled rates as the sole criterion for
selecting a network interface, and the sanity checks described above apply.

Relay agents and snooping devices that instantiate traffic management functions based on the rate
options consume finite resources, such as hardware queues and shapers. Because only server messages
received on trusted ports are processed and rates are applied per client-facing port (see
{{processing}}), the amount of such resources is bounded by the number of client-facing ports.
However, a client can trigger frequent DHCP exchanges, potentially causing repeated reconfiguration
of these functions. In addition to not reconfiguring unchanged rates (see {{processing}}),
implementations MAY rate-limit DHCP messages received on client-facing ports or the reconfiguration
of traffic management functions.

# Privacy Considerations {#privacy}

The general privacy considerations for DHCP specified in {{?RFC7844}} apply to this document.

The rate options convey the rates provisioned for a client, which typically reflect the service tier
of a subscriber. Because DHCP messages are typically transmitted as plaintext, this information is
exposed to on-path observers, as well as to relay agents and snooping devices that process the
options. The rate information itself does not identify a subscriber, but in combination with other
information it can reveal commercial details of the service contract.

Clients that include the rate option codes in the Parameter Request List or Option Request option
disclose additional information that can contribute to fingerprinting the device. Clients that
implement the anonymity profiles specified in {{?RFC7844}} MAY omit the rate option codes from the
Parameter Request List or Option Request option.

The operational practice of exposing and logging signaled and effective rates (see {{operations}})
can provide operators with information about the equipment installed by a subscriber. Such data
is subject to the operator's applicable privacy policies.

# IANA Considerations {#iana}

IANA is requested to replace TBD1, TBD2, TBD3, and TBD4 throughout this document with the values
assigned as specified below. RFC Editor: please replace "RFC XXXX" throughout this document with the
RFC number assigned to this document, and remove this note.

## DHCPv4 Options

IANA is requested to assign two new DHCP Option Codes in the "BOOTP Vendor Extensions and DHCP
Options" registry as follows:

| Tag  | Name                      | Data Length | Meaning                    | Reference |
| :--- | :------------------------ | ----------: | :------------------------- | :-------- |
| TBD1 | OPTION_V4_UPSTREAM_RATE   |           8 | DHCPv4 Upstream Rate       | RFC XXXX  |
| TBD2 | OPTION_V4_DOWNSTREAM_RATE |           8 | DHCPv4 Downstream Rate     | RFC XXXX  |
{: #tab-iana-v4 title="DHCPv4 Option Codes"}

## DHCPv6 Options

IANA is requested to assign two new DHCPv6 Option Codes in the "Option Codes" registry as follows:

| Value | Description               | Client ORO | Singleton Option | Reference |
| :---- | :------------------------ | :--------- | :--------------- | :-------- |
| TBD3  | OPTION_V6_UPSTREAM_RATE   | Yes        | Yes              | RFC XXXX  |
| TBD4  | OPTION_V6_DOWNSTREAM_RATE | Yes        | Yes              | RFC XXXX  |
{: #tab-iana-v6 title="DHCPv6 Option Codes"}

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
ensures that this enforcement is not triggered under normal operation. CPE that do not support
these options continue to be subject to the enforcement at the BNG, so that the provisioned
service tier is maintained in all cases.

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

## Why Separate Options Instead of a Container Option?

Each option defined in this document carries exactly one fixed-size value. This avoids sub-option
parsing, a dedicated sub-option registry, and the corner cases of empty or partially valid
containers, and it allows each direction to be requested, signaled, and processed independently,
consistent with the guidance for simple option formats in {{?RFC7227}}. A client that only applies
the upstream rate can request and process only the upstream rate option. Should additional
parameters be required in the future, they can be defined as separate options.

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
  restriction to Layer 3 rates, the use of separate options, and the relation to access line
  information and to existing proprietary solutions.
- Added applicability to local area networks.

Protocol changes (not backward compatible with -00):

- Replaced the container option with sub-options by four top-level options:
  OPTION_V4_UPSTREAM_RATE, OPTION_V4_DOWNSTREAM_RATE, OPTION_V6_UPSTREAM_RATE, and
  OPTION_V6_DOWNSTREAM_RATE, each carrying a single 64-bit rate in bits per second.
- All rates are Layer 3 rates. Removed the rate type and layer selection, the informational rates,
  and the DHCP Rate Sub-Options registry.

Processing and behavior:

- Added the sections Traffic Management Using Signaled Rates, Rate Processing, and Error Handling,
  and specified the messages in which the options can appear.
- Clarified the requirement levels for applying rates, the precedence of administrator
  configuration, and the behavior of servers, relay agents, and snooping devices.

Other:

- Changed the intended status to Standards Track; replaced RFC 8415 by RFC 9915.
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
