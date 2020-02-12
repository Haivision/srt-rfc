---
title: The SRT Protocol
abbrev: SRT
docname: draft-ietf-sharabayko-srt-latest
category: std

ipr: trust200902
area: opsarea
workgroup: mops
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: "M. Sharabayko"
    name: "Maxim Sharabayko"
    organization: "Haivision"
    email: maxlovic@gmail.com
 -
    ins: "J. Kim"
    name: "Jeongseok Kim"
    organization: "SK Telecom Co., Ltd."
    email: jeongseok.kim@sk.com

normative:
  RFC2119:

informative:
  RFC8174:


--- abstract

TODO Abstract

--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Data Packets

The data packet structure is as following.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|F|                    Packet Sequence Number                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| PP|O|Enc|R|                   Message Number                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Destination Socket ID                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                              Data                             +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #datapacket title="data packet structure"}

F (1 bit): 
: Packet Flag. The flag value of data packet must be 0.

Packet Sequence Number (31 bits):
: 

PP (2 bits):
: Position of Packet. This field indicates the position of packet.
  The value "10b" means the first packet, "00b" is packets in the middle, "01b" is the last packet.
  If it is a single data packet, the value is "11b".

O (1 bit):
: Order.  Indicates whether the message should be delivered in order or not.
  This value is not used in Live Transmission Mode({{live-transmission-mode}}).

Enc (2 bits):
: Encryption Flag. The flag bits indicate whether or not data is encrypted.
  The value "00b" is not encrypted, "01b" means ecnrypted data with even key, 
 "11b" signifies ecrypted data with odd key.

R (1 bit):
: Retransmitted Packet. This flag bit is clear when a packet is transmitted very first time.
  If a packet is retransmitted, the flag is set as "1".

Message Number (26 bits):
: 

Timestamp (32 bits):
: The time stamp when the packet is sent. 
  This is relative time value starting from when the connection is established.

Destination Socket ID (32 bits):
: 

Data (variable length):
:

# Control Packets

If the flag bit of SRT packet is set as "1", it should have a following structure as a control packet.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|F|         Message Type        |            Reserved           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Type-specific Information                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Destination Socket ID                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                   Control Information Field                   +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #controlpacket title="control packet structure"}

F (1 bit): 
: Packet Flag. The control packet must set this flag as "1".

Message Type (15 bits):
: Control Message Type. The use of these bits is determined
  by the control message type definition.

  - "0": Handshake
  - "1": Keep Alive
  - "2": ACK
  - "3": NACK (Loss Report)
  - "4": Congestion Warning
  - "5": Shutdown
  - "6": ACKACK
  - "7": Drop Request
  - "8": Peer Error

Reserved (16 bits):
: This field is reserved for future definition.

Type-specific Information (32 bits):
: The use of this field is defined by the particular control
  message type. Handshake messages don't use this field.

Timestamp (32 bits):
: The time stamp when the packet is sent. 
  This is relative time value starting from when the connection is established.

Destination Socket ID (32 bits):
: 

Control Information Field (variable length):
: The use of this field is defined by Message Type.

## Handshake

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            Version                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Encryption Field       |        Extension Field        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Initial Packet Sequence Number                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Maximum Packet Size                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Maximum Flow Window Size                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Connection Type                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Socket ID                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           SYN Cookie                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Peer IP Address                        |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|         Extension Type        |        Extension Length       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                       Extension Contents                      +
|                                                               |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
~~~
{: #handshake-packet-structure title="handshake packet structure"}

### Handshake Extension

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          SRT Version                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           SRT Flags                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Receiver TsbPd Delay     |       Sender TsbPd Delay      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #handshake-extension-structure title="handshake extension structure"}


### Key Material Exchange

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|S|  V  |   PT  |              Sign             |    Rev    | KK|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              KEKI                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Cipher    |      Auth     |       SE      |  SLen |  KLen |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              Salt                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                          Wrapped Key                          +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~

{: #keymaterial-extension-structure title="key material extension structure"}

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                              ICV                              +
|                                                               |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|                              xSEK                             |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|                              oSEK                             |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
~~~
{: #unwrapped-key-structure title="unwrapped key structure"}

## Keep Alive

Keep-alive control packets are exchaged approximately every 10ms to 
enable SRT streams to be automattically restored after a connection loss.

Message Type: 
: The type value of keep-alive control packet is "1".

Type-specific Information: 
: This field is reserved for future definition.

Control Information Field:
: This field must not appear in keep-alive control packets.

## ACK (Acknowledgement)

Acknowledgement control packets are used to provide delivery status of data packets and RTT information.

Message Type: 
: The type value of ACK control packet is "2".

Type-specific Information: 
: This field is used for ACK sequence number.

The Control Information Field of ACK control packet is extended as following.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Last Acknowledged Packet Sequence Number           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              RTT                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          RTT variance                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Available Buffer Size                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Packets Receiving Rate                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Estimated Link Capacity                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Receiving Rate                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #ack-control-information title="control information field of ACK control packet"}

## NAK (Loss Report)

Negative acknowledgement control packets are used to signal failed
data packet deliveries.

Message Type: 
: The type value of NAK control packet is "3".

Type-specific Information: 
: This field is reserved for future definition.

The Control Information Field of NAK control packet is a lost packet number,
or list of lost packet sequence numbers. See packet sequence number coding in
{{packet-seq-list-coding}}.

## Congestion Warning

## Shutdown

Shutdown control packets are used to initiate the closing of an SRT connection.

Message Type: 
: The type value of Shutdown control packet is "5".

Type-specific Information: 
: This field is reserved for future definition.

Control Information Field:
: This field must not appear in keep-alive control packets.

## ACKACK

ACKACK control packets are used to acknowledge the reception of an ACK, 
and are instrumental in the ongoing caculation of RTT.

Message Type: 
: The type value of ACKACK control packet is "6".

Type-specific Information: 
: This field is used for ACK sequence number.

Control Information Field:
: This field must not appear in keep-alive control packets.

## Drop Request

## Peer Error

# Bandwidth Estimation

# Control Events

# Timers

# Congestion Control

## Live Transmission mode {#live-transmission-mode}

## File Transmission mode

# Security Considerations

TODO Security


# IANA Considerations

TODO IANA


# Acknowledgments
{:numbered="false"}

TODO acknowledge.

--- back

# Packet Sequence List coding {#packet-seq-list-coding}

For any single packet sequence number or two consecutive sequence numbers, 
it uses the original sequence number in the field. The first bit 
must start with "0".

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|                   Sequence Number                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #single-sequence-number title="single or two consecutive sequence numbers coding"}

For any consectutive packet seqeunce numbers that the differnece between
the last and first is more than 1, only record the first (a) and the
the last (b) sequence numbers in the list field, and modify the
the first bit of a to "1".

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1|                   Sequence Number a (first)                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|                   Sequence Number b (last)                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #list-sequence-numbers title="list of sequence numbers coding"}


