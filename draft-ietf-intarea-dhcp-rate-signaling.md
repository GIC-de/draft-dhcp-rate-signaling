---
title: "DHCPv6 Options for Explicit Rate Signaling"
abbrev: "DHCPv6 Rate Signaling"
docname: draft-ietf-intarea-dhcp-rate-signaling-latest
submissiontype: IETF
ipr: trust200902
category: std
v: 3
area: Internet Area
workgroup: Internet Area Working Group
keyword:
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
  latest: "https://GIC-de.github.io/draft-dhcp-rate-signaling/\
    draft-ietf-intarea-dhcp-rate-signaling.html"
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
   IANA-DHCPV6:
      title: "Dynamic Host Configuration Protocol for IPv6 (DHCPv6)"
      target: https://www.iana.org/assignments/dhcpv6-parameters
      author:
      - org: IANA
   TR101:
      title: "Migration to Ethernet-Based Broadband Aggregation"
      target: https://www.broadband-forum.org/pdfs/tr-101-2-0-0.pdf
      author:
      - org: Broadband Forum
      date: 2011-07
      seriesinfo:
         TR: 101 Issue 2
   TR069:
      title: "CPE WAN Management Protocol"
      target: https://www.broadband-forum.org/pdfs/tr-069-1-6-1.pdf
      author:
      - org: Broadband Forum
      date: 2020-06
      seriesinfo:
         TR: 069 Issue 1 Amendment 6 Corrigendum 1
   OPENWRT:
      title: "OpenWrt Project"
      target: https://openwrt.org
      author:
      - org: OpenWrt Project
   SQM:
      title: "SQM Scripts: Smart Queue Management"
      target: https://github.com/tohojo/sqm-scripts
      author:
      - org: The Bufferbloat Project
   G.984.1:
      title: "Gigabit-capable passive optical networks (GPON): General characteristics"
      target: https://www.itu.int/rec/T-REC-G.984.1
      author:
      - org: ITU-T
      date: 2008-03
      seriesinfo:
         ITU-T Recommendation: G.984.1

--- abstract

This document defines options for the Dynamic Host Configuration Protocol for IPv6 (DHCPv6) that
signal the intended upstream and downstream rates of a network connection to clients. Each rate
applies to the combined IPv4 and IPv6 traffic of the corresponding logical interface. In broadband
access networks, the Customer Premises Equipment (CPE) typically sends at the speed of its physical
link while the upstream rate is enforced by policers at the service edge. By learning the upstream
rate, the CPE can shape its traffic and manage the queue using prioritization and active queue
management (AQM). Two options carry rates counted at the IP layer (Layer 3 rates). Two additional,
optional options carry rates counted at the link layer together with the per-packet overhead (Layer
2 rates), for networks in which the rate is not limited at the IP layer.

--- middle

# Introduction

## Problem Statement {#problem-statement}

In broadband access networks, the rate available to a subscriber is typically lower than the speed
of the physical link that connects the Customer Premises Equipment (CPE) to the network. The service
edge is typically a Broadband Network Gateway (BNG) {{TR101}}. The available rate can also be
limited by an access device connected to the CPE, such as an Optical Network Terminal (ONT)
{{G.984.1}}, a Digital Subscriber Line (DSL) modem, or a cable modem, or within the access network,
for example by the synchronization rate of a DSL line. This document refers to the node or link that
limits the rate as the enforcement point. The CPE is typically unaware of these rates and sends at
the speed of its physical link.

Per-subscriber queuing in the upstream direction is not supported by all BNG platforms. Where it
is supported, it is often restricted to specific hardware or available only at reduced scale, as
it competes for the same queuing and scheduling resources as downstream hierarchical scheduling.
Upstream enforcement is therefore commonly implemented using ingress policers, which discard
packets that exceed the configured rate and burst size instead of queuing them. As the speed of
the physical link of the CPE can exceed the provisioned rate by an order of magnitude, traffic
bursts are discarded at the policer. Congestion control algorithms react to these losses by
repeatedly reducing their sending rate, resulting in an unstable sawtooth throughput pattern well
below the provisioned rate. Users experience this as poor and fluctuating results, for example in
speed tests, even though no part of the network is congested.

Where the upstream bottleneck is located at the access device or within the access network, the
resulting queue is typically managed by neither active queue management (AQM) {{?RFC7567}} nor
prioritization. Depending on the buffering of the access device, this results in packet loss or in
increased latency under load, affecting all of the subscriber's traffic.

In both cases, upstream prioritization at the CPE, for example of DNS, Voice over IP (VoIP), or
video conferencing traffic, or of TCP acknowledgments, is ineffective. As long as the CPE sends at
the speed of its physical link, its queues remain empty, and packets are discarded or delayed at the
actual bottleneck regardless of their priority.

Because the downstream rate is typically enforced at the BNG using egress shapers, it is of lower
relevance for traffic management at the CPE. It remains relevant for informational purposes, such
as displaying the provisioned service tier in the CPE user interface.

## Solution Overview {#solution-overview}

This document defines four DHCPv6 {{!RFC9915}} options: two that signal rates counted at the IP
layer (Layer 3 rates), and two optional ones that signal rates counted at the link layer together
with the per-packet overhead (Layer 2 rates).

The Upstream Rate option and the Downstream Rate option signal Layer 3 rates. Each carries one
unsigned 64-bit integer: the intended rate for its direction in bits per second (bit/s). Each rate
counts the IP header and its payload at the IP layer of the link to which the option applies (see
{{layer3}}) and covers the combined IPv4 and IPv6 traffic of that link (see {{dual-stack}}).

These two Layer 3 Rate Options are the baseline and a complete solution for the most common case:
a service rate that is enforced at the IP layer on an access line of higher capacity, for example
the rate of a subscribed product on a fiber access line. Every client that supports this document
can use them.

Where the enforcement point does not count IP packet sizes, for example because it counts
link-layer frames or is a physical link such as a copper access line, a Layer 3 rate can only
approximate it, because the IP rate that such an enforcement point carries depends on the packet
size. For this case, the Upstream Layer 2 Rate option and the Downstream Layer 2 Rate option each
carry the rate as counted by the enforcement point together with the per-packet overhead and the
minimum packet size that the enforcement point counts for each IP packet (see {{layer2}}). These
Layer 2 Rate Options are optional for servers and clients. A server never sends one without the
Layer 3 Rate Option for the same direction, and a client that does not support them uses the Layer
3 Rate Options. {{layer2}} explains why the two kinds of rate are carried in separate options.

This document specifies the encoding, transport, and meaning of the options. The usage examples
are illustrative; clients choose whether and how to use the information according to local policy
(see {{application}}).

