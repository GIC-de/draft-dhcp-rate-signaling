---
title: "DHCP Explicit Rate Signalling"
docname: draft-giese-dhcp-rate-signalling-latest
submissiontype: IETF
date: 2026-05-22
category: info
v: 3

venue:
   github: GIC-de/draft-dhcp-rate-signalling

author:
-  fullname: "Christian Giese"
   organization: "RtBrick"
   email: "christian@rtbrick.com"
-  fullname: "Richard Patterson"
   organization: "Sky UK"
   email: "Richard.Patterson@sky.uk"

informative:
   TR101:
      title: "Migration to Ethernet-Based Broadband Aggregation"
      author:
      - org: Broadband Forum
      date: 2011-07
      seriesinfo:
         TR: 101 Issue 2
   TR069:
      title: "CPE WAN Management Protocol"
      author:
      - org: Broadband Forum
      date: 2020-06
      seriesinfo:
         TR: 069 Issue 6
   G.984.1:
      title: "Gigabit-capable passive optical networks (GPON): General characteristics"
      author:
      - org: ITU-T
      date: 2008-03
      seriesinfo:
         ITU-T Recommendation: G.984.1

--- abstract

This document defines new Dynamic Host Configuration Protocol (DHCP)
options for both DHCPv4 and DHCPv6 to explicitly
signal available upstream and downstream data rates. In many broadband
access networks, Customer Premises Equipment (CPE) and intermediate
nodes lack visibility into the subscriber's provisioned service tier.
By communicating these capacities natively via DHCP, clients, relay agents,
and snooping switches can dynamically configure localized traffic shaping
and queuing. This explicit signaling improves overall network performance
by reducing the reliance on indiscriminate packet dropping and policing at the
service edge. Additionally, it provides the necessary capacity awareness
to enable effective Active Queue Management (AQM) and the Low Latency,
Low Loss, and Scalable Throughput (L4S) architecture.

--- middle

# Introduction

In typical broadband access networks, the Customer Premises Equipment (CPE)
is often unaware of the actual available data rates. This lack of visibility
often occurs when an external modem or Optical Network Terminal (ONT) {{G.984.1}}
connects to the CPE at a physical link speed that significantly exceeds the
subscriber's provisioned service rate. Furthermore, operators commonly deploy
unified access profiles where the available rate is artificially limited at
the service edge, such as the Broadband Network Gateway (BNG) {{TR101}},
to match the subscriber's purchased service tier.

When the network bottleneck resides between the BNG and the CPE, intermediate
devices are typically not well equipped to provide deep buffering, priority
scheduling, or Active Queue Management (AQM) {{?RFC7567}}. Relying on indiscriminate
packet dropping or policing at the service edge severely degrades the user experience.
Conversely, network performance improves significantly by performing intelligent shaping
and prioritization. When combined with AQM and the Low Latency, Low Loss, and
Scalable Throughput (L4S) architecture {{?RFC9330}},
these localized traffic management benefits are further amplified.

In many IP over Ethernet (IPoE) {{TR101}} architectures, the
Broadband Network Gateway (BNG) operates strictly as a DHCP relay agent.
While per-subscriber traffic management policies, such as queues, shapers,
and policers, are typically provisioned out-of-band via
Authentication, Authorization, and Accounting (AAA) protocols like RADIUS {{?RFC2865}},
deployments lacking direct AAA integration at the service edge require an in-band
signaling alternative.

Transporting available data rates natively within DHCP options addresses this
architectural constraint. This mechanism allows the BNG to dynamically instantiate
downstream shapers and upstream policers based on the DHCP payload, while concurrently
equipping the Customer Premises Equipment (CPE) with the parameters required to manage
upstream traffic locally.

Furthermore, intermediate Layer 2 access nodes performing DHCP snooping can passively
extract these explicitly signaled rate parameters to optimize their local queues and
shapers. By distributing capacity awareness across the entire forwarding path, this
in-band signaling mitigates buffer congestion within the access segment and significantly
improves end-to-end transport performance.

