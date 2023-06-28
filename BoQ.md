## Terminology
* TODO - do we need to define channel?  
* Multi-channel BGP using QUIC: Running the BGP protocol over multiple QUIC streams as defined in this document.
* QUIC connection (defined in RFC 9000)
* QUIC streams - A bidirectional or unidirectional bytestream provided by the QUIC transport (defined in RFC 9000)
* BGP channel: Instance of BGP protocol state machine mapped to specific QUIC stream.
* BGP control channel (implemented as bidirectional stream)
* (function/MP channel) BGP per AFI/SAFI channel (implemented asymmetrically as unidirectional streams)

## Introduction
The Border Gateway Protocol (BGP) <xref target="RFC4271"/> is the routing protocol used to exchange routing and reachability information among autonomous systems. BGP uses TCP as its transport protocol to provide reliable communication.  BGP establishes peer relationships between routers using a TCP session on port 179.

The Multiprotocol Extensions for BGP-4 (MP-BGP) <xref target="RFC4760"/> allow BGP to carry information for multiple Network Layer protocols. However, only a single TCP connection can reach the Established state between a pair of peers <xref target="RFC4271"/>.  As a consequence, an error related to a particular Network Layer protocol may result in the termination of the connection for all.

QUIC <xref target="RFC9000"/> is a UDP-based multiplexed and secure transport protocol that provides connection-oriented and stateful interaction between a client and server. It can provide low latency and encrypted transport with resilient connections.

In QUIC, application protocols exchange information using streams.
Each stream is a separate unidirectional or bidirectional channel consisting of an
ordered stream of bytes. Moreover, each stream has its own flow
control, which limits bytes sent on a stream, together with flow
control of the connection.

This document specifies the procedures for BGP to use QUIC as a transport protocol with a mechanism to carry Network Layer protocols (AFI/SAFI) over individual streams.  The Network layer protocols are identified using a combination of Address Family (AF) and Subsequent Address Family (SAFI), as described in <xref target="RFC4760"/>.  These per-AFI/SAFI streams (function channels) and the associated control mechanism (control channel) for the session are called "BGP channels". In one BGP over QUIC (BoQ) connection, one control channel and one or more function channels are used to carry routing information.


## Summary of Operations

### Establish BGP/QUIC Connection

Before two BoQ speakers start exchanging routing information, they must establish a BGP session. It is established in two phases:

 - Establish a transport layer connection. TLS 1.3 is integrated with QUIC. The TLS authentication parameters used for this connection are out of the scope of this draft. 
 - Establish a BoQ session over this transport connection. This document specifies the details of such an operation.

QUIC connections are established as described in [RFC9000].  During connection establishment, a BoQ speaker SHOULD use UDP port TBD1 and MUST select the Application-Layer Protocol Negotiation (ALPN) [RFC7301] token "boq" in the TLS handshake.  Support for other application-layer protocols MUST NOT be offered in the same handshake. A connection MUST be closed if the ALPN token is not as indicated or if other application-layer protocols are offered in the same handshake.

[RFC4271] defines the operations for a single BGP session between two BoQ speakers using TCP.  This document defines the ability to carry BGP over multiple QUIC streams as "BGP channels".  

On a BoQ connection, the BoQ speaker first establishes a bidirectional stream for the "BGP control channel".  The control channel is used to establish a BGP peer relationship between two BoQ speakers, similar to RFC 4271.  OPEN messages are exchanged on the control channel, and if the BoQ session parameters are acceptable, the peering session is established.  Similar to RFC 4271, the BoQ session is terminated with a NOTIFICATION message if the parameters are unacceptable.

After establishing the control channel, each BoQ speaker may create function channels using unidirectional QUIC streams.  These function channels are used to carry BGP routes for a specific AFI/SAFI.  Only one function channel per AFI/SAFI can exist from one BoQ speaker to another (see "Channel Collision Avoidance"). Unlike RFC 4271 BGP, there is no requirement for both BoQ speakers to have a symmetric and matching set of function channels.

BGP channels largely use the mechanisms of the RFC 4271 Finite State Machine (FSM) for their establishment. For the control channel carried over a bidirectional QUIC stream, the FSM is identical to the RFC 4271 FSM.  However, since the function channels are unidirectional, the RFC 4271 FSM procedures cannot be carried out solely using the unidirectional channel from one BoQ speaker to another.  Instead, the responding BoQ speaker must carry its replies for the unidirectional streams over the control channel and address them to a specific BGP function channel.

### Establish BGP/QUIC Control Channel

