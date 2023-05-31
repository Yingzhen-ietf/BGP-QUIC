## Terminology
QUIC connection  RFC9000
QUIC streams  9000
BGP channel:  Instance of BGP protocol state machine mapped to specific QUIC stream.
BGP control channel
(function/MP channel) BGP per AFI/SAFI channel
Multi-channel BGP

## Introduction
The Border Gateway Protocol (BGP) <xref target="RFC4271"/> is the routing protocol used to exchange routing and reachability information among autonomous systems, and it uses TCP as its transport protocol to provide reliable packet communication.  BGP establishes peer relationships between routers using a TCP session on port 179.

The Multiprotocol Extensions for BGP-4 (MP-BGP) <xref target="RFC4760"/> allow BGP to carry information for multiple Network Layer protocols. However, only a single TCP connection can reach the Established state between a pair of peers <xref target="RFC4271"/>.  As a consequence, an error related to a particular Network Layer protocol may result in the termination of the connection for all.

QUIC <xref target="RFC9000"/> is a UDP-based multiplexed and secure transport protocol that provides connection-oriented and stateful interaction between a client and server. It can provide low latency and encrypted transport with resilient connections.

In QUIC, application protocols exchange information using streams.
Each stream is a separate unidirectional or bidirectional channel of
"order stream of bytes." Moreover, each stream has its own flow
control which limits bytes sent on a stream, together with flow
control of the connection.

This document specifies the procedures for BGP to use QUIC as a transport protocol with a mechanism to establish multiple BGP channels using QUIC streams. In one BGP over QUIC connection, there is always a control channel, and one or more channels used to advertise routing information.


## Summary of Operations

### Establish BGP/QUIC Connection

Before two BGP speakers start exchanging routing information, they need to establish a BGP session. It is established in two phases:

 - Establish a transport layer connection. TLS 1.3 is integrated with QUIC, how TLS authentication should be done is out of scope of this draft. We assume a QUIC connection can be established here.
 - Establish a BGP session. This draft specifies the details that allow multiple BGP sessions in a single QUIC connection.

[RFC4271] defines the BGP FSM operation. BoQ extends RFC4271 FSM to be able to adddress individual BGP channels.

QUIC connections are established as described in [RFC9000].  During connection establishment, a BGP speaker SHOULD use UDP port xxx and MUST select the Application-Layer Protocol Negotiation (ALPN) [RFC7301] token "boq" in the TLS handshake.  Support for other application-layer protocols MUST NOT be offered in the same handshake. A connection MUST be closed if the ALPN token is not as indicated, or if other application-layer protocols are offered in the same handshake.

### Establish BGP/QUIC Control Channel

The control channel is a bidirectional QUIC stream, and is created by sending a BGP OPEN message. This OPEN message is unchanged but it doesn't include the multiprotocol capability. The control channel has to reach established state before any other channel can be created.

### Establish BGP/QUIC Function Channel

QUIC supports connection migration, however only the client side can move.  The role of the QUIC endpoints is important.  For future extensibility, a new BoQ Capability indicates the configured role of the BGP speaker: Client, Server, or Any.  It is expected that the BGP configuration and QUIC roles match.  The QUIC connection can be reset if they don't.  See Section x for details.

The procedure to reach the Established state in the control channel is the same as what is currently specified in rfc4271.

In addition to the control channel, function channels are used to
exchange specific types of routing information. After the control
channel reaches established state, function channels can be created as
unidirectional QUIC streams to advertise specific AFI/SAFI routes.

BoQ supports two modes of route advertisement, symmetric and
asymmetric.  Symmetric route advertisement occurs when both BGP
speakers need to send updates to their neighbors, for a specific
AFI/SAFI.  Asymmetric route advertisement occurs when only one BGP
speaker needs to send updates to its neighbor.  BGP-LS or flow-spec
may be examples of asymmetric route advertisement.

A BGP speaker that needs to advertise routes to its peer opens a
unidirectional functional channel to its neighbor and sends an OPEN
message indicating the particular AFI/SAFI to be used.  Only one
AFI/SAFI is permitted per functional channel, and only one functional
channel per AFI/SAFI can exist.  The receiver replies with an OPEN_ACK
(Section y) message in the control channel.  Once the BGP session is
established the sender can advertise routes.

A single functional channel for an AFI/SAFI pair results in asymmetric
route advertisements.  Both BGP speakers can create each a functional
channel to implement symmetric route advertisements.

Each functional channel is created independently, to naturally support
multi-channel BGP. The neighbor state machines are decoupled, in case
of error it is possible to reset only one functional channel (one
direction of the symmetric route exchange).

### Channel Reset

If the entire BGP connection needs to be reset for any reason, such as
a configuration change or a network outage, a notification is sent
over the control channel to inform the other router that the
connection is being reset.

If the control channel is closed, the QUIC connection needs to be
terminated using a CONNECTION_CLOSE frame, and an error message (TBD)
should be included to indicate that the connection has been terminated
by BGP. If there are other open channels, they are implicitly closed
when the connection is closed.

A function channel can be reset independently without impacting any
other function channels.

### Channel Coordination

A single QUIC stream provides ordered and reliable delivery, however
there is no guarantee of transmission and deliver order across
streams. Therefore, if specific data from one channel needs to be
received before data from other channels, this requirement must be
accomplished through BGP.


## Protocol Definitions

### BGP QUIC Capability

A new ”BGP over QUIC” capability in the OPEN message to signal the BGP speaker is a QUIC client, a QUIC server or any (Don’t care). To be a client or server, it needs to be explicitly configured for a BGP speaker, otherwise the default value is “any”.
BoQ capability:
  Code: TBD (to be assigned by IANA)
  Length: 1(octet)
  Value:
    0 Any
    1 Client
    2 Server

This ONLY applies to the control channel. If the BoQ capability is sent in OPEN message for any AFI/SAFI channel, it MUST be ignored. A BGP speaker SHOULD support the explicit configuration of it as a client, a server or any. Before a control channel is created, a BGP speaker SHOULD check whether its configuration matches the QUIC connection role. If they don't match, the QUIC connection SHOULD be terminated. For example, if a BGP speaker is configured as a client, but the QUIC connection comes up as a QUIC server, the QUIC connection should be terminated.

If a BGP speaker is configured as QUIC client, it SHOULD try to initiate the QUIC connection. If a BGP speaker is configured as QUIC server, it SHOULD wait for a QUIC connection.
When both BGP peers are explicitly configured (one side as client and the other side as server), no collision will happen.
When both BGP peers are “any”, existing session collision mechanism are used.
When both BGP peers are explicitly configured as client (configuration error), a new OPEN message error subcode (BoQ error) MUST be sent.
When both BGP peers are explicitly configured as server (configuration error), no connection will be initiated.
For peering between router A and router B, if A is configured as client and B is any, the following rules are followed:
When A starts a connection as the client, B accepts it.
When B starts a connection as the client, A should start a connection to B as the client, and this stream should be used.
When there are two simultaneous connections, the one with A as the client wins (modification to existing collision resolution). 
For peering between router A and router B, if A is configured as server and B is any, A will wait for the connection from B.