While auto-configuration protocols such as TR-069 {{TR069}} can provision rate
information, they are not universally deployed by all service providers. Furthermore,
the increasing prevalence of customer owned, unmanaged CPEs limits the effectiveness
of operator-managed configuration servers. A standardized DHCP option addresses this
gap by providing a universal mechanism to explicitly signal available data rates
directly from the DHCP server, across BNGs and access nodes, down to the CPE.
This localized approach is particularly advantageous because it serves the entire path,
whereas auto-configuration servers exclusively target the end device. This method also
integrates seamlessly with architectures where RADIUS servers inject DHCP options.

Although primarily designed for IPoE deployments, this mechanism is equally applicable
to Point-to-Point Protocol over Ethernet (PPPoE) {{?RFC2516}} environments. Since DHCP
is frequently utilized over PPPoE, most notably for IPv6 Prefix Delegation. This option
provides a standardized method for rate signaling. Consequently, this approach can supersede
the fragmented, proprietary methods currently in use, such as embedding rate limits within
PPP {{?RFC1661}} authentication reply messages.

Conveying rates via DHCP natively supports dynamic updates through regular lease renewals
or triggered reconfiguration requests. As an auxiliary benefit, this explicit rate
information can be exposed in the CPE user interface to satisfy customer expectations
regarding bandwidth visibility. Ultimately, introducing a DHCP option to signal available
data rates represents a simple, standardized enhancement that yields widespread improvements
in internet service delivery.

Operational requirements necessitate the definition of this option for both DHCPv4 and DHCPv6.
This ensures coverage for networks lacking IPv6 support and prevents configuration gaps in
dual-stack scenarios where DHCPv4 is established prior to DHCPv6.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document makes use of the terms defined in {{!RFC2131}} and {{!RFC8415}}.
The following additional terms are used:

DHCP
: The abbreviation DHCP is used throughout this document to refer to both DHCPv4 and DHCPv6 protocols.

DHCPv4
: Dynamic Host Configuration Protocol {{!RFC2131}}

DHCPv6
: Dynamic Host Configuration Protocol for IPv6 {{!RFC8415}}

Subscriber
: >
   The individual, organization, or entity that maintains a contractual relationship with a
   Broadband Service Provider for network access services. Within the network infrastructure,
   a subscriber is typically represented by an authenticated logical session (e.g., IPoE or PPPoE)
   and an associated policy profile that dictates service attributes, including provisioned
   upstream and downstream data rates.


# DHCP Rate Option

The DHCP Rate Option specified in this document employs a unified sub-option structure for
both DHCPv4 and DHCPv6, utilizing the format explicitly known from DHCPv4 Option 82
(the Relay Agent Information Option) {{!RFC3046}}. The top-level option encapsulation strictly
conforms to the requirements of each base protocol. To simplify cross-protocol implementation,
this document proposes the uniform assignment of OPTION_RATE with the option code TBD1
(value to be assigned by IANA) for both IP versions. Specifically, DHCPv4 utilizes an 8-bit option
code set to TBD1 alongside an 8-bit length field, whereas DHCPv6 utilizes a 16-bit option code
set to TBD1 alongside a 16-bit length field. Despite these differences in outer header sizing,
the internal payload remains completely identical. The encapsulated sub-options maintain
consistent 8-bit sub-option code and 8-bit length fields across both protocol versions,
ensuring common parsing and processing logic regardless of the underlying IP version.

## Sub-Options Format

The rate information field consists of a sequence of SubOpt/Length/Value tuples for each
sub-option, encoded in the following manner:

~~~
 SubOpt  Len     Sub-option Value
+------+------+------+------+------+------+--...-+------+
|  1   |   N  |  s1  |  s2  |  s3  |  s4  |      |  sN  |
+------+------+------+------+------+------+--...-+------+
 SubOpt  Len     Sub-option Value
+------+------+------+------+------+------+--...-+------+
|  2   |   N  |  i1  |  i2  |  i3  |  i4  |      |  iN  |
+------+------+------+------+------+------+--...-+------+
~~~

