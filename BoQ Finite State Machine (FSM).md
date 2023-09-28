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
| AllowAutomaticStart                  |     Y           |       Y          |
| AllowAutomaticStop                   |     Y           |       Y          |
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