After BoQ session establishment, the BoQ speakers will create the control channel.  The control channel is a bidirectional QUIC stream.  It is created by sending a BGP OPEN message. BGP OPEN messages carry parameters such as the Autonomous System number, BGP Identifier (router-id), Hold Time, and Capabilities.  These parameters are used by a BoQ speaker to decide whether a BGP session is permitted to be established.

The capabilities carried in this OPEN message for the control channel are the BoQ connection-specific parameters, i.e. those that apply to the entire connection.  An example of this is the BGP Role Capability <xref target="https://www.rfc-editor.org/rfc/rfc9234.html"/>.  If a function-only capability, as categorized in the "BGP Capability Category" table, is included in the OPEN message, it MUST be ignored.

The control channel uses BGP hold time procedures as specified in RFC 4271.  KEEPALIVE messages are sent periodically in the absence of other messages on the control channel.  If no messages are received within the negotiated hold time on the control channel, the BoQ connection is closed with a NOTIFICATION as specified in <xref target="RFC4760" section="6.5"/>.  In short, the BoQ control channel is used to establish the peering relationship and connection parameters between the two BoQ speakers, ensure connectivity over this session is verified, and further is used as the response channel for the function channels as specified in <xref target="Establish BGP/QUIC Function Channel"/>.

(TODO)
QUIC supports connection migration. However, only the client side can move.  The role of the QUIC endpoints is important.  For future extensibility, a new BoQ Capability indicates the configured role of the BoQ speaker: Client, Server, or Any.  It is expected that the BGP configuration and QUIC roles match.  The QUIC connection can be reset if they don't.  See Section x for details.

### Establish BGP/QUIC Function Channel

Per-AFI/SAFI Function channels are used to exchange routing information. After the control channel reaches the Established state, function channels are created as unidirectional QUIC streams and advertise routes for a single AFI/SAFI using BGP UPDATE messages.  Only one function channel per AFI/SAFI can exist from one BoQ speaker to another (see "Channel Collision Avoidance").

BoQ speakers asymmetrically create their function channels.  While it might be the typical case for there to be a symmetric set of per-AFI/SAFI function channels, one for each speaker, this is not a requirement.  For example, BGP-LS <xref target="https://datatracker.ietf.org/doc/html/rfc7752"/> may only require that a BoQ speaker asymmetrically receive BGP-LS NLRI and may not need to send them.

A BoQ speaker that needs to advertise routes to its peer opens a unidirectional stream to its neighbor and sends an OPEN message indicating the particular AFI/SAFI to be used.  The BoQ connection-wide parameters have previously been exchanged over the control channel. The function channel OPEN messages MUST contain identical BGP Autonomous System number and BGP Identifier as the previously established control channel.  It is RECOMMENDED that the BGP Hold Time value exchanged in the function channels be significantly longer than the hold time negotiated for the control channel.  It is the responsibility of the hold timer for the control channel to provide connection verification for the BoQ connection.  The purpose of the function channel negotiated hold time is to provide verification of the communication between the two BoQ speakers for that AFI/SAFI.

(TODO) Errors for mismatched ASN and ID.

The BGP Capabilities carried on the function channel SHOULD only be those that are function-specific, as categorized in the "BGP Capability Category" table.  Conflicting BoQ connection-wide parameters exchanged over the function channel MAY result in the BoQ speaker sending a NOTIFICATION message and not permitting the per-AFI/SAFI function channel to become Established. 

(**) We should identify which capabilities can result in a NOTIFICATION and which can be ignored.

The receiving BoQ speaker replies to those messages as defined in the RFC 4271 FSM by sending its messages (OPEN/NOTIFICATION/KEEPALIVE) addressed to the sender over the control channel.

(TODO - now is the point to discuss changes to BGP messages to support BGP Channels, including how we address specific BGP PDUs to the receiver and is this in the QUIC frame or the BGP PDU)

Once the function channel has reached the Established state, BGP UPDATE messages may be sent to the remote BoQ speaker.

A single function channel for an AFI/SAFI pair results in asymmetric route advertisements.  Both BoQ speakers can create a function channel to implement symmetric route advertisements.

Each function channel is created independently to naturally support multi-channel BGP. The neighbor state machines are decoupled; in case of error, it is possible to reset only one function channel (one direction of a symmetric route exchange). If one function channel is blocked for some reason, other channels can still progress and operate.  


### Channel Reset

A NOTIFICATION is sent over the control channel if the entire BGP connection needs to be reset for any reason, such as a configuration change or a network outage.

If the control channel is closed, the QUIC connection MUST be terminated using a CONNECTION_CLOSE frame, and an error message (TBD) should be included to indicate that the connection has been terminated by BGP. If there are other open channels, they are also closed when the connection is closed.