No "pad" sub-option is defined, and the Information field shall NOT be terminated with a 255
sub-option.  The length N of the Rate Information Option shall include all bytes of the sub-option
code/length/value tuples. The length N of the sub-options shall be the number of octets in only that
sub-option's value field.  A sub-option length may be zero.  The sub-options need not appear in
sub-option code order.

The initial assignment of DHCP Rate Sub-options is as follows:

| Sub-option Code | Length | Description                                        |
| ---------------:| ------:| -------------------------------------------------- |
|               1 |      8 | Available Rate Upstream in Bits per second (bps)   |
|               2 |      8 | Available Rate Downstream in Bits per second (bps) |
|               3 |      1 | Rate Type (L2 or L3)                               |

## Sub-Options

### Available Rate Upstream

The sub-option Available Rate Upstream defines the rate in Bits per second (bps) available from the
DHCP client towards the DHCP server direction. The rate format is a 64-bit unsigned integer in
network byte order.

~~~
 SubOpt   Len     Available Rate Upstream
+------+------+------+------+------+------+------+------+--
|  1   |   8  |  64 Bit bps
+------+------+------+------+------+------+------+------+--
~~~

A value of 0 in this context signifies an unrestricted rate. It can be interpreted as a request to
remove a previously set rate, thereby resetting to the device default configuration.

### Available Rate Downstream

The available rate downstream defines the rate in Bits per second (bps) available from the DHCP server towards the DHCP client direction. The rate format is a 64-bit unsigned integer in network byte order.

~~~
 SubOpt   Len     Available Rate Downstream
+------+------+------+------+------+------+------+------+--
|  1   |   8  |  64 Bit bps
+------+------+------+------+------+------+------+------+--
~~~

A value of 0 in this context signifies an unrestricted rate. It can be interpreted as a request to
remove a previously set rate, thereby resetting to the device default configuration.

### Rate Type

The rate type defines the networking layer to which the stated rates apply. The default value of 2 is defined as the Layer 2 rate, which signifies that the rate encompasses the entire Ethernet frame. Implementations SHOULD calculate this rate using the Ethernet header and all payload, excluding the Ethernet Frame Check Sequence (FCS) and Inter-Packet Gap (IPG).

~~~
 SubOpt   Len   Rate Type
+------+------+------+
|  1   |   1  | i    |
+------+------+------+
~~~

If the rate type is set to 3, the rate applies to Layer 3, encompassing only the IP header and its
payload. This rate calculation is frequently utilized by end-user speed test applications and is
often regarded as the marketable "product bandwidth". Its primary advantage is its independence from
the variable overhead introduced by differing numbers of VLAN tags or tunnel encapsulations.

In the absence of the Rate Type sub-option, the client MUST assume that the signaled values are
defined as the Layer 2 rate.

A client MAY include the Rate Type sub-option within its initial requests to serve as a hint to the
server regarding its preferred calculation method (e.g., requesting a Layer 3 rate instead of a
Layer 2 rate). Sending this hint is OPTIONAL for the client, and honoring the hint is OPTIONAL
for the server.

# DHCPv4

## DHCPv4 Rate Option

The DHCPv4 OPTION_RATE code is TBD1.

~~~
 Code   Len     Rate Information Field
+------+------+------+------+------+------+--...-+------+
| TBD1 | N    |  i1  |  i2  |  i3  |  i4  |      |  iN  |
+------+------+------+------+------+------+--...-+------+
~~~

The length (N) gives the total number of octets in the Rate Information Field, which is either zero
or longer than 2 bytes, which is the sub-options header length.

## DHCPv4 Client Behavior

DHCPv4 clients that support the DHCP Rate Option SHOULD include the corresponding OPTION_RATE code in
the Parameter Request List (PRL). This inclusion explicitly signals client support, enabling the DHCP
server to determine whether to include the rate parameters in its response. Furthermore, this
signaling provides network operators with visibility into client capabilities, which aids in
troubleshooting and resolving customer service quality complaints.

