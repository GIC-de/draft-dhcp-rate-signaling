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
   IEEE802.1Q:
      title: "IEEE Standard for Local and Metropolitan Area Networks--Bridges and Bridged Networks"
      author:
      - org: IEEE
      date: 2022
      seriesinfo:
         IEEE Std: 802.1Q-2022
         DOI: 10.1109/IEEESTD.2022.10004498
   IEEE802.3:
      title: "IEEE Standard for Ethernet"
      author:
      - org: IEEE
      date: 2022
      seriesinfo:
         IEEE Std: 802.3-2022
         DOI: 10.1109/IEEESTD.2022.9844436

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

This document defines new Dynamic Host Configuration Protocol (DHCP) options for both DHCPv4 and
DHCPv6 to signal available upstream and downstream data rates to clients and intermediate nodes. In
many broadband access networks, the provisioned upstream rate is enforced at the service edge by
ingress policers, while the Customer Premises Equipment (CPE) sends at the speed of its physical
link without knowledge of this rate. The resulting policer drops cause unstable and reduced
throughput and render upstream prioritization at the CPE ineffective. By learning the provisioned
rates via DHCP, the CPE can shape its upstream traffic to these rates, moving the queue to the CPE,
where it can be managed by prioritization, Active Queue Management (AQM), and the Low Latency, Low
Loss, and Scalable Throughput (L4S) architecture, while enforcement at the service edge remains in
place. Intermediate nodes acting as DHCP relay agents or performing DHCP snooping can use the same
information to instantiate per-subscriber shapers and policers, without requiring a separate
provisioning interface. The options are also applicable within local area networks, for example to
inform end devices about the capacity of the upstream connection.

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
the CPE. For this purpose, the option also allows the Advertised Rates to be signaled separately
(see {{advertised}}).

A similar lack of visibility exists within local area networks (LANs), such as home, enterprise,
or public Wi-Fi hotspot networks, where end devices are typically connected at link speeds that
exceed the capacity of the upstream connection (see {{lan}}).

## Solution Overview {#solution-overview}

This document defines new DHCP options for both DHCPv4 and DHCPv6 to signal the available upstream
and downstream rates. If the CPE knows the available upstream rate, it can shape its upstream
traffic to this rate, regardless of where the bottleneck is located. The queue then builds at the
CPE, where it can be managed using prioritization, Active Queue Management (AQM) {{?RFC7567}}, and
flow queuing {{?RFC8290}}. Under normal operation, the policer at the BNG is not triggered, and the
access device is not congested (see {{fig-policer}}). The resulting upstream throughput is stable
and close to the available rate, and latency under load remains low. When combined with the Low
Latency, Low Loss, and Scalable Throughput (L4S) architecture {{?RFC9330}}, the queue at the CPE can
additionally provide low latency for L4S traffic.

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

The same rate information can be used by intermediate nodes. In many IP over Ethernet (IPoE)
{{TR101}} architectures, the BNG operates as a DHCP relay agent. While per-subscriber traffic
management policies are typically provisioned via Authentication, Authorization, and Accounting
(AAA) protocols such as RADIUS {{?RFC2865}}, deployments lacking AAA integration at the service
edge require an in-band alternative. The BNG can use the signaled rates to instantiate downstream
shapers and upstream policers, and access nodes performing DHCP snooping can use them to
configure their local queues and shapers. DHCPv6 additionally allows the server to signal
different rates to the client and to each relay agent in the path (see {{dhcpv6-server}}).

Where the actual rate of the access line is lower than the provisioned rate, for example due to the
synchronization rate of a DSL line, the server can signal the lower of the two rates. The actual
rate of the access line is typically known at the BNG, for example from access loop characteristics
reported by the access node {{TR101}}, such as the Actual-Data-Rate-Upstream and
Actual-Data-Rate-Downstream attributes {{?RFC4679}} or the corresponding Access Node Control
Protocol (ANCP) TLVs {{?RFC6320}} (see {{access-loop}}).

## Applicability

The mechanism defined in this document is primarily designed for broadband access networks using
IPoE or Point-to-Point Protocol over Ethernet (PPPoE) {{?RFC2516}}. In PPPoE deployments, DHCPv6 is
commonly used over the PPP session for IPv6 Prefix Delegation and conveys the options in the same
way (see {{pppoe}}). The options are defined for both DHCPv4 and DHCPv6 to support IPv4-only,
IPv6-only, and dual-stack deployments. A minimal implementation, which supports only the upstream
rate and its Applied Rate Layer, is described in {{minimal}}.

Unlike auto-configuration protocols such as TR-069 {{TR069}}, which are not deployed by all
operators and are typically not available for customer-owned CPE, DHCP is supported by virtually
all CPE, including retail routers and devices running open-source firmware (e.g., OpenWrt), and
it is visible to intermediate nodes along the path. Rates conveyed via DHCP are refreshed with
every lease renewal and can be updated dynamically. The options are also applicable within LANs
(see {{lan}}).

{{rationale}} discusses the reasoning behind this mechanism and addresses frequently raised
questions, such as why rate enforcement at the BNG alone is not sufficient and why upstream
shaping is not performed at the BNG.

# Conventions and Definitions {#definitions}

{::boilerplate bcp14-tagged}

This document makes use of the terms defined in {{!RFC2131}} and {{!RFC9915}}.
The following additional terms are used:

Advertised Rate
: The rate communicated to the subscriber, for example the rate of the subscribed service as
  specified in the service agreement. The Advertised Rate can differ from the provisioned rate.

Applied Rate
: The rate signaled by the Applied Rate sub-options, which is intended to be applied to traffic
  management functions. It is typically derived from the provisioned rate.

DHCP
: The abbreviation DHCP is used throughout this document to refer to both DHCPv4 and DHCPv6
  protocols.

DHCPv4
: The Dynamic Host Configuration Protocol {{!RFC2131}}.

DHCPv6
: The Dynamic Host Configuration Protocol for IPv6 {{!RFC9915}}.

Downstream
: The direction of traffic from the network towards the client. For a client, this corresponds to
  traffic received over the link on which the DHCP Rate Option was received. For a relay agent or
  snooping device, this corresponds to traffic sent on the client-facing interface or port.

Enforcement Point
: The node or link at which the signaled rates are enforced or which constitutes the bottleneck
  of the path, for example a policer at the BNG or an access link.

Intermediate Node
: A relay agent or snooping device on the path between a client and a server that processes the DHCP
  Rate Option, for example a BNG acting as a DHCP relay agent or an access node performing DHCP
  snooping.

Provisioned Rate
: The rate configured for a subscriber according to the subscribed service, typically enforced
  at the enforcement point (e.g., the BNG). The rates signaled by the DHCP Rate Option are
  typically derived from the provisioned rates.

Receiver
: Any node that processes a received DHCP Rate Option, including DHCP clients, relay agents
  (including Lightweight DHCPv6 Relay Agents {{?RFC6221}} and Layer 2 DHCPv4 Relay Agents
  {{TR101}}), and DHCP snooping devices.

Subscriber
: The customer of a network access service. Within the network, a subscriber is typically
  represented by a subscriber session at the BNG (e.g., an IPoE or PPPoE session), to which the
  provisioned rates and other per-subscriber policies are applied.

Upstream
: The direction of traffic from the client towards the network. For a client, this corresponds to
  traffic sent over the link on which the DHCP Rate Option was received. For a relay agent or
  snooping device, this corresponds to traffic received on the client-facing interface or port.

# DHCP Rate Option

The DHCP Rate Option (OPTION_RATE) specified in this document is a container option with a
sub-option structure for both DHCPv4 and DHCPv6. This document uses OPTION_RATE to refer to both the
DHCPv4 option (OPTION_V4_RATE) and the DHCPv6 option (OPTION_V6_RATE). The top-level option
encapsulation conforms to the requirements of each base protocol. Specifically, DHCPv4 uses an 8-bit
option code set to TBD1 alongside an 8-bit length field, whereas DHCPv6 uses a 16-bit option code
set to TBD2 alongside a 16-bit length field.

While the underlying semantics of the rate information remain identical across both IP versions, the
internal payload sizing differs to align with the conventions of each respective protocol. For
DHCPv4, the encapsulated sub-options use an 8-bit sub-option code and an 8-bit sub-option length
field, mirroring the format of DHCPv4 Option 82 (Relay Agent Information Option) {{?RFC3046}}.
Conversely, for DHCPv6, the encapsulated sub-options use a 16-bit sub-option code and a 16-bit
sub-option length field.

Section 9 of {{?RFC7227}} favors encapsulated options, which use the DHCPv6 option code space, over
sub-options with an option-specific code space. This document nevertheless uses sub-options with a
code space shared by DHCPv4 and DHCPv6, so that a single registry and identical semantics apply to
both protocols. This simplifies implementations supporting both protocols and ensures consistent
processing in dual-stack deployments (see {{processing}}). The sub-options are defined on a single
level only and, for those defined in this document, have a fixed length.

## Sub-Options Format

The value of OPTION_RATE consists of a sequence of sub-option (SubOpt) Code/Length/Value tuples.
Because the field sizing differs between the two protocols to align with their respective base
specifications, the sub-options are encoded as follows:

DHCPv4 Sub-Option Format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  SubOpt Code  | SubOpt Length | SubOpt Value                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

DHCPv6 Sub-Option Format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          SubOpt Code          |         SubOpt Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         SubOpt Value                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

No "pad" or "end" sub-option is defined. Sub-option codes 0 and 255 are reserved and MUST NOT be
sent. The length of OPTION_RATE MUST include all octets of the sub-option code/length/value tuples.
The sub-option length MUST equal the number of octets in the sub-option's value field. The
sub-options need not appear in sub-option code order.

