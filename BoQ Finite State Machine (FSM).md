# BoQ Finite State Machine (FSM)

The data structures and FSM described in this document are conceptual and do not have to be implemented precisely as described here, as long as the implementations support the described functionality and they exhibit the same externally visible behavior.

A BoQ implementation is expected to maintain a separate FSM for each channel. The control channel in a BoQ connection is required to reach the Established state before any function channel can be created. This means the setup of the QUIC connection and any related errors are processed by the control channel.

The BGP messages and events handled by the control and function channels vary. In general, what is specified in RFC 4271 section 8 that applies to a BGP peer connection is applicable to a BoQ channel unless explicitly specified in this document.

RFC 4271 section 8 defines the mandatory and optional session attributes for each connection. For a BoQ implementation, some of these attributes are applicable to both the control and function channels. However some attributes only apply to the control or function channels. The following tables list the applicability of each attribute.

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
| AllowAutomaticStart                  |     Y           |       Y (send)   |
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
The Inputs to the BGP FSM are events.  Events can either be mandatory or optional.  Some optional events are linked to optional session attributes.  Optional session attributes enable several groups of FSM functionality.

The linkage between FSM functionality, events, and the optional session attributes are as described in RFC4271, Section 8.1.1.  Any updates of deviations are indicated below, and the applicability is summarized in the tables above. 

Group 1: Automatic Administrative Events (start/Stop)
Optional Channel Attributes:
AllowAutomaticStart,
AllowAutomaticStop,
DampPeerOscillations,
IdleHoldTime, IdleHoldTimer


Group 2: Unconfigured Peers
Optional Channel Attributes: AcceptConnectionsUnconfiguredPeers


The TCP procssing is updated to be QUIC processing as the following.
Group 3: QUIC processing
Optional Channel Attributes: 
PassiveQUICEstablishment,
TrackQUICState

Option 1: PassiveQUICEstablishment
Description: This option indicated that the BoQ FSM will passivley wait for the remote BGP peer to establish the BGP QUIC connecton. As specified in <Section 5.1>, a BoQ speaker's role can be a QUIC client, a QUIC server, or any. If a BoQ speaker is configured as a QUIC client, the PassiveQUICEstablishment MUST be set to FALSE; when a BoQ speaker is configured as a QUIC server, the PassiveQUICEstablishment MUST be set to TRUE. 
Value: TURE or FALSE

Option 2: TrackQUICState
Description: The BoQ FSM normally tracks the end result of a QUIC connection attempt rather than individual QUIC messages. Optionally, the BoQ FSM can support additional interaction with the QUIC connection negotiation.  The interaction with the QUIC events may increase the amount of logging the BGP peer connection requires and the number of BoQ FSM changes.
Value: TRUE or FALSE

Group 4: BGP Message Processing
Optional Session Attributes: DelayOpen, DelayOpenTime,
                                      DelayOpenTimer,
                                      SendNOTIFICATIONwithoutOPEN,
                                      CollisionDetectEstablishedState

### Adminstrative Events
An administrative event is an event in which the operator interface and BGP Policy engine signal the BoQ FSM to start or stop the BGP state machine.  The basic start and stop indications are augmented by optional connection attributes that signal a certain type of start or stop mechanism to the BoQ FSM.

The settings of optional session attributes may be implicit in some implementations, and therefore may not be set explicitly by an external operator action. The administrative states described below may also be implicit in some implementations and not directly configurable by an external operator.

Event 1: ManualStart
Definition: Local system administrator manually starts a BoQ channel. For the control channel, this event indicates the start of the QUIC connection and the control channel. For a function channel, it is to start an unidirectional channel to the BoQ peer.
Status:     Mandatory
Optional Attribute Status:
For the control channel, the PassiveQUICEstablishment attribute SHOULD be set to FALSE.

Event 2: ManualStop
Definition: Local system administrator manually stops a BoQ channel. For the control channel, this event indicates the end of the BoQ connection.
Status:     Mandatory
Optional Attribute Status: No interaction with any optional attributes.

Event 3: AutomaticStart
Definition: Local system automatically starts a BoQ channel. For the control channel, this event indicates the start of the QUIC connection and the control channel. For a function channel, it is to start an unidirectional channel to the BoQ peer.
Status:     Optional
Optional Attribute Status: 1) The AllowAutomaticStart attribute SHOULD be set to TRUE if this event occurs.
                           2) If the PassiveQUICEstablishment optional session attribute is supported, it SHOULD be set to FALSE.
                           3) If the DampPeerOscillations optional session attribute is supported, it SHOULD be set to FALSE when this event occurs.

Event 4: ManualStart_with_PassiveQUICEstablishment
Definition: Local system administrator manually starts the peer
                     connection, but has PassiveQUICEstablishment
                     enabled.  The PassiveQUICEstablishment optional
                     attribute indicates that the peer will listen prior
                     to establishing the connection. This event only applies to the control channel.