A client MAY include the DHCP Rate Option directly within its DHCPREQUEST messages to serve as a hint
to the server proposing their maximum data rates or preferred rate type (L2 or L3 rates). For
example, when a CPE device is connected via a 1 Gbps WAN interface to an external Optical Network
Terminal (ONT) or modem capable of exceeding 1 Gbps on the WAN side, the CPE can explicitly signal
this physical limitation to the service provider. This enables the network to align its shaping
parameters directly with the device's actual capacity, ensuring traffic does not exceed the physical
limit.

However, providing this hint is OPTIONAL. The manner in which a DHCP server processes or utilizes
these client-provided hints is implementation-specific and outside the scope of this document.
Because the server remains authoritative, the client MUST accept and apply the rate type ultimately
provided by the server in the DHCPACK message, regardless of the hint it originally sent.

Clients MUST ignore the OPTION_RATE when received within a message other than DHCPACK. If the
OPTION_RATE is present in a DHCPOFFER message, the client MUST NOT apply the specified rate limits to
its interfaces. However, a client MAY evaluate the rate information provided in a DHCPOFFER as a
selection criterion to prefer one server's offer over another.

## DHCPv4 Server Behavior

When a DHCPv4 server is configured to support the DHCP Rate Option and receives a client request
(e.g., DHCPDISCOVER or DHCPREQUEST) that includes the OPTION_RATE within the Parameter Request List
(PRL), the server MUST include the OPTION_RATE in the resulting DHCPOFFER and DHCPACK messages.

The server MAY derive the specific upstream and downstream rates and rate type from local
configuration profiles, centralized Authentication, Authorization, and Accounting (AAA) systems such
as RADIUS, or external policy servers. If no non-zero values are configured or signaled to be used,
the server MAY return rate values of 0.

A server MAY include the OPTION_RATE in its responses even if the client did not explicitly request
it via the Parameter Request List (PRL), provided the operator has explicitly configured the server
to forcefully inject the option to provision intermediate nodes, such as DHCP relay agents or Layer 2
snooping switches, which MAY drop these options before forwarding the message to the client.

## DHCPv4 Relay Agent Behavior

DHCPv4 Relay Agents, including L2 DHCPv4 Relay Agents {{TR101}}, MAY extract the OPTION_RATE from
DHCPACK messages traversing the network. Relay agents that perform localized traffic management MAY
utilize these extracted values to dynamically instantiate shapers and policers on their
subscriber-facing interfaces.

Furthermore, a relay agent MAY add, modify, or remove the OPTION_RATE before forwarding the DHCP
message to the client. This accommodates deployments where the relay agent (e.g. a BNG) is
responsible for policy enforcement and populates or overrides the OPTION_RATE based on subscriber
attributes retrieved directly from an external Authentication, Authorization, and Accounting (AAA)
server, such as RADIUS.

# DHCPv6

## DHCPv6 Rate Option

The DHCPv6 OPTION_RATE code is TBD1.

~~~
 Code          Len            Rate Information Field
+------+------+------+------+------+------+------+--...-+------+
| TBD1        | N           |  i1  |  i2  |  i3  |      |  iN  |
+------+------+------+------+------+------+------+--...-+------+
~~~

The length (N) gives the total number of octets in the Rate Information Field, which is either zero
or longer than 2 bytes, which is the sub-options header length.

## DHCPv6 Client Behavior

DHCPv6 clients that support the DHCP Rate Option SHOULD include the corresponding OPTION_RATE code in
the Option Request Option (ORO) {{!RFC8415}}. This inclusion explicitly signals client support,
enabling the DHCPv6 server to determine whether to include the rate parameters in its response.
Furthermore, this signaling provides network operators with visibility into client capabilities,
which aids in troubleshooting and resolving customer service quality complaints.

A client MAY include the OPTION_RATE with any or all sub-options within its DHCPv6 Request messages
to serve as a hint to the server proposing their maximum data rates or preferred rate type
(L2 or L3 rates).