A function channel can be reset independently without impacting any other function channels or the control channel.

### Channel Coordination

Each function channel has its own keepalive messages with its own timers.  These timers are used to ensure that the session stays alive even if no routing updates are being exchanged.  Non-routing related function channels, such as BGP FlowSpec, may have longer keepalive
timers, for example 120 seconds, as they do not need to exchange
routing updates as frequently as other function channels and can run
in a relatively lower priority.

(**) 
(1) Earlier, we RECOMMENDED that all functional channels have higher hold times.  Here' we're saying "non-routing".

(2) The times and the priority are not the same thing.
   
A single QUIC stream provides ordered and reliable delivery. However, there is no guarantee of transmission and delivery orders across streams. Therefore, if specific data from one channel needs to be received before data from other channels, this requirement must be accomplished through BGP.

As defined in [RFC9000], a QUIC implementation SHOULD provide ways in
which an application can indicate the relative priority of streams.

A BGP implementation utilizing QUIC as its transport protocol MUST support a prioritization mechanism for BGP streams.  This is
essential for ensuring that critical routing information can be transmitted
with higher priority compared to non-routing information.

How to implement the supported priorities using QUIC congestion control at the connection level, stream level flow control, and packetization are out of the scope of this document.

## Protocol Definitions

### BGP Over QUIC Capability

A new ”BGP over QUIC” capability is defined below to signal whether the BoQ speaker is a QUIC client, a QUIC server, or any (Don’t care). The default value is "any"; other values MUST be explicitly configured.

BoQ capability:
  Code: TBD2 (to be assigned by IANA)
  Length: 1(octet)
  Value:
    0 Any
    1 Client
    2 Server

The BoQ Capability is a control-only capability (see the "BGP Capability Category" table).  It MUST be ignored if received in the OPEN message of any function channel. 

A QUIC connection SHOULD be terminated (how?) if the BoQ speaker configuration and the QUIC connection role don't match. For example, if a BoQ speaker is configured as a client, but the QUIC connection comes up as a QUIC server, the QUIC connection should be terminated.  The "any" configuration matches both the QUIC client and QUIC server roles.

If a BoQ speaker is configured as QUIC client, it SHOULD try to initiate the QUIC connection. If a BoQ speaker is configured as QUIC server, it SHOULD wait for a QUIC connection.

(**) There are too many SHOULDs here, and I think a bit of a logic loop.  Should the BoQ and QUIC roles be compared before or after?  If the roles are considered when starting the connection, then there shouldn't be a mismatch later -- but these are only recommendations.  If a server, why would the router wait forever?

When both BGP peers are explicitly configured (one side as client and the other side as server), no collision will happen.
When both BGP peers are “any”, existing session collision mechanism are used.
When both BGP peers are explicitly configured as client (configuration error), a new OPEN message error subcode (BoQ error) MUST be sent.
When both BGP peers are explicitly configured as server (configuration error), no connection will be initiated.
For peering between router A and router B, if A is configured as client and B is any, the following rules are followed:
When A starts a connection as the client, B accepts it.
When B starts a connection as the client, A should start a connection to B as the client, and this connection should be used.
When there are two simultaneous connections, the one with A as the client wins (modification to existing collision resolution). 
For peering between router A and router B, if A is configured as server and B is any, A will wait for the connection from B.

### Capability Category

```md
|Value| Name                              | Ref     | Control/Function |
|-----| --------------------------------- | ------- | -----------------|
| 1   | Multiprotocol Extensions for BGP-4| RFC2858 | Function         |
| 2   | Route Refresh Capability for BGP-4| RFC2918 | Function         |
| 3   |Outbound Route Filtering Capability| RFC5291 | C/F             |
| 5   | Extended Next Hop Encoding        | RFC8950 | C/F
| 6   | BGP Extended Message              | RFC8654 | C
| 7   | BGPsec Capability                 | RFC8205 | C
| 8   | Multiple Labels Capability        | RFC8277 | C
| 9   | BGP Role                          | RFC9234 | C		
| 64  | Graceful Restart Capability       | RFC4724 | F
| 65  | Support for 4-octet AS number     | RFC6793 | C
|     |  capability                       |
| 67  | Support for Dynamic Capability    |         |
|     | (capability specific)             |draft-ietf-idr-dynamic-cap
| 68  | Multisession BGP Capability       |draft-ietf-idr-bgp-multisession
| 69  | ADD-PATH Capability               | RFC7911 | F
| 70  | Enhanced Route Refresh Capability | RFC7313 | F
| 71  | Long-Lived Graceful Restart (LLGR)|
|     | Capability                        |draft-uttaro-idr-bgp-persistence
| 72  | Routing Policy Distribution       |draft-ietf-idr-rpd|	
| 73  | FQDN Capability                   |draft-walton-bgp-hostname-capability
| 74  | BFD Capability                    |draft-ietf-idr-bgp-bfd-strict-mode
| 75  | Software Version Capability	[draft-abraitis-bgp-version-capability-11]	IETF
| ... | ...  |

Table: BGP Capability Category.
```

