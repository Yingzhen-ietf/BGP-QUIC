# BoQ Finite State Machine (FSM)

The data structures and FSM described in this document are conceptual and do not have to be implemented precisely as described here, as long as the implementations support the described functionality and they exhibit the same externally visible behavior.

A BoQ implementation is expected to maintian a separate FSM for each channel. The control channel in a BoQ connection is required to reach the Established state before any function channel can be created. This means the set up of the QUIC connection and any related errors are processed by the control channel.

The BGP messages and events handled by the control channel and function channels are different. In general, what is specified in RFC 4271 section 8 that applies to a BGP peer connection is applicable to a BoQ channel unless explicitly specified in this document.

RFC 4271 section 8 defines the mandatory and optional session attributes for each connection. For a BoQ implementation, some of these attributes are applicable to both the control channel and function channels, however there are some attributes that are applicable to only to the control channel or  funtion channels. The following tables list applicability of each attribute.

```md
| Channel Attributes | Control Channel | Function Channel |
| ------------------ | --------------- | -----------------|
| State              |     Y           |       Y          |
| ConnectRetryCounter|     Y           |       Y (receive)|
| ConnectRetryTimer  |     Y           |       Y (send)   |
| ConnectRetryTime   |     Y           |       Y (send)   |
| HoldTimer          |     Y           |       Y          |
| HoldTime           |     Y           |       Y          |
| KeepaliveTimer     |     Y           |       Y          |
| KeepaliveTime      |     Y           |       Y          |
| Stream ID          |     N           |       Y          |

Table: Mandatory Chanel Attributes
```

<!--- 
For the Function Channel, I changed ConnectRetryCounter to "Y" because it is applicable on the receive side.  How many times has my peer attempted to start the AFI/SAFI session?

ConnectRetryCounter doesn't make sense on the send side of a Function Channel.

The ConnectRetryTimer/ConnectRetryTime make sense in the send side of the Function Channel.  rfc4271 talks about these times as related to the TCP session -- in BoQ, the QUIC connection is already up so these attributes apply to the stream setup.

Do we need to expand the table to indicate the send/receive separation?
 --->

For a BoQ function channel, a new mandatory attribute StreamID is required.  This attribute indicates the QUIC Stream ID used by the Function Channel.

```md
| Channel Attributes                   | Control Channel | Function Channel |
| ------------------------------------ | --------------- | -----------------|
| AcceptConnectionsUnconfiguredPeers   |     Y           |       N          |
<!--- 
AcceptConnectionsUnconfiguredPeers doesn't apply to the function channel because the QUIC connection is already up.  
--->
| AllowAutomaticStart                  |     Y           |       Y          |
| AllowAutomaticStop                   |     Y           |       Y          |
<!--- 
We should be able to srart/stop specific AFs at any time, based on "implementation-specific" logic.
--->
| CollisionDetectEstablishedState      |     Y           |       N          |
| DampPeerOscillations                 |     Y           |       Y          |
| DelayOpen                            |     Y           |       N          |
| DelayOpenTime                        |     Y           |       N          |
| DelayOpenTimer                       |     Y           |       N          |
| IdleHoldTime                         |     Y           |       Y          |
| IdleHoldTimer                        |     Y           |       Y          |
| PassiveQUICEstablishment             |     Y           |       N          |    
| SendNOTIFICATIONwithoutOPEN          |     Y           |       Y (receive)|
<!--- 
The receive side of the function channel should be able to send a NOTIFICATION without sending an OPEN first.
--->
| TrackQUICState                       |     Y           |       N          |

Table: Optional Chanel Attributes
```

## Events for the BoQ FSM

### Optional Events
Same as defined in RFC 4271, BoQ events can either be mandatory or optional. 

Group 1: Automatic Administrative Events (start/Stop)
Optional Channel Attributes:
AllowAutomaticStart,
AllowAutomaticStop,
DampPeerOscillations,
IdleHoldTime, IdleHoldTimer
Applicability: the control and funtion channels.

Group 2: Unconfigured Peers
Optional Channel Attributes: AcceptConnectionsUnconfiguredPeers
Applicability: the control and funtion channels