Providing this hint is OPTIONAL. The manner in which a DHCPv6 server processes these client-provided
hints is implementation-specific. Because the server remains authoritative, the client MUST accept
and apply the rate type ultimately provided by the server in the REPLY message, regardless of the
hint it originally sent.

Clients MUST ignore the OPTION_RATE when received within a message other than REPLY. If the
OPTION_RATE is present in an ADVERTISE message, the client MUST NOT apply the specified rate limits
to its interfaces, however, the client MAY evaluate this early rate visibility as a selection
criterion to prefer one server's advertisement over another.

## DHCPv6 Server Behavior

A DHCPv6 server MAY embed the OPTION_RATE directly within a REPLY message encapsulated inside
RELAY-REPL messages to explicitly provision the end client. The server MAY include the option within
the RELAY-REPL message to target the corresponding relay agent, instructing it to apply rate limits
locally. The nested relay header architecture of DHCPv6 empowers the server to explicitly address
each relay agent in the path, in addition to the end client, ensuring precise targeting of signaling
parameters.

If network policy dictates localized traffic management at both the Customer Premises Equipment (CPE)
and the relay node, the server MAY include the OPTION_RATE at all encapsulation levels
simultaneously. When provisioning multiple levels, the server MAY supply different rate values to
each respective node. For example, an operator might configure a relay agent's upstream policer with
a slightly higher rate limit than the CPE's upstream shaper. This operational delta accommodates
minor traffic burstiness from the CPE and prevents premature packet drops at the intermediate access
node. This capability to independently target different nodes along the forwarding path is unique to
the nested relay header architecture of DHCPv6, as DHCPv4 lacks a comparable mechanism for addressing
multiple relay agents distinctly.

Furthermore, to dynamically update a client’s rate limits mid-lease, the server MAY utilize
RECONFIGURE messages to apply updates before the T1 timer expires. By triggering the client to
initiate a Renew or Information-request transaction, this mechanism allows the server to push newly modified rate parameters without waiting for timer expiration.

## DHCPv6 Relay Agent Behavior

DHCPv6 Relay Agents, including Lightweight DHCPv6 Relay Agents (LDRA) {{?RFC6221}}, MUST extract and
consume the OPTION_RATE from their corresponding Relay-Reply header. Because the DHCPv6 architecture
provides a dedicated signaling channel for intermediate nodes, relay agents MUST NOT passively
inspect the encapsulated client-facing REPLY payload to extract rate information. Relay agents that
perform localized traffic management MAY utilize these explicitly targeted values to dynamically
instantiate shapers, policers, or Active Queue Management (AQM) disciplines on their
subscriber-facing interfaces.

In many architectures, the Broadband Network Gateway (BNG) or similar intermediary device serves as
the authoritative policy enforcement point rather than a transparent relay. In such deployments, the
intermediary device MAY populate, modify, or remove the OPTION_RATE destined for the client. This
allows the network edge to inject dynamic rate parameters based on subscriber attributes retrieved
directly from an external Authentication, Authorization, and Accounting (AAA) server, such as RADIUS,
before the message reaches the downstream client.

# DHCP Snooping

DHCP snooping switches are typically deployed as intermediate Layer 2 devices that passively monitor
DHCP message exchanges to enforce security policies and build binding databases, all without
modifying the DHCP payloads. These devices MAY passively inspect the DHCP Rate Option within messages
destined for clients. By extracting these explicit rate parameters, snooping devices can dynamically
provision appropriate traffic shapers, policers, or hardware queues on the corresponding downstream,
client-facing ports.

# PPPoE

In Point-to-Point Protocol over Ethernet (PPPoE) {{?RFC2516}} architectures, the Customer Premises
Equipment (CPE) typically employs DHCPv6 over the PPP {{RFC1661}} link to request an IPv6
Delegated Prefix (IA_PD) {{!RFC8415}}. This encapsulated DHCPv6 exchange provides a standardized
transport mechanism for the explicit DHCP Rate Option. While less prevalent in modern deployments,
DHCPv4 transactions operating within a PPPoE session MAY similarly convey these rate options.

