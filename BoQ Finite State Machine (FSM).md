# BoQ Finite State Machine (FSM)

A BoQ implementation MUST maintian a separate FSM for each channel. The control channel in each BoQ connection needs to reach the Established state before any function channel can be created. This means the set up of the QUIC connection and any related errors are processed by the control channel.

The BGP messages and events handled by by the control channel and function channels are different. In general, what specified in RFC 4271 section 8 that applies to a BGP peer connection is applicable to a BoQ channel unless explicitly specified in this document.

RFC 4271 section 8 defines the mandatory and optional session attributes for each connection. For a BoQ implementation, some of these attributes are applicable to both the control channel and function channels, however there are some attributes that are applicable to only to the control channel or  funtion channels. The following tables list applicability of each attribute.

```md
| Channel Attributes | Control Channel | Function Channel |
| ------------------ | --------------- | -----------------|
| State              |     Y           |       Y          |
| ConnectRetryCounter|     Y           |       N          |
| ConnectRetryTimer  |     Y           |       N          |
| ConnectRetryTime   |     Y           |       N          |
| HoldTimer          |     Y           |       Y          |
| HoldTime           |     Y           |       Y          |
| KeepaliveTimer     |     Y           |       Y          |
| KeepaliveTime      |     Y           |       Y          |

Table: Mandatory Chanel Attributes
```

```md
| Channel Attributes                   | Control Channel | Function Channel |
| ------------------------------------ | --------------- | -----------------|
| AcceptConnectionsUnconfiguredPeers   |     Y           |       Y          |
| AllowAutomaticStart                  |     Y           |       N          |
| AllowAutomaticStop                   |     Y           |       N          |
| CollisionDetectEstablishedState      |     Y           |       N          |
| DampPeerOscillations                 |     Y           |       Y          |
| DelayOpen                            |     Y           |       N          |
| DelayOpenTime                        |     Y           |       N          |
| DelayOpenTimer                       |     Y           |       N          |
| IdleHoldTime                         |     Y           |       Y          |
| IdleHoldTimer                        |     Y           |       Y          |
| PassiveQUICEstablishment             |     Y           |       N          |    
| SendNOTIFICATIONwithoutOPEN          |     Y           |       N          |
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
Description: This option indicated that the BoQ FSM will passivley wait for the remote BGP peer to establish the BGP QUIC connecton. As specified in Section 5.1, a BoQ speaker's role can be a QUIC client, a QUIC server, or any. If a BoQ speaker is configured as a QUIC client, the PassiveQUICEstablishement SHOULD be set to FALSE. 
Value: TURE or FALSE
Applicability: the control channel

Option 2: TrackQUICState
Description: The BoQ FSM normally tracks the end result of a
                      QUIC connection attempt rather than individual QUIC
                      messages.  Optionally, the BoQ FSM can support
                      additional interaction with the QUIC connection
                      negotiation.  The interaction with the QUIC events
                      may increase the amount of logging the BGP peer
                      connection requires and the number of BoQ FSM
                      changes.
Value:       TRUE or FALSE
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
Optional Attribute Status:  The TrackQUICState attribute SHOULD be set to
                        TRUE if this event occurs.
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

### Control Channel

#### Connect State
After a QUIC connection is successfully established, the BoQ speaker sends an OPEN message to its peer and changes it's state to OpenSent.

#### OpenSent
A BoQ waits for an OPEN message from its peer. When an OPEN message is received, it's checked for correctness. In case of error, the local BoQ speaker sends a NOTFICATION message, terminates the QUIC connection and changes its state to Idle.
If there is no error in the OPEN message, the BoQ speaker sends a KEEPALIVE message and sets a KeepAlive timer, and moves it's state to OpenConfirm.
If a disconnect notification from QUIC or a administrative shutdown is received, the BoQ speaker terminates the BGP/QUIC connection and goes into the Active state.

#### OpenConfirm
Waiting for a KEEPALIVE. When it is received, the BoQ speaker's state moved Established.
If the HoldTimer expires, the BoQ speaker sends a NOTIFICATION with error code Hold Timer Expired and changes its state to Idle.
If a NOTIFICATION sent to the control channel is received, it changes its state to IDLE.
When a BoQ speaker changes its state from OpenConfirm to Idle, it closes both the BGP and QUIC connection.

#### Established
When the control channel reaches established state, it sends NOTIFCATION and KEEPALIVE messages to its peer. (There is no UPDATE message in the control channel)


### Function Channel

Function channels can only be created after the control channel has reached Established state. This means the FSM for a function channel starts from the Connect state. 

### Connect State
After the control channel reaches established state, the BoQ speaker sends an OPEN message to its peer and changes its state for this function channel to OpenSent.

### OpenSent
A BoQ waits for an OPEN_ACK message from its peer in the control channel with destination stream ID matches its own stream ID. When an OPEN_ACK message is received, it's checked for correctness. In case of error, the local BoQ speaker sends a NOTFICATION message, closes the channel/QUIC stream, and changes its state to Connect
If there is no error in the OPEN_ACK message, the BoQ speaker sends a KEEPALIVE message and sets a KeepAlive timer, and moves it's state to OpenConfirm.


### OpenConfirm
Waiting for a KEEPALIVE. When it is received in the control channel, the BoQ speaker's state moved Established.
If the HoldTimer expires, the BoQ speaker sends a NOTIFICATION with error code Hold Timer Expired and terminates the channel.
If a NOTIFICATION sent to the control channel with the matching destination stream ID is received, it closes the channel.

### Established
When the control channel reaches established state, it sends UPDATE, NOTIFCATION and KEEPALIVE messages to its peer.