The TCP procssing is updated to be QUIC processing as the following.
Group 3: QUIC processing
Optional Channel Attributes: 
PassiveQUICEstablishment,
TrackQUICState

Option 1: PassiveQUICEstablishment
Description: This option indicated that the BoQ FSM will passivley wait for the remote BGP peer to establish the BGP QUIC connecton. As specified in <Section 5.1>, a BoQ speaker's role can be a QUIC client, a QUIC server, or any. If a BoQ speaker is configured as a QUIC client, the PassiveQUICEstablishment MUST be set to FALSE; when a BoQ speaker is configured as a QUIC server, the PassiveQUICEstablishment MUST be set to TRUE. 
Value: TURE or FALSE
Applicability: the control channel

Option 2: TrackQUICState
Description: The BoQ FSM normally tracks the end result of a QUIC connection attempt rather than individual QUIC messages. Optionally, the BoQ FSM can support additional interaction with the QUIC connection negotiation.  The interaction with the QUIC events may increase the amount of logging the BGP peer connection requires and the number of BoQ FSM changes.
Value: TRUE or FALSE
Applicability: the control channel

Group 4: BGP Message Processing
Optional Session Attributes: DelayOpen, DelayOpenTime,
                                      DelayOpenTimer,
                                      SendNOTIFICATIONwithoutOPEN,
                                      CollisionDetectEstablishedState
Applicability: the control channel

### Adminstrative Events

Event 1: ManualStart
Definition: Local system administrator manually starts a BoQ channel.
Status:     Mandatory
Optional Attribute Status:
For the control channel, the PassiveQUICEstablishment attribute SHOULD be set to FALSE.
Applicability: the control and function channels. For the control channel, this means the start of the QUIC connection and the control channel. For a function channel, it is to start the unidirectional channel to the BoQ peer.

Event 2: ManualStop
Definition: Local system administrator manually stops a BoQ channel.
Status:     Mandatory
Optional Attribute Status: No interaction with any optional attributes.
Applicability: For the control channel, this means to stop the BoQ connection. For a function channel, the default behavior is to stop the unidirectional QUIC stream connection. When an optional parameter is included to inidcate that it is to stop the stream connection, as well as the unidrectional stream (receiving side). In this case, a Cease NOTIFICATION SHOULD be sent to the BoQ peer with Administrative Shutdown subcode. 

Event 3: AutomaticStart
Definition: Local system automatically starts the BoQ connection.
Applicability: the control channel

Event 4: ManualStart_with_PassiveQUICEstablishment
Definition: Local system administrator manually starts the peer
                     connection, but has PassiveQUICEstablishment
                     enabled.  The PassiveQUICEstablishment optional
                     attribute indicates that the peer will listen prior
                     to establishing the connection.
Status:     Optional, depending on local system
Optional Attribute Status: 1) The PassiveQUICEstablishment attribute SHOULD be
                        set to TRUE if this event occurs.
                     2) The DampPeerOscillations attribute SHOULD be set
                        to FALSE when this event occurs.
Applicability: the control channel

Event 5: AutomaticStart_with_PassiveQUICEstablishment
Definition: Local system automatically starts the BGP
                     connection with the PassiveQUICEstablishment
                     enabled.  The PassiveQUICEstablishment optional
                     attribute indicates that the peer will listen prior
                     to establishing a connection.

Status:     Optional, depending on local system
Optional  Attribute Status: 1) The AllowAutomaticStart attribute SHOULD be set         to TRUE.
               2) The PassiveTcpEstablishment attribute SHOULD be set to TRUE.
               3) If the DampPeerOscillations attribute is supported, the DampPeerOscillations SHOULD be set to FALSE.
Applicability: the control channel

Event 6: AutomaticStart_with_DampPeerOscillations
Applicability: the control channel

Event 7: AutomaticStart_with_DampPeerOscillations_and_PassiveTcpEstablishment
Applicability: the control channel

Event 8: AutomaticStop
Applicability: the control channel

### Timer Events
Event 9: ConnectRetryTimer_Expires
Applicability: the control channel

Event 10: HoldTimer_Expires
Applicability: the control and function channels