A CPE that knows the upstream rate can use it to configure an upstream shaper. Queuing at the CPE
can then be managed using prioritization, AQM, and flow queuing {{?RFC8290}}. Where the queue at
the CPE implements an AQM that supports the Low Latency, Low Loss, and Scalable Throughput (L4S)
architecture {{?RFC9330}}, such as the DualQ Coupled AQM {{?RFC9332}}, it can additionally provide
low latency for L4S traffic, provided that the shaper at the CPE is the bottleneck. When the shaper
accounts for the bottleneck rate and burst tolerance, this can reduce policer drops and queuing in
the access network (see {{fig-policer}}). Observations from the deployment of dual-queue low-latency
networking by an operator, covering the home network, the CPE, and the aggregation network, are
described in {{?I-D.livingood-low-latency-deployment}}.

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
| 100 Mbit/s  |                                 | 100 Mbit/s: pass  |
+-------------+                                 +-------------------+
~~~
{: #fig-policer title="Upstream Policing at the BNG with and without CPE Shaping"}

Enforcement at the BNG remains in place. The rate of the BNG policer can be configured slightly
above the signaled rate to accommodate CPE shaping and burst behavior.

## Applicability {#applicability}

The mechanism defined in this document is primarily designed for broadband access networks using
IP over Ethernet (IPoE) {{TR101}} or Point-to-Point Protocol over Ethernet (PPPoE) {{?RFC2516}}. In
both cases, DHCPv6 is commonly used by the CPE to obtain addresses and delegated prefixes; in PPPoE
deployments, it runs over the Point-to-Point Protocol (PPP) {{?RFC1661}} session (see {{pppoe}}).

The options are defined for DHCPv6 only. DHCPv6 is available in dual-stack and IPv6-only
deployments, including those that provide IPv4 service over IPv6. Signaling the same rates in both
DHCPv4 and DHCPv6 would require every client to retain the rates of each protocol separately, as the
withdrawal of a rate in one protocol, or the expiry of its lease, leaves the rates of the other
protocol in effect. With a single protocol, the rates are processed like any other DHCPv6
configuration option (see {{client}}). The rates nevertheless cover IPv4 traffic (see
{{dual-stack}}); DHCPv6 options that configure IPv4 service have precedent, for example the Address
Family Transition Router (AFTR) Name option for Dual-Stack Lite (DS-Lite) {{?RFC6334}} and the
options for Mapping of Address and Port (MAP) {{?RFC7598}}. Networks that do not use DHCPv6 on the
link, such as IPv4-only access networks, cannot use the options and are outside the scope of this
document.

DHCPv6 is widely deployed on CPE, including customer-owned devices and devices running open-source
firmware. It provides an existing path for initial configuration and updates, complementing manual
configuration and management protocols such as TR-069 {{TR069}}.

The options are applicable to any access link on which DHCPv6 is used to configure the
network-facing interface and on which the intended rates are determined by a subscription, contract,
or policy, including enterprise access links. They convey provisioned, slowly changing values and do
not describe the instantaneous capacity of links whose capacity varies over time, such as wireless
or cellular links; on such links, they can serve as an upper bound. Signaling rates to relay agents
or other network nodes is outside the scope of this document.

The Layer 3 rates describe the access service and not a property of the link between the CPE and
the network. A CPE can therefore make them available to devices in the local area network (LAN)
behind it, for example to a VPN gateway that shapes its tunnel traffic to the rate of the access
service, or to hosts that display the rate or use it as an upper bound when selecting an initial
media quality. This document permits this use and states its limits (see {{application}}), but it
does not define the mechanism by which a CPE conveys the rates to devices in the LAN.

The Standard Communication with Network Elements (SCONE) protocol {{?I-D.ietf-scone-protocol}}
addresses a different part of the problem. SCONE carries throughput advice from a network element
to the endpoints of an individual QUIC flow, so that applications can adapt, for example the
bitrate of a video stream. The options defined in this document instead configure the CPE with the
rates of the access service as a whole, independent of flows and applications, so that the CPE can
shape and schedule its traffic. The two mechanisms are complementary: a CPE that knows its rates
can manage its queue, and it could also act as a network element that provides throughput advice to
endpoints.

# Conventions and Definitions {#definitions}

{::boilerplate bcp14-tagged}

This document makes use of the terms defined in {{!RFC9915}}. The following additional terms are
used:

Downstream
: The direction of traffic from the network toward the client, received by the client over the
  link on which the Rate Option was received.

Enforcement Point
: The node or link at which the rates of a subscriber are enforced or which constitutes the
  bottleneck of the path, for example a policer at the BNG or an access link.

Intended Rate
: The maximum rate, in a given direction, at which the network intends to forward the traffic of a
  client. Traffic exceeding the intended rate is subject to being dropped or delayed at the
  enforcement point. The intended rate is not a guarantee: the achievable rate can be lower, for
  example due to congestion elsewhere in the network. The Rate Options signal the intended rates;
  this document also refers to them as the signaled rates.

Layer 2 Rate
: A rate calculated over each IP packet plus a per-packet Overhead, subject to a Minimum Packet
  Unit (MPU), both of which are signaled together with the rate (see {{layer2}}).

Layer 2 Rate Option
: The Upstream Layer 2 Rate option (OPTION_UPSTREAM_L2_RATE) or the Downstream Layer 2 Rate
  option (OPTION_DOWNSTREAM_L2_RATE) defined in this document.

Layer 3 Rate
: A rate calculated over the IP header and its payload at the IP layer of the link to which the
  Rate Option applies (see {{layer3}}).

Layer 3 Rate Option
: The Upstream Rate option (OPTION_UPSTREAM_RATE) or the Downstream Rate option
  (OPTION_DOWNSTREAM_RATE) defined in this document.

Rate Option
: Any of the four options defined in this document, that is, a Layer 3 Rate Option or a Layer 2
  Rate Option.

Subscriber
: The customer of a network access service. Within the network, a subscriber is typically
  represented by a subscriber session at the BNG, for example an IPoE or PPPoE session, to which the
  subscribed rates and other per-subscriber policies are applied.

Upstream
: The direction of traffic from the client toward the network, sent by the client over the link on
  which the Rate Option was received.

# Rate Semantics {#semantics}

## Rate Values {#values}

Each Rate Option carries a rate as an unsigned 64-bit integer in network byte order
(big-endian), expressing the rate in bits per second (bit/s), not in bytes (octets) per second.
Throughout this document, Mbit/s and Gbit/s denote 10^6 and 10^9 bits per second, respectively. The
largest representable rate is 2^64 - 1 bit/s (18,446,744,073,709,551,615 bit/s), approximately
18.45 Ebit/s, where Ebit/s denotes 10^18 bits per second.

The value 0 is not a valid rate. A server MUST NOT send a Rate Option with a rate of 0. A direction
for which no rate is signaled is indicated by the absence of the respective option.

The two directions are independent: a server can send Rate Options for either or both, and the
presence or absence of the options for one direction does not affect the processing of the other.

## Layer 3 Rates {#layer3}

Servers and clients MUST interpret the rate of a Layer 3 Rate Option as a Layer 3 rate: the number
of bits per second counted over the IP header and its entire payload, including IP options, IPv6
extension headers, and transport headers. The reference is the IP layer of the physical or logical
link on which the DHCPv6 exchange takes place; this document refers to it as the reference link.

This reference remains the same when the link is carried over a tunnel. If DHCPv6 runs inside a
tunnel, the rate refers to the IP packets on that logical link, excluding the tunnel's carrier
headers. If the link instead carries tunneled data, such as IPv4-in-IPv6 traffic from a DS-Lite
Basic Bridging BroadBand (B4) element {{?RFC6333}}, the IPv6 header and its encapsulated payload are
counted. Thus the reference is the IP layer of the DHCPv6-configured link, not necessarily the
outermost IP header on the underlying physical interface. Translation is accounted for as seen on
the reference link.

Layer 3 accounting excludes link-layer headers, trailers, padding, and physical-layer overhead.
Virtual LAN (VLAN) tags or PPPoE and PPP headers added or removed along the path do not change the
rate. IP encapsulation or translation between the client and the enforcement point can change the
counted packet size, which operators need to take into account when selecting the signaled rate.

For 1500-octet TCP/IP packets without options or extension headers, TCP payload throughput is
approximately 2.7% (IPv4) to 4% (IPv6) lower than the Layer 3 rate due to the IP and transport
headers.

Traffic management that counts IP packet sizes can use the signaled rate directly. Traffic
management that counts link-layer frames needs to exclude the counted link-layer overhead to
reproduce Layer 3 accounting. This header compensation is local: only overhead actually counted by
the implementation is removed, including padding where applicable. For example, a shaper counting
an Ethernet header, one VLAN tag, PPPoE, and PPP, but no frame check sequence (FCS) or padding,
subtracts 14 + 4 + 6 + 2 = 26 octets per packet. Overhead inserted later by hardware is not
subtracted. Applying the same numeric rate without compensation yields a lower IP rate that
depends on packet size; it does not change the meaning of the signaled value.

The meaning of the Layer 3 Rate Options does not depend on the presence of a Layer 2 Rate Option
({{layer2}}) or of any other option. A client that does not support the Layer 2 Rate Options
ignores them and uses the Layer 3 Rate Options as specified in this section.

## Layer 2 Rates {#layer2}

This document defines separate options for Layer 3 rates and Layer 2 rates because the two
describe different enforcement points and cannot replace each other.

The Layer 3 Rate Options are a complete solution wherever the rate is limited at the IP layer. The
most common case is a service rate that is enforced at the service edge on an access line of higher
capacity, for example the rate of a subscribed product on a fiber access line. Where the enforcement
point counts IP packet sizes, a client that shapes IP packets at the signaled Layer 3 rate matches
it for every packet size. By definition, a Layer 3 rate needs no overhead information, and a client
implements it by reading one integer per direction.

Other enforcement points do not count IP packet sizes. A policer or shaper that counts link-layer
frames, or a physical link such as a copper access line or an Ethernet handoff, consumes
additional octets per packet, so the IP rate it can carry depends on the packet size. For example,
an Ethernet link with one VLAN tag consumes 42 octets per packet in addition to the IP packet and
at least 84 octets per packet in total. At 100 Mbit/s, such a link carries 97.3 Mbit/s of
1500-octet IP packets but only 47.6 Mbit/s of 40-octet IP packets.

A Layer 2 rate therefore cannot be converted into a single Layer 3 rate, or the reverse, without
assuming a packet size; the result is wrong for traffic with other packet sizes. The conversion is
exact only when it is applied to each packet and the per-packet overhead is known. For this reason,
a Layer 2 rate is always signaled together with its overhead, and it is carried in options of its
own, so that the meaning of the Layer 3 Rate Options never depends on other values.

The guidelines for DHCPv6 options advise against options that are aliases of each other, that is,
options that provide input to the same configuration parameter ({{Section 7 of ?RFC7227}}). The
Layer 3 and Layer 2 Rate Options are not aliases, because they carry different information:

- A Layer 3 rate describes the service. It is independent of the encapsulation of any particular
  link, so it remains meaningful beyond the link on which it was received, for example for display
  in a user interface or as information for devices in the LAN behind the client (see
  {{application}}).
- A Layer 2 rate, with its Overhead and MPU, describes how the enforcement point of this link
  counts packets. These values are meaningful only for traffic management by the DHCPv6 client on
  this link and are not passed on to other devices.

Neither can be derived from the other, and a client can use both at the same time: the Layer 2
values to shape traffic on the link, and the Layer 3 rate for display or other purposes. For the
traffic management of one direction, a client applies exactly one of them, selected by the rule in
{{client}}, and a server never sends a Layer 2 Rate Option without the Layer 3 Rate Option for the
same direction (see {{placement}}).

This difference is also the reason why the Layer 3 Rate Options are mandatory and the Layer 2 Rate
Options are optional. Every use of the rates other than traffic management on the link, such as
display or making the rate available to devices in the LAN, requires the Layer 3 rate and cannot
be served by Layer 2 values. The Layer 3 rate is also the only value that every client can apply.
The Layer 2 Rate Options refine a single use, traffic management on the link, for enforcement
points that do not count at the IP layer.

A Layer 2 Rate Option describes the enforcement point for one direction with three values:

Layer 2 Rate:
: The rate in bits per second, with the encoding specified in {{values}}, counted over the
  accounted size of each packet as defined below.

Overhead:
: The number of octets that the enforcement point counts per packet in addition to the size of
  the IP packet. It includes all link-layer headers and trailers counted by
  the enforcement point, whether or not they are present on the link of the client, and, where the
  enforcement point is a physical link, physical-layer overhead that consumes transmission time,
  such as the Ethernet preamble and inter-packet gap. The Overhead is always present and can be 0.

Minimum Packet Unit (MPU):
: The minimum number of octets that the enforcement point counts per packet, after the Overhead
  has been added. The value 0 indicates that no minimum applies.

For an IP packet of S octets, the accounted size L in octets is:

~~~
L = max(S + Overhead, MPU)
~~~

S is the size of the IP packet as defined for Layer 3 rates in {{layer3}}, on the same reference
link. The Overhead thus states which octets below the IP layer are counted, so this document does
not need a separate definition of where Layer 2 begins and ends. As the Overhead is relative to the
IP packet and not to the frames on the link of the client, the values are independent of the
encapsulation used by the client and of where its traffic management counts packets: an
implementation that counts IP packets adds the Overhead, and an implementation that counts
link-layer frames adds the Overhead minus the link-layer octets it already counts. For example, a
shaper that counts the Ethernet header and one VLAN tag (18 octets) and receives an Overhead of 42
configures a per-packet adjustment of 24 octets.

The Layer 2 rate and the Layer 3 rate of the same direction describe the same enforcement point at
different precision. The Layer 3 rate is the value that the operator selects for clients that count
IP packets without the signaled Overhead (see {{operations}}). The Layer 2 rate is meaningful only
together with the Overhead and MPU of the same option; a client MUST NOT apply it as a Layer 3 rate
or without the Overhead and MPU.

Effects that scale all packets equally, such as line coding, are not part of the Overhead; the
operator reflects them in the Layer 2 rate. Link layers that pad each packet to a whole number of
fixed-size cells, such as Asynchronous Transfer Mode (ATM) Adaptation Layer 5 (AAL5) with 48 octets
of payload in each 53-octet cell, cannot be described exactly by these values. They can be described
conservatively by signaling the rate of the cell stream multiplied by 48/53 as the Layer 2 rate and
by adding 47 octets to the Overhead, which never underestimates the size of a packet on the link. In
this case, the Overhead also includes the encapsulation within the AAL5 frame and the 8-octet AAL5
trailer.

{{dual-stack}} and {{application}} apply to Layer 2 rates in the same way as to Layer 3 rates.

## IPv4 and IPv6 Traffic {#dual-stack}

Although the Rate Options are carried in DHCPv6, each rate describes the aggregate IP traffic of
the service on the reference link in its direction, including both IPv4 and IPv6. The rates are not
an allowance for IPv6 traffic only, and there is no separate allowance for IPv4: for example, a
signaled upstream rate of 1 Gbit/s describes 1 Gbit/s in total for IPv4 and IPv6 together.

This applies regardless of how IPv4 is provided on the logical interface, for example natively and
configured by DHCPv4 {{?RFC2131}} or by the IP Control Protocol (IPCP) {{?RFC1332}}, or over IPv6
using a transition mechanism such as DS-Lite {{?RFC6333}}. In the latter case, IPv4 traffic is
counted as the encapsulating IPv6 packets on the reference link (see {{layer3}}).

The rates cover only the traffic of the logical interface on which the DHCPv6 exchange takes place.
Where IPv4 is provided on a different logical interface, for example in a separate PPPoE session or
on a separate VLAN, the signaled rates do not apply to that IPv4 traffic.

## Use of Signaled Rates {#application}

A client MAY use the signaled rates for traffic management, display, or other local purposes. This
document does not require shaping, AQM, or a particular response to rates that exceed local link or
processing capacity. These choices, including precedence over locally configured policies, are
implementation and deployment decisions. For example, a CPE with a 1 Gbit/s connection to an
external modem can display a provisioned rate of 2 Gbit/s together with the local link limit,
whether or not it configures a shaper. A client that uses the rates for traffic management
accounts for their meaning ({{layer3}}, {{layer2}}) and for the traffic to which they apply
({{dual-stack}}).

A client MAY make the Layer 3 rates available to devices on its other links, for example to
devices in the LAN behind a CPE, as information about the access service. Where it does so, it
conveys the Layer 3 rates of the Layer 3 Rate Options, also where it uses a Layer 2 Rate Option for
its own traffic management. The capacity available to a device in the LAN can be lower than the
signaled rate, for example because of the LAN link or because the access service is shared with
other devices, so that such a device can use the rate only as an upper bound. A client MUST NOT
pass the Layer 2 Rate, Overhead, or MPU of a Layer 2 Rate Option on to other devices: these values
describe how the enforcement point counts packets on the link of the client and have no meaning on
any other link.

The rates are neither a guarantee of available capacity nor a replacement for congestion control.
Clients MUST NOT use them to bypass congestion control or to increase their sending rates beyond
what the congestion control algorithm would otherwise permit. Clients that make the rates available
to applications MUST NOT present them as available capacity or as a sending rate; applications can
use them only as an upper bound or for display. Enforcement of a service limit remains the
responsibility of the network.

# DHCPv6 Rate Options {#options}

## Upstream Rate Option {#upstream-option}

The Upstream Rate option (OPTION_UPSTREAM_RATE) carries the intended Layer 3 rate for the upstream
direction. It has the following format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   option-code (TBD1)          |         option-len=8          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                    Upstream Rate (64 bits)                    +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-upstream-format title="Upstream Rate Option Format"}