Status:     Optional
Optional Attribute Status: 1) The PassiveQUICEstablishment attribute SHOULD be
                        set to TRUE if this event occurs.
                     2) The DampPeerOscillations attribute SHOULD be set
                        to FALSE when this event occurs.

Event 5: AutomaticStart_with_PassiveQUICEstablishment
Definition: Local system automatically starts the BGP
                     connection with the PassiveQUICEstablishment
                     enabled.  The PassiveQUICEstablishment optional
                     attribute indicates that the peer will listen prior
                     to establishing a connection. This event only applies to the control channel.
Status:     Optional
Optional  Attribute Status: 1) The AllowAutomaticStart attribute SHOULD be set to TRUE.
               2) The PassiveTcpEstablishment attribute SHOULD be set to TRUE.
               3) If the DampPeerOscillations attribute is supported, the DampPeerOscillations SHOULD be set to FALSE.

Event 6: AutomaticStart_with_DampPeerOscillations
Definition: Local system automatically starts the BGP peer
            connection with peer oscillation damping enabled.
            The exact method of damping persistent peer
            oscillations is determined by the implementation
            and is outside the scope of this document.

Status:     Optional

Optional Attribute Status:     1) The AllowAutomaticStart attribute SHOULD be set to TRUE.
                     2) The DampPeerOscillations attribute SHOULD be set to TRUE.
                     3) The PassiveQUICEstablishment attribute SHOULD be set to FALSE.
<!---
For function channels...  AutomaticStart only applies in the send direction.

DampPeerOscillations should apply in both directions. 
--->

Event 7: AutomaticStart_with_DampPeerOscillations_and_PassiveQUICEstablishment
Definition: Local system automatically starts the BGP peer
                     connection with peer oscillation damping enabled
                     and PassiveQUICEstablishment enabled.  The exact
                     method of damping persistent peer oscillations is
                     determined by the implementation and is outside the
                     scope of this document. This event only applies to the control channel.

Status:     Optional

Optional Attributes Status:     1) The AllowAutomaticStart attribute SHOULD be set to TRUE.
                     2) The DampPeerOscillations attribute SHOULD be set to TRUE.
                     3) The PassiveQUICEstablishment attribute SHOULD be set to TRUE.


Event 8: AutomaticStop
Definition: Local system automatically stops a BoQ channel. For the control channel, this event indicates the end of the BoQ connection.

An example of an automatic stop event for a function channel is exceeding the number of prefixes for a given peer and the local system automatically disconnecting the peer.

Status:     Optional

Optional Attribute Status:     1) The AllowAutomaticStop attribute SHOULD be TRUE.

### Timer Events
The definition of events 9 through 11 are the same as in Section 8.1.3 of RFC 4271.

### QUIC Connection-Based Events
Event 14: QUICConnection_Valid
Definition: Event indicating the local system reception of a QUIC connection request with a valid source IP address, UDP port, destination IP address, and UDP Port.  The definition of invalid source and invalid destination IP address is determined by the implementation.
BoQ's destination port SHOULD be port TDB1 (Section 4.1).
A QUIC connection request is denoted by the local system receiving the QUIC Intitial packet [RFC9000].
Status:     Optional
Optional Attribute Status:  The TrackQUICState attribute SHOULD be set to TRUE if this event occurs.

Event 15: QUIC_CR_Invalid
    Definition: Event indicating the local system reception of a
                     QUIC connection request with either an invalid
                     source address or port number, or an invalid
                     destination address or port number.

                     BOQ's destination port number SHOULD be TBD1 (Section 4.1).

                     A QUIC connection request occurs when the local
                     system receives a QUIC Initial packet [RFC9000].

         Status:     Optional

         Optional
         Attribute
         Status:     1) The TrackTcpState attribute should be set to
                        TRUE if this event occurs.

Applicability: the control channel on the QUIC server

Even 16: QUIC_CR_Acked
Definition: Event indicating the local system's request to establish a QUIC connection to the remote peer has been completed. 
The local system's QUIC connection request has completed and a HANDSHAKE_DONE frame is recieved [RFC9000].
Status:     Mandatory

Event 17: QUICConnectionConfirmed
Definition: Event indicating that the local system has received a confirmation that the QUIC connection has been established by the remote site. 
The HANDSHAKE_DONE frame has been acknowledged [RFC9000].
Status:     Mandatory

Event 18: QUICConnectionFails
Definition: Event indicating that the local system has received a QUIC connection failure notice. 
The local peer has received a QUIC CONNECTION_CLOSE frame [RFC9000].
Status:     Mandatory

### BGP Message-Based Events

