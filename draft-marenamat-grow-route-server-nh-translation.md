---
title: "Route Server Next Hop Translation"
#abbrev: "TODO - Abbreviation"
category: std

docname: draft-marenamat-grow-route-server-nh-translation-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Operations and Management Area"
workgroup: "Global Routing Operations"
keyword:
 - coexistence of IPv4 and IPv6
 - ARP proxying
 - address translation
venue:
  group: "GROW (Global Routing Operations)"
  type: Working Group
  mail: grow@ietf.org
  github: marenamat/ietf-draft-marenamat-grow-route-server-nh-translation

author:
  - name: Maria Matejka
    org: CZ.NIC
    street: Milesovska 1136/5
    city: Praha
    code: 13000
    country: Czechia
    email:
    - maria.matejka@nic.cz
    - mq@jmq.cz

normative:
  RFC4271: bgp
  RFC6890: special-purpose-ip
  RFC7947: internet-exchange
  RFC8950: bgp-mixed-nh
  RFC9161: peering-evpn-arp-proxy
  I-D.chroboczek-intarea-v4-via-v6: mixed-nh

informative:
  I-D.schoen-intarea-unicast-240: schoen-240
  DE-CIX-EVPN:
    title: Peering LAN 2.0 — Introduction of EVPN at DE-CIX
    target: https://blog.apnic.net/2023/08/16/peering-lan-2-0-introduction-of-evpn-at-de-cix/
    date: 2023-08-23
    author:
      ins: T. King
      name: Dr. Thomas King
      org: DE-CIX


--- abstract

Internet Exchange BGP Route Server (RFC 7947) is an interconnection broker
for three or more External BGP speakers on a shared LAN.

To support IPv6 Next Hops for IPv4 NLRIs (RFC 8950) on an Internet Exchange,
traditionally, all BGP speakers connected to the Route Server must support it.

This document defines how to allow coexistence of speakers supporting RFC 8950
with others not supporting it.


--- middle

# Introduction

Traditionally, Internet Exchange BGP Route Servers {{-internet-exchange}}
serve IPv6 NLRIs with IPv6 next hops, and IPv4 NLRIs with IPv4 next hops.

Recently, there have been networks running IPv4 only on end hosts and
forwarding the IPv4 traffic over IPv6-only intermediate hosts {{-mixed-nh}}.
In the Internet Exchange environment, however, these networks need an IPv4 address
assigned to allow routing from and to legacy-only networks where IPv6 nexthops
for IPv4 NLRIs {{-bgp-mixed-nh}} are not supported.

This document specifies how to extend the ARP Proxy {{-peering-evpn-arp-proxy}}
functionality to allow deployment of IPv6 next hops for IPv4 NLRIs
{{-bgp-mixed-nh}} in networks where some BGP speakers do not support that,
without the need to assign public IPv4 addresses to all BGP speakers.

This document does not cover IPv6 NLRIs with IPv4 next hops.

# Conventions and Definitions

The terminology of {{-peering-evpn-arp-proxy}}, {{-internet-exchange}}
and {{-bgp}} applies.

Client:
: A BGP speaker which is connected to the IXP's Route Server. The client
  may be a Legacy speaker, Supporting speaker or Unnumbered speaker.

Legacy speaker:
: Any BGP speaker with no support for IPv4 NLRIs with IPv6 next hops
  in context of an IXP.

Supporting speaker:
: Any BGP speaker with support for IPv4 NLRIs with IPv6 next hops,
  and with an assigned IPv4 address.

Unnumbered speaker:
: Any BGP speaker with support for IPv4 NLRIs with IPv6 next hops,
  with no IPv4 address assinged.

{::boilerplate bcp14-tagged}

# Providing reachability between Legacy and Unnumbered speakers

All IPv4 routes announced to and from Legacy speakers must have IPv4 next hops,
while all IPv4 routes announced to and from Unnumbered speakers must have IPv6
next hops.  To facilitate reachability between these speakers, we need to
translate between IPv4 and IPv6 next hops in BGP, IPv6 ND and ARP.

## Speaker configuration

All speakers SHOULD have a fixed MAC address set and registered with the IXP.

All speakers MUST have their IPv6 LLA and IPv6 GUA and assigned
by the IXP. They do not have to set these addresses up on the respective interfaces
as long as their BGP sessions are able to run.

These assignments MUST be unique, such that for any two triples
`(MAC, IPv6 LLA, IPv6 GUA)` and `(MAC', LLA', GUA')` it holds
that `MAC != MAC'`, `LLA != LLA'` and `GUA != GUA'`.

This set of triplets is called Local Address Table (LAT).

## IPv4 assignment

Contrary to IPv6 where both the LLA and GUA space allow for sharing the same
prefix, in IPv4 this isn't always possible as the IXP may run out of global IPv4
addresses for the number of speakers present in the local network.

Therefore, the IXP, in cooperation with every Supporting Speaker and Legacy Speaker,
MUST decide on an IPv4 prefix (or a set of IPv4 prefixes) short enough to
accomodate the number of speakers in the IXP network. This prefix MAY be
different for different clients. This prefix is called Speaker-specific local prefix.