Event 11: KeepaliveTimer_Expires
Applicability: the control and function channels

Event 12: DelayOpenTimer_Expires
Applicability: the control channel

Event 13: IdleHoldTimer_Expires
Applicability: the control and function channels

### QUIC Connection-Based Events
Event 14: QUICConnection_Valid
Definition: Event indicating the local system reception of a QUIC connection request with a valid source IP address, UDP port, destination IP address, and UDP         Port.  The definition of invalid source and invalid destination IP address is determined by the implementation.
BGP's destination port SHOULD be port TDB1, Please refer to the IANA Considerations.
QUIC connection request is denoted by the local system receiving the QUIC Intitial packet.
Status:     Optional
Optional Attribute Status:  The TrackQUICState attribute SHOULD be set to TRUE if this event occurs.
Applicability: the control channel on the QUIC server

Event 15: QUIC_CR_Invalid
Applicability: the control channel on the QUIC server

Even 16: QUIC_CR_Acked
Definition: Event indicating the local system's request to establish a QUIC connection to the remote peer. The local system's QUIC connection sent a QUIC Initial (ClientHello), received a QUIC Initial (ServerHello), and sent a QUIC Ack. When this event is received, the QUIC client has reached the Handshake Complete state.
Status:     Mandatory
Applicability: the control channel on the QUIC client

Event 17: QUICConnectionConfirmed
Definition: Event indicating that the local system has received a confirmation that the QUIC connection has been established by the remote site. This applies to both QUIC client and server, indicating that the Handshake Confirmed state has been reached.
Status:     Mandatory
Applicability: the control channel

Event 18: QuicConnectionFails
Definition: Event indicating that the local system has received a QUIC connection failure notice. It means that an error occurs in the QUIC handshake before the system enters the Handshake Confirmed state.
Status:     Mandatory
Applicability: the control channel

### BGP Message-Based Events

Event 19: BGPOpen
Applicability: the control and function channels

Event 20: BGPOpen with DelayOpenTimer running
Applicability: the control channel

Event 21: BGPHeaderErr
Applicability: the control and function channels

Event 22: BGPOpenMsgErr
Applicability: the control and function channels

Event 23: OpenCollisionDump
Applicability: the control and function channels

Event 24: NotifMsgVerErr
Applicability: the control and function channels

Event 25: NotifMsg
Applicability: the control and function channels

Event 26: KeepAliveMsg
Applicability: the control and function channels

Event 27: UpdateMsg
Applicability: function channels

Event 28: UpdateMsgErr
Applicability: function channels


## Description of FSM

BoQ MUST maintain a sparate FSM for each BoQ channel. A BoQ implementation will have one FSM for the control channel, plus one FSM for each function channel for each direction. For example between two BoQ speakers, bidirectional IPv4 Unicast AFI/SFI is enabled, as well as a unidirectional FlowSpec. For this case, each BoQ speaker will have 4 FSMs: one for the control channel, plus two for the IPv4 unicast, plus one for the unidirectional FlowSpec.


### Terms "active" and "passive"
For BGP over TCP, once the TCP connection is completed, it doesn't matter which end was active and which was passive. Please refer to <RFC 4271 section 8.2.1.1>.

However, in a BoQ implementation, because QUIC supports connection migration, and only the client side can move. The role of the BoQ speaker is significant. Please refer to <section 5.1> for role configurations.

### FSM and Collision Detection

For BoQ implementations using sepcifications in this document, there is only one QUIC connection between two BGP peers. Same as what defined in RFC 4271, when the connection collision occors, it is resolved using the steps defined in section 6.8 of RFC 4271. 


### Control Channel FSM

The FSM defined in section 8.2.2 of RFC 4271 still applies to the control channel of a BoQ speaker after the TCP related events and state are replaced with the corresponding QUIC events and state.

The changes and difference are explictly defined below.
   
#### Idle state

Initially, the control channel FSM is in the Idle state. 

