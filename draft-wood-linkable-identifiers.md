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
    RFC793:
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

Privacy is an increasingly important topic in networks and protocols. {{RFC6973}} defines correlation of information relevant to or associated with a specific user as a significant attack on privacy. Such information can be exposed at any network stack layer. For example, a MAC address, if not properly rotated or randomized, may be used to link network packets to a single device. Many standards suggest randomizing or otherwise rotating such identifiers on a regular basis. Often, such advice is followed by recommendations to change other identifiers, e.g., IP addresses, host names, etc., in unison. For example, Huitema et al. {{I-D.ietf-dnssd-privacy}} say, "it is important that the obfuscation of instance names is performed at the right time, and that the obfuscated names change in synchrony with other identifiers, such as MAC Addresses, IP Addresses or host names." Consider the following example where this advice is not followed, wherein an IP address is changed yet the MAC address is not. 

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

An adversary may trivially link these packets together. In this document, we outline simple rules that SHOULD be followed by protocol implementations to avoid such trivial linkability. We then survey protocols developed inside the IETF and out, and identify their sticky identifiers. Results were obtained by analyzing protocol documentation and specifications, and also scanning packet traces captured from protocols in practice on common systems.

# Limiting Linkable Identifiers

The introductory example illustrating packet linkability using MAC addresses is one of many possible ways in which an attacker may link packets. As another hypothetical example, assume that IP address and MAC addresses were properly rotated. Moreover, assume TLS session IDs were reused over time, as shown below.

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

Despite rotating all protocol identifiers beneath TLS, a static session identifier makes packet linkability trivial. 
Thus, a strict, yet safe rule for removing packet linkability is to rotate all identifiers in unison. 
Unfortunately, this strategy is problematic in practice. 

XXX: describe why — involves cutting off connections, possibly. Need to align rotation events to minimize or limit linkability leakage: if identifier at layer (N) changes, then identifiers at layer (N - 1) and below should also change. (If TLS session ID changes for resumption, then all IP address, TCP port, Ethernet MAC address, etc. should also rotate)

XXX: discuss difference in linkability across time and space (multiple paths). time is easier to enforce. space is harder to accomplish.

# Sticky Protocol Identifiers

In this section, we survey existing protocols developed inside and out of the IETF, and identify sticky protocol identifiers for each. A stick identifier is one that does not change across protocol “sessions,” regardless of whether it is transmitted in the clear or not. This may include state issued XXXy servers or, commonly, client algorithm, software configuration, or device-specific fields. We categorize surveyed protocols by OSI layer at which they operate. Specifically, we focus on Link, Internet, Transport, Session, and Application layers. (Our taxonomy may not match traditional OSI models, though we consider it sufficiently representative.)

## Link Layer

XXX: address

Ethernet: MAC address
XXX: include other link layer types from DHCP spec?

## Internet Layer

- IPv4 and IPv6: address

## Transport Layer

- TCP {{RFC793}}: TCP source ports may be sticky if reused across senders. For example,
most operating systems allocate allocate ephemeral (short lived) ports to each new 
connection. Per IANA allocations, ephemeral ports range from 49152 to 65535 (2^15+2^14 to 2^16−1) [http://www.iana.org/assignments/port-numbers]. However, this does not prevent an application
from re-using port across connections. Destination are also intentionally sticky, since they
identify services offered by endpoints. Therefore, reusing a destination port does not lead to decreased
linkability.

- MPTCP {{RFC6824}}: Connection tokens or IDs are explicitly used to link MPTCP subflows between IP
address pairs. These tokens are only exposed during flow management operations, e.g., when creating
new subflows. Normal data transfer uses TCP sequence numbers to bypass middlebox interference and
an additional data sequence number (DSN) TCP option to allow receivers to deal with out-of-order
subflow packet arrival. The union of packet DSNs across subflows should yield a contiguous packet
number sequence. 

- SCTP: multihome (IP sending addresses in INIT chunks, this is not a mandatory parameter as per RFC4960), Initiate tag (used across multiple addresses), Address Families

- RTP: application performance reporting

## Session Layer

- TLS: session IDs, and possibly algorithms (though the space of possible values is large here), DTLS 1.3 connection IDs
- QUIC: connection ID

## Application Layer:

- HTTP ((CITE)): Cookies are sticky identifiers created chosen by servers to carry application state across
requests. Cookies might contain information about client identity or authentication preferences

- DNS: SRV hostnames
- DHCP: broadcast discover requests, hardware type, client hardware address, IP addresses (http://www.networksorcery.com/enp/protocol/dhcp.htm), transaction identifier

- NTP: (mode 3 — client to server, https://tools.ietf.org/html/draft-ietf-ntp-data-minimization-01#section-3): Timestamps, poll field, precision field, all other fields (Stratum, Root Delay, Root Dispersion, Ref ID, Ref Timestamp, Origin Timestamp, Receive Timestamp) 

- IMAP: username/password login...


Others: IMAP, LDAP, IKE/ESP, SRTP, oath, SIP, SSH, SNMP, ICMP, ARP


# Timing Considerations

Advice in this document SHOULD NOT be interpreted as guarantees for preventing linkability. Rather,
they aim to increase linkability complexity. 


Consider, for example, an application using QUIC to send
data to a peer. If the sender invokes a voluntary path change, thereby obtaining a new source port
and, presumably, using a new connection ID, 


XXX:
say "increases the complexity" as opposed to "reduces linkability" because, as a measurement geek, I'm deeply pessimistic about the ability to eliminate linkability of packets sent by the same sender. After you have added a runtime requirement to keep some amount of per-flow state in order to link packets to a flow, any additional complexity in packet number obfuscation only serves to increase the amount of work to design an algorithm for linking packets, i.e., it increases the number of masters' and/or doctoral degrees awarded to people who work on solving the problem of linking QUIC flows.)


# IANA Considerations

This document has on request to IANA.

# Security Considerations

TODO

# Acknowledgments

TODO

