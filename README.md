<!-- regenerate: off (set to off if you edit this file) -->

# DHCP Explicit Rate Signaling

This is the working area for the individual Internet-Draft, "DHCP Explicit Rate Signaling".

* [Editor's Copy](https://GIC-de.github.io/draft-dhcp-rate-signaling/#go.draft-giese-intarea-dhcp-rate-signaling.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-giese-intarea-dhcp-rate-signaling)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-giese-intarea-dhcp-rate-signaling)
* [Compare Editor's Copy to Individual Draft](https://GIC-de.github.io/draft-dhcp-rate-signaling/#go.draft-giese-intarea-dhcp-rate-signaling.diff)


## Abstract

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

## Contributing

See the
[guidelines for contributions](https://github.com/GIC-de/draft-dhcp-rate-signaling/blob/main/CONTRIBUTING.md).

The contributing file also has tips on how to make contributions, if you
don't already know how to do that.

## Command Line Usage

Formatted text and HTML versions of the draft can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed.  See
[the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).

