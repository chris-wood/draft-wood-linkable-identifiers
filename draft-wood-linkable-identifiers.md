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
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: cawood@apple.com

normative:
    RFC0793:
    RFC2508:
    RFC2616:
    RFC6265:
    RFC6824:
    RFC6973:
    RFC7232:
    RFC7413:
    I-D.ietf-ntp-data-minimization:
    I-D.ietf-dnssd-privacy:

--- abstract

Rotating public identifiers is often encouraged as best practice to protect
against privacy violations. For example, regular MAC address randomization
is a technique implemented to prevent device tracking across time and space. 
Other protocols beyond those in the link layer also have public identifiers
or parameters that should rotate over time. This document surveys such 
privacy-related identifiers exposed by common Internet protocols at various 
layers in a network stack. It provides advice for rotating linked identifiers
such that privacy violations do not occur while rotating one identifier and 
neglecting to rotate a related identifier.

--- middle

# Introduction

{{RFC6973}} defines the correlation of information relevant to or
associated with a specific user as a significant attack on privacy.
Different layers of the network stack use identifiers to uniquely
address hosts or information flows.
To mitigate the privacy concern, many standards suggest randomizing
or otherwise rotating such identifiers on a regular basis.
For example, a MAC address may be used to link otherwise unrelated
network packets to a single device. Rotating the MAC address prevents
this association at the link layer.
However, when multiple identifiers are simultaneously present on
different layers of the stack, breaking the association at any
individual layer might be insufficient to disassociate a host
from their network traffic.
For example, Huitema et al. {{I-D.ietf-dnssd-privacy}} say, "it is important
that the obfuscation of instance names is performed at the right time, and
that the obfuscated names change in synchrony with other identifiers, such as
MAC Addresses, IP Addresses or host names."
Consider the following example where this advice is not followed,
wherein an IP address is changed yet the MAC address is not.

~~~
+---------------+        +---------------+
|      ...      |  ....  |      ...      |
+---------------+        +---------------+
| IP Address  A <---//---> IP Address  B |
+---------------+        +---------------+
| MAC Address A <--------> MAC Address A |
+---------------+        +---------------+

+---------------------------------------->
                  time
~~~

A network adversary may trivially link these packets based on their common
MAC address and continue to associate traffic with this particular host based
on IP address B even if the MAC address eventually changes in the future.
In this document, we outline simple rules that SHOULD be followed by protocol
implementations to avoid such linkability.
We then survey protocols developed inside the IETF and out, and identify their
sticky identifiers. Results were obtained by analyzing protocol documentation
and specifications, and also scanning packet traces captured from protocols in
practice on common systems.

# Identifier Scope and Threat Model

