## Terminology
* TODO - do we need to define channel?  
* Multi-channel BGP using QUIC: Running the BGP protocol over multiple QUIC streams as defined in this document.
* QUIC connection (defined in RFC 9000)
* QUIC streams (defined in RFC 9000)
* BGP channel: Instance of BGP protocol state machine mapped to specific QUIC stream.
* BGP control channel (implemented as bidirectional stream)
* (function/MP channel) BGP per AFI/SAFI channel (implemented asymmetrically as unidirectional streams)

## Introduction
The Border Gateway Protocol (BGP) <xref target="RFC4271"/> is the routing protocol used to exchange routing and reachability information among autonomous systems. BGP uses TCP as its transport protocol to provide reliable packet communication.  BGP establishes peer relationships between routers using a TCP session on port 179.

The Multiprotocol Extensions for BGP-4 (MP-BGP) <xref target="RFC4760"/> allow BGP to carry information for multiple Network Layer protocols. However, only a single TCP connection can reach the Established state between a pair of peers <xref target="RFC4271"/>.  As a consequence, an error related to a particular Network Layer protocol may result in the termination of the connection for all.

Previous proposals attempt to address the issue of errors for individual Network Layer protocols (AFI/SAFI) by providing extensions to run multiple BGP sessions between two BGP speakers.  One such example, BGP multisession <xref target="https://datatracker.ietf.org/doc/html/draft-ietf-idr-bgp-multisession"/>, uses multiple TCP sessions between the two participating BGP speakers.  However, this proposal has not gained widespread adoption.

QUIC <xref target="RFC9000"/> is a UDP-based multiplexed and secure transport protocol that provides connection-oriented and stateful interaction between a client and server. It can provide low latency and encrypted transport with resilient connections.

In QUIC, application protocols exchange information using streams.
Each stream is a separate unidirectional or bidirectional channel consisting of an
ordered stream of bytes. Moreover, each stream has its own flow
control which limits bytes sent on a stream, together with flow
control of the connection.

This document specifies the procedures for BGP to use QUIC as a transport protocol with a mechanism to carry multiple BGP address families (AFI/SAFI) over individual streams.  These per-AFI/SAFI streams (function channels), and the associated control mechanism (control channel) for the session are called "BGP channels". In one BGP over QUIC connection, there is always one control channel, and one or more per-AFI/SAFI function channels used to carry routing information.


## Summary of Operations

### Establish BGP/QUIC Connection

Before two BGP speakers start exchanging routing information, they need to establish a BGP session. It is established in two phases:

 - Establish a transport layer connection. TLS 1.3 is integrated with QUIC. The TLS authentication parameters used for this connection are out of scope of this draft. 
 - Establish a BGP over QUIC session over this connection. This draft specifies the details that allow BGP channels to be carried over this QUIC connection.

QUIC connections are established as described in [RFC9000].  During connection establishment, a BGP speaker SHOULD use UDP port TBD and MUST select the Application-Layer Protocol Negotiation (ALPN) [RFC7301] token "boq" in the TLS handshake.  Support for other application-layer protocols MUST NOT be offered in the same handshake. A connection MUST be closed if the ALPN token is not as indicated, or if other application-layer protocols are offered in the same handshake.

[RFC4271] defines the BGP Finite State Machine (FSM) operations for a single BGP session between two BGP speakers using TCP.  This document defines the ability to carry BGP over multiple QUIC streams as "BGP channels".  

On a BGP over QUIC connection, the BGP over QUIC speaker first establishes a bidirectional stream for the "BGP control channel".  The control channel is used to establish a BGP peer relations between two BGP over QUIC speakers similar to RFC 4271.  Open messages are exchanged on the control channel, and if the BGP over QUIC session parameters are acceptable, the peering session comes up. Afterward the per-AFI/SAFI function channels may then be created.  Similar to RFC 4271, if the parameters are not acceptable, the BGP over QUIC session is terminated with a NOTIFICATION message.

After the control channel is Established, each BGP over QUIC speaker may create per-AFI/SAFI function channels using a unidirectional QUIC stream.  These function channels are used to carry BGP routes for that AFI/SAFI.  There SHALL NOT be more than one function channel per AFI/SAFI from one BGP over QUIC speaker to the other. Unlike RFC 4271 BGP, there is no requirement that both BGP over QUIC speakers have a symmetric and matching set of per-AFI/SAFI channels.

BGP channels largely use the mechanisms of the RFC 4271 FSM for their establishment. For the control channel carried over a bidirectional QUIC stream, the FSM is identical to the RFC 4271 FSM.  However, since the per-AFI/SAFI function channels are unidirectional, the RFC 4271 FSM procedures cannot be carried solely using the unidirectional channel from one BGP over QUIC speaker to another.  Instead, responding BGP speaker must carry its replies for the unidirectional streams over the control channel and are addressed to a specific BGP function channel.

### Establish BGP/QUIC Control Channel

After BGP over QUIC session establishment, the BGP over QUIC speakers will create their control channels.  The control channel is a bidirectional QUIC stream.  It is created by sending a BGP OPEN message. BGP OPEN messages carry parameters such as the Autonomous System number, BGP Identifier (router-id), Hold Time, and Capabilities.  These parameters are used by a BGP speaker to decide whether a BGP session is permitted to be established.