In this state, the control channel FSM refuses all incoming BGP connections for this peer.  No resources are allocated to the peer.  In response to a ManualStart event (Event 1) or an AutomaticStart event (Event 3), the local system:
 
 - initializes all BGP resources for the peer connection,
 - sets ConnectRetryCounter to zero,
 - starts the ConnectRetryTimer with the initial value,
 - initiates a QUIC connection to the other BGP peer if the local system is configured as BoQ client or any
 - listens for a connection that may be initiated by the remote BGP peer, and
 - changes its state to Connect.

 The ManualStop event (Event 2) and AutomaticStop (Event 8) event are ignored in the Idle state.
 
In response to a ManualStart_with_PassiveQUICEstablishment event (Event 4) or AutomaticStart_with_PassiveQUICEstablishment event (Event 5), the local system:
 - initializes all BGP resources,
 - sets the ConnectRetryCounter to zero,
 - starts the ConnectRetryTimer with the initial value,
 - listens for a connection that may be initiated by the remote peer, and
 - changes its state to Active.

The exact value of the ConnectRetryTimer is a local matter, but it SHOULD be sufficiently large to allow QUIC initialization.

If the DampPeerOscillations attribute is set to TRUE, the following three additional events may occur within the Idle state:

- AutomaticStart_with_DampPeerOscillations (Event 6),

- AutomaticStart_with_DampPeerOscillations_and_
          PassiveTcpEstablishment (Event 7),

- IdleHoldTimer_Expires (Event 13).

Upon receiving these 3 events, the local system will use these events to prevent peer oscillations.  The method of preventing persistent peer oscillation is outside the scope of this document.

Any other event (Events 9-12, 15-28) received in the Idle state does not cause change in the state of the local system.
   
      
#### Connect State
In this state, the control channel FSM is waiting for the QUIC connection to be completed.

After a QUIC connection is successfully established, the BoQ speaker sends an OPEN message to its peer and changes it's state to OpenSent.

#### Active State

In this state, the control channel FSM is trying to acquire a peer by listening for, and accepting, a QUIC connection.

#### OpenSent
The control channel FSM waits for an OPEN message from its peer. 


#### OpenConfirm
The control channel FSM waits for a KEEPALIVE or NOTIFICATION message.

#### Established
In this state, the control channel FSM can exchange NOTIFICATION, KEEPALIVE and ROUTEREFRESH messages with its peer. 
There is no UPDATE message in the control channel.

The following text is not applicable to BoQ control channel any more.
   > One reason for an AutomaticStop event is: A BGP receives an UPDATE
   messages with a number of prefixes for a given peer such that the
   total prefixes received exceeds the maximum number of prefixes
   configured.  The local system automatically disconnects the peer.

When a KEEPALIVE message is received, whether it is destined to the control channel or a function channel, the local system:
- restarts its HoldTimer, if the negotiated HoldTime value is non-zero, and
- remains in the Established state.

If the local system receives an UPDATE message, it SHOULD be ignored.


### Function Channel

Function channels can only be created after the control channel has reached Established state. This means that a funtion channel SHOULD NOT send any QUIC connection related events (14-18).

Function channels are unidirectional. For one AFI/SAFI, there might be one or two FSMs, one function channel sending FSM and one funtion channel receiving FSM. 

### Funtion Channel Sending FSM

#### Idle State
Function channel sending FSM starts from idle state as well. 

In response to a ManualStart event (Event 1) or an AutomaticStart event (Event 3), the local system:
  - initializes the function channel BGP resources for the peer connection,
  - changes its state to Connect.

The ManualStop event (Event 2) and AutomaticStop (Event 8) event are ignored in the Idle state.

If the DampPeerOscillations attribute is set to TRUE, the following two additional events may occur within the Idle state:
 - AutomaticStart_with_DampPeerOscillations (Event 6),
 - IdleHoldTimer_Expires (Event 13).

Upon receiving these 2 events, the local system will use these events to prevent peer oscillations.  The method of preventing persistent peer oscillation is outside the scope of this document.

Any event related with PassiveQUICEstablishment SHOULD NOT be received in a function channel:
ManualStart_with_PassiveTcpEstablishment event (Event 4), AutomaticStart_with_PassiveTcpEstablishment event (Event 5),  or AutomaticStart_with_DampPeerOscillations_and_PassiveTcpEstablishment (Event 7).
Any other event received in the Idle state does not cause change in the state of the local system.