option-code
: OPTION_UPSTREAM_RATE (TBD1).

option-len
: 8.

Upstream Rate
: Unsigned 64-bit integer in network byte order, in bits per second, as specified in
  {{semantics}}.

## Downstream Rate Option {#downstream-option}

The Downstream Rate option (OPTION_DOWNSTREAM_RATE) carries the intended Layer 3 rate for the
downstream direction. It has the following format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   option-code (TBD2)          |         option-len=8          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                   Downstream Rate (64 bits)                   +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-downstream-format title="Downstream Rate Option Format"}

option-code
: OPTION_DOWNSTREAM_RATE (TBD2).

option-len
: 8.

Downstream Rate
: Unsigned 64-bit integer in network byte order, in bits per second, as specified in
  {{semantics}}.

## Layer 2 Rate Options {#layer2-options}

The Upstream Layer 2 Rate option (OPTION_UPSTREAM_L2_RATE) and the Downstream Layer 2 Rate
option (OPTION_DOWNSTREAM_L2_RATE) carry the Layer 2 Rate, Overhead, and MPU of the enforcement
point for the upstream and downstream direction, respectively (see {{layer2}}). Both options
have the following format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          option-code          |         option-len=12         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                     Layer 2 Rate (64 bits)                    +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Overhead            |              MPU              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-layer2-format title="Layer 2 Rate Option Format"}