The definition of events 19 through 28 is the same as in Section 8.1.5 of RFC 4271.


## Description of FSM

BoQ implementations are expected to maintain a sparate FSM for each BoQ channel. As described later in this section, a BoQ implementation will have one FSM for the control channel, plus one FSM for each direction of a function channel. For example, two BoQ speakers configured to advertise IPv6 routes in both directions will have 3 active FSMs: one for the control channel, and one for each direction (send and receive) of the IPv6 route exchange.


### Terms "active" and "passive"
There is only one active side and one passive side to any one QUIC connection. In BoQ, the distinction is significant and they correspond to the QUIC client and server roles.  Refer to <Section 5.1> for more details.,

### FSM and Collision Detection

There is one control clannel FSM (Section x) per BoQ connection. When a connection collision occurs prior to determining what peer a connection is associated with, there may be two connections for one peer. Connection collisions are resolved using the specification in Section 6.8 of RFC 4271. After the connection collision is resolved, the FSM for the connection that is closed SHOULD be disposed.  

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

Any Start event (Events 1, 3, 6) is ignored in the Established state.

In response to a ManualStop event (initiated by an operator)(Event 2), the local system:

- sends the NOTIFICATION message with a Cease,
- deletes all routes associated with this connection,
- releases related BGP resources,
- drops the stream/channel connection,
- changes its state to Idle.

In response to an AutomaticStop event (Event 8), the local system:

- sends a NOTIFICATION with a Cease,
- deletes all routes associated with this connection,
- releases related BGP resources,
- drops the stream/channel connection,
- (optionally) performs peer oscillation damping if the DampPeerOscillations attribute is set to TRUE, and
- changes its state to Idle.

If the HoldTimer_Expires event occurs (Event 10), the local system:
- sends a NOTIFICATION message with the Error Code Hold Timer Expired,
- releases related BGP resources,
- drops the stream/channel connection,
- (optionally) performs peer oscillation damping if the DampPeerOscillations attribute is set to TRUE, and
- changes its state to Idle.

If the KeepaliveTimer_Expires event occurs (Event 11), the local system:
- sends a KEEPALIVE message, and
- restarts its KeepaliveTimer, unless the negotiated HoldTime value is zero.

Each time the local system sends a KEEPALIVE or UPDATE message, it restarts its KeepaliveTimer, unless the negotiated HoldTime value is zero.

Any QUIC event (Event 14-17) received SHOULD be ignored.

If the local system receives a NOTIFICATION message (Event 24 or 25) from the control channel with matching StreamID, the local system:

 - deletes all routes associated with this AFI/SAFI,
 - releases the related BGP resources,
 - drops the stream/channel connection,
 - changes its state to Idle.

If the local system receives a KEEPALIVE message (Event 26) in the control channel with matching StreamID, the local system:
- restarts its HoldTimer, if the negotiated HoldTime value is non-zero, and
- remains in the Established state.

A functon channel is unidirecitonal, it SHOULD NOT receive any UPDATE message from the control channel with matching StreamID. In case an UPDATE message is received, it SHOULD be ignored.

### Funtion Channel Receiving FSM

In BoQ, function channels are unidirectional, hence after the control channel reaches established state, as the receiving side a BoQ speaker doesn't know what function channels will be created by its peer. A BoQ implementation is suggested to have a central process or thread to handle BGP packets received from its peer in function channels that don't have receiving FSMs yet, we can call this the packet dispatcher.
This despatcher SHOULD only handle OPEN messages received from its BoQ peer, any other BGP messages SHOULD be ignored.
When an OPEN message is received by the dispatcher, and there is no receiving FSM created for the funtion channel, a funtion chanel receiving FSM SHOULD be created to handle packets from the stream. For example, when an OPEN packet for IPv6 Unicast is received on stream #2, and no receiving FSM for IPv6 unicast exists, a receiving FSM SHOULD be created for IPv6 unicast, and the StreamID should be set to 2. When a receiving FSM for IPv6 Unicast with different StreamID already exists, a NOTIFICATION with BoQ error code, BoQ channel conflict subcode SHOULD be sent in the control channel (see section 5.2). 

#### Idle State
After a function channel receiving FSM is created, it starts from the Idle state.

In this state, the local system:
- initialize related BGP resources for the peer connection for the AFI/SAFI
- changes its state to Connect.

The ManualStop event (Event 2) and AutomaticStop (Event 8) event are ignored in the Idle state.

#### Connect State
In this state, the function channel receiving FSM process the received OPEN message, if there is no error in the OPEN message, the local system:
- sends an OPEN message in the control channel as acknowledgement with the StreamID,
- sends a KEEPALIVE message in the control channel,
- if the HoldTimer initial value is non-zero,
  - starts the KeepaliveTimer with the initial value and
  - resets the HoldTimer to the negotiated value,

  else, if the HoldTimer initial value is zero,
  - resets the KeepaliveTimer and
  - resets the HoldTimer value to zero,
