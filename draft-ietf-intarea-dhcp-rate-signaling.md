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

This document defines two options for the Dynamic Host Configuration Protocol for IPv6 (DHCPv6)
that signal the intended upstream and downstream rates of a network connection to clients. Each
rate applies to the combined IPv4 and IPv6 traffic of the corresponding logical interface. In
broadband access networks, the Customer Premises Equipment (CPE) typically sends at the speed of its
physical link while the upstream rate is enforced by policers at the service edge. By learning the
upstream rate, the CPE can shape its traffic and manage the queue using prioritization and active
queue management (AQM).

--- middle

# Introduction

## Problem Statement {#problem-statement}

In broadband access networks, the rate available to a subscriber is typically lower than the speed
of the physical link that connects the Customer Premises Equipment (CPE) to the network. The
service edge is typically a Broadband Network Gateway (BNG) {{TR101}}. The available rate can also
be limited by an access device connected to the CPE, such as an Optical Network Terminal (ONT)
{{G.984.1}}, a Digital Subscriber Line (DSL) modem, or a cable modem, or within the access
network, for example by the synchronization rate of a DSL line. The CPE is typically unaware of
these rates and sends at the speed of its physical link.

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

This document defines two DHCPv6 {{!RFC9915}} options: the Upstream Rate option and the Downstream
Rate option. Each carries one unsigned 64-bit integer: the intended rate for its direction in bits
per second (bit/s). Each rate counts the IP header and its payload at the IP layer of the link to
which the option applies (see {{layer3}}) and covers the combined IPv4 and IPv6 traffic of that
link (see {{dual-stack}}). This document specifies the encoding, transport, and meaning of the
options. The usage examples are illustrative; clients choose whether and how to use the
information according to local policy (see {{application}}).

A CPE that knows the upstream rate can use it to configure an upstream shaper. Queuing at the CPE
can then be managed using prioritization, AQM, and flow queuing {{?RFC8290}}. Where the queue at
the CPE implements an AQM that supports the Low Latency, Low Loss, and Scalable Throughput (L4S)
architecture {{?RFC9330}}, such as the DualQ Coupled AQM {{?RFC9332}}, it can additionally provide
low latency for L4S traffic, provided that the shaper at the CPE is the bottleneck. When the shaper
accounts for the bottleneck rate and burst tolerance, this can reduce policer drops and queuing in
the access network (see {{fig-policer}}).

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
deployments, including those that provide IPv4 service over IPv6, and a single protocol avoids
reconciling rates signaled by two protocols for the same traffic. The rates nevertheless cover IPv4
traffic (see {{dual-stack}}); DHCPv6 options that configure IPv4 service have precedent, for example
the Address Family Transition Router (AFTR) Name option for DS-Lite {{?RFC6334}} and the options for
Mapping of Address and Port (MAP) {{?RFC7598}}. Networks that do not use DHCPv6 on the link are
outside the scope of this document.

DHCPv6 is widely deployed on CPE, including customer-owned devices and devices running open-source
firmware. It provides an existing path for initial configuration and updates, complementing manual
configuration and management protocols such as TR-069 {{TR069}}.

The options are applicable to any access link on which DHCPv6 is used to configure the
network-facing interface and on which the intended rates are determined by a subscription, contract,
or policy, including enterprise access links. They convey provisioned, slowly changing values and do
not describe the instantaneous capacity of links whose capacity varies over time, such as wireless
or cellular links; on such links, they can serve as an upper bound. Signaling rates to relay agents
or other network nodes, and to end devices in local area networks, is outside the scope of this
document.

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

Layer 3 Rate
: A rate calculated over the IP header and its payload at the IP layer of the link to which the
  Rate Option applies (see {{layer3}}).

Rate Option
: The Upstream Rate option (OPTION_UPSTREAM_RATE) or the Downstream Rate option
  (OPTION_DOWNSTREAM_RATE) defined in this document. Where the distinction is not relevant, this
  document refers to either as a Rate Option.