For every particular speaker, the IXP then assigns an IPv4 address from the
Speaker-specific local prefix for every triplet in the LAT, creating a Specific
Local Address Table (SLAT).

The Unnumbered Speakers need no such allocation.

Legacy speakers SHOULD set up their NEXT_HOP Attribute handling so that they
never propagate the SLAT IPv4 addresses.

## ARP and ND Proxy configuration

For each speaker, the IXP MUST set up ARP and ND snooping. The IXP MUST NOT
forward any ARP nor ND traffic between speakers. The IXP MUST answer
all ARP and ND requests from the speakers themselves, using the appropriate SLAT
for that speaker.

## NEXT_HOP Attribute management on Route Servers

When the Route Server receives a route where the NEXT_HOP Attribute contains
When a route with IPv4 NLRI and IPv4 NEXT_HOP Attribute is received from any
speaker, the Route Server MUST rewrite the NEXT_HOP according to the sender's
SLAT to the IPv6 GUA or LLA.

When the Route Server sends a route to a Legacy speaker, it MUST rewrite
the NEXT_HOP according to the receiver's SLAT to the assigned IPv4 address.

When the Route Server sends a route to a Supporting speaker, it SHOULD NOT
rewrite the NEXT_HOP. When the Route server sends a route to an Unnumbered speaker,
it MUST NOT rewrite the NEXT_HOP.

The Route Server MUST NOT propagate any route where the NEXT_HOP Attribute
holds an address not assigned to any speaker by the appropriate SLAT.

{{Section 2.2.1 of -internet-exchange}} does not apply.

# IXP Interconnection Space {#ixp-interconnection-space}

This document requests an allocation of an IPv4 IXP Interconnection Space
from the experimental range. By previous efforts {{-schoen-240}}, it has already
been shown that these addresses are technically feasible to be used in limited
environments. Here, the use is limited for local next hop resolution and possibly
BGP session addressing.

It is RECOMMENDED that the prefix used is the IXP Interconnection Space for every
speaker supporting this allocation, to reduce the size of the SLATs.

BGP speakers MUST NOT propagate any routes with IPv4 NLRI from the
IXP Interconnection Space.

{{Section 2.2.2 of -special-purpose-ip}} is updated by adding the following
record:

+----------------------+---------------------------+
| Attribute            | Value                     |
+----------------------+---------------------------+
| Address Block        | TBD                       |
| Name                 | IXP Interconnection Space |
| RFC                  | TBD                       |
| Allocation Date      | TBD                       |
| Termination Date     | N/A                       |
| Source               | False                     |
| Destination          | False                     |
| Forwardable          | False                     |
| Global               | False                     |
| Reserved-by-Protocol | False                     |
+----------------------+---------------------------+
Table 3: Shared Address Space

Following the claims in {{-schoen-240}}, there are already networks which have
completely exhausted all the private space addresses, and some of them have
already been squatting the experim

# Operational and Management Considerations

This setup should be possible to be rolled out in steps. First, the ARP and ND
snooping is not dependent on anything else in this document. Then, setting up
a new route server supporting IPv6 next hops for IPv4 NLRI, and allowing Supporting
clients to use that server while keeping also the traditional one.

The SLATs may be started as uniform for every client reflecting the current address
assignment, allowing the Legacy Speakers into the new route server, and gradual
renumbering may occur later, client-by-client, when the original IPv4 range starts
being exhausted.

The clients have to properly assess which address range is suitable for them to use
for IXP interconnection. If using the IXP Interconnection Space, they also have to
check whether these addresses are considered eligible as next hops by their routing
equipment.

The IXPs may have to rethink how they are displaying the route next hops in
their human-facing interfaces (looking glasses). It may be handy to display
the original next hop (if it was IPv4), the actual IPv6 next hop, and also
the result of the egress translation for a selected client.

# Security Considerations

Implementing the ARP and ND snooping should improve the overall security of IXPs
by blocking possible ARP or ND spoofing, both inadvertent and intended. {{DE-CIX-EVPN}}

Mistakes in the MAC address registration and manual management of IP address assignment
may lead to inadvertent invalid route announcement. It's recommended to run automated
address management with a single source of truth.

Mistakes in the next hop address translation may lead to inadvertent invalid
route announcement. It's recommended to run periodic automated checks whether
the next hops actually resolve to the same address by the appropriate SLAT.

Mistakes in route announcements are contained to the route not being propagated further.

Mistakes in the client setup may lead to spreading unreachable routes across
their autonomous systems, causing inefficient routing.

It is recommended to log rogue GARP or IPv6 DAD communication to detect
possible misconfigurations.

# IANA Considerations

IANA is asked to record the allocation of an IPv4 /8 from the 240/4 range
for use as IXP Interconnection Space as requested in {{ixp-interconnection-space}}.

The IXP Interconnection Space address range is: x.0.0.0/8.

*\[Note to RFC Editor: this address range to be added before publication\]*


--- back

# Acknowledgments
{:numbered="false"}

TODO
