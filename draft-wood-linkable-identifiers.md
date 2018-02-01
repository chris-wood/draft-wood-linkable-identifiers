---
title: Linkable Identifiers
abbrev: linkable-identifiers
docname: draft-wood-linkable-identifiers-latest
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: C. A. Wood
    name: Christopher A. Wood
    org: Apple Inc.
    street: 1 Infinite Loop
    city: Cupertino, California 95014
    country: United States of America
    email: cawood@apple.com

normative:
    RFC0793:
    RFC2508:
    RFC6824:
    RFC6973:
    I-D.ietf-dnssd-privacy:
    Noise:
      title: The Noise Protocol Framework
      url: http://noiseprotocol.org/noise.pdf
      authors:
        -
          ins: Trevor Perrin

--- abstract

TODO

--- middle

# Introduction

Privacy is an increasingly important topic in networks and protocols. {{RFC6973}} defines 
correlation of information relevant to or associated with a specific user as a significant attack 
on privacy. Such information can be exposed at any network stack layer. For example, a MAC address, 
if not properly rotated or randomized, may be used to link network packets to a single device. Many 
standards suggest randomizing or otherwise rotating such identifiers on a regular basis. Often, 
such advice is followed by recommendations to change other identifiers, e.g., IP addresses, host 
names, etc., in unison. For example, Huitema et al. {{I-D.ietf-dnssd-privacy}} say, "it is 
important that the obfuscation of instance names is performed at the right time, and that the 
obfuscated names change in synchrony with other identifiers, such as MAC Addresses, IP Addresses or 
host names." Consider the following example where this advice is not followed, wherein an IP 
address is changed yet the MAC address is not. 

~~~
+---------------+        +---------------+
|      ...      |  ....  |      ...      |
+---------------+        +---------------+
| IP Address  A <---//---> IP Address  B |
+---------------+        +---------------+
| MAC Address A <--------> MAC Address A |
+---------------+        +---------------+

+------------------------------------------>
                  time
~~~

An adversary may trivially link these packets together. In this document, we outline simple rules 
that SHOULD be followed by protocol implementations to avoid such trivial linkability. We then 
survey protocols developed inside the IETF and out, and identify their sticky identifiers. Results 
were obtained by analyzing protocol documentation and specifications, and also scanning packet 
traces captured from protocols in practice on common systems.

# Identifier Scope and Threat Model

Not all packet or datagram identifiers are visible end-to-end. For example, MAC addresses are only
visible on local subnets. IP addresses are only visible between endpoints. (In systems such as Tor,
source and destination addresses change at each circuit hop.) Thus, threats to identifier linkability
depend on the threat model under consideration. Off-path adversaries are generally not a problem since
they do not have access to datagrams in flight. On-path adversaries may exist at various locations 
relative to an endpoint (sender or receiver) on a path, e.g., in a local subnet, as an intermediate router 
or middlebox between two endpoints, or as a TLS terminating reverse proxy. In this document, we categorize 
these three types of adversaries as follows:

1. Local: An on-path adversary belonging to the same local subnet as an endpoint, a switch.
2. Intermediate: An on-path adversary that observes datagrams in flight but does not
terminate a (TCP or TLS) connection, e.g., a middlebox or performance enhancing proxy (PEP).
3. Terminator: An on-path adversary that terminates a connection, e.g., a TLS-terminating reverse proxy.

# Sticky Protocol Identifiers

In this section, we survey existing protocols developed inside and out of the IETF, and identify 
sticky protocol identifiers for each. A stick identifier is one that does not change across 
protocol "sessions," regardless of whether it is transmitted in the clear or not. This may include 
state-generating servers or, commonly, client algorithm, software configuration, or 
device-specific fields. We categorize surveyed protocols by OSI layer at which they operate. 
Specifically, we focus on Link, Internet, Transport, Session, and Application layers. (Our taxonomy 
may not match traditional OSI models, though we consider it sufficiently representative.)

## Link Layer

XXX: address

Ethernet: MAC address
XXX: include other link layer types from DHCP spec?

## Internet Layer

- IPv4 and IPv6: address

## Transport Layer