- and changes its state to OpenConfirm.

If BGP message header checking (Event 21) or OPEN message checking detects an error (Event 22), the local system:
- (optionally) If the SendNOTIFICATIONwithoutOPEN attribute is set to TRUE, then the local system first sends a NOTIFICATION message with the appropriate error code, and then
- releases all related BGP resources,
- delete the function channel receiving FSM.

If a NOTIFICATION message is received with a version error (Event 24), the local system:
- releases all related BGP resources,
- delete the function channel receiving FSM.

In response to any other events (Events 8, 10-11, 13, 19, 23, 25-28), the local system:
- releases all related BGP resources,
- delete the function channel receiving FSM.

#### OpenConfirm State

In the state, the receiving FSM wait for a KEEPALIVE or NOTIFICATION message.

Any start event (Events 1, 3-7) is ignored in the OpenConfirm state.

In response to a ManualStop event (Event 2) initiated by the oerator or the AutomaticStop event initiated by the system (Event 8), the local system:
- sends the NOTIFICATION message with a Cease in the control channel,
- releases all related BGP resources,
- delete the function channel receiving FSM.

If the HoldTimer_Expires event (Event 10) occurs before a KEEPALIVE message is received, the local system:
- sends the NOTIFICATION message with the Error Code Hold Timer Expired,
- releases all related BGP resources,
- delete the function channel receiving FSM.

If the local system receives a KeepaliveTimer_Expires event (Event 11), the local system:
- sends a KEEPALIVE message in the control channel,
- restarts the KeepaliveTimer, and
- remains in the OpenConfirmed state.

If the local system receives a NOTIFICATION message with a version error (NotifMsgVerErr (Event 24)), the local system:
- restarts the KeepaliveTimer, and
- remains in the OpenConfirmed state.

If a valide OPEN message is received on the same stream, it SHOULD be ignored.

If an OPEN message is received with error,  (BGPHeaderErr (Event 21)or BGPOpenMsgErr (Event 22)), the local system:
- sends a NOTIFICATION message with the appropriate error code in the control channel,
- releases all related BGP resources,
- delete the function channel receiving FSM.

If the local system receives a KEEPALIVE message (KeepAliveMsg (Event 26)), the local system:
- restarts the HoldTimer and
- changes its state to Established.

In response to any other event (Events 9, 12-13, 20, 27-28), the local system:

- sends a NOTIFICATION with a code of Finite State Machine Error,
- releases all related BGP resources,
- delete the function channel receiving FSM.

#### Established State
In the Established state, a function channel receiving FSM can receive UPDATE, NOTIFICATION, and KEEPALIVE messages from its peer.

Any Start event (Events 1, 3-7) is ignored in the Established state.

In response to a ManualStop event (initiated by an operator) (Event 2), or an AutomaticStop event (Event 8), the local system:
- sends the NOTIFICATION message with a Cease in the control channel,
- deletes all routes associated with this connection,
- releases all related BGP resources,
- delete the function channel receiving FSM.

If the HoldTimer_Expires event occurs (Event 10), the local system:
- sends a NOTIFICATION message with the Error Code Hold Timer Expired in the control channel,
- releases all related BGP resources,
- delete the function channel receiving FSM.

If the KeepaliveTimer_Expires event occurs (Event 11), the local system:
- sends a KEEPALIVE message in the control channel, and
- restarts its KeepaliveTimer, unless the negotiated HoldTime value is zero.

Each time the local system sends a KEEPALIVE in the control channel, it restarts its KeepaliveTimer, unless the negotiated HoldTime value is zero.

If the local system receives a NOTIFICATION message (Event 24 or Event 25), the local system:
- deletes all routes associated with this connection,
- releases all related BGP resources,
- delete the function channel receiving FSM.

If the local system receives a KEEPALIVE message (Event 26), the local system:
 - restarts its HoldTimer, if the negotiated HoldTime value is non-zero, and
 - remains in the Established state.

If the local system receives an UPDATE message (Event 27), the local system:
- processes the message,
- restarts its HoldTimer, if the negotiated HoldTime value is non-zero, and
- remains in the Established state.

If an UPDATE message is received with error (Event 28), the lcoal system:
- sends a NOTIFICATION message with an Update error,
- deletes all routes associated with this connection,
- releases all related BGP resources,
- delete the function channel receiving FSM.

In response to any other event (Events 9, 12-13, 20-22), the local system:
- sends a NOTIFICATION message with the Error Code Finite State Machine Error,
- deletes all routes associated with this connection,
- releases all related BGP resources,
- delete the function channel receiving FSM.