#### Connect State
The start events (Events 1, 3-7) are ignored in the Connect state.

In response to a ManualStop event (Event 2), the local system:
- releases all BGP resources for the channel,
- changes its state to Idle.

The local system:
- sends an OPEN message to its peer using a unidirectional QUIC stream,
- sets the HoldTimer to a large value,
- changes its state to OpenSent,
- record the StreamID.

If the value of the autonomous system field is the same as the local Autonomous System number, set the connection status to an internal connection; otherwise it will be "external".

If a NOTIFICATION message is received with a version error (Event 24) in the control channel destined to the function channel, the local system:
- release all BGP resources for the function channel,
- terminates the function channel/QUIC stream,
- performs peer oscillation damping if the DampPeerOscillations attribute is set to True, and
- changes its state to Idle.


#### OpenSent
A BoQ speaker waits for an OPEN_ACK message from its peer in the control channel with destination StreamID matches its own StreamID.
When an OPEN_ACK message is received, it's checked for correctness. In case of error, the local BoQ speaker:
- sends a NOTFICATION message
- closes the channel/QUIC stream, 
- releases the corresponding BGP resources,
- (optionally) performs peer oscillation damping if the DampPeerOscillations attribute is TRUE, and
- and changes its state to Idle.
If there is no error in the OPEN_ACK message, the BoQ speaker sends a KEEPALIVE message and sets a KeepAlive timer, sets the HoldTimer and moves it's state to OpenConfirm.
If the HoldTimer_Expires (Event 10), the local system:
- sendssends a NOTIFICATION message with the error code Hold Timer Expired,
- releases all BGP resources,
- (optionally) performs peer oscillation damping if the DampPeerOscillations attribute is set to TRUE, and
- changes its state to Idle.

If a NOTIFICATION message is received in the control channel  with version error (Event 24) and matching StreamID, the local system:
- relase the related BGP resources,
- drop the stream, and
- changes its state to Idle.

In response to any other event (Events 11, 13, 25-28), the local system:

- sends the NOTIFICATION with the Error Code Finite State Machine Error,
- releases related BGP resources,
- drops the QUIC stream connection/channel,
- (optionally) performs peer oscillation damping if the  DampPeerOscillations attribute is set to TRUE, and
- changes its state to Idle.

#### OpenConfirm
Waiting for a KEEPALIVE. When it is received in the control channel with matcing StreamID, the BoQ speaker's state moved Established.
If the HoldTimer expires, the BoQ speaker sends a NOTIFICATION with error code Hold Timer Expired and terminates the channel.
If a NOTIFICATION sent to the control channel with the matching destination StreamID is received, it closes the channel.

In response to the AutomaticStop event initiated by the system (Event 8), the local system:

- sends the NOTIFICATION message with a Cease,
- releases all BGP resources,
- drops the stream/channel connection,
- (optionally) performs peer oscillation damping if the DampPeerOscillations attribute is set to TRUE, and
- changes its state to Idle.

If the HoldTimer_Expires event (Event 10) occurs before a
KEEPALIVE message is received, the local system:
 - sends the NOTIFICATION message with the Error Code Hold Timer Expired,
- releases related BGP resources,
- drops the stream/channel connection,
- (optionally) performs peer oscillation damping if the DampPeerOscillations attribute is set to TRUE, and
- changes its state to Idle.

If the local system receives a KeepaliveTimer_Expires event (Event 11), the local system:

- sends a KEEPALIVE message,
- restarts the KeepaliveTimer, and
- remains in the OpenConfirmed state.

If the local system receives a KEEPALIVE message (KeepAliveMsg (Event 26)) from the control channel, the local system:
- restarts the HoldTimer and
- changes its state to Established.

#### Established
When the sending function channel reaches established state, it sends UPDATE, NOTIFCATION and KEEPALIVE messages to its peer.

In response to a ManualStop event (initiated by an operator)(Event 2), the local system:

- sends the NOTIFICATION message with a Cease,
- deletes all routes associated with this connection,
- releases related BGP resources,
- drops the stream/channel connection,
- changes its state to Idle.