Subscriber
: The customer of a network access service. Within the network, a subscriber is typically
  represented by a subscriber session at the BNG, for example an IPoE or PPPoE session, to which the
  subscribed rates and other per-subscriber policies are applied.

Upstream
: The direction of traffic from the client toward the network, sent by the client over the link on
  which the Rate Option was received.

# Rate Semantics {#semantics}

## Rate Values {#values}

Each Rate Option carries a single rate as an unsigned 64-bit integer in network byte order
(big-endian), expressing the rate in bits per second (bit/s), not in bytes (octets) per second.
Throughout this document, Mbit/s and Gbit/s denote 10^6 and 10^9 bits per second, respectively. The
largest representable rate is 2^64 - 1 bit/s (18,446,744,073,709,551,615 bit/s), approximately
18.45 Ebit/s, where Ebit/s denotes 10^18 bits per second.

The value 0 is not a valid rate. A server MUST NOT send a Rate Option with the value 0. A direction
for which no rate is signaled is indicated by the absence of the respective option.

The two options are independent: a server can send either or both, and the presence or absence of
one does not affect the processing of the other.

## Layer 3 Rates {#layer3}

Servers and clients MUST interpret each rate as a Layer 3 rate: the number of bits per second
counted over the IP header and its entire payload, including IP options, IPv6 extension headers,
and transport headers. The reference is the IP layer of the physical or logical link on which the
DHCPv6 exchange takes place.

This reference remains the same when the link is carried over a tunnel. If DHCPv6 runs inside a
tunnel, the rate refers to the IP packets on that logical link, excluding the tunnel's carrier
headers. If the link instead carries tunneled data, such as IPv4-in-IPv6 traffic from a Dual-Stack
Lite (DS-Lite) basic bridging broadband (B4) element {{?RFC6333}}, the IPv6 header and its
encapsulated payload are counted. Thus the reference is the IP layer of the DHCPv6-configured link,
not necessarily the outermost IP header on the underlying physical interface. Translation is
accounted for as seen on the reference link.

Layer 3 accounting excludes link-layer headers, trailers, padding, and physical-layer overhead.
Virtual LAN (VLAN) tags or PPPoE and PPP headers added or removed along the path do not change the
rate. IP encapsulation or translation between the client and the enforcement point can change the
counted packet size, which operators need to take into account when selecting the signaled rate.

For 1500-octet TCP/IP packets without options or extension headers, TCP payload throughput is
approximately 2% to 4% lower than the Layer 3 rate due to the IP and transport headers.

Traffic management that counts IP packet sizes can use the signaled rate directly. Traffic
management that counts link-layer frames needs to exclude the counted link-layer overhead to
reproduce Layer 3 accounting. This header compensation is local: only overhead actually counted by
the implementation is removed, including padding where applicable. For example, a shaper counting
an Ethernet header, one VLAN tag, PPPoE, and PPP, but no frame check sequence (FCS) or padding,
subtracts 14 + 4 + 6 + 2 = 26 octets per packet. Overhead inserted later by hardware is not
subtracted. Applying the same numeric rate without compensation yields a lower IP rate that
depends on packet size; it does not change the meaning of the signaled value.

The Layer 3 meaning of the Rate Options is fixed. Future documents can define additional options,
for example to signal link-layer overhead or rates at another layer; such options complement the
Rate Options and do not change their meaning. A client that does not support such an option
ignores it and uses the Rate Options as specified in this document.

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
accounts for their Layer 3 meaning ({{layer3}}) and for the traffic to which they apply
({{dual-stack}}).

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

## Option Placement {#placement}

The Rate Options are top-level options, not Identity Association (IA) sub-options. They can appear
in Advertise messages, where they are informational (see {{client}}), and in Reply messages sent in
response to Solicit, Request, Renew, Rebind, and Information-request. A server MUST NOT include
them in replies to Confirm, Release, or Decline, or in Reconfigure messages. A client MUST ignore a
Rate Option in any other placement or exchange.

