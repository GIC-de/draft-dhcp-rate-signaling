---
title: "DHCP Explicit Rate Signalling"
docname: draft-giese-dhcp-rate-signalling-00
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
scheduling, or Active Queue Management (AQM) {{?RFC7567}}. Relying on traffic
indiscriminate packet dropping or policing at the service edge severely degrades
the user experience. Conversely, network performance improves significantly by
performing intelligent shaping and prioritization. When combined with AQM and
the Low Latency, Low Loss, and Scalable Throughput (L4S) architecture {{?RFC9330}},
these localized traffic management benefits are further amplified.

In many IP over Ethernet (IPoE) {{TR101}} architectures the
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
to Point-to-Point Protocol over Ethernet (PPPoE){{?RFC2516}} environments. Since DHCP
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
   Broadband Service Provider for network access services. Within the network infrastructure.
   A subscriber is typically represented by an authenticated logical session (e.g., IPoE or PPPoE)
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

# Security Considerations

TODO Security

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