### Channel Collision Avoidance

A function channel for a specific Network layer protocol MUST NOT be created if one already exists.

If a BoQ speaker receives a function channel creation request for an AFI/SAFI that already exists, the local BoQ speaker SHOULD send a notification with Error Code Sease and subcode BOQ_CHANNEL_CONFLICT through the control channel. (or OPEN Message Error Subcode???? in this case, not only the AFI/SAFI info needed, the stream ID is also needed)

(**) This error is specific to using multiple channels in QUIC; it wouldn't apply to multiple TCP sessions.  IMO, we should create a new error code (BoQ) with the corresponding sub-codes for problems that are BoQ-specific.

If a BoQ speaker receives a functional channel creation request for an AFI/SAFI that it doesn't support, the local BoQ speaker SHOULD send a notification with Erro Code Cease and Subcode BOQ_CHANNEL_NOSUPPORT through the control channel.

(**) This error is a normal AFI/SAFI mismatch, and the normal error should be used.  BTW, which one is that?
"Unsupported AFI/SAFI" or "Unsupported Attribute" (error code 3) in the Open Message Error NOTIFICATION message 

Unless allowed via configuration, a channel collision with an existing BGP channel in the Established state causes the closing of the newly created channel.

(**) I think we may have put this here for the graceful restart case...

### Message Format

In version 1 of QUIC, BoQ messages are carried by QUIC STREAM frames. In BoQ, the control channel always uses QUIC stream 0, which is a client-initiated bidirectional stream.

For function channels, which are unidirectional streams, can be client or server initiated.

Some BoQ messages, although sent in the control channel, are meant for a function channel, such as the responding OPEN message or KEEPALIVE message for a function channel. These messages need to carry the corresponing channel/stream ID information. 

#### BoQ Framing Layer

There are two types of BoQ Frames: Data and Control Data.

Data Frames have the following format:
```yml
---
BoQ Frame Format {
  Type (0),
  Length (),
  Frame Payload (...)
}
---
```
Control Data Frames have the following format:
```yml
---
BoQ Frame Format {
  Type (1),
  Length (),
  Stream ID (),
  Frame Payload (...)
}
---
```
Type: One octcet, it identifies the frame type.
Length: The two-byte unsigned integer that describes the length in bytes of the frame payload.
Stream ID: A Variable-length integer indicating the receiving stream ID of this message.
Frame Payload: BGP messages.
```md
+===============+=================+==================+
|               | Control Channel | Function Channel |
+===============+=================+==================+
| OPEN          | Control Data    |       Data       |
+---------------+-----------------+------------------+
| UPDATE        |                 |       Data       |            
+---------------+-----------------+------------------+
| KEEPALIVE     | Control Data    |       Data       |
+---------------+-----------------+------------------+
| NOTFICATION   | Control Data    |       Data       |
+---------------+-----------------+------------------+
| Route-Refresh |                 |                  |
+-------+------------------------+---------------- --+
```

- OPEN
OPEN message sent in the control channel for the control channel creation MUST NOT contain Multiprotocl Extensions Capability (value 1) in the Capabilites.
OPEN message sent in a function channel and the responding OPEN message sent in the control channel for one AFI/SAFI MUST contain only one Multiprotocl Extensions Capability (value 1) in the Capabilites. 

- UPDATE
There is no UPDATE message sent in the control channel. 

- KEEPALIVE and NOTIFICATION
For the keepalive and NOTIFICATION messages sent in the control channel for one function channel, the BoQ Control Data frame MUST be used, and the stream ID in the frame is to indiate the the target AFI/SAFI.

- Route-Refresh
Route-refresh messages are sent in the control channel using BoQ Control Data Frame.

## Error Handling
* channel connection error: 
** AFI/SAFI conflict: already exists or conflicts with existing AFI/SAFI

* Connection error

## BGP Finite State Machine
TBD

## Operational Considerations