option-code
: OPTION_UPSTREAM_L2_RATE (TBD3) or OPTION_DOWNSTREAM_L2_RATE (TBD4).

option-len
: 12.

Layer 2 Rate
: Unsigned 64-bit integer in network byte order, in bits per second. The value 0 is not valid.

Overhead
: Unsigned 16-bit integer in network byte order, in octets.

MPU
: Unsigned 16-bit integer in network byte order, in octets.

The format has a fixed length and no conditional fields, in line with the guidelines in
{{?RFC7227}}. The two directions use separate options, so the Overhead and MPU can differ between
the upstream and downstream direction.

## Option Placement {#placement}

The Rate Options are top-level options, not Identity Association (IA) sub-options. They can appear
in Advertise messages, where they are informational (see {{client}}), and in Reply messages sent in
response to Solicit (with Rapid Commit), Request, Renew, Rebind, and Information-request. A server
MUST NOT include them in replies to Confirm, Release, or Decline, or in Reconfigure messages. A
client MUST ignore a Rate Option in any other placement or exchange.

Clients MUST NOT send Rate Options, and servers MUST ignore Rate Options received from clients.
This document does not define the use of Rate Options in the options field of Relay-forward or
Relay-reply messages.

All four options are singleton options ({{Section 21 of !RFC9915}}). A server MUST NOT include more
than one instance of the same Rate Option in a message. {{!RFC9915}} treats multiple instances of
an option as separate instances; for the Rate Options, this document additionally specifies that a
client that receives more than one instance of the same Rate Option MUST discard all instances of
that option and treat it as absent.

A Layer 3 Rate Option whose option-len is not 8, or whose rate is 0, is invalid. A Layer 2 Rate
Option whose option-len is not 12, or whose Layer 2 Rate is 0, is invalid. A client MUST discard an
invalid option and treat it as absent. Invalid options do not invalidate otherwise valid DHCPv6
configuration.

A server MUST NOT include a Layer 2 Rate Option in a message unless it includes the Layer 3 Rate
Option for the same direction in that message. A client MUST discard a Layer 2 Rate Option and treat
it as absent if the message does not contain a valid Layer 3 Rate Option for the same direction.
This ensures that a Layer 3 rate is available to every client whenever rates are signaled.

# Client Behavior {#client}