Clients MUST NOT send Rate Options, and servers MUST ignore Rate Options received from clients.
This document does not define the use of Rate Options in the options field of Relay-forward or
Relay-reply messages.

Both options are singleton options ({{Section 21 of !RFC9915}}). A server MUST NOT include more
than one instance of the same Rate Option in a message. {{!RFC9915}} treats multiple instances of
an option as separate instances; for the Rate Options, this document additionally specifies that a
client that receives more than one instance of the same Rate Option MUST discard all instances of
that option and treat it as absent.

A Rate Option whose option-len is not 8, or whose value is 0, is invalid. A client MUST discard an
invalid Rate Option and treat it as absent. Invalid Rate Options do not invalidate otherwise valid
DHCPv6 configuration.

# Client Behavior {#client}

DHCPv6 clients that support the Rate Options SHOULD include both option codes in the Option Request
option (ORO) of every Solicit, Request, Renew, Rebind, and Information-request message. This
requests rate information and indicates support for receiving it; it does not indicate whether the
client will use the rates for traffic management.

A client MUST NOT apply rates received in any message other than Reply. Rates in an Advertise
message are informational and indicate the rates the server intends to supply; see {{security}}
regarding their use in server selection.

The Rate Options are configuration information received from the server and are processed like
other non-IA configuration options of {{!RFC9915}}:

- A valid Rate Option in a Reply to Solicit (with Rapid Commit), Request, Renew, Rebind, or
  Information-request replaces the previously received rate for its direction, regardless of which
  of these exchanges supplied the previous rate.
- If a Reply to Renew, Rebind, or Information-request does not include a requested Rate Option that
  the client previously received, the client SHOULD stop using the rate for that direction, as
  specified in {{Section 18.2.10.5 of !RFC9915}}. This applies regardless of which exchange
  supplied the rate; a client does not need to track the source of a rate.
- The rates are part of the client's DHCPv6 configuration and are not associated with individual
  bindings. The client discards them when it discards that configuration, for example because all
  addresses and delegated prefixes have become invalid, because it switches to another server, or
  because the underlying link or PPP session terminates. Clients that use only Information-request
  refresh them as specified in {{Section 21.23 of !RFC9915}}.

Replies to Confirm, Release, or Decline, and Replies that contain a top-level Status Code option
({{Section 21.13 of !RFC9915}}) with a value other than Success, do not change the rates; the effect
of binding changes is covered by the rules above. A client MUST NOT treat a rate that it has
stopped using as current DHCPv6-provided information; subsequent use of local defaults remains a
local decision.

# Server Behavior {#server}

When a DHCPv6 server that supports the Rate Options receives a Solicit, Request, Renew, Rebind, or
Information-request message whose ORO includes the code of a Rate Option, and a rate is configured
for the client for the respective direction, the server MUST include that Rate Option in the
resulting Advertise and Reply messages. If no rate is configured for a direction, the server omits
the respective option; for a client that previously received it, this withdraws the rate as
specified in {{client}}. Where several servers can respond to a client, for example after a Rebind,
operators need to configure the rates consistently on all of them; otherwise, a Reply from a server
without that configuration withdraws the rates.

The server MAY derive the rates from local configuration profiles, from Authentication,
Authorization, and Accounting (AAA) systems such as Remote Authentication Dial-In User Service
(RADIUS) {{?RFC2865}}, or from external policy servers.

Where the rates are provisioned via RADIUS, they can be carried as Rate Options in the
DHCPv6-Options attribute {{!RFC9445}}, in which both options are permitted (see {{iana}}). Where the
BNG operates as a relay agent and a separate DHCPv6 server configures the client, the BNG can
convey this attribute to the server in the RADIUS option {{!RFC7037}}. As the omission of a
requested Rate Option withdraws the rate ({{client}}), such a relay agent SHOULD include the RADIUS
option in every Relay-forward message relaying a Solicit, Request, Renew, Rebind, or
Information-request from the client, unless the server retains the rates for the client. In all
cases, the server includes the Rate Options in its responses as specified in this section.

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