The capabilities carried in this OPEN message for the control channel are the BGP over QUIC connection specific parameters; i.e. those that apply to the entire connection.  An example of this is the BGP Role Capability <xref target="https://www.rfc-editor.org/rfc/rfc9234.html"/>. 

The control channel uses BGP holdtime procedures as specified in RFC 4271.  A Keepalive timer is used to periodically send KEEPALIVE messages in the absence of other messages on the control channel.  If no messages are received within the negotiated holdtime on the control channel, the BGP over QUIC connection is closed with a NOTIFICATION sent on the control channel.  In short, the BGP over QUIC control channel is used to establish the peering relationship and connection parameters between the two BGP speakers, ensure connectivity over this session is verified, and further is used as the response channel for the per-AFI/SAFI function channels as specified in the next section.

If the parameters for the BGP over QUIC session carried on the control channel are acceptable, the control channel enters the Established state.  Afterwards the per-AFI/SAFI function channels can then be created. The BGP function channels carry the per-AFI/SAFI routing information in BGP Updates.  Per-AFI/SAFI capabilities only need to be carried on those function channels.

(TODO)
QUIC supports connection migration, however only the client side can move.  The role of the QUIC endpoints is important.  For future extensibility, a new BoQ Capability indicates the configured role of the BGP speaker: Client, Server, or Any.  It is expected that the BGP configuration and QUIC roles match.  The QUIC connection can be reset if they don't.  See Section x for details.

### Establish BGP/QUIC Function Channel

Per-AFI/SAFI Function channels are used to
exchange routing information. After the control
channel reaches Established state, function channels are created as
unidirectional QUIC streams and advertise per-AFI/SAFI routes using BGP Update messages.  There SHALL NOT be more than one per-AFI/SAFI function channel for each BGP speaker; they are unique.

BGP over QUIC speakers asymmetrically create their per-AFI/SAFI function channels.  While it might be the typical case for there to be a symmetric set of per-AFI/SAFI function channels, one for each speaker, this is not a requirement.  For example, BGP-LS <xref target="https://datatracker.ietf.org/doc/html/rfc7752"/> may only require that a BGP speaker asymmetrically receive BGP-LS NLRI, and may not need to send them.

(XXX Alvaro)
A BGP speaker that needs to advertise routes to its peer opens a
unidirectional stream to its neighbor and sends an OPEN
message indicating the particular AFI/SAFI to be used.  The BGP over QUIC connection-wide parameters have previously been exchanged over the control channel. The per-AFI/SAFI function channel OPEN messages MUST contain identical BGP Autonomous System number and BGP Identifier as the previously Established control channel.  It is RECOMMENDED that the BGP Holtime value exchanged in the per-AFI/SAFI function channels is significantly longer than the holdtime negotiated for the control channel.  It is the responsibility of the holdtimer for the control channel to provide connection verification for the BGP over QUIC connection.  The purpose of the per-AFI/SAFI function channel negotiated holdtime is to provide verification of communication between  the two BGP over QUIC speakers for that AFI/SAFI.

The BGP Capabilities carried on the per-AFI/SAFI channel SHOULD only be those that are per-AFI/SAFI specific.  Conflicting BGP over QUIC connection-wide parameters exchanged over the per-AFI/SAFI function channels MAY BE reason for the BGP speaker to send a NOTIFICATION message and not to permit the per-AFI/SAFI function channel to become Established. 

The BGP over QUIC speaker creates the per-AFI/SAFI channel by sending its OPEN message over the newly created QUIC unidirectional stream.  The receiving BGP speaker replies to those messages as defined in the RFC 4271 FSM by sending its messages (OPEN/NOTIFICATION/KEEPALIVE) addressed to the sender over the control channel.

(TODO - now is the point to discuss changes to BGP messages to support BGP Channels, including how do we address specific BGP PDUs to the receiver and is this in the QUIC frame or the BGP PDU)


-----

Once the per-AFI/SAFI function channel has reached the Established state, it may send BGP Update messages to the remote BGP over QUIC speaker.


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

Each function channnel has its own keepalive messages with its own
timers.  These timers are used to ensure that the session stays alive
even if no routing updates are being exchanged.  Non-routing related
function channels, such as BGP FLowSpec, may have longer keepalive
timers, for example 120 seconds, as they do not need to exchange
routing updates as frequently as other function channels and can run
in a relatively lower priority.
   
A single QUIC stream provides ordered and reliable delivery, however
there is no guarantee of transmission and deliver order across
streams. Therefore, if specific data from one channel needs to be
received before data from other channels, this requirement must be
accomplished through BGP.

As defined in [RFC9000], a QUIC implementation SHOULD provide ways in
which an application can indicate the relative priority of streams.

For a BGP implementation utilizing QUIC as its transport protocol,
it MUST support a prioritization mechanism for BGP streams.  This is
essential for ensuring that critical routing information can be transmitted
with higher priority compared to non-routing information.

How to implement the supported priorities using QUIC congestion control at connection level, stream level flow control, and packetization are out of the scope of this document.

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
When B starts a connection as the client, A should start a connection to B as the client, and this connection should be used.
When there are two simultaneous connections, the one with A as the client wins (modification to existing collision resolution). 
For peering between router A and router B, if A is configured as server and B is any, A will wait for the connection from B.

### Collision Avoidance

Before creating a new channel, a BGP speaker should check that no channel exists for the same Network Layer protcol. If a channel already exists, the BGP speaker SHOULD NOT attemp to create a new one.

## Error Handling