Each sub-option defined in this document has a fixed length, as listed in {{sub-option-table}}.
Sub-options defined in the future specify their own length requirements, which can include a length
of zero.

DHCPv4 and DHCPv6 share a single sub-option code space, which is limited to the values 0-255
(see {{iana-sub-option-table}}). Although DHCPv6 encodes the sub-option code in 16 bits, DHCPv6
sub-option codes 256-65535 MUST NOT be sent. Receivers treat reserved sub-option codes and DHCPv6
sub-option codes above 255 as unrecognized sub-option codes (see {{errors}}).

The initial assignment of sub-options is as follows:

| Code  | Length | Description                    |
|------:| ------:| ------------------------------ |
|     0 |      - | Reserved                       |
|     1 |      8 | Advertised Rate Upstream       |
|     2 |      8 | Advertised Rate Downstream     |
|     3 |      8 | Applied Rate Upstream          |
|     4 |      8 | Applied Rate Downstream        |
|     5 |      1 | Applied Rate Layer             |
|     6 |      1 | Applied Rate Layer Downstream  |
|     7 |      6 | Per-Packet Overhead            |
|     8 |      6 | Per-Packet Overhead Downstream |
| 9-254 |      - | Unassigned                     |
|   255 |      - | Reserved                       |
{: #sub-option-table title="DHCP Rate Sub-Options"}

## Sub-Options

### Advertised Rates {#advertised}

The Advertised Rate Upstream (code 1) and Advertised Rate Downstream (code 2) sub-options convey the
Advertised Rates (see {{definitions}}), for example the rates of the subscribed service as specified
in the service agreement, which are typically specified as payload rates. In local area networks,
they can convey the nominal capacity available to the clients (see {{lan}}). Their format is
identical to the format of the Applied Rate sub-options (see {{rate-upstream}}), using the
sub-option codes 1 and 2 and a sub-option length of 8. The rate is a 64-bit unsigned integer in bits
per second (bps) in network byte order.

Advertised Rates are informational only. Receivers SHOULD NOT apply them to traffic management
functions, unless explicitly configured otherwise by local policy. They are not subject to the
Applied Rate Layer or the Per-Packet Overhead sub-options. The layer to which an Advertised Rate
refers is not specified; it reflects the rate as communicated to the subscriber. Receivers can use
them, for example, to populate user interfaces, for telemetry, as an initial estimate for
applications, or as a metric to select a path with higher throughput. A value of 0, or the absence
of the sub-option, indicates that no Advertised Rate is provided for the respective direction. Relay
agents and snooping devices do not use Advertised Rates for their traffic management.

Advertised Rates can differ from the Applied Rates. For example, an operator can signal an Applied
Rate accounted at Layer 2, configured such that the payload rate achieved with maximum-sized packets
meets the contracted rate, together with the contracted rate as the Advertised Rate (see
{{operations}}).

### Applied Rate Upstream {#rate-upstream}

The Applied Rate Upstream sub-option defines the rate in bits per second (bps) that is intended to
be applied in the upstream direction (see {{application}}). The rate format is a 64-bit unsigned
integer in network byte order. This allows rates beyond the approximately 4.29 Gbit/s that can be
represented by 32-bit rate attributes in bits per second, such as those defined in {{?RFC4679}}.

DHCPv4 Sub-Option Format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| SubOpt Code=3 | SubOpt Len=8  |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
| Applied Rate Upstream                                         |
+                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

DHCPv6 Sub-Option Format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| SubOpt Code=3                 | SubOpt Len=8                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Applied Rate Upstream                                         |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

A value of 0 indicates that no rate is signaled for the upstream direction. A receiver that has
previously applied an upstream rate learned from the DHCP Rate Option MUST remove that rate and
revert to its default configuration for the upstream direction.

### Applied Rate Downstream {#rate-downstream}

The Applied Rate Downstream sub-option defines the rate in bits per second (bps) that is intended to
be applied in the downstream direction (see {{application}}). The rate format is a 64-bit unsigned
integer in network byte order.

DHCPv4 Sub-Option Format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| SubOpt Code=4 | SubOpt Len=8  |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
| Applied Rate Downstream                                       |
+                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

DHCPv6 Sub-Option Format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| SubOpt Code=4                 | SubOpt Len=8                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Applied Rate Downstream                                       |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

A value of 0 indicates that no rate is signaled for the downstream direction. A receiver that has
previously applied a downstream rate learned from the DHCP Rate Option MUST remove that rate and
revert to its default configuration for the downstream direction.

### Applied Rate Layer {#rate-layer}

The Applied Rate Layer indicates the layer at which the enforcement point accounts traffic, and
thereby the layer to which the Applied Rates apply, so that receivers can model this accounting.

Two sub-options with identical format and semantics are defined. The Applied Rate Layer sub-option
(code 5) applies to both directions. If the Applied Rate Layer Downstream sub-option (code 6) is
present, it overrides the Applied Rate Layer sub-option for the downstream direction. This allows a
server, for example, to signal rates accounted at different layers in each direction. Throughout
this document, the Applied Rate Layer of a direction refers to the Applied Rate Layer applicable to
that direction.

The format is shown for the Applied Rate Layer sub-option. The Applied Rate Layer Downstream
sub-option uses the same format with a sub-option code of 6.

DHCPv4 Sub-Option Format:

~~~ aasvg
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| SubOpt Code=5 | SubOpt Len=1  |     Layer     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

DHCPv6 Sub-Option Format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| SubOpt Code=5                 | SubOpt Len=1                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Layer     |
+-+-+-+-+-+-+-+-+
~~~

The Applied Rate Layer sub-options carry the number of the layer at which the Applied Rates are
accounted, as follows:

| Value | Description                |
|------:| -------------------------- |
|     2 | Layer 2 rate               |
|     3 | Layer 3 rate               |
{: #rate-layer-table title="Applied Rate Layer Values"}

A Layer 2 rate encompasses the Ethernet frame as transmitted on the link over which the receiver is
connected to the network, including the Ethernet header, all Virtual LAN (VLAN) tags {{IEEE802.1Q}}
present on that link, and the complete Ethernet payload, including any encapsulation headers carried
within it (e.g., PPPoE and PPP headers). It excludes the Ethernet Frame Check Sequence (FCS),
preamble, start frame delimiter (SFD), and inter-packet gap (IPG) {{IEEE802.3}}. For link layers
other than Ethernet, the Layer 2 rate is calculated analogously over the link-layer header and
payload, excluding link-layer trailers and physical-layer framing overhead.

A Layer 3 rate encompasses only the IP header and its payload. This rate calculation is frequently
used by end-user speed test applications and is often regarded as the marketable "product
bandwidth". Its primary advantage is its independence from the variable overhead introduced by
differing numbers of VLAN tags or tunnel encapsulations, which makes it suitable for comparison with
end-user measurements.

The Layer 2 rate refers to the frames on the link of the receiver, independent of the logical
interface on which the receiver applies its traffic management functions. A receiver that applies
traffic management functions on a logical interface on which the frames differ from those on the
link, such as a VLAN interface whose tags are added by an underlying device, or a PPP interface,
accounts for the difference locally, for example by adjusting the per-packet overhead of its traffic
management functions. The per-packet overhead can also differ between the link of the receiver and
the enforcement point, for example when an access node adds or removes a VLAN tag. Servers can
signal such differences using the Per-Packet Overhead sub-options (see {{overhead}}) or, in DHCPv6,
by signaling node-specific rates (see {{dhcpv6-server}}).

For a direction for which no Applied Rate Layer sub-option applies, a receiver MUST interpret the
Applied Rate as a Layer 3 rate. The Layer 3 default corresponds to the rate measured by end-user
speed tests and to existing deployments of proprietary rate signaling mechanisms.

Values other than 2 and 3 are invalid, and no registry is defined for this field. Other accounting
models can be defined as additional sub-options (see {{iana-sub-option-table}}). If a receiver
receives an Applied Rate Layer sub-option with an invalid value, it does not apply the Applied Rate
of that direction and treats it as information only (see {{application}}). The remaining
sub-options are processed as usual (see {{errors}}).

Servers SHOULD signal the Applied Rates at the layer at which the enforcement point accounts
traffic and SHOULD include the Per-Packet Overhead sub-options whenever the accounting of the
enforcement point is known, including when all of their values are 0. A Layer 2 rate together with
the per-packet overhead allows the most precise modeling of a bottleneck that accounts traffic at
Layer 2. A Layer 3 rate is useful where the enforcement point itself accounts traffic at Layer 3, or
where the effective rate is intended to match the rates measured by end-user speed tests.

A receiver whose traffic management functions account traffic at Layer 3 can apply a Layer 3 rate
directly. The effective rate then closely corresponds to the rate measured by end-user speed tests
and is identical on every node, independent of the encapsulation used on the link of the receiver or
on any other segment of the path.

The traffic management functions of a receiver do not necessarily account traffic at the layer
indicated by the Applied Rate Layer. For example, shapers commonly account for the Layer 2 frame
length, while the signaled rate can be a Layer 3 rate. A receiver whose traffic management functions
support per-packet overhead accounting compensates for the difference between its own accounting and
the accounting of the enforcement point, as indicated by the Applied Rate Layer and the Per-Packet
Overhead sub-options (see {{overhead}}). For a Layer 3 rate, this includes the headers between Layer
3 and its own accounting layer, such as Ethernet, VLAN, and PPPoE headers, which are known locally,
for example by configuring a negative overhead, so that the effective rate corresponds to the
signaled Layer 3 rate. A receiver without such support applies the signaled rate directly or
approximates the difference, for example by reducing the rate by a margin.

A client MAY indicate its preferred layer as a hint (see {{hints}}).

### Per-Packet Overhead {#overhead}

The Per-Packet Overhead sub-options describe the size that the enforcement point of the signaled
rates accounts for each packet, relative to the size of the packet at the layer indicated by the
Applied Rate Layer of the respective direction, on the link of the receiver. They allow a receiver
to configure its traffic management functions such that they accurately model the enforcement point,
independent of differences in encapsulation between the receiver and the enforcement point, such as
additional VLAN tags or cell-based link layers.

Two sub-options with identical format and semantics are defined. The Per-Packet Overhead sub-option
(code 7) applies to both directions. If the Per-Packet Overhead Downstream sub-option (code 8) is
present, it overrides the Per-Packet Overhead sub-option for the downstream direction. As the
per-packet overhead is typically identical in both directions, and clients typically apply only the
upstream rate (see {{application}}), servers usually include only the Per-Packet Overhead
sub-option. As the Overhead is relative to the layer indicated by the Applied Rate Layer of the
respective direction, a server that signals different Applied Rate Layers for the two directions
includes a Per-Packet Overhead Downstream sub-option where the Overhead differs as a result.

The format is shown for the Per-Packet Overhead sub-option. The Per-Packet Overhead Downstream
sub-option uses the same format with a sub-option code of 8.

DHCPv4 Sub-Option Format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| SubOpt Code=7 | SubOpt Len=6  |           Overhead            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Minimum Packet Unit (MPU)  | Cell Payload  |   Cell Size   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

DHCPv6 Sub-Option Format:

~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| SubOpt Code=7                 | SubOpt Len=6                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Overhead            |    Minimum Packet Unit (MPU)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Cell Payload  |   Cell Size   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

Overhead:
: A 16-bit signed integer in two's complement and network byte order, indicating the number of
  octets the enforcement point accounts per packet in addition to the size of the packet at the
  layer indicated by the Applied Rate Layer of the respective direction, on the link of the
  receiver. The value can be negative, for example if the enforcement point does not account for
  headers that are present on the link of the receiver. Where the bottleneck is a physical link
  rather than a policer, the Overhead includes physical-layer overhead that consumes transmission
  time, such as the Ethernet preamble, SFD, and IPG.

Minimum Packet Unit (MPU):
: A 16-bit unsigned integer in network byte order, indicating the minimum size in octets that
  the enforcement point accounts per packet, after the Overhead has been added. A value of 0
  indicates that no minimum applies.

Cell Payload:
: An 8-bit unsigned integer indicating the number of octets of packet data carried per cell for
  link layers that transport packets in fixed-size cells. A value of 0 indicates that no
  cell-based accounting applies.

Cell Size:
: An 8-bit unsigned integer indicating the number of octets accounted per cell, including the
  cell header. If the Cell Payload is 0, the Cell Size MUST be set to 0 by the sender and is
  ignored by the receiver.

For a packet with a size of S octets at the layer indicated by the Applied Rate Layer of the
respective direction, the size L accounted by the enforcement point is calculated as follows:

1. L = max(S + Overhead, MPU)
2. If the Cell Payload is not 0: L = ceil(L / Cell Payload) * Cell Size

Cell-based accounting describes link layers that pad each packet to an integer number of cells. The
most common example is Asynchronous Transfer Mode (ATM) Adaptation Layer 5 (AAL5), as used for
example on Asymmetric Digital Subscriber Line (ADSL), ADSL2, and ADSL2+ lines, which is described by
a Cell Payload of 48 and a Cell Size of 53. In this case, the Overhead includes all encapsulation
overhead carried within the AAL5 frame, including the 8-octet AAL5 trailer, and the signaled rate
corresponds to the rate of ATM cells, for example the synchronization rate of the ADSL line. A Cell
Size smaller than the Cell Payload is invalid; a receiver that receives such values ignores the
cell-based accounting. Accounting models that cannot be described by these parameters can be defined
as additional sub-options (see {{iana-sub-option-table}}). Examples of parameter values for common
deployments are given in {{examples}}.

The Per-Packet Overhead sub-options are optional and supplement the signaled rates to improve the
accuracy of traffic management. A receiver that applies the signaled rates and whose traffic
management functions support per-packet overhead accounting SHOULD configure them such that the rate
of the traffic in the respective direction, calculated over the accounted sizes L, does not exceed
the signaled rate for that direction.

A receiver whose traffic management functions do not support per-packet overhead accounting, or
support it only partially (e.g., without cell-based accounting or only for a limited range of
values), MAY ignore the sub-options or those parts of them that it cannot apply, and applies the
signaled rates as specified in {{rate-layer}}. Such a receiver MAY reduce the rate it applies by a
margin to approximate the accounting of the enforcement point. The inability of a receiver to apply
the Per-Packet Overhead sub-options, fully or partially, does not render the OPTION_RATE invalid.

For a direction for which no Per-Packet Overhead sub-option applies, the server does not provide
information about the accounting of the enforcement point. In this case, the receiver uses its
local configuration, if any, or otherwise assumes an Overhead of 0, an MPU of 0, and no cell-based
accounting. If a Per-Packet Overhead sub-option applies to a direction, its values take precedence
over the default values of the receiver for that direction, including values of 0, which indicate
that the enforcement point does not account any additional overhead. Explicit configuration by the
administrator of the receiver takes precedence over the signaled values (see {{application}}).

## Traffic Management Using Applied Rates {#application}

The Applied Rate sub-options convey rates that are intended to be applied to local traffic
management functions, such as traffic shapers, policers, or AQM parameters, at the layer indicated
by the Applied Rate Layer (see {{rate-layer}}). A client that supports such traffic management
functions SHOULD apply the upstream rate and MAY apply the downstream rate. Relay agents and
snooping devices MAY apply the rates in either direction (see {{dhcpv4}}, {{dhcpv6}}, and
{{snooping}}).

From the perspective of a client, the upstream rate is the relevant one: downstream traffic is
typically already shaped by the network before it reaches the client, whereas upstream traffic is
enforced by the network only after it has left the client. A client therefore typically applies only
the upstream rate to its traffic management functions, which is why applying the downstream rate is
optional for clients. Where the downstream shaper at the BNG does not apply AQM, a client can
additionally shape received traffic slightly below the downstream rate to control queuing delay.
Intermediate nodes, such as relay agents and snooping devices, typically apply the rates in both
directions.

A receiver that does not apply an Applied Rate, for example because it does not support the required
traffic management functions, because its processing capacity is not sufficient to apply traffic
management at the signaled rate, or because applying the downstream rate is optional for clients,
treats the rate as information only. In particular, such a receiver can expose the rate in its
management interfaces (see {{operations}}).

The presence of an Applied Rate sub-option always indicates a rate that is intended to be applied.
A server that intends to provide rate information for informational purposes only, without the rates
being applied, uses the Advertised Rate sub-options (see {{advertised}}) instead of the Applied
Rate sub-options.

Explicit configuration by the administrator of the receiver, such as disabling the application of
signaled rates or configuring fixed rates or overhead values, takes precedence over the signaled
values. In the absence of such configuration, the signaled values take precedence over the default
values of the receiver.

## Rate Processing {#processing}

The processing of the DHCP Rate Option is intentionally stateless with respect to the DHCP protocol
version and the DHCP lease, such that it can be implemented by simple event-driven scripts on the
client. A receiver only needs to retain the rates currently in effect.

Rates are maintained per link. DHCPv4 and DHCPv6 messages received over the same link share the
rates in effect, even if they are processed on different logical interfaces of the receiver. The
rules in this section apply independently to each link on which an OPTION_RATE is received.

Each valid OPTION_RATE received in a DHCPv4 DHCPACK message or a DHCPv6 Reply message (or, for
DHCPv6 relay agents, in the corresponding Relay-reply message) replaces the rates previously in
effect in their entirety, regardless of whether the previous rates were learned via DHCPv4 or
DHCPv6:

- If an Applied Rate sub-option is present, the receiver applies the signaled rate for the
  corresponding direction as specified in {{application}}.
- If an Applied Rate sub-option is absent, or carries a value of 0, the receiver MUST remove any
  rate previously applied for the corresponding direction and revert to its default configuration
  for that direction.
- Advertised Rate sub-options replace any previously received Advertised Rates. If an Advertised
  Rate sub-option is absent, no Advertised Rate is provided for the respective direction.

Consequently, an OPTION_RATE without any sub-options instructs the receiver to revert to its default
configuration for both directions. Because some DHCPv4 client implementations discard options with a
length of zero, servers SHOULD NOT send an OPTION_RATE without sub-options in DHCPv4. To remove
previously signaled rates for both directions, a server instead includes an OPTION_RATE containing
Applied Rate sub-options with a value of 0. These semantics apply to an OPTION_RATE received from
a server; an OPTION_RATE without sub-options sent by a relay agent in a Relay-forward message
indicates support for this option (see {{dhcpv6-relay}}).

If the rates, Applied Rate Layers, and per-packet overhead contained in a valid OPTION_RATE are
identical to the currently applied ones, the receiver SHOULD NOT reconfigure its traffic management
functions.

If a DHCPACK or Reply message does not contain an OPTION_RATE, the receiver MUST retain the
rates currently in effect unchanged.

An OPTION_RATE that is discarded in accordance with {{errors}} is treated as if it were not
present, and the rates currently in effect remain unchanged.

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
1%). This allows a client that supports this option to shape its traffic to the signaled rate
without triggering the enforcement at the intermediate node, while the intermediate node still
enforces the provisioned rate for clients that do not support or ignore this option.
Alternatively, in DHCPv6, the server can signal node-specific rates (see {{dhcpv6-server}}).

A client can receive an OPTION_RATE indicating an Applied Rate that exceeds the maximum physical
link speed of its upstream or downstream interfaces (e.g., signaling 2 Gbit/s to a CPE with a
1 Gbit/s Wide Area Network (WAN) port). In such scenarios, the client SHOULD cap the applied traffic
shaping, policing, or AQM parameters to the maximum capacity of the physical link.

In dual-stack deployments, servers SHOULD either signal identical rate information via both
DHCPv4 and DHCPv6, or signal rate information via only one of the two protocols that is expected
to be negotiated by every client. In the latter case, messages of the other protocol do not carry
an OPTION_RATE and therefore leave the rates in effect unchanged. If neither approach is followed,
the rates in effect depend on which message was received last.

## Minimal Implementation {#minimal}

The DHCP Rate Option is designed such that receivers can implement a subset of its sub-options. A
minimal client implementation supports the Applied Rate Upstream and Applied Rate Layer sub-options,
applies the upstream rate at the layer indicated by its Applied Rate Layer, and implements the rules
specified in {{processing}} and {{errors}}. Such an implementation MAY treat sub-options that it
does not implement, such as the Per-Packet Overhead or Advertised Rate sub-options, as unrecognized
sub-option codes (see {{errors}}). Support for the remaining sub-options improves the accuracy and
flexibility of traffic management but is not required for interoperability.

## Client Hints {#hints}

A client MAY include an OPTION_RATE in its messages to the server as a hint. Hints reuse the formats
of the Applied Rate and Applied Rate Layer sub-options, with the semantics defined in this section.
In a client message, the sub-options have the following meaning:

- An Applied Rate sub-option indicates the maximum rate the client is able to handle in the
  corresponding direction, for example due to the capacity of its physical interface. It does not
  request a specific service rate.
- The Applied Rate Layer sub-option indicates the layer preferred by the client. A client
  MAY include an Applied Rate Layer Downstream sub-option to indicate a different preference for the
  downstream direction.
- An absent sub-option, or a rate value of 0, indicates that the client has no preference or
  limitation for the corresponding direction.
- Clients do not include Per-Packet Overhead or Advertised Rate sub-options in hints; servers ignore
  them if present.

The rules specified in {{processing}} do not apply to hints. A client MAY include hints in
DHCPDISCOVER, DHCPREQUEST, and DHCPINFORM messages for DHCPv4, and in Solicit, Request, Renew,
Rebind, and Information-request messages for DHCPv6. A client that includes hints SHOULD include
the same hints in all of these messages.

Sending hints is OPTIONAL for the client, and honoring them is OPTIONAL for the server. A server
that honors a hint uses it to limit the signaled rates to the capacity indicated by the client,
i.e., it signals the lower of the provisioned rate and the hinted rate. How a server otherwise
processes hints is implementation specific. The server remains authoritative: the client MUST
apply the rate information provided by the server in accordance with {{processing}}, regardless
of any hint it sent.

Hints are addressed to the server only. Relay agents and snooping devices MUST NOT use an
OPTION_RATE contained in a client message to configure their traffic management. As intermediate
nodes can derive their enforcement from the signaled rates, honoring a hint can also lower the
enforcement in the network for that client. As the resulting rate never exceeds the provisioned
rate, this only affects the client that sent the hint.

## Appearance in Messages {#appearance}

{{tab-appearance-v4}} and {{tab-appearance-v6}} list the messages in which the OPTION_RATE can
appear. The OPTION_RATE MUST NOT be included in any other message. Servers and receivers MUST
ignore an OPTION_RATE contained in any other message.

| Message                                  | Use of OPTION_RATE                         |
| :--------------------------------------- | :----------------------------------------- |
| DHCPDISCOVER, DHCPREQUEST, DHCPINFORM    | Hint (see {{hints}})                       |
| DHCPOFFER                                | Server selection (see {{dhcpv4-server}})   |
| DHCPACK                                  | Rate information (see {{processing}})      |
{: #tab-appearance-v4 title="Appearance of OPTION_RATE in DHCPv4 Messages"}

| Message                                  | Use of OPTION_RATE                         |
| :--------------------------------------- | :----------------------------------------- |
| Solicit, Request, Renew, Rebind, Information-request | Hint (see {{hints}})           |
| Advertise                                | Server selection (see {{dhcpv6-server}})   |
| Reply to Solicit, Request, Renew, Rebind, or Information-request | Rate information (see {{processing}}) |
| Relay-forward                            | Support indication, length 0 (see {{dhcpv6-relay}}) |
| Relay-reply                              | Rate information for the relay agent (see {{dhcpv6-relay}}) |
{: #tab-appearance-v6 title="Appearance of OPTION_RATE in DHCPv6 Messages"}

In particular, the OPTION_RATE is not included in DHCPv4 DHCPNAK, DHCPDECLINE, DHCPRELEASE, or
DHCPFORCERENEW messages, nor in DHCPv6 Reconfigure messages, as specified in Section 18.3.11 of
{{!RFC9915}}.

## Error Handling {#errors}

Conflicting rate information received via DHCPv4 and DHCPv6 is resolved as specified in
{{processing}}: the most recently received valid OPTION_RATE takes precedence.

To ensure forward compatibility, receivers and servers MUST ignore unrecognized sub-option codes
and continue processing the remainder of the OPTION_RATE.

If a receiver processes a recognized sub-option containing an unrecognized value, it handles the
value as specified for the respective sub-option. For an invalid Applied Rate Layer value, the
receiver does not apply the Applied Rate of the affected direction, as applying a rate without
understanding its accounting could result in incorrect traffic management (see {{rate-layer}}).
Values of the Per-Packet Overhead sub-options that a receiver cannot apply do not affect the
processing of the remaining sub-options (see {{overhead}}). Specifications defining additional
sub-options specify the handling of unrecognized values.

A receiver MUST also discard the entire OPTION_RATE if any of the following conditions is met:

- the length of a recognized sub-option differs from the length specified for that sub-option;
- the length of a sub-option exceeds the number of octets remaining in the OPTION_RATE;
- the number of octets remaining in the OPTION_RATE is non-zero but smaller than the size of a
  sub-option header (2 octets for DHCPv4, 4 octets for DHCPv6).

A discarded OPTION_RATE is treated as if it were not present (see {{processing}}).

A sender MUST NOT include more than one OPTION_RATE in a DHCPv4 message, or more than one
OPTION_RATE at the same encapsulation level of a DHCPv6 message (i.e., within the message exchanged
with the client or within a given Relay-reply message). A sender MUST NOT include the same
sub-option more than once within an OPTION_RATE.

If a DHCPv4 receiver nevertheless receives multiple instances of OPTION_RATE, it MUST concatenate
them as specified in {{!RFC3396}} and process the result as a single OPTION_RATE. If a DHCPv6
receiver receives more than one OPTION_RATE at the same encapsulation level, it SHOULD discard
all instances, which are then treated as if they were not present (see {{processing}}).

If multiple instances of the same sub-option code are present within an OPTION_RATE, the last
instance MUST be processed.

# DHCPv4 {#dhcpv4}

## DHCPv4 Rate Option

The option code of OPTION_V4_RATE is TBD1. The messages in which it can appear are listed in
{{appearance}}.

~~~ aasvg
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | Code TBD1     | Length N      | SubOpt Code   | SubOpt Length |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | SubOpt Value                                                  |
 |                                                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | SubOpt Code   | SubOpt Length | SubOpt Value                  |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               |
 |                                                               |
 |                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                               | SubOpt Code   | SubOpt Length |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | SubOpt Value
 +-+-+-+-+-+-+-...
~~~

The option length (N) gives the total number of octets of all sub-options. A length of 0 is
valid and indicates an OPTION_RATE without any sub-options (see {{processing}}).

## DHCPv4 Client Behavior

DHCPv4 clients that support the DHCP Rate Option SHOULD include the corresponding OPTION_RATE code
in the Parameter Request List (PRL). This inclusion signals client support, enabling the DHCP server
to determine whether to include the rate parameters in its response. Furthermore, this signaling
provides network operators with visibility into client capabilities, which aids in troubleshooting
and resolving customer service quality complaints.

A client MAY include hints as specified in {{hints}}. For example, when a CPE is connected via a
1 Gbit/s WAN interface to an external ONT or modem capable of exceeding 1 Gbit/s on the WAN side,
the CPE can signal this physical limitation to the service provider. This enables the network to align
its shaping parameters with the device's actual capacity, ensuring traffic does not exceed the
physical limit.

A client MUST NOT apply rates received in any DHCPv4 message other than DHCPACK. A client MAY use
the rate information contained in a DHCPOFFER message as a criterion for selecting among multiple
offers. Rates received in a DHCPACK message are processed as specified in {{processing}}.

## DHCPv4 Server Behavior {#dhcpv4-server}

When a DHCPv4 server that supports the DHCP Rate Option receives a client message whose PRL includes
OPTION_RATE, and rate information is configured for the client, the server MUST include an
OPTION_RATE reflecting the current rate information in the resulting DHCPOFFER and DHCPACK messages.
If no rate information is configured for the client, the server MAY omit the OPTION_RATE. Because a
missing OPTION_RATE leaves the rates in effect unchanged, a server that needs to remove previously
signaled rates MUST include an OPTION_RATE without the respective Applied Rate sub-options or with a
rate value of 0, as specified in {{processing}}.

The server MAY derive the specific upstream and downstream rates and Applied Rate Layers from local
configuration profiles, centralized AAA systems such as RADIUS, or external policy servers.

A server MAY include the OPTION_RATE in its responses even if the client did not request it via the
PRL, provided the operator has explicitly configured the server to do so, for example to provision
Layer 2 DHCPv4 Relay Agents or snooping devices.

To update the rates of a client before its next scheduled renewal, a server MAY send a
DHCPFORCERENEW message {{?RFC3203}}, causing the client to initiate a renewal transaction.
DHCPFORCERENEW messages are subject to the authentication requirements of {{?RFC3203}}, which
can be satisfied using the mechanism specified in {{?RFC6704}}. For clients that do not support
DHCPFORCERENEW, updated rates take effect with the next renewal, the timing of which is
controlled by the renewal (T1) time value.

## DHCPv4 Relay Agent Behavior

DHCPv4 relay agents, including Layer 2 DHCPv4 Relay Agents {{TR101}}, MAY extract the OPTION_RATE
from DHCPACK messages traversing the network. Relay agents that perform localized traffic management
MAY use these extracted values to dynamically instantiate shapers and policers on their
client-facing interfaces. Relay agents SHOULD remove rates applied for a client when the
corresponding binding is removed. Layer 2 DHCPv4 Relay Agents MUST only process an OPTION_RATE
contained in DHCPACK messages received on trusted, network-facing ports.

In many architectures, the BNG or a similar device acts as the authoritative policy enforcement
point. Modifying the message destined for the client is outside the relay agent behavior specified
in {{!RFC2131}} and {{?RFC3046}}. A device that populates, modifies, or removes the OPTION_RATE
destined for the client based on subscriber attributes (e.g., retrieved from an AAA server such as
RADIUS) therefore acts as a DHCPv4 server or DHCPv4 proxy towards the client, and the server
behavior specified in this document applies to it.

# DHCPv6 {#dhcpv6}

## DHCPv6 Rate Option

The option code of OPTION_V6_RATE is TBD2. The messages in which it can appear are listed in
{{appearance}}.

~~~ aasvg
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | Code TBD2                     | Length N                      |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | SubOpt Code                   | SubOpt Length                 |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | SubOpt Value                                                  |
 |                                                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | SubOpt Code                   | SubOpt Length                 |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | SubOpt Value
 +-+-+-+-+-+-+-...
~~~

The option length (N) gives the total number of octets of all sub-options. A length of 0 is
valid and indicates an OPTION_RATE without any sub-options (see {{processing}}).

## DHCPv6 Client Behavior

DHCPv6 clients that support the DHCP Rate Option SHOULD include the corresponding OPTION_RATE code
in the Option Request option (ORO) {{!RFC9915}}. This inclusion signals client support, enabling the
DHCPv6 server to determine whether to include the rate parameters in its response. Furthermore, this
signaling provides network operators with visibility into client capabilities, which aids in
troubleshooting and resolving customer service quality complaints.

A client MAY include hints as specified in {{hints}}.

A client MUST NOT apply rates received in any DHCPv6 message other than Reply. A client MAY use
the rate information contained in an Advertise message as a criterion for selecting among
multiple Advertise messages. Rates received in a Reply message are processed as specified in
{{processing}}.

## DHCPv6 Server Behavior {#dhcpv6-server}

When a DHCPv6 server that supports the DHCP Rate Option receives a Solicit, Request, Renew, Rebind,
or Information-request message whose ORO includes OPTION_RATE, and rate information is configured
for the client, the server MUST include an OPTION_RATE reflecting the current rate information in
the resulting Advertise and Reply messages. If no rate information is configured for the client, the
server MAY omit the OPTION_RATE. Because a missing OPTION_RATE leaves the rates in effect unchanged,
a server that needs to remove previously signaled rates MUST include an OPTION_RATE without the
respective Applied Rate sub-options or with a rate value of 0, as specified in {{processing}}.

A server MAY include the OPTION_RATE in its responses even if the client did not request it via
the ORO, provided the operator has explicitly configured the server to do so.

In relayed exchanges, the OPTION_RATE for the client is included in the Reply message, which is
encapsulated in one or more Relay-reply messages. The server MAY additionally include the
option in a Relay-reply message to provision the corresponding relay agent. The nested
Relay-reply structure of DHCPv6 allows the server to address each relay agent in the path
individually, in addition to the client.

DHCPv6 does not define a mechanism for relay agents to request options from the server.
A server therefore includes the OPTION_RATE in a Relay-reply message based on local
configuration, for example by identifying the relay agent by the link-address or peer-address
field of the Relay-forward message, or by relay agent options such as Interface-Id {{!RFC9915}}
or Remote-Id {{?RFC4649}}.
A server MAY additionally take into account a support indication received from the relay agent
(see {{dhcpv6-relay}}).

If network policy dictates localized traffic management at both the CPE and the relay node, the
server MAY include the OPTION_RATE at all encapsulation levels simultaneously. When provisioning
multiple levels, the server MAY supply different rate values to each respective node. For example,
an operator might configure a relay agent's upstream policer with a slightly higher rate limit than
the CPE's upstream shaper. This operational delta accommodates minor traffic burstiness from the CPE
and prevents premature packet drops at the intermediate access node. DHCPv4 does not provide a
comparable mechanism for addressing multiple relay agents individually.

To update the rates of a client before its next scheduled renewal, a server MAY send a
Reconfigure message as specified in Section 18.3.11 of {{!RFC9915}}, causing the client to
initiate a Renew, Rebind, or Information-request transaction. As specified in {{!RFC9915}},
a server can only send Reconfigure messages to clients that have included the Reconfigure Accept
option in their messages, and Reconfigure messages are authenticated using the Reconfiguration
Key Authentication Protocol (RKAP; see Section 20.4 of {{!RFC9915}}).

For clients that do not support Reconfigure, updated rates take effect with the next Renew or
Rebind transaction or, for stateless clients, with the next Information-request transaction.
Operators can control the timing of such updates through the T1 value and the Information
Refresh Time option (Section 21.23 of {{!RFC9915}}).

## DHCPv6 Relay Agent Behavior {#dhcpv6-relay}

DHCPv6 relay agents that support the DHCP Rate Option, including Lightweight DHCPv6 Relay Agents
(LDRA) {{?RFC6221}}, process an OPTION_RATE contained in a Relay-reply message addressed to them as
specified in {{processing}}. Such relay agents MAY use the signaled rates to instantiate shapers,
policers, or AQM disciplines on their client-facing interfaces. Relay agents SHOULD remove rates
applied for a client when the corresponding binding is removed. Relay agents MUST only process
Relay-reply messages received from the network side; in particular, an LDRA discards Relay-reply
messages received on client-facing interfaces as specified in Section 6.1.1 of {{?RFC6221}}.

A relay agent that supports the DHCP Rate Option MAY include an OPTION_RATE with a length of 0 in
its Relay-forward messages to indicate its support to the server. A server MUST NOT interpret
such an option as a rate hint.

An OPTION_RATE contained in the encapsulated message destined for the client targets the client
only. Relay agents MUST NOT use such an option to configure their own traffic management, since
the DHCPv6 relay architecture provides the Relay-reply message as a dedicated channel for
signaling rates to relay agents.

For an OPTION_RATE contained in a Relay-reply message, the Layer 2 rate and the Per-Packet
Overhead sub-options refer to the frames on the client-facing link of the relay
agent (see {{rate-layer}} and {{overhead}}), which can differ from the frames as seen by the client.

In many architectures, the BNG or a similar device acts as the authoritative policy enforcement
point. Modifying the message destined for the client is outside the relay agent behavior specified
in {{!RFC9915}}. A device that populates, modifies, or removes the OPTION_RATE destined for the
client based on subscriber attributes (e.g., retrieved from an AAA server such as RADIUS) therefore
acts as a DHCPv6 server or DHCPv6 proxy towards the client, and the server behavior specified in
this document applies to it.

# DHCP Snooping {#snooping}

DHCP snooping switches are typically deployed as intermediate Layer 2 devices that passively monitor
DHCP message exchanges to enforce security policies and build binding databases, all without
modifying the DHCP payloads. These devices MAY passively inspect the DHCP Rate Option within
messages destined for clients. By extracting these rate parameters, snooping devices can dynamically
provision appropriate traffic shapers, policers, or hardware queues on the corresponding downstream,
client-facing ports. Snooping devices SHOULD remove rates applied for a client when the
corresponding binding is removed.

Snooping devices MUST only process an OPTION_RATE contained in DHCPACK (DHCPv4) and Reply
(DHCPv6) messages received on trusted, network-facing ports.
An OPTION_RATE contained in messages received on client-facing ports MUST NOT be used to
configure traffic management.

# Deployment Scenarios

This section describes the application of the DHCP Rate Option in specific deployment
scenarios.

## PPPoE {#pppoe}

In PPPoE {{?RFC2516}} architectures, the CPE typically employs DHCPv6 over the PPP {{?RFC1661}} link
to obtain an IPv6 prefix via Prefix Delegation (IA_PD) {{!RFC9915}}. This encapsulated DHCPv6
exchange provides a standardized transport mechanism for the DHCP Rate Option. While less prevalent
in modern deployments, DHCPv4 transactions operating within a PPPoE session can similarly convey the
DHCP Rate Option.

The processing rules and client behavior for an OPTION_RATE received over PPPoE are identical to
those defined for IPoE environments, as specified in {{processing}}. In particular, the rates in
effect revert to the default configuration of the device when the underlying PPPoE session is
terminated.

If a client receives rate information embedded within a PPP authentication message and also receives
an OPTION_RATE, the client SHOULD give precedence to the OPTION_RATE. This ensures that the
standardized, dynamic DHCP signaling supersedes proprietary rate information conveyed during the PPP
authentication phase.

In PPPoE deployments, the OPTION_RATE is received on the PPP interface, while a Layer 2 rate refers
to the Ethernet frames on the link, including the PPPoE and PPP headers (see {{rate-layer}}). A
receiver that applies its traffic management functions to the PPP interface accounts for the
Ethernet, PPPoE, and PPP headers as per-packet overhead.

## Local Area Networks {#lan}

The DHCP Rate Option can also be used by DHCP servers in local area networks, for example a DHCP
server co-located with a CPE or another gateway, to inform clients about the rates available to
them. In this case, the signaled rates indicate the maximum rate a client can expect through the
network it is attached to. Examples include:

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
MAY trigger an earlier update as specified in {{dhcpv4-server}} and {{dhcpv6-server}}.

Unless the signaled rates are enforced for each client individually, they describe a capacity shared
by all clients of the LAN. In this case, the server SHOULD signal them using the Advertised Rate
sub-options (see {{advertised}}) rather than the Applied Rate sub-options, so that clients use the
rates as information rather than configuring traffic management functions based on them.

Clients cap the effective rates to the capacity of their physical link (see {{processing}}), for
example when a client is connected at 1 Gbit/s to a LAN whose upstream connection provides a higher
rate. For links with variable capacity, such as wireless links, the effective rate is the lower of
the signaled rate and the current link capacity.

The signaled rates do not account for other bottlenecks along the path to a remote peer. Clients
MUST NOT use the signaled rates as a replacement for congestion control or other adaptive rate
mechanisms, and MUST NOT use them to increase the sending rate of a congestion control algorithm,
such as its initial sending rate, beyond the rate that the algorithm would otherwise use. Where rate
limits need to be enforced, enforcement remains the responsibility of the network; the DHCP Rate
Option only informs clients about such limits.

# Operational Considerations {#operations}

Deploying explicit rate signaling via DHCP introduces several operational benefits and deployment
considerations for network management. When combined with appropriately configured AQM and, where
deployed, L4S-compatible queue management, these per-subscriber rate parameters help to concentrate
congestion control at a well-defined bottleneck and minimize queuing delay in the access segment.

Operators should signal the rate at the layer at which the enforcement point accounts traffic,
together with the per-packet overhead of the enforcement point whenever it is known (see
{{overhead}}). This allows clients to model the actual bottleneck precisely, including additional
encapsulations and cell-based link layers. Where contracted rates are specified as payload rates,
for example due to regulatory requirements, operators typically configure the enforcement point such
that the payload rate achieved with maximum-sized packets meets the contracted rate; the signaled
rate and overhead then describe the configuration of the enforcement point, not the contracted
payload rate. The contracted rate can be conveyed in addition using the Advertised Rate sub-options
(see {{advertised}}).

Very-high-bit-rate Digital Subscriber Line 2 (VDSL2) commonly uses Packet Transfer Mode (PTM) with
64/65-octet encapsulation instead of ATM cells. Its effect corresponds to a constant reduction of
the available rate and can be included in the signaled rate, without requiring cell-based
accounting. Per-packet overhead of PTM, such as the Ethernet FCS, can be signaled using the Overhead
field (see {{overhead}}).

Where intermediate nodes enforce the signaled rates, operators should ensure that the enforcement at
the intermediate node is not triggered by a client that shapes its traffic to the signaled rate, for
example by applying a small correction factor at the intermediate node or by signaling node-specific
rates (see {{processing}} and {{dhcpv6-server}}). As enforcement points at intermediate nodes
commonly account traffic at Layer 2, servers typically signal the Applied Rate Layer explicitly when
provisioning intermediate nodes.

Where the actual rate of the access line is known and lower than the provisioned rate, for example
due to the synchronization rate of a DSL line, operators should signal the lower of the two rates,
so that the client shapes its traffic to the actual bottleneck. {{access-loop}} describes how the
parameters of the DHCP Rate Option can be derived from the access loop characteristics reported by
the access node.

Without the use of DHCPFORCERENEW or Reconfigure, changes of the provisioned rates take effect
with the next renewal of the client. Operators should consider this delay when choosing lease
and renewal times, for example when rates are changed due to service upgrades or fair-use
policies.

The inclusion of OPTION_RATE in the Parameter Request List or Option Request option allows
operators to determine the share of clients supporting this option, for example to plan the
introduction of correction factors or to identify clients that require enforcement at the
network.

Implementations are encouraged to expose the signaled, DHCP-learned rate parameters, including
Advertised Rates, within CPE management interfaces, such as the local Web User Interface (UI) or
remote management protocols. Providing end-users and operators with immediate visibility into the
locally provisioned service tier can reduce support inquiries related to perceived bandwidth issues.

As noted in {{processing}}, a client can receive an OPTION_RATE indicating an Applied Rate that
exceeds the maximum physical link speed of its interfaces. In such scenarios, management interfaces
should expose both the originally signaled rate value and the effective, capped rate value applied
by the device. Additionally, implementations should log this discrepancy if logging facilities are
enabled. Capturing and exposing these specific events provides useful information to network
operators, as they can indicate a mismatch between the subscriber's provisioned service tier and
their installed physical equipment.

# Implementation Status {#impl-status}
{:removeinrfc="true"}

{::boilerplate rfc7942}

At the time of writing, no implementations of this document are known to the authors.

## Implementation Feasibility on Open-Source CPE

The authors analyzed the feasibility of implementing the client behavior on OpenWrt without
modifying the DHCP client software. The DHCPv4 client (BusyBox udhcpc) and DHCPv6 client (odhcp6c)
used by OpenWrt can request OPTION_RATE, send hints, and expose received options of unknown type as
hexadecimal environment variables to user-defined hook scripts. All client behavior specified in
this document can therefore be implemented in such scripts, which parse the option, apply the
processing rules of {{processing}}, and configure local traffic management (e.g., via Smart Queue
Management (SQM) or the Common Applications Kept Enhanced (CAKE) queue discipline). The parameters
of CAKE can be derived from the Per-Packet Overhead sub-options (see {{overhead}}): for a Layer 3
rate, the Overhead corresponds directly to the overhead parameter of CAKE, which is relative to the
network-layer packet, while for a Layer 2 rate, the length of the Layer 2 headers on the link is
added. The MPU corresponds to the mpu parameter of CAKE, and cell-based accounting with a Cell
Payload of 48 and a Cell Size of 53 corresponds to its atm parameter. The rates learned on the WAN
interface can be propagated to LAN clients using the existing custom-option support of the OpenWrt
DHCPv4 server (dnsmasq) and DHCPv6 server (odhcpd).

# Security Considerations {#security}

DHCP messages are typically transmitted as plaintext and are unauthenticated. Consequently, the DHCP
Rate Options defined in this document are vulnerable to interception, modification, or spoofing by
on-path attackers. This applies to any path the option traverses, including the path between
servers and clients as well as the path between servers and relay agents, where an OPTION_RATE
carried in a Relay-reply message configures the relay agent itself (see {{dhcpv6-relay}}).
A malicious actor or a rogue DHCP server could inject artificially low rate limits to
throttle a client's connection, resulting in a localized Denial of Service (DoS).

The severity of this specific risk is generally no greater than standard DHCP threat vectors, such
as rogue default gateway assignment, DNS hijacking, or IP pool exhaustion, which typically yield a
much more critical and immediate loss of service.

Operators can mitigate rogue DHCP servers by deploying DHCP snooping for DHCPv4 and DHCPv6-Shield
{{?RFC7610}} for DHCPv6 on access nodes, which discard server messages received on client-facing
ports. These mechanisms protect both the clients and the intermediate nodes that apply signaled
rates.

To mitigate the impact of malicious or malformed options, client implementations MAY apply sanity
checks before applying signaled rates. In particular, a client implementation MAY assume a minimum
rate below which basic connectivity is no longer ensured, and treat an OPTION_RATE signaling a
non-zero rate below this minimum as if it were not present (see {{processing}}). The appropriate
minimum depends on the deployment and the use case of the device and is therefore left to the client
implementation. For example, a broadband CPE might reject rates below 1 Mbit/s.

Excessive values of the Overhead or MPU fields of the Per-Packet Overhead sub-options reduce the
effective rate. Receivers MAY limit these values to a plausible range or ignore the sub-option,
which does not invalidate the OPTION_RATE (see {{overhead}}). Manipulated Advertised Rate
sub-options only affect informational uses, such as the display of rates or estimates used by
applications.

Conversely, an attacker could inject an excessively high rate, a rate value of 0, or an OPTION_RATE
without Applied Rate sub-options, thereby removing any effective rate limit (see {{processing}}).
Because clients are expected to cap the effective rate to the capacity of the physical link (see
{{processing}}), the impact is limited to the loss of the benefits of rate signaling, such as
increased queuing delay at the bottleneck, which corresponds to the behavior of a device without
support for this option. Where an intermediate node derives the enforcement of the provisioned rate
from the DHCP Rate Option (see {{solution-overview}}), however, an attacker on the path between the
server and that node that raises or removes the signaled rates also raises or removes the
enforcement, allowing a client to exceed its provisioned rate. Mechanisms for securing the messages
exchanged between servers and relay agents are described in {{?RFC8213}}.

In local area networks, DHCP snooping is often not deployed, making rogue DHCP servers more
likely. A rogue server signaling artificially low rates could cause applications on end devices
to reduce their quality or to prefer another network interface, which can incur additional cost
(e.g., cellular data). Clients SHOULD NOT use the signaled rates as the sole criterion for
selecting a network interface, and the sanity checks described above apply.

Relay agents and snooping devices that instantiate traffic management functions based on the
DHCP Rate Option consume finite resources, such as hardware queues and shapers. Because only
server messages received on trusted ports are processed and rates are applied per client-facing
port (see {{processing}}), the amount of such resources is bounded by the number of client-facing
ports. However, a client can trigger frequent DHCP exchanges, potentially causing repeated
reconfiguration of these functions. In addition to not reconfiguring unchanged rates (see
{{processing}}), implementations MAY rate-limit DHCP messages received on client-facing ports or
the reconfiguration of traffic management functions.

Hints are unauthenticated input from the client. Servers ought not to grant rates above the
provisioned rate on the basis of hints, and relay agents and snooping devices do not use hints
for their traffic management (see {{hints}}).

# Privacy Considerations {#privacy}

The general privacy considerations for DHCP specified in {{?RFC7844}} apply to this document.

The DHCP Rate Option conveys the rates provisioned for a client, which typically reflect the service
tier of a subscriber. Because DHCP messages are typically transmitted as plaintext, this information
is exposed to on-path observers, as well as to relay agents and snooping devices that process the
option. The rate information itself does not identify a subscriber, but in combination with other
information it can reveal commercial details of the service contract. This applies in particular to
the Advertised Rate sub-options, which can convey the contracted rate directly; servers include them
only where they are useful, for example not towards every client in a local area network.

Clients that include OPTION_RATE in the Parameter Request List or Option Request option, or that
send hints (see {{hints}}), disclose additional information that can contribute to fingerprinting
the device. Hints can additionally reveal hardware characteristics, such as the capacity of the
client's physical interface. Clients that implement the anonymity profiles specified in
{{?RFC7844}} SHOULD NOT send hints and MAY omit OPTION_RATE from the Parameter Request List or
Option Request option.

The operational practice of exposing and logging signaled and effective rates (see {{operations}})
can provide operators with information about the equipment installed by a subscriber. Such data
is subject to the operator's applicable privacy policies.

# IANA Considerations

IANA is requested to replace TBD1 and TBD2 throughout this document with the values assigned as
specified below. RFC Editor: please replace "RFC XXXX" throughout this document with the RFC number
assigned to this document, and remove this note.

If feasible, IANA is requested to assign the same numeric value for TBD1 and TBD2. This is a
convenience for implementers and not a requirement.

## DHCPv4 Option

IANA is requested to assign a new DHCP Option Code (TBD1) in the "BOOTP Vendor Extensions and
DHCP Options" registry as follows:

- Tag: TBD1
- Name: OPTION_V4_RATE
- Data Length: N
- Meaning: DHCP Rate Option
- Reference: RFC XXXX

## DHCPv6 Option

IANA is requested to assign a new DHCPv6 Option Code (TBD2) in the "Option Codes" registry as
follows:

- Value: TBD2
- Description: OPTION_V6_RATE
- Client ORO: Yes
- Singleton Option: Yes
- Status: (empty)
- Reference: RFC XXXX

## DHCP Rate Sub-Options Registry

IANA is requested to create a new registry titled "DHCP Rate Sub-Options" within the "Dynamic Host
Configuration Protocol (DHCP) and Bootstrap Protocol (BOOTP) Parameters" registry group. This single
registry applies to the sub-options encapsulated within both the DHCPv4 OPTION_RATE (TBD1) and the
DHCPv6 OPTION_RATE (TBD2).

The allocation policy of this new registry is Specification Required (Section 4.6 of {{!RFC8126}}).
Guidance for the designated experts is provided in Section {{experts}}{:format="counter"}.
Registration requests include the Description, the Length, and a reference to the specification.

The registry covers the values 0-255. Each assigned value identifies the same sub-option in both the
DHCPv4 and the DHCPv6 OPTION_RATE. Values 256-65535 of the 16-bit DHCPv6 sub-option code field are
not available for assignment (see {{sub-option-table}}).

The registry contains the following columns:

Value:
: The sub-option code.

Description:
: The name of the sub-option.

Length:
: The length of the sub-option value in octets. For sub-options with a variable length, this column
  contains "variable". For reserved and unassigned values, it contains "-".

Reference:
: The specification that defines the sub-option.

The initial content of this registry is listed in {{iana-sub-option-table}}.

| Value | Description                    | Length | Reference |
| ----: | :----------------------------- | -----: | :-------- |
|     0 | Reserved                       |      - |           |
|     1 | Advertised Rate Upstream       |      8 | RFC XXXX  |
|     2 | Advertised Rate Downstream     |      8 | RFC XXXX  |
|     3 | Applied Rate Upstream          |      8 | RFC XXXX  |
|     4 | Applied Rate Downstream        |      8 | RFC XXXX  |
|     5 | Applied Rate Layer             |      1 | RFC XXXX  |
|     6 | Applied Rate Layer Downstream  |      1 | RFC XXXX  |
|     7 | Per-Packet Overhead            |      6 | RFC XXXX  |
|     8 | Per-Packet Overhead Downstream |      6 | RFC XXXX  |
| 9-254 | Unassigned                     |      - |           |
|   255 | Reserved                       |      - |           |
{: #iana-sub-option-table title="Initial Contents of the DHCP Rate Sub-Options Registry"}

As the "DHCP Rate Sub-Options" registry also applies to DHCPv6, IANA is requested to add a reference
to this registry in the "Dynamic Host Configuration Protocol for IPv6 (DHCPv6)" registry group.

No registry is required for the values of the Applied Rate Layer sub-options, as their values are
fully defined by this document (see {{rate-layer}}).

## Guidelines for the Designated Experts {#experts}

It is suggested that multiple designated experts be appointed for registration requests to the
registry defined in this document.

Criteria that should be applied by the designated experts include determining whether the proposed
registration duplicates existing entries and whether the registration description is clear and fits
the purpose of the registry.

The designated experts should also verify that the referenced specification is publicly available
and defines the semantics of the requested value with sufficient clarity to allow interoperable
implementations. Suitable specifications include, but are not limited to, RFCs and documents
published by other standards development organizations, such as Broadband Forum Technical Reports.

Registration requests are submitted to IANA, which forwards them to the designated experts for
evaluation. The designated experts aim to complete their evaluation within three weeks and inform
IANA of their decision. Denials should include an explanation and, if applicable, suggestions as
to how to make the request successful.

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
this option continue to be subject to the enforcement at the BNG, so that the provisioned
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
refreshed with every lease renewal and can be updated dynamically. The option is simple enough
to be processed by event-driven scripts in existing DHCP clients, without changes to the DHCP
client software (see {{processing}}).

## Relation to Access Line Information

Standardized mechanisms exist to convey the characteristics of an access line from the access node
to the network: access loop characteristics are carried in DHCP relay agent options and PPPoE tags
{{TR101}}, mapped to RADIUS attributes such as Actual-Data-Rate-Upstream and
Actual-Data-Rate-Downstream {{?RFC4679}}, and reported via ANCP {{?RFC6320}}. These mechanisms
inform the BNG and the AAA infrastructure about what the access line can carry. No standardized
mechanism, however, conveys the resulting rates to the CPE and to intermediate nodes. This document
closes this gap, using the access line information as input for the signaled rates (see
{{access-loop}}).

## Relation to Existing Proprietary Solutions

Several operators have successfully deployed proprietary mechanisms to provide the CPE with its
provisioned rate, for example by embedding rate information in PPP authentication messages or by
vendor-specific management interfaces. The deployment of such mechanisms demonstrates the need for
this information at the CPE, and CPE implementations already provide the corresponding traffic
management functions, including shaping at Layer 2 or at Layer 3. However, these mechanisms have
significant limitations: they are typically supported only by operator-provided CPE, so that
customer-owned CPE fall back to the policing behavior described above; rate changes often take
effect only after the session is re-established; and the information is not available to
intermediate nodes. This document defines a standardized and interoperable alternative that
addresses these limitations.

# Examples of Per-Packet Overhead Parameters {#examples}

This appendix provides examples of the parameters of the Per-Packet Overhead sub-options (see
{{overhead}}) for common deployments. The ADSL examples use bridged Ethernet with Logical Link
Control (LLC) and Subnetwork Access Protocol (SNAP) encapsulation {{?RFC2684}}, and PPP over ATM
(PPPoA) with Virtual Circuit (VC) multiplexing {{?RFC2364}}. The values are illustrative; the actual
values depend on the configuration of the access network and of the enforcement point and need to be
determined by the operator.

In all examples, S denotes the size of the packet at the layer indicated by the Applied Rate Layer
of the respective direction, on the link of the receiver. For a Layer 2 rate, S includes the
Ethernet header and any VLAN tags and encapsulation headers present on the link of the receiver, but
excludes the Ethernet FCS (see {{rate-layer}}).

| Scenario                                    | Applied Rate Layer | Overhead | MPU | Cell Payload / Size |
| :------------------------------------------ | :----------------- | -------: | --: | ------------------: |
| Ethernet handoff as bottleneck              | 2 (Layer 2)        |       24 |  84 |               0 / 0 |
| Policer, VLAN tag added by access node      | 2 (Layer 2)        |        4 |   0 |               0 / 0 |
| Policer accounting IP packets               | 3 (Layer 3)        |        0 |   0 |               0 / 0 |
| ADSL, PPPoE, bridged Ethernet with LLC/SNAP | 2 (Layer 2)        |       18 |   0 |             48 / 53 |
| ADSL, PPPoA with VC multiplexing            | 2 (Layer 2)        |        8 |   0 |             48 / 53 |
{: #tab-examples title="Example Per-Packet Overhead Parameters"}

Ethernet handoff as bottleneck:
: Where the bottleneck is an Ethernet link, for example an Ethernet handoff at the provisioned rate,
  each frame additionally consumes the transmission time of the FCS (4 octets), the preamble (7
  octets), the SFD (1 octet), and the inter-packet gap (12 octets) {{IEEE802.3}}, resulting in an
  Overhead of 24 octets. As the minimum Ethernet frame size is 64 octets including the FCS, the
  MPU is 84 octets.

Policer, VLAN tag added by access node:
: The enforcement point accounts Ethernet frames without FCS, but the access node adds a VLAN tag
  that is not present on the link of the receiver, resulting in an Overhead of 4 octets.

Policer accounting IP packets:
: The enforcement point accounts IP packets. The Per-Packet Overhead sub-option is included with all
  values set to 0, indicating that no additional overhead applies. A receiver that accounts
  traffic at Layer 2 compensates for its own headers implicitly (see {{rate-layer}}).

ADSL, PPPoE, bridged Ethernet with LLC/SNAP:
: The receiver sends PPPoE frames over Ethernet, which are transported over ATM using bridged
  Ethernet encapsulation with LLC/SNAP without preserved FCS {{?RFC2684}}. Within the AAL5 frame,
  the Ethernet frame without FCS is preceded by the LLC/SNAP header including padding (10 octets)
  and followed by the AAL5 trailer (8 octets), resulting in an Overhead of 18 octets and
  cell-based accounting with a Cell Payload of 48 and a Cell Size of 53. If the FCS is preserved,
  the Overhead increases by 4 octets.

ADSL, PPPoA with VC multiplexing:
: The receiver sends PPP frames over ATM using PPP over AAL5 with VC multiplexing {{?RFC2364}}.
  S includes the 2-octet PPP protocol field. The AAL5 trailer (8 octets) results in an Overhead
  of 8 octets, with cell-based accounting using a Cell Payload of 48 and a Cell Size of 53.

# Deriving Parameters from Access Loop Characteristics {#access-loop}

This appendix describes how a server or BNG can derive the parameters of the DHCP Rate Option from
the access loop characteristics reported by the access node, for example via DHCP relay agent
options or PPPoE tags {{TR101}}, RADIUS attributes {{?RFC4679}}, or ANCP {{?RFC6320}}.

Where the access line is the bottleneck, the Applied Rate is derived from the actual net data rate
of the access line, as conveyed by the Actual-Data-Rate-Upstream and Actual-Data-Rate-Downstream
attributes {{?RFC4679}} in bits per second, or by the corresponding ANCP TLVs {{?RFC6320}} in
kbit/s. The Applied Rate is the lower of the provisioned rate and the rate derived from the actual
net data rate. As the net data rate refers to the ATM cells or Ethernet frames transported over the
access line, the Applied Rate Layer is set to 2, or to 3 for IP over AAL5, where the frame on the
link of the receiver is the IP packet. For lines using PTM with 64/65-octet encapsulation, the rate
available to Ethernet frames is 64/65 of the net data rate (see {{operations}}).

The Per-Packet Overhead can be derived from the Access-Loop-Encapsulation TLV {{?RFC6320}} or the
corresponding RADIUS attribute {{?RFC4679}}, which indicate the data link and encapsulation of the
access loop. {{tab-access-loop}} lists the resulting parameters for the encapsulations defined for
ATM AAL5, where NULL denotes VC multiplexing. The values are relative to the frame on the link of
the receiver: the Ethernet frame without FCS for Ethernet over AAL5, the PPP frame including the
2-octet PPP protocol field for PPP over AAL5 {{?RFC2364}}, and the IP packet for IP over ATM (IPoA).

| Encapsulation over ATM AAL5         | Applied Rate Layer | Overhead | Cell Payload / Size |
| :---------------------------------- | :----------------- | -------: | ------------------: |
| Ethernet over AAL5 LLC without FCS  | 2 (Layer 2)        |       18 |             48 / 53 |
| Ethernet over AAL5 LLC with FCS     | 2 (Layer 2)        |       22 |             48 / 53 |
| Ethernet over AAL5 NULL without FCS | 2 (Layer 2)        |       10 |             48 / 53 |
| Ethernet over AAL5 NULL with FCS    | 2 (Layer 2)        |       14 |             48 / 53 |
| PPPoA LLC                           | 2 (Layer 2)        |       12 |             48 / 53 |
| PPPoA NULL                          | 2 (Layer 2)        |        8 |             48 / 53 |
| IPoA LLC                            | 3 (Layer 3)        |       16 |             48 / 53 |
| IPoA NULL                           | 3 (Layer 3)        |        8 |             48 / 53 |
{: #tab-access-loop title="Parameters Derived from the Access Loop Encapsulation"}

The Overhead values include the encapsulation headers within the AAL5 frame, such as the LLC and
SNAP headers {{?RFC2684}} and the padding of bridged frames, and the 8-octet AAL5 trailer. For an
Ethernet data link, no cell-based accounting applies, and the Overhead accounts for each VLAN tag
that is present at the enforcement point but not on the link of the receiver (4 octets per tag), as
indicated by the VLAN tagging reported in the Access-Loop-Encapsulation TLV, as well as for any
per-packet overhead of PTM, if known.

# Example Receiver Processing {#example-processing}

This appendix summarizes the processing of the DHCP Rate Option by a receiver as specified in
{{rate-layer}}, {{overhead}}, {{processing}}, and {{errors}}. It is informative; in case of
discrepancies, the referenced sections take precedence.

~~~ pseudocode
For each DHCPACK or Reply message received on a link:
  if no OPTION_RATE is present:
    keep the rates in effect unchanged; done
  if the OPTION_RATE is malformed:
    keep the rates in effect unchanged; done
  layer_up := Applied Rate Layer (5), or 3 if absent
  layer_down := Applied Rate Layer Downstream (6), or layer_up
  ovh_up := Per-Packet Overhead (7), or local configuration
            or defaults if absent
  ovh_down := Per-Packet Overhead Downstream (8), or ovh_up
  advertised rates := Advertised Rates (1, 2), if present
    (information only, e.g., for display)
  for each direction (upstream, downstream):
    rate := Applied Rate (3, 4) of the direction, or 0
    if rate is 0:
      target := default configuration
    else if layer is invalid or the rate is not applied
        (e.g., not supported, or disabled by the
        administrator):
      target := default configuration
      treat the rate as information only
    else:
      rate := min(rate, capacity of the physical link)
      target := rate with the overhead of the direction,
        converted to the shaping interface, if supported;
        otherwise rate, optionally reduced by a margin
    if target differs from the configuration in effect:
      apply target to the direction

On termination of the link or session:
  revert to the default configuration
~~~
{: #fig-example-processing title="Example Receiver Processing"}

A client typically applies only the upstream direction and uses the downstream rate as
information only (see {{application}}).

# Change Log
{:removeinrfc="true"}

## Since draft-ietf-intarea-dhcp-rate-signaling-00

The following changes are based on the feedback received during the presentation at the last IETF
meeting, the subsequent mailing list discussions, and the GitHub issues of this document.

Motivation and scope:

- Restructured the Introduction into Problem Statement, Solution Overview, and Applicability,
  explaining upstream policing at the BNG, other upstream bottlenecks, and the limitations of
  prioritization at the CPE; added a figure.
- Added the Design Rationale appendix addressing frequently raised questions, including the relation
  to access line information and to existing proprietary solutions.
- Added applicability to local area networks.

Protocol changes (not backward compatible with -00):

- Replaced the informational rate type with the new Advertised Rate sub-options (codes 1 and 2),
  conveying informational rates such as the contracted rate.
- Renamed the Available Rate sub-options to Applied Rate (codes 3 and 4); they always convey rates
  that are intended to be applied.
- Replaced the Rate Type sub-option by the Applied Rate Layer sub-options (codes 5 and 6), carrying
  the layer number (2 or 3) without an IANA registry. The layer values 2 and 3 are unchanged, the
  informational value 0 was removed, and the default changed from Layer 2 to Layer 3.
- Added the optional Per-Packet Overhead sub-options (codes 7 and 8) with overhead, minimum packet
  unit, and cell-based accounting.
- Added per-direction overrides for the Applied Rate Layer and the Per-Packet Overhead.
- Specified that a Layer 2 rate refers to the frames on the link of the receiver.

Processing and behavior:

- Added the sections Traffic Management Using Applied Rates, Rate Processing, Minimal
  Implementation, Client Hints, Appearance in Messages, and Error Handling.
- Clarified the requirement levels for applying rates, the precedence of administrator
  configuration, and the behavior of servers, relay agents, and snooping devices.

Other:

- Changed the intended status to Standards Track; replaced RFC 8415 by RFC 9915.
- Extended the Security Considerations; added Privacy Considerations, Operational Considerations,
  Implementation Status, and appendices with example parameters and example receiver processing.
- Added an appendix on deriving the parameters from access loop characteristics {{?RFC4679}}
  {{?RFC6320}}, and explained the use of 64-bit rates.
- Addressed IANA and RFC Editor review comments, including the option names OPTION_V4_RATE and
  OPTION_V6_RATE.

# Acknowledgments
{:numbered="false"}

The authors would like to thank Glenn Deen, Jason Livingood, and
Sebastian Moeller for their valuable review comments and discussion,
which helped to significantly improve the clarity,
applicability, and operational guidance of this document.