- TCP {{RFC0793}}: TCP source ports may be sticky if reused across senders. For example,
most operating systems allocate allocate ephemeral (short lived) ports to each new 
connection. Per IANA allocations, ephemeral ports range from 49152 to 65535 (2^15+2^14 to 2^16−1) 
[http://www.iana.org/assignments/port-numbers]. However, this does not prevent an application
from re-using port across connections. Destination are also intentionally sticky, since they
identify services offered by endpoints. Therefore, reusing a destination port does not lead to 
decreased
linkability.

- MPTCP {{RFC6824}}: Connection tokens or IDs are explicitly used to link MPTCP subflows between IP
address pairs. These tokens are only exposed during flow management operations, e.g., when creating
new subflows. Normal data transfer uses TCP sequence numbers to bypass middlebox interference and
an additional data sequence number (DSN) TCP option to allow receivers to deal with out-of-order
subflow packet arrival. The union of packet DSNs across subflows should yield a contiguous packet
number sequence. 

- SCTP: multihome (IP sending addresses in INIT chunks, this is not a mandatory parameter as per 
RFC4960), Initiate tag (used across multiple addresses), Address Families

- RTP: application performance reporting

## Session Layer

- TLS: session IDs, and possibly algorithms (though the space of possible values is large here), 
DTLS 1.3 connection IDs
- QUIC: connection ID

## Application Layer:

- HTTP ((CITE)): Cookies are sticky identifiers created chosen by servers to carry application 
state across
requests. Cookies might contain information about client identity or authentication preferences

- DNS: SRV hostnames
- DHCP: broadcast discover requests, hardware type, client hardware address, IP addresses 
(http://www.networksorcery.com/enp/protocol/dhcp.htm), transaction identifier

- NTP: (mode 3 — client to server, 
https://tools.ietf.org/html/draft-ietf-ntp-data-minimization-01#section-3): Timestamps, poll field, 
precision field, all other fields (Stratum, Root Delay, Root Dispersion, Ref ID, Ref Timestamp, 
Origin Timestamp, Receive Timestamp) 

- IMAP: username/password login...

Others: IMAP, LDAP, IKE/ESP, SRTP, oath, SIP, SSH, SNMP, ICMP, ARP

# Limiting Linkable Identifiers

The introductory example illustrating packet linkability using MAC addresses is one of many 
possible ways in which an attacker may link packets. As another hypothetical example, assume that 
IP address and MAC addresses were properly rotated. Moreover, assume TLS session IDs were reused 
over time, as shown below.

~~~
+---------------+        +---------------+
|TLS SessionID X<-------->TLS SessionID X|
+---------------+        +---------------+
|      ...      |  ....  |      ...      |
+---------------+        +---------------+
| IP Address  A <---//---> IP Address  B |
+---------------+        +---------------+
| MAC Address A <---//---> MAC Address C |
+---------------+        +---------------+

+------------------------------------------>
                  time
~~~

Despite rotating all protocol identifiers beneath TLS, a static session identifier makes packet 
linkability trivial. Thus, a strict, yet safe rule for removing packet linkability is to rotate 
all identifiers in unison. Unfortunately, this strategy is problematic in practice. It would
imply terminating active connections whenever an identifier changes (otherwise, linkability remains trivial). 
If MAC addresses are rotated on a regular basis, e.g., every 15 minutes, then connection lifetimes 
are bounded by this window.

Moreover, there are multiple dimensions along which identifiers may be linked: in time, as identifiers
are used and re-used by senders, and space, as identifiers are duplicated across multiple disjoint
network paths. We refer to these dimensions as time and path linkability, respectively. 

Time linkability is arguably simpler to mitigate, since new connections over time may opt to use
new identifiers. For example, instead of resuming a TLS session with an existing session ID, a
client may initiate a fresh handshake. As a simple rule, if an identifier at layer (N) changes, 
endpoints SHOULD use fresh identifiers for all lower layers, i.e., 1,.., (N-1). This means that
new TLS sessions SHOULD be initiated from an endpoint with a fresh MAC address, IP addres, and
TCP source port. 

In contrast, path linkability is more difficult to achieve, as it requires using fresh identifiers 
for each protocol field. (This may not always be technically feasible.) Moreover, protocols such
as QUIC explicitly try to enable path linkability via connection-level identifiers (CIDs) to support 
multihoming endpoints. This makes path linkability impossible to mitigate. However, as multiple,
disjoint paths may be operated by different entities, it may be the case that collusion is less common.

# Timing Considerations

Advice in this document SHOULD NOT be interpreted as guarantees for preventing linkability. Rather,
they aim to increase linkability complexity. It is difficult to prevent path-linkability without
modifying protocols above the layer at which identifiers rotate. For example, assuming MPTCP subflows
were unlinkable across paths, shared transport state controlling the rate of data transmission may
be sufficient to link these flows. 

# IANA Considerations

This document has no request to IANA.

# Security Considerations

TODO

# Privacy Considerations

TODO

# Acknowledgments

TODO