The foundational processing rules and client behavior for rate options received over PPPoE are
identical to those defined for IP over Ethernet (IPoE) environments.

Because the PPP session dictates the primary logical link state, the applied rate MUST revert to
the device's default configuration under two specific conditions. First, the client MUST reset the
applied limits if it receives a valid DHCP message explicitly signaling rate removal, such as an
empty Rate Option or a sub-option containing a rate value of zero. Second, the rate MUST be
implicitly revoked if the underlying PPPoE session itself is terminated.

# Errors and Conflicts {#errors}

Clients receiving conflicting rate information across DHCPv4 and DHCPv6 protocols SHOULD apply the
most recently received value.

Clients and servers MUST ignore unrecognized sub-option codes and continue processing the rest of the
Rate Option.

If multiple instances of the same sub-option code are present, the last instance MUST be processed.

A client may receive an OPTION_RATE indicating an Available Rate that exceeds the maximum physical
link speed of its upstream or downstream interfaces (e.g., signaling 2 Gbps to a CPE with a 1 Gbps
WAN port). In such scenarios, the client SHOULD cap the applied traffic shaping, policing, or Active
Queue Management (AQM) parameters to the maximum capacity of the physical link.

# Operational or Manageability Considerations {#operations}

Deploying explicit rate signaling via DHCP introduces several operational benefits and deployment
considerations for network management.

Implementations SHOULD expose the explicitly signaled, DHCP-learned rate parameters within
Customer Premises Equipment (CPE) management interfaces, such as the local Web User Interface (UI)
or remote management protocols. Providing end-users and operators with immediate visibility into the
locally provisioned service tier drastically reduces support inquiries related to perceived bandwidth
issues and improves overall user satisfaction.

As noted in {{<<errors}}, a client may receive an OPTION_RATE indicating an available rate that
exceeds the maximum physical link speed of its interfaces. In such scenarios, management interfaces
SHOULD expose both the originally signaled rate value and the effective, capped rate value applied by
the device. Additionally, implementations SHOULD log this discrepancy if logging facilities are
enabled. Capturing and exposing these specific events provides critical telemetry for network
operators, as they frequently indicate a mismatch between the subscriber's provisioned service tier
and their installed physical equipment.

# Security Considerations {#security}

DHCP messages are typically transmitted as plaintext and are unauthenticated. Consequently, the DHCP
Rate Options defined in this document are vulnerable to interception, modification, or spoofing by
on-path attackers. A malicious actor or a successfully deployed rogue DHCP server could inject
artificially low rate limits to severely throttle a client's connection, resulting in a localized
Denial of Service (DoS).

The severity of this specific risk is generally no greater than standard DHCP threat vectors, such as
rogue default gateway assignment, DNS hijacking, or IP pool exhaustion, which typically yield a much
more critical and immediate loss of service.

To mitigate the impact of malicious or malformed options, clients MAY implement basic sanity checks
and threshold validations before applying rate parameters. For example, clients MAY ignore downstream
or upstream rates that fall below a basic operational minimum (e.g., 1000 bps) to prevent complete
session starvation. Furthermore, as mandated in the client behavior specification, if a maliciously
injected rate is impractically high, the client implicitly mitigates this by capping the applied rate
to the physical link capacity.

# IANA Considerations

This document requests that IANA assign the same numeric value in both registries
for DHCPv4 and DHCPv6, if feasible.

## DHCPv4 Option

IANA is requested to assign a new DHCP Option Code (TBD1) in the "BOOTP Vendor Extensions and DHCP Options" registry for OPTION_RATE.

## DHCPv6 Option

IANA is requested to assign a new DHCPv6 Option Code (TBD1) in the "Option Codes" registry for OPTION_RATE.

## DHCP Rate Sub-Options Registry

IANA is requested to create a new registry titled "DHCP Rate Sub-Options".

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