### Using Multi Channel BGP over QUIC
The decision to use BoQ instead of the TCP-based mechanism defined in
[RFC4271] is an operational decision and out of the scope of this
document.  An implementation MUST provide a configuration mechanism
to enable BoQ on a per-peer basis. 
Connectivity problems (e.g., blocking UDP) can result in a failure to
establish a QUIC connection; BGP speakers SHOULD attempt to establish
a TCP-based BGP session in this case.

### BGP Multi Channel Prioritization
One of the drawbacks of a single BGP over TCP session is that control plane messages for all supported Network Layer protocols use the same connection, which may cause resource contention.

QUIC [RFC9000] does not provide a mechanism for exchanging
prioritization information.  Instead, it recommends that
implementations provide ways for an application to indicate the
relative priority of streams, in this case, mapped to BGP channels.
An operator should prioritize BGP channels (streams) that carry
critical control plane information if the functionality is available.
The definition of this functionality and the determination of the
importance of a BGP channel are both outside the scope of this
document.

An example implementation is to have four priority (0-3) defined, and
smaller number means higher priority.  Each AFI/SAFI should be
assigned a default priority and optional configuration to modify the
default value.  For example, IPv4 and IPv6 unicast AFI/SAFI (1/1 and
2/1) may have priority of 1, while BGP-LS (16388/71 and 16388/72) may
have a priority of 3, and BGP FlowSpec (1/133 and 1/134) may have a
priority of 4.

## Security Considerations
This document replaces the transport protocol layer of BGP from TCP
   to QUIC.  It does not modify the basic protocol specifications of
   BGP, and therefore does not introduce new security risks to the basic
   BGP protocol.  The non-TCP-related considerations of [RFC4271],
   [RFC4272], and [RFC7454] apply to the specification in this document.

BoQ enhances transport-layer security for BGP sessions, refer to
   [RFC7454] :

   (1) Supports QUIC server identity authentication.

   (2) (Optional) Supports QUIC client identity authentication.

   (3) Confidentiality protection of BGP messages is supported.  All BGP
   messages are encrypted for transmission.

   (4) Supports integrity protection for BGP messages.

   The use of a specific UDP port number and an ALPN token
   protects a BoQ speaker from attempts to establish an unexpected BGP
   session.  Additionally, all packets directed to UDP port 179 on the
   local device and sourced from an address not known or permitted to
   become a BGP neighbor SHOULD be discarded.

   With BGP multi channel support using QUIC streams, it separates the
   control plane traffic over multiple channels, the effect of a
   session-based vulnerability is reduced; only a single channel is
   affected and not the whole connection.  The result is increased
   resiliency.

   On the other hand, a high number of BGP channels may result in higher
   resource utilization and the risk of depletion.  Also, more channels
   may imply additional configuration and operational complexity.

## IANA Considerations

### UDP Port for BoQ

IANA is requested to assign a UDP port (TBD1) from the "Service Name and Transport Protocol Port Number Registry" as follows:

Service Name | Port Number | Transport Protocol | Description | Assignee | Contact | Registration Date | Modification Date | Reference | Service Code | Unauthorized Use Reported | Assignment Notes 

boq |	TBD1 |	udp |	BGP over QUIC | IEFT | IDR WG | TBD | | [this document] | | idr@ietf.org |


### Registration of the BGP4 Identification String

This document creates a new registration for the identification of
BGP [RFC4271] in the "TLS Application-Layer Protocol Negotiation
(ALPN) Protocol IDs" registry.

   The "boq" string identifies BGP-4 [RFC4271] over QUIC:

   Protocol: Multi-Channel BGP over QUIC

   Identification Sequence: 0x62 0x6f 0x71 ("boq")

   Specification: This document
   
### BGP Over QUIC Capability

   IANA is asked to assign a new Capability Code for the BGP over QUIC
   Capability (Section 3.2.2) as follows:

    +=======+========================+===========+===================+
    | Value | Description            | Reference | Change Controller |
    +=======+========================+===========+===================+
    | TBD2  | BoQ Capability         | [This     | IETF              |
    |       |                        | Document] |                   |
    +-------+------------------------+-----------+-------------------+

                     Table 1: BoQ Capability
### Error Codes
IANA is asked to assign two values from the Cease NOTIFICATION
   Message Error subcodes registry as follows:

     +=======+=====================+=============+================+
     | Value | Name                | Action      |Reference       |
     +=======+=====================+=============+================+
     | TBD   |BOQ_CHANNEL_CONFLICT |Stream Closed|[This Document] |
     +-------+---------------------+-------------+----------------+
     | TBD   |BOQ_CHANNEL_NOSUPPORT|Stream Closed|[This Document] |
     +-------+---------------------+-------------+----------------+