DHCPv6 clients that support this document MUST support the Layer 3 Rate Options. Support for the
Layer 2 Rate Options is OPTIONAL. A client requests the Rate Options that it supports using the
Option Request option (ORO) as specified in {{Section 21.7 of !RFC9915}}. Requesting an option
indicates support for receiving it; it does not indicate whether the client will use the rates for
traffic management.

A Layer 2 Rate Option depends on the Layer 3 Rate Option for its direction: a client discards it
if the same message does not contain a valid Layer 3 Rate Option for that direction (see
{{placement}}). A Layer 3 Rate Option does not depend on a Layer 2 Rate Option.

For each direction, a client that uses the signaled values for traffic management selects them as
follows:

- If the client holds a valid Layer 2 Rate Option for the direction and its traffic management can
  apply the Overhead and MPU per packet, it SHOULD use the Layer 2 Rate, Overhead, and MPU. The
  server sends a Layer 2 Rate Option only where it describes the enforcement point more precisely
  than the Layer 3 rate (see {{server}}).
- Otherwise, the client uses the rate of the Layer 3 Rate Option as specified in {{layer3}}.

A client that displays a rate, for example in its user interface, uses the Layer 3 rate.

A client MUST NOT apply rates received in any message other than Reply. Rates in an Advertise
message are informational and indicate the rates the server intends to supply; see {{security}}
regarding their use in server selection.

The Rate Options are configuration information received from the server and are processed as
specified in {{!RFC9915}} for other configuration options that are not associated with an IA. In
particular:

- A valid Rate Option in a Reply to Solicit (with Rapid Commit), Request, Renew, Rebind, or
  Information-request replaces the previously received values of that option, regardless of which
  of these exchanges supplied them.
- If a Reply to Renew, Rebind, or Information-request does not include a requested Rate Option that
  the client previously received, the client SHOULD stop using the values of that option, as
  specified in {{Section 18.2.10.5 of !RFC9915}}. This applies regardless of which exchange
  supplied them; a client does not need to track the source of a rate. An option that the client
  discards and treats as absent (see {{placement}}) is handled like an option that is not included.
- The rates are part of the client's DHCPv6 configuration and are not associated with individual
  bindings. The client discards them when it discards that configuration, for example because all
  addresses and delegated prefixes have become invalid, because it switches to another server, or
  because the underlying link or PPP session terminates. Clients that use only Information-request
  refresh them as specified in {{Section 21.23 of !RFC9915}}.

Replies to Confirm, Release, or Decline do not carry Rate Options (see {{placement}}) and do not
change the rates. A client MUST NOT treat a rate that it has stopped using as current
DHCPv6-provided information; subsequent use of local defaults remains a local decision.

# Server Behavior {#server}

A server sends Rate Options only to clients for which rates are configured, and it includes
requested options in Advertise and Reply messages as specified in {{!RFC9915}}. This document adds
the following rules:

- As the omission of a requested Rate Option from a Reply to Renew, Rebind, or Information-request
  withdraws its values (see {{client}}), a server for which values are configured for a client
  MUST include the corresponding requested Rate Options in every such Reply. To withdraw a rate,
  the server omits the option.
- A server MUST NOT include a Layer 2 Rate Option without the Layer 3 Rate Option for the same
  direction (see {{placement}}). A server therefore cannot signal Layer 2 rate values for a
  direction for which no Layer 3 rate is configured.
- Where several servers can respond to a client, for example after a Rebind, operators need to
  configure the rates consistently on all of them; otherwise, a Reply from a server without that
  configuration withdraws the rates.

Operators configure Layer 2 rate values only where they know how the enforcement point accounts
for packets. Where the enforcement point counts IP packet sizes, the Layer 3 Rate Option describes
it exactly and a Layer 2 Rate Option is not needed.

The server MAY derive the rates from local configuration profiles, from Authentication,
Authorization, and Accounting (AAA) systems such as Remote Authentication Dial-In User Service
(RADIUS) {{?RFC2865}}, or from external policy servers. As the rates are per-client values, the
server or its policy source needs to hold them for each client.

Where the rates are provisioned via RADIUS, they can be carried as Rate Options in the
DHCPv6-Options attribute {{!RFC9445}}, in which all four options are permitted (see {{iana}}). Where
the BNG operates as a relay agent and a separate DHCPv6 server configures the client, the BNG can
convey this attribute to the server in the RADIUS option {{!RFC7037}}. As the omission of a
requested Rate Option withdraws the rate ({{client}}), such a relay agent SHOULD include the RADIUS
option in every Relay-forward message relaying a Solicit, Request, Renew, Rebind, or
Information-request from the client, unless the server retains the rates for the client. This goes
beyond {{!RFC7037}}, which recommends the RADIUS option for messages relaying Solicit, Request, and
Rebind. As specified in {{Section 3 of !RFC9445}}, options supplied via RADIUS take precedence over
local configuration. How rates that are updated during a session, for example by a RADIUS Change of
Authorization {{?RFC5176}}, reach a separate DHCPv6 server is outside the scope of this document. In
all cases, the server includes the Rate Options in its responses as specified in this section.

A server MAY include the Rate Options in its responses even if the client did not request them via
the ORO, according to its configuration. As revocation by omission applies only to requested
options ({{Section 18.2.10.5 of !RFC9915}}), a server cannot withdraw unrequested rates by omitting
them; such rates are discarded only as part of the client's configuration (see {{client}}). This is
a further reason for clients to request the Rate Options.

To update the rates of a client before its next scheduled renewal, a server MAY send a Reconfigure
message as specified in {{Section 18.3.11 of !RFC9915}}, causing the client to initiate a Renew,
Rebind, or Information-request transaction. As specified in {{!RFC9915}}, a server can send
Reconfigure messages only to clients that have included the Reconfigure Accept option in their
messages, and Reconfigure messages are authenticated using the Reconfiguration Key Authentication
Protocol (RKAP; see {{Section 20.4 of !RFC9915}}). Updated rates are conveyed in the subsequent
Reply.

For clients that do not support Reconfigure, updated rates take effect with the next Renew or
Rebind transaction or, for stateless clients, with the next Information-request transaction.
Operators can control the timing of such updates through the T1 value and the Information
Refresh Time option ({{Section 21.23 of !RFC9915}}). As support for Reconfigure is limited in
deployed clients, operators cannot generally rely on it for updates before the next renewal or
information refresh.

# PPPoE Deployments {#pppoe}

In PPPoE {{?RFC2516}} architectures, the CPE typically employs DHCPv6 over the PPP link to obtain an
IPv6 prefix via prefix delegation {{!RFC9915}}, while IPv4 is configured using IPCP {{?RFC1332}}.
The DHCPv6 exchange conveys the Rate Options in the same way as in IPoE deployments, and the
signaled rates apply to the combined IPv4 and IPv6 traffic of the PPP session (see {{dual-stack}}).

As the rates of the Layer 3 Rate Options are Layer 3 rates, a CPE that attaches its shaper to the
PPP interface applies them directly, without accounting for the PPPoE, PPP, Ethernet, or VLAN
headers. Where a Layer 2 Rate Option is used, its Overhead includes the PPPoE and PPP headers if the
enforcement point counts them, as the Overhead is relative to the IP packet (see {{layer2}}).