As the signaled rates are Layer 3 rates, a CPE that attaches its shaper to the PPP interface applies
them directly, without accounting for the PPPoE, PPP, Ethernet, or VLAN headers.

If a client receives rate information from other sources, for example within PPP authentication
messages, precedence is a matter of local policy. DHCPv6 can refresh the information during a
session.

# Operational Considerations {#operations}

By default, a server does not send Rate Options; rates are signaled only when configured by the
operator. Implementations are expected to allow operators to inspect the rates in effect, for
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
Choosing not to apply a value does not change its Layer 3 meaning. Clients SHOULD NOT use signaled
rates as the sole criterion for selecting a server or network interface.

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
of the service contract.

# IANA Considerations {#iana}

RFC Editor: please replace TBD1 and TBD2 throughout this document with the values assigned by IANA,
replace "RFC XXXX" with the RFC number assigned to this document, and remove this note.

IANA is requested to assign two new option codes in the "Option Codes" registry within the
"Dynamic Host Configuration Protocol for IPv6 (DHCPv6)" registry group {{IANA-DHCPV6}} as
follows:

| Value | Description            | Client ORO | Singleton Option | Status | Reference |
| :---- | :--------------------- | :--------- | :--------------- | :----- | :-------- |
| TBD1  | OPTION_UPSTREAM_RATE   | Yes        | Yes              |        | RFC XXXX  |
| TBD2  | OPTION_DOWNSTREAM_RATE | Yes        | Yes              |        | RFC XXXX  |
{: #tab-iana title="DHCPv6 Option Codes"}

IANA is requested to add the following entries to the "DHCPv6 Options Permitted in the RADIUS
DHCPv6-Options Attribute" registry within the "Dynamic Host Configuration Protocol for IPv6
(DHCPv6)" registry group {{IANA-DHCPV6}}, as specified in {{Section 8.4.1 of !RFC9445}}:

| Value | Description            | Reference |
| :---- | :--------------------- | :-------- |
| TBD1  | OPTION_UPSTREAM_RATE   | RFC XXXX  |
| TBD2  | OPTION_DOWNSTREAM_RATE | RFC XXXX  |
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
  SQM overhead settings provide the header compensation described in {{layer3}}.

Shell arithmetic on OpenWrt uses signed 64-bit integers, so a script-based implementation needs to
handle values above 2^63 - 1, for example by treating them as exceeding any local limit. A native
implementation could instead decode the options in odhcp6c and expose the rates as interface data
via netifd, which also makes them available to other components, such as the web interface.

# Design Rationale {#rationale}
{:removeinrfc="true"}

This appendix explains major design decisions to the working group. Permanent rationale is
included in the relevant sections of this document.

## DHCPv6 Only {#rationale-dhcpv6}

Earlier versions of this document defined the Rate Option for both DHCPv4 and DHCPv6. This version
defines options for DHCPv6 only; the permanent rationale is given in {{applicability}}.

- DHCPv6 is available where the mechanism is needed. Broadband CPE commonly use DHCPv6 for prefix
  delegation in both IPoE and PPPoE deployments, so the options reach the target devices without
  a second protocol.
- IPv4-only deployments of the new options are unlikely. Providers and CPE vendors that implement
  new DHCP options for current services are very likely to support IPv6 as well, so an IPv4-only
  variant would add specification and implementation effort for a small and shrinking set of
  deployments.
- A single protocol removes considerable complexity. Signaling the same rates through DHCPv4 and
  DHCPv6 required rules for selecting between the two protocols, for keeping their values
  consistent, and for withdrawal and lifetime that differed between the protocols. These rules
  are no longer needed.
- IPv4 traffic remains covered. The rates describe the service on the logical interface, not an IP
  version, and apply to the combined IPv4 and IPv6 traffic, whether IPv4 is provided natively,
  configured by DHCPv4 or IPCP, or carried over IPv6 (see {{dual-stack}}).
- The DHCPv6 option space allows a simpler format. DHCPv4 option codes are scarce, which motivated
  a single option carrying both directions. With DHCPv6, separate Upstream Rate and Downstream Rate
  options make each direction independent and allow withdrawal by omission instead of a reserved
  zero value.
- Access networks that do not use DHCPv6, such as IPv4-only access, and signaling to end devices in
  local area networks are outside the scope of this document.
- If the working group identifies a need, a DHCPv4 option can be defined in a separate document
  that reuses the semantics specified in {{semantics}}.

The authors invite feedback from the working group on whether a DHCPv4 option is needed.

## Overhead Signaling {#rationale-layer3}

The accounting reference of the signaled rates has been discussed at length. Proposals included
signaling Layer 2 rates instead of Layer 3 rates, signaling the per-packet link-layer overhead
together with the rates, and a selector indicating the layer at which a rate is counted. Earlier
versions of this document included such a layer selector. This version defines a single Layer 3
reference (see {{layer3}}) for the following reasons:

- Layer 3 is the most common case. A CPE typically attaches its shaper to the IP interface, such as
  the PPP interface or the IPoE interface, where packet sizes are counted at the IP layer, so the
  signaled rate can be applied directly.
- A Layer 3 rate is independent of the encapsulation along the path. VLAN tags and PPPoE and PPP
  headers are added and removed between the CPE and the enforcement point, so the link-layer
  overhead seen by the CPE generally differs from the overhead counted at the enforcement point.
  A Layer 2 rate would only be meaningful together with a precise definition of the link and
  encapsulation to which it refers.
- Overhead is often not known to the server. The per-packet overhead counted at the bottleneck
  depends on the access technology, framing, padding, and implementation, which a DHCPv6 server
  frequently cannot determine. Signaling an inaccurate overhead would not improve on a
  conservative Layer 3 rate.
- A single reference avoids ambiguity. With one fixed meaning, all clients interpret a value
  identically, and no negotiation or capability signaling is needed. Implementations that count
  link-layer frames apply local header compensation as described in {{layer3}}, and operators
  account for overhead at the enforcement point, for example by allowing headroom or by signaling a
  conservative rate.

Layer 3 remains the default reference of the options defined in this document and is not changed
by any extension. If the working group identifies a need, signaling of Layer 2 rates or of
link-layer overhead, for example the per-packet overhead and framing of the bottleneck link, can be
defined in a separate document as additional options that complement the Rate Options defined
here.

The authors invite feedback from the working group on whether such an extension is needed.

## Relay-Directed Rates {#rationale-relay}

Intermediate versions of this document allowed the server to signal separate rates to relay agents
in Relay-reply messages, so that, for example, a BNG or an access node operating as a Lightweight
DHCPv6 Relay Agent (LDRA) could configure per-subscriber shapers and policers with values that
differ from those signaled to the client. This mechanism has been removed from this document for
the following reasons:

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
  Removed the DHCPv4 option, DHCPv6 snooping, rates signaled to relay agents, and the LAN,
  enterprise, and variable-capacity scenarios (see {{rationale}}).
- Replaced the sub-option format with two fixed-format options, Upstream Rate and Downstream Rate,
  each an unsigned 64-bit value in bit/s. The value 0 is invalid; omission withdraws a rate. Not
  backward compatible with -00.
- Defined a fixed Layer 3 rate for the combined IPv4 and IPv6 traffic of the DHCPv6-configured
  link, with local header compensation; future options can complement but not change it.
- Made the use of the rates a local decision and aligned client processing with RFC 9915,
  independent of the exchange that supplied a rate.
- Added optional provisioning via RADIUS (RFC 9445, RFC 7037) and the corresponding IANA
  registration.
- Condensed operational, security, privacy, and IANA considerations; removed the implementation
  example; added OpenWrt applicability and a design rationale appendix.

# Acknowledgments
{:numbered="false"}

The authors would like to thank Glenn Deen, Jason Livingood, and Sebastian Moeller for their
valuable review comments and discussion, which helped to significantly improve the clarity,
applicability, and operational guidance of this document.