Not all packet or datagram identifiers are visible end-to-end in a client-server
interaction. For example, MAC addresses are only visible within on local subnets.
IP addresses are only visible between endpoints. (In systems such as Tor,
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
Note that there can be distinct terminators for individual layers of network stack.
E.g., one for TLS and another for HTTP.

# Sticky Protocol Identifiers

In this section, we survey existing protocols developed inside and out of the
IETF, and identify sticky protocol identifiers for each. A sticky identifier
is one that persists across logically grouped data exchanges between a client
and server.
This may include state-generating servers or, commonly, client algorithm,
software configuration, or device-specific fields.
We categorize surveyed protocols by the OSI layer at which they operate.
Specifically, we focus on Link, Internet, Transport, Session, and Application
layers.
(Our taxonomy may not match traditional OSI models, though we consider it
sufficiently representative.)

## Internet and Link Layer

- IPv4 and IPv6: Static or infrequently rotating addresses are sticky identifiers
when exposed on the network.

- Ethernet: MAC addresses are fixed to specific devices. Unless frequently rotated,
they are sticky identifiers.

- IKEv2: Initiator Security Parameters Indexes (SPIs) are used as connection 
identifiers instead of IP addresses. They are required to rotate for each new SA.

<!-- ARP: ??? -->

## Transport and Session Layer

- TCP {{RFC0793}}: TCP source ports may be sticky if reused across senders. For example,
most operating systems allocate allocate ephemeral (short lived) ports to each new
connection. Per IANA allocations, ephemeral ports range from 49152 to 65535 (2^15+2^14 to 2^16−1)
[http://www.iana.org/assignments/port-numbers]. However, this does not prevent an application
from re-using port across connections. Destination are also intentionally sticky, since they
identify services offered by endpoints. Therefore, reusing a destination port does not lead to
decreased linkability. Moreover, with TCP Fast Open (TFO) {{RFC7413}}, servers give clients
plaintext cookies that must be re-used when resuming a TCP+TFO connection. Clients do not modify
these server cookies, which therefore means they can be tracked.

- MPTCP {{RFC6824}}: Connection tokens or IDs are explicitly used to link MPTCP subflows between IP
address pairs. These tokens are only exposed during flow management operations, e.g., when creating
new subflows. Normal data transfer uses TCP sequence numbers to bypass middlebox interference and
an additional data sequence number (DSN) TCP option to allow receivers to deal with out-of-order
subflow packet arrival. The union of packet DSNs across subflows should yield a contiguous packet
number sequence.

- TLS ((CITE)): Prior to TLS 1.3, significant information is exposed during TLS handshakes, including:
session identifiers (or re-used PSK identifiers in TLS 1.3), timestamps, random nonces, supported ciphersuites,
certificates, and extensions. Many of these are common across all TLS clients -- specifically, ciphersuites,
nonces, and timestamps. However, others may persist across active sessions, including: session identifiers (in TLS 1.2
and earlier versions) and re-used PSK identifiers (in TLS 1.3). Without rotation, these re-used identifers
are sticky.

- DTLS ((CITE)): Datagram TLS is a slightly modified variant of TLS aimed to run over datagram protocols
such as UDP. In addition to identifiers exposed via TLS, DTLS adds cookie-based denial-of-service
countermeasures. Servers issue stateless cookies to clients during a handshake, which must be replayed
in cleartext by clients to prove ownership of its IP address. (This is similar to TFO cookies described
above.) Additionally, DTLS 1.3 ((CITE)) is considering support of a static connection identifier (CID), which permits
client address mobility. CIDs are specifically designed to not change across addresses.

- QUIC ((CITE)): QUIC is another secure transport protocol originally developed by Google and now being
standardized by the IETF. IETF-QUIC ((CITE)) uses TLS 1.3 for its handshake. In addition to identifiers
exposed by TLS 1.3, QUIC has its own connection identifier (CID) used to permit address mobility.

<!-- SCTP: multihome (IP sending addresses in INIT chunks, this is not a mandatory parameter as per RFC4960), Initiate tag (used across multiple addresses), Address Families -->
<!-- RTP: application performance reporting -->
<!-- SRTP: ??? -->

## Application Layer:

- HTTP {{RFC2616}}: While HTTP is a stateless protocol, it enables applications
to define state-keeping mechanisms in header fields. The fields might carry
the state itself or tokens pointing to state kept at the endpoints.
The Cookie header field {{RFC6265}} is de-facto the mechanism for web
applications to uniquely identify their clients by generating a token
and instructing the client to attach to any future requests.
The ETag header field {{RFC7232}} enables applications to uniquely reference
a resource which the client may cache. Applications may return unique
reference tokens to distinct clients.

- DNS ((CITE)): SRV records often contain human-readable information specific to 
particular devices, clients, or users. For example, printers may advertise
its services with SRV records that contain a human-readable instance name.
These are often not rotated as services change.

- NTP ((CITE)): By default, mode 3 for NTP -- client to server -- sends several 
source-specific fields in the clear to NTP servers, including: timestamps, poll, 
and precision. These fields should be left empty or randomized as per {{I-D.ietf-ntp-data-minimization}}.
Other fields that may link to clients include: Stratum, Root Delay, Root Dispersion, 
Ref ID, Ref Timestamp, Origin Timestamp, and Receive Timestamp. 

- DHCP ((CITE)): broadcast discover requests, hardware type, client hardware address, IP addresses
(http://www.networksorcery.com/enp/protocol/dhcp.htm), transaction identifier

<!-- IMAP: ??? -->
<!-- LDAP: ??? -->
<!-- OAuth: ??? -->
<!-- SIP: ??? -->
<!-- SSH: ??? -->
<!-- SNMP: ??? -->
<!-- ICMP: ??? -->

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
all linked identifiers in unison. Unfortunately, this strategy is problematic in practice. It would
imply terminating active connections whenever an identifier changes (otherwise, linkability remains trivial).
If MAC addresses are rotated on a regular basis, e.g., every 15 minutes, then connection lifetimes
would be limited to this window.

Moreover, there are multiple dimensions along which identifiers may be linked: in time, as identifiers
are used and re-used by senders, and space, as identifiers are duplicated across multiple disjoint
network paths. We refer to these dimensions as time and path linkability, respectively.

Time linkability is arguably simpler to mitigate, since new connections over time may opt to use
new identifiers. For example, instead of resuming a TLS session with an existing session ID, a
client may initiate a fresh handshake. As a simple rule, if an identifier at layer (N) changes,
endpoints SHOULD use fresh identifiers for all lower layers, i.e., 1,.., (N-1). This means that
new TLS sessions SHOULD be initiated from an endpoint with a fresh MAC address, IP address, and
TCP source port. Note that clients behind NATs may not need to generate a fresh MAC or IP address,
as they enjoy some measure of anonymity by design.

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