If a client receives rate information from other sources, for example within PPP authentication
messages, precedence is a matter of local policy. DHCPv6 can refresh the information during a
session.

# Operational Considerations {#operations}

By default, a server does not send Rate Options; rates are signaled only when
configured by the operator.

Where the enforcement point counts IP packet sizes, for example a policer at the BNG configured with
a Layer 3 rate, the operator signals that rate in the Layer 3 Rate Option, and no Layer 2 Rate
Option is needed. Where the enforcement point accounts for additional octets per packet, the IP rate
it can carry depends on the packet size (see {{layer2}}), and the operator has to choose the Layer 3
rate for clients that do not use the Layer 2 Rate Option. A Layer 3 rate derived for large packets,
for example 1500 octets, matches the throughput of bulk transfers, but traffic with smaller packets
can exceed the enforcement point, so that queuing or loss moves back into the network. A lower Layer
3 rate keeps the queue at the client for more traffic at the cost of throughput. The Layer 2 Rate
Option avoids this trade-off for clients that support it. In both cases, the signaled values need to
stay at or below what the enforcement point forwards; a client that shapes below the enforcement
point remains the bottleneck, whereas a client that shapes above it does not.

Where the bottleneck is the access line itself, for example a DSL line, its rate is not a
provisioned value but the result of line synchronization. A BNG typically learns this rate from
access line information, for example the access loop characteristics conveyed via PPPoE or DHCP
{{TR101}}, the corresponding RADIUS attributes {{?RFC4679}}, or the Access Node Control Protocol
(ANCP) {{?RFC6320}}. The operator can derive the signaled rates from this information, for example
as the lower of the subscribed rate and the line rate. As the line rate can change when the line
resynchronizes, such deployments need to consider how quickly updated rates reach the client (see
{{server}}).

Implementations are expected to allow operators to inspect the rates in effect, for
example per interface at a client, and to log changes to them. This document does not define a YANG
data model; the Rate Options can be added to the YANG data model for DHCPv6 configuration
{{?RFC9243}} in a separate document.

# Security Considerations {#security}

The security considerations of DHCPv6 {{!RFC9915}} apply. Without protection against modification
and spoofing, an attacker can advertise false rates. A client that uses an artificially low rate
for traffic management could throttle its own connection. A falsely high rate or a withdrawal
could remove a locally derived limit and increase queuing or loss. As the absence of a Rate Option
withdraws a rate, an on-path attacker that removes Rate Options from DHCPv6 messages achieves the
same effect as a forged withdrawal. Even a client that only displays rates can be misled;
applications using the values could reduce quality or choose a more costly network interface.

Local policy can restrict whether received rates affect traffic management and can validate values
against deployment-specific bounds. The full unsigned 64-bit range is valid on the wire; decoding,
arithmetic, and conversion to local APIs need to avoid overflow, truncation, and unit errors.
Choosing not to apply a value does not change its meaning. A forged Layer 2 Rate Option can reduce
the effective rate through a low Layer 2 Rate or through large Overhead or MPU values, and can raise
it through a high Layer 2 Rate. Clients MAY limit the Overhead and MPU to a
plausible range and ignore a Layer 2 Rate Option outside that range, in which case they use the
Layer 3 rate. Clients SHOULD NOT use signaled rates as the sole criterion for selecting a server or
network interface.

Operators can use DHCPv6-Shield {{?RFC7610}} to reject server messages arriving from client-facing
ports. This does not protect against a compromised authorized server or an attacker elsewhere on
the server-to-client path. A client accepts rate information only as part of a valid DHCPv6
exchange and uses the restrictions specified in {{placement}}. As the rates also govern IPv4
traffic, protecting only the IPv4 configuration protocol does not protect them.

The security considerations of {{!RFC9445}} and {{!RFC7037}} apply to rates provisioned via RADIUS.
A server MUST ignore Rate Options conveyed in the RADIUS option unless they are received from a
relay agent that it trusts, for example over a path protected as described in {{?RFC8213}}.

As the rates are signaled to clients, they do not by themselves enforce a service limit; a forged
rate affects only the traffic management of the client that receives it.

# Privacy Considerations {#privacy}

The privacy considerations for DHCPv6 specified in {{?RFC7824}} apply to this document. Clients can
additionally use the anonymity profiles specified in {{?RFC7844}}.

The Rate Options convey the rates provisioned for a client, which typically reflect the service tier
of a subscriber. Because DHCPv6 messages are typically transmitted in cleartext, this information is
exposed to on-path observers, including relay agents. The rate information itself does not
identify a subscriber, but in combination with other information it can reveal commercial details
of the service contract. A client that makes the Layer 3 rates available to devices in the LAN (see
{{application}}) exposes this information to those devices.

# IANA Considerations {#iana}

RFC Editor: please replace TBD1, TBD2, TBD3, and TBD4 throughout this document with the values
assigned by IANA, replace "RFC XXXX" with the RFC number assigned to this document, and remove this
note.

IANA is requested to assign four new option codes in the "Option Codes" registry within the
"Dynamic Host Configuration Protocol for IPv6 (DHCPv6)" registry group {{IANA-DHCPV6}} as
follows:

| Value | Description                 | Client ORO | Singleton Option | Status | Reference |
| :---- | :-------------------------- | :--------- | :--------------- | :----- | :-------- |
| TBD1  | OPTION_UPSTREAM_RATE        | Yes        | Yes              |        | RFC XXXX  |
| TBD2  | OPTION_DOWNSTREAM_RATE      | Yes        | Yes              |        | RFC XXXX  |
| TBD3  | OPTION_UPSTREAM_L2_RATE     | Yes        | Yes              |        | RFC XXXX  |
| TBD4  | OPTION_DOWNSTREAM_L2_RATE   | Yes        | Yes              |        | RFC XXXX  |
{: #tab-iana title="DHCPv6 Option Codes"}

IANA is requested to add the following entries to the "DHCPv6 Options Permitted in the RADIUS
DHCPv6-Options Attribute" registry within the "Dynamic Host Configuration Protocol for IPv6
(DHCPv6)" registry group {{IANA-DHCPV6}}, as specified in {{Section 8.4.1 of !RFC9445}}:

| Value | Description                 | Reference |
| :---- | :-------------------------- | :-------- |
| TBD1  | OPTION_UPSTREAM_RATE        | RFC XXXX  |
| TBD2  | OPTION_DOWNSTREAM_RATE      | RFC XXXX  |
| TBD3  | OPTION_UPSTREAM_L2_RATE     | RFC XXXX  |
| TBD4  | OPTION_DOWNSTREAM_L2_RATE   | RFC XXXX  |
{: #tab-iana-radius title="DHCPv6 Options Permitted in the RADIUS DHCPv6-Options Attribute"}

This document does not create any new registries.

--- back

# Implementation Status {#impl-status}
{:removeinrfc="true"}

{::boilerplate rfc7942}

At the time of writing, no implementations of this document are known to the authors.

## Applicability to OpenWrt {#impl-openwrt}

Once option codes are assigned, the Rate Options can be supported on OpenWrt {{OPENWRT}} without
changes to its DHCPv6 client, using the following existing components:

Requesting the options:
: The netifd DHCPv6 protocol handler accepts additional option codes in the `reqopts` list of the
  interface configuration and passes them to the DHCPv6 client odhcp6c, which includes them in the
  ORO of its requests.

Receiving the options:
: odhcp6c passes options that it does not process itself to its status script as environment
  variables named `OPTION_<code>`, containing the hex-encoded option value. The status script
  (`/lib/netifd/dhcpv6.script`) invokes user scripts in `/etc/odhcp6c.user.d/` on each state
  change, such as bound, updated, rebound, informed, unbound, and stopped. A variable that is absent
  in an update corresponds to the withdrawal of the rate, and the unbound and stopped events
  correspond to the discarding of the client's configuration (see {{client}}).

Applying the rates:
: A user script converts the rates from bit/s to the kbit/s used by the Smart Queue Management (SQM)
  package {{SQM}} and updates the upload and download rates of the SQM instance on the WAN
  interface, which applies them using CAKE or fq_codel. On a PPPoE WAN interface, the shaper
  typically counts IP packets, so the rates apply directly; on Ethernet-based WAN interfaces, the
  SQM overhead settings provide the header compensation described in {{layer3}}. Where a
  Layer 2 Rate Option is received, the script instead sets the rate to the Layer 2 Rate and passes
  the Overhead and MPU to the per-packet overhead and minimum packet size settings of SQM, after
  subtracting any link-layer octets that the shaper already counts (see {{layer2}}).

Shell arithmetic on OpenWrt uses signed 64-bit integers, so a script-based implementation needs to
handle values above 2^63 - 1, for example by treating them as exceeding any local limit. A native
implementation could instead decode the options in odhcp6c and expose the rates as interface data
via netifd, which also makes them available to other components, such as the web interface.

# Design Rationale {#rationale}
{:removeinrfc="true"}

This appendix explains major design decisions to the working group. Permanent rationale is
included in the relevant sections of this document.

## Why Not Solve This in the Network? {#rationale-network}

Reviewers asked why the CPE needs to know its rates when the network already enforces them, and
whether the operator could instead shape or apply AQM on its own equipment.

- Enforcement remains in the network. The network does not depend on the CPE: the policer or shaper
  at the enforcement point stays in place, and a CPE that ignores or misapplies the signaled rates
  affects only its own traffic. The signaled rates let the CPE improve the service of its
  subscriber; they are not a means of enforcement.
- A policer has no queue to which AQM or marking could be applied. Replacing upstream policers by
  per-subscriber shapers with AQM at the BNG is limited by platform support and scale (see
  {{problem-statement}}).
- Prioritization has to take place before the bottleneck. The policy for prioritizing the traffic
  of a household, for example between devices and applications, is configured at the CPE. A queue
  in the network can apply AQM, but not that policy.
- Where the bottleneck is the access line or an access device, the queue forms before the traffic
  reaches the BNG, in a modem, ONT, or access node that is typically managed by neither AQM nor
  prioritization. Only the sender in front of that bottleneck, the CPE, can keep this queue short.
- Management protocols such as TR-069 can configure rates on operator-managed CPE, but they are not
  deployed by all operators and do not reach customer-owned CPE. Manual configuration requires the
  subscriber to know the rates and to update them when the service changes. DHCPv6 is already used
  by these devices and can update the rates during a session.

## DHCPv6 Only {#rationale-dhcpv6}

Earlier versions of this document defined the Rate Option for both DHCPv4 and DHCPv6. This version
defines options for DHCPv6 only; the permanent rationale is given in {{applicability}}.

- Two protocols require per-protocol state on every client. In DHCPv6, a requested option that is
  missing from a Reply revokes the information ({{Section 18.2.10.5 of !RFC9915}}). If the same
  rates are also signaled in DHCPv4, a client that loses the rate in one protocol, through
  omission, lease expiry, or release, has to fall back to the rate of the other protocol if that
  is still valid. It therefore has to retain the rates of each protocol together with their
  validity. Version -00 required this explicitly.
- Simpler selection rules do not avoid this state. Rules such as "use the most recently received
  value" or "use the larger (or the lower) value" were proposed. They resolve conflicting values, but not
  withdrawal and lifetime: the client still has to know the value of the other protocol when one
  of them goes away.
- DHCPv4 and DHCPv6 are commonly handled by separate client programs with separate hook scripts,
  so that shared state has to be built by the implementer. DHCPv4 also has no general rule that a
  missing option revokes earlier information, so the two protocols would behave differently.
- With DHCPv6 only, the options are processed as specified in {{!RFC9915}}, and a client can
  implement them with a single hook script (see {{impl-openwrt}}).
- IPv4 traffic remains covered. The rates describe the service on the logical interface, not an IP
  version, and apply to the combined IPv4 and IPv6 traffic, whether IPv4 is provided natively,
  configured by DHCPv4 or IPCP, or carried over IPv6 (see {{dual-stack}}).
- DHCPv6 is available where the mechanism is used. Broadband CPE commonly use DHCPv6 for prefix
  delegation in both IPoE and PPPoE deployments.
- The DHCPv6 option space allows a simpler format. DHCPv4 option codes are scarce, which motivated
  a single container option with sub-options in -00. With DHCPv6, each direction has its own
  fixed-format option, no sub-option registry is needed, and a rate is withdrawn by omission
  instead of a reserved zero value.
- The cost is that access networks that do not use DHCPv6 on the subscriber link, such as
  IPv4-only access networks, cannot use the options. The number of such networks is decreasing:
  most of them depend on carrier-grade NAT (CGNAT), the number of flows per subscriber keeps
  growing, and operators deploy IPv6 to reduce the load on the CGNAT.
- This limitation can be removed later, whereas the reverse is not possible. A DHCPv4 option can be
  defined in a separate document that reuses the semantics specified in {{semantics}} and
  specifies the interaction between the two protocols. Once a DHCPv4 option is published, every
  client has to implement that interaction.

The authors invite feedback from the working group on whether a DHCPv4 option is needed.

## Layer 3 and Layer 2 Rates {#rationale-layer3}

This topic is an open issue. The text below describes the proposal of the authors in this version;
the working group has not yet been asked to confirm it.

The accounting reference of the signaled rates has been discussed at length. Version -00 carried a
Rate Type sub-option that selected between a Layer 2 rate, which was the default, and a Layer 3
rate; it did not signal per-packet overhead. Reviewers pointed out that a Layer 2 rate is not well
defined without the per-packet overhead of the bottleneck, and that a shaper at the CPE keeps
control of the queue only if it accounts for at least the overhead of the actual bottleneck.
Others, including the authors, consider a Layer 3 rate sufficient for most deployments and
essential for simple implementations.

This version keeps Layer 3 rates as the mandatory baseline (see {{layer3}}) and adds optional
Layer 2 Rate Options (see {{layer2}}), for the following reasons:

- A Layer 3 rate has an unambiguous boundary: the IP header and its payload. A Layer 2 rate needs
  further qualification, such as whether VLAN tags, the frame check sequence, the preamble, and the
  inter-packet gap are counted, and the encapsulation changes between the CPE and the enforcement
  point.
- Layer 3 rates match deployed practice. Existing proprietary mechanisms that signal rates to CPE
  within PPP authentication messages use Layer 3 rates by default. A CPE reads one value per
  direction and applies it, with local header compensation where its shaper counts frames.
- A Layer 3 rate is exact only where the enforcement point counts IP packet sizes. Where it counts
  frames or is a physical link, the IP rate depends on the packet size, and the per-packet overhead
  is difficult to determine at the CPE but known to the operator.
- The overhead cannot be added to the Layer 3 Rate Options as a third value. The rate that belongs
  to an overhead is the rate counted by the enforcement point, which is higher than the Layer 3
  rate. A client that ignored the overhead would apply that higher rate to IP packets and exceed the
  enforcement point. The Layer 2 Rate Options therefore carry their own rate, and the Layer 3 Rate
  Options remain safe for clients that ignore the Layer 2 Rate Options.
- The two kinds of rate serve different purposes. A Layer 3 rate describes the service and remains
  meaningful beyond the link, for example for display or for devices in the LAN. Layer 2 values
  describe the accounting of one link, are used only for traffic management on that link, and are
  not passed on. A client can use both at the
  same time, which a single option with a layer selector would not allow.
- The Overhead is relative to the IP packet, not to a Layer 2 frame. This avoids defining a Layer 2
  reference and makes the values independent of the encapsulation and the accounting of the
  client. As an enforcement point counts at least the IP packet, the Overhead is unsigned.
  Enforcement points that count less than the IP packet on the reference link, for example after
  decapsulation or translation, are not covered.
- The Layer 3 Rate Options are mandatory because the Layer 3 rate is needed in every deployment:
  it is the value for display and for devices in the LAN, and the only value that every client can
  apply. The Layer 2 Rate Options are optional because they refine one use, traffic management on
  the link, and only where the enforcement point does not count at the IP layer.
- The server always sends the Layer 3 Rate Option together with a Layer 2 Rate Option, so that
  every client has a usable value. A client that can apply the Layer 2 Rate Option prefers it, as the
  server sends it only where it is more precise.
- The format is fixed, with one option per direction, so that the overhead can differ between the
  directions. Cell-based accounting, as needed for ATM, and hints sent by the client, as in -00, are not
  included.

The following alternatives were considered:

- Layer 3 rates only, with Layer 2 rates defined in a separate document. This is the smallest
  specification, but it does not describe enforcement points that count frames, and a separate
  document might not be implemented.
- A selector indicating the layer of a single rate, as in -00. Every client would need to support
  both layers, and the Layer 2 reference would remain ambiguous.
- A single overhead option that applies to both directions and to the rates of the Layer 3 Rate
  Options. As explained above, a client that ignored it would exceed the enforcement point.
- A variable-length option in which the overhead fields are optional. A rate without overhead is
  already provided by the Layer 3 Rate Options, and a fixed format is simpler to implement and
  validate.

The authors invite feedback from the working group on the following questions:

- Should the Layer 2 Rate Options remain in this document, move to a separate document, or be
  removed?
- Are the Overhead and the MPU sufficient? Is the MPU needed, and is cell-based accounting needed?
- Is the rule that a client prefers the Layer 2 Rate Option over the Layer 3 Rate Option
  appropriate?

## Burst Tolerance {#rationale-burst}

This topic is an open issue. A policer admits bursts only up to its configured burst size, and
access devices have limited buffers. A shaper at the CPE that releases larger bursts can therefore
cause loss even when it shapes to the signaled rate. This version does not signal a burst size.
The authors invite feedback on whether a burst size or burst duration needs to be signaled, and
whether it belongs in the Layer 2 Rate Options or in a separate option.

## Relay-Directed Rates {#rationale-relay}

Version -00 and intermediate versions of this document allowed the server to signal separate rates
to relay agents in Relay-reply messages, so that, for example, a BNG or an access node operating as
a Lightweight DHCPv6 Relay Agent (LDRA) could configure per-subscriber shapers and policers with
values that differ from those signaled to the client. This mechanism has been removed from this
document for the following reasons:

- Server-to-relay configuration without a request by the relay agent is a new pattern for DHCPv6
  and would require relay agents to maintain per-client state, contrary to the stateless relay
  agent model.
- Withdrawal by omission and the loss of state after a relay agent restart would remove
  enforcement until the client's next exchange, and updates would depend on client renewal timing.
- The server has no indication of whether a relay agent applied the rates, and the mapping of
  Relay-reply encapsulation levels to devices depends on the relay topology.
- BNGs commonly obtain per-subscriber policies via AAA, which also supports updates during a
  session.

Signaling of rates to relay agents can be specified in a separate document if the working group
identifies a need. The core mechanism defined here, rates signaled to clients, is independent of
such an extension.

# Change Log
{:removeinrfc="true"}

## Since draft-ietf-intarea-dhcp-rate-signaling-00

- Changed the intended status to Standards Track; DHCPv6 base reference is now RFC 9915.
- Limited the scope to DHCPv6 and to rates signaled to clients in broadband access networks.
  Removed the DHCPv4 option, DHCPv6 snooping, rates signaled to relay agents, and client hints
  (see {{rationale}}).
- Replaced the sub-option format with two fixed-format options, Upstream Rate and Downstream Rate,
  each an unsigned 64-bit value in bit/s. The value 0 is invalid; omission withdraws a rate. Not
  backward compatible with -00.
- Defined the rates of these options as Layer 3 rates for the combined IPv4 and IPv6 traffic of the
  DHCPv6-configured link, with local header compensation. Layer 3 replaces Layer 2 as the default,
  and the Rate Type sub-option is removed.
- Added the optional Upstream Layer 2 Rate and Downstream Layer 2 Rate options, which carry
  the rate, per-packet overhead, and minimum packet size as counted by the enforcement point. This
  part is an open issue (see {{rationale-layer3}}).
- Made the use of the rates a local decision and aligned client processing with RFC 9915,
  independent of the exchange that supplied a rate.
- Allowed a client to make the Layer 3 rates available to devices in the LAN, without defining
  the mechanism; Layer 2 values are not passed on.
- Added optional provisioning via RADIUS (RFC 9445, RFC 7037) and the corresponding IANA
  registration.
- Condensed operational, security, privacy, and IANA considerations; removed the implementation
  example; added OpenWrt applicability and a design rationale appendix.

# Acknowledgments
{:numbered="false"}

The authors would like to thank Sebastian Moeller for the extensive discussion of per-packet
overhead and Layer 2 accounting, which led to the Layer 2 Rate Options; Mikael Abrahamsson for
raising the question of what the signaled rate measures; Bernie Volz for guidance on the DHCPv6
option format; Alan DeKok and Ian Farrer for input on RADIUS provisioning; Juliusz Chroboczek for
input on applicability; and Glenn Deen and Jason Livingood for their review comments and discussion.