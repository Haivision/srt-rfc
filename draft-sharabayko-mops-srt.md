---
title: The SRT Protocol
abbrev: SRT
docname: draft-sharabayko-mops-srt
category: std

ipr: trust200902
area: opsarea
workgroup: mops
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: "M.P. Sharabayko"
    name: "Maxim Sharabayko"
    organization: "Haivision"
    email: maxsharabayko@haivision.com
 -
    ins: "J. Kim"
    name: "Jeongseok Kim"
    organization: "SK Telecom Co., Ltd."
    email: jeongseok.kim@sk.com

normative:
  RFC2119:
  RFC0768:

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

# Packet Structure

SRT packets are transmitted in UDP packets {{RFC0768}}. Every UDP packet carrying SRT 
traffic contains an SRT header (immediately after the UDP header).

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            SrcPort            |            DstPort            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|              Len              |            ChkSum             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                          SRT Packet                           +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #srt-in-udp title="SRT packet as UDP payload"}

SRT has two types of packets distinguished by the Packet Type Flag:
data packet and conrol packet.
The structure of the SRT packet is shown in {{srtpacket}}.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|F|        (Field meaning depends on the packet type)           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          (Field meaning depends on the packet type)           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Destination Socket ID                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+    (Packet contents: depends on the packet type)              +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #srtpacket title="SRT packet structure"}

F (1 bit): 
: Packet Type Flag. The control packet has this flag set to "1".
  The data packet has this flag set to "0".

Timestamp (32 bits):
: The time stamp of the packet in microseconds.
  The value is relative to the time the SRT connection was established.
  Depending on the transmission mode {{data-transmission-mode}},
  the field stores the packet send time or the packet origin time.

Destination Socket ID (32 bits):
: A fixed-width field providing the SRT socket ID to which a packet should be dispatched.
  The field may have the special value "0" when the packet is a connection request.


## Data Packets

The structure of the SRT data packet is shown in {{srtdatapacket}}.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|                    Packet Sequence Number                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|P P|O|K K|R|                   Message Number                  |
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
{: #srtdatapacket title="data packet structure"}

Packet Sequence Number (31 bits):
: The sequential number of the data packet.

PP (2 bits):
: Packet Position Flags. This field indicates the position of the data packet in the message.
  The value "10b" means the first packet of the message. "00b" indicates a packet in the middle,
  "01b" is the last packet. If a single data packet forms the whole message,
  the value is "11b".

O (1 bit):
: Order Flag. Indicates whether the message should be delivered by the receiver
  in order (1) or not (0). Certain restrictions apply depending on the data transmission mode used ({{data-transmission-mode}}).

KK (2 bits):
: Encryption Flag. The flag bits indicate whether or not data is encrypted.
  The value "00b" means data is not encrypted, "01b" indicates that data is
  encrypted with even key, and "11b" is used for odd key encryption.

R (1 bit):
: Retransmitted Packet Flag. This flag is clear when a packet is transmitted the very first time.
  The flag set to "1" means the packet is retransmitted.

Message Number (26 bits):
: The sequential number of the message formed by consecutive data packets (see PP field).

Data (variable length):
: The payload of the data packet. The length of the data is the remaining length of the UDP packet.


## Control Packets

SRT control packet has the following structure.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1|         Control Type        |            Subtype            |
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

Control Type (15 bits):
: Control Packet Type. The use of these bits is determined
  by the control packet type definition. See {{srt-ctrl-pkt-type-table}}

Subtype (16 bits):
: This field specifies additional subtype of specific packets.
  See {{srt-ctrl-pkt-type-table}}

Type-specific Information (32 bits):
: The use of this field is defined by the particular control
  packet type. Handshake packets don't use this field.

Control Information Field (variable length):
: The use of this field is defined by the Control Type field of the control packet.


The types of SRT control packets are shown in {{srt-ctrl-pkt-type-table}}.
The value, "0x7ffff", is reserved for user-defined type.

| ----------------- | ------------ | ------- | -------------------------- |
| Packet Type       | Control Type | Subtype | Section                    |
| ----------------- | :----------: | :-----: | -------------------------- |
| HANDSHAKE         |  0x0000      |   0x0   | {{ctrl-pkt-handshake}}     |
| KEEPALIVE         |  0x0001      |   0x0   | {{ctrl-pkt-keepalive}}     |
| ACK               |  0x0002      |   0x0   | {{ctrl-pkt-ack}}           |
| NAK (Loss Report) |  0x0003      |   0x0   | {{ctrl-pkt-nak}}           |
| SHUTDOWN          |  0x0005      |   0x0   | {{ctrl-pkt-shutdown}}      |
| ACKACK            |  0x0006      |   0x0   | {{ctrl-pkt-ackack}}        |
| User-Defined Type |  0x7FFF      |    -    | N/A                        |
| ----------------- | ------------ | ------- | -------------------------- |
{: #srt-ctrl-pkt-type-table title="SRT Control Packet Types"}


### Handshake {#ctrl-pkt-handshake}

The handshake control packets are used to exchange peer configurations,
agree on the and connection parameters and establish the connection.

The CIF of handshake control packet is shown in {{handshake-packet-structure}}.

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
|                Maximum Transmission Unit Size                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Maximum Flow Window Size                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Handshake Type                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         SRT Socket ID                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           SYN Cookie                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                        Peer IP Address                        +
|                                                               |
+                                                               +
|                                                               |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|         Extension Type        |        Extension Length       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                       Extension Contents                      +
|                                                               |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
~~~
{: #handshake-packet-structure title="handshake packet structure"}

Version (32 bits):
: A base protocol version number. Currently used values are 4 and 5.
  The value greater than 5 is reserved for future definition.

Encryption Field (16 bits):
: Block cipher family and block size. The values of this field are
  described as the following.

 | Value | Cipher family and block size |
 | ----- | :--------------------------: |
 |     0 | No Encryption                |
 |     2 | AES-128                      |
 |     3 | AES-192                      |
 |     4 | AES-256                      |
{: #handshake-encr-fld title="Handshake Encryption Field Values"}

Extension Field (16 bits):
: This field is message specific extension related to Handshake Type field.
  The value must be set as 0 except for the following cases.
  If the handshake control packet is a INDUCTION message, this field is 
  sent back by the Listener. In case of CONCLUSION message, this field value 
  should contain a combination of Extension Type value. For more details, see in
  {{caller-listener-handshake}}.

Initial Packet Sequence Number (32 bits):
: The sequence number for the first data packet to be sent.

Maximum Transmission Unit Size (32 bits):
: The value is typically 1500 to follow default size of MTU (Maximum Transmission Unit)
  in Ethernet, but can be less.

Maximum Flow Window Size (32 bits):
: The value of this field is the maximum number of data packets allowed
  to be "in flight" which means that the number of sent packets 
  without receiving ACK control packet.

Handshake Type (32 bits):
: This field indicates the handshake packet types.
  The possible values are described in {{handshake-type}}.
  For more details refer to {{handshake-messages}}.

 | Value      | Handshake type |
 | ---------- | :--------------------------: |
 | 0xFFFFFFFD | DONE                         |
 | 0xFFFFFFFE | AGREEMENT                    |
 | 0xFFFFFFFF | CONCLUSION                   |
 | 0x00000000 | WAVEHAND                     |
 | 0x00000001 | INDUCTION                    |
{: #handshake-type title="Handshake Type"}

SRT Socket ID (32 bits):
: The value of this field has the source SRT socket id from which the handshake packet is issued.

SYN Cookie (32 bits):
: Randomized value for processing handshake. The value of this field is specified
  by handshake message type. See in {{handshake-messages}}.

Peer IP Address (128 bits):
: The sender's IPv4 or IPv6 IP address. The value consists of four 32-bit fields.

Extension Type (16 bits):
: The value of this field is used for processing integrated handshake.
  There are two basic extension; Handshake Extension Message ({{handshake-extension-msg}})
  and Key Material Exchange ({{key-material-exchange}}).
  Each extension can have a pair of request and response type.  

Extension Length (16 bits):
: The length of Extension Contents.

Extension Contents (variable length):
: The payload of the extension.


#### Handshake Extension Message {#handshake-extension-msg}

In Handshake Extension, the value of the Extension Field of 
handshake control packet is defined as 1 for Handshake Extension Request, 
or 2 for Handshake Extension Response.

The HS Extension Message Contents is revealed as the following.

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
{: #handshake-extension-msg-structure title="Handshake extension message structure"}

SRT Version (32 bits):
: SRT library version.

SRT Falgs (32 bits):
: SRT configuration flags.

 | Bitfield   | Flag |
 | ---------- | :---------------: |
 | 0xFFFFFFFD | TSBPDSND          |
 | 0xFFFFFFFE | TSBPDRCV          |
 | 0xFFFFFFFF | CRYPT             |
 | 0x00000000 | TLPKTDROP         |
 | 0x00000001 | PERIODICNAK       |
 | 0x00000001 | REXMITFLG         |
 | 0x00000001 | STREAM            |
{: #hs-ext-msg-flags title="HS Extension Message Flags"}

Receiver TsbPd Delay (16 bits):
: TSBPD delay of the receiver. Refer to {{tsbpd}}.

Sender TsbPd Delay (16 bits):
: TSBPD delay of the sender. Refer to {{tsbpd}}.


#### Key Material Exchange {#key-material-exchange}

The Key Material Exchange has both request and response type extensions.
The value of request is 3, and the response value is 4.

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




### Keep Alive {#ctrl-pkt-keepalive}

Keep-alive control packets are exchaged approximately every 10ms to 
enable SRT streams to be automatically restored after a connection loss.

Control Type: 
: The type value of keep-alive control packet is "1".

Type-specific Information: 
: This field is reserved for future definition.

Control Information Field:
: This field must not appear in keep-alive control packets.


### ACK (Acknowledgement) {#ctrl-pkt-ack}

Acknowledgement control packets are used to provide delivery status of data packets.
These packets may also carry some additional information from the receiver like
RTT, bandwidth, receiving speed, etc. The CIF of ACK control packet is expanded
as the following.

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
{: #ack-control-packet title="ACK control packet"}

Last Acknowledged Packet Sequence Number (32 bits):
: The sequence number of the last acknowledged data packet +1.

RTT (32 bits):
: RTT value estimated by the receiver based on the ACK-ACKACK packets exchange.

RTT variance (32 bits):
: The variance of the RTT estimation.

Available Buffer Size (32 bits):
: Available size of the receiver's buffer.

Packets Receiving Rate (32 bits):
: The receiving rate of the packets in packets / second.

Estimated Link Capacity (32 bits):
: Estimated bandwidth of the link.

Receiving Rate (32 bits):
: Estimated receiving rate.

### NAK (Loss Report) {#ctrl-pkt-nak}

Negative acknowledgement control packets are used to signal failed
data packet deliveries. The receiver notifies the sender about the lost packets using the NAK packets.
The NAK packet contains a list of sequence numbers of lost packets.

Control Type: 
: The type value of NAK control packet is "3".

Type-specific Information: 
: This field is reserved for future definition.

Control Information Field:
: One or list of lost packet sequence number. See packet sequence number coding in
  {{packet-seq-list-coding}}.

### Shutdown {#ctrl-pkt-shutdown}

Shutdown control packets are used to initiate the closing of an SRT connection.

Control Type: 
: The type value of Shutdown control packet is "5".

Type-specific Information: 
: This field is reserved for future definition.

Control Information Field:
: This field must not appear in shutdown control packets.

### ACKACK {#ctrl-pkt-ackack}

ACKACK control packets are used to acknowledge the reception of an ACK.
Furthermore, these packets are used in the calculation of RTT by the receiver.

Control Type: 
: The type value of ACKACK control packet is "6".

Type-specific Information: 
: ACK Sequence Number. This field is used for the sequence number of
  the ACK packet acknowledged.

Control Information Field:
: This field must not appear in ACKACK control packets.

# SRT Data Transmission and Control

After handshakes and exchanges of capability information, packet data 
can be sent and received over the established connection. To fully utilize 
the features of low latency and error recovery provided by SRT, the sender 
and receiver MUST handle control packets, timers and buffers for the connection
as specified in this section.

## Data Transmission Mode {#data-transmission-mode}

In file transfer mode this a message with O=0 that is sent later 
(but reassembled before an earlier message which may be incomplete due to packet loss)
is allowed to be delivered immediately, without waiting for the earlier message to be completed.
In Live Transmission Mode the only valid value is "1".

### Live mode

### File mode

## Handshake Messages {#handshake-messages}

### Caller-Listener Handshake {#caller-listener-handshake}

### Rendezvous Handshake

## SRT Buffer Latency

The sender and receiver have buffers to store packets.
On the sender, latency is the time that SRT holds a packet to give it a chance to be
delivered successfully while maintaining the rate of the sender at the receiver.
If an ACK is missing or late for the configured latency, the packet is dropped
from the sender buffer. The packet can be retransmitted, while the packet exists
in the buffer for the latency window. On the receiver, a packet delivered to an 
application from a buffer after latency time passed, to recover from a potential
packet loss. 

Latency is a value specified in milliseconds, which can cover hundreds or even thousands
of packets at high bitrate. Latency can be thought of as a window that slides over time,
during which a number of activities take place, such as report of ACKs({{packet-acks}})
or NAKs({{packet-naks}}).
Latency is configured through capability exchange during extended handshake process 
between initiator and responder. The handshake extension({{handshake-extension-msg}}) has 
receiver and sender TsbPd Delay information in milliseconds. The maximum value of latencies
from initiator and responder will be established. 

## Timestamp Based Packet Delivery {#tsbpd}

This feature uses the timestamp of the SRT data packet header.
TsbPD allows a receiver to deliver packets to the decoder at the same pace they were
provided to the SRT sender by an encoder. Basically, the sender timestamp
in the received packet is adjusted to the receiver’s local time
(compensating for time drift or different time zone)
before releasing the packet to the application.
Packets can be withheld by SRT for a configured receiver delay in milliseconds.
Higher delay can accommodate data traffic which could lead to a larger uniform packet drop
rate or larger packet burst drop.
Packets received after their “play time” are dropped.
The packet timestamp (in microseconds) is relative to the SRT connection creation time.
The origin time (in microseconds) of the packet is already sampled when a packet is first
submitted by the application to the SRT sender.
The TsbPD feature uses this time to stamp the packet for first transmission
and any subsequent re-transmission. This timestamp and the configured latency
control the recovery buffer size and the instant (aforementioned "play time"
which is decided by adding timestamp to configured latency) that packets 
are delivered at the destination.
Latency is agreed during handshake process as maximum value of Receiver/Sender TsbPd Delay
from initiator/responder (see section #ctrl-pkt-handshake).  

## Too-Late-Packet-Drop

Too-Late-Packet Drop allows the sender to drop packets that have no chance to be delivered in time.
In the SRT sender, when Too-Late Packet Drop is enabled, and a packet timestamp
is older than 125% of the SRT latency, it is considered too late to be delivered and may be dropped
by the sender. Packets of an IFrame tail can then be dropped before being delivered.
In the receiver, tail packets of a big I-Frame may be quite late and not held by the SRT receive buffer.
They pass through to the application. The receiver buffer depletes and there is no time left
for retransmission if missing packets are discovered. Missing packets are then skipped by the receiver.

## Packet Acknowledgement (ACKs) {#packet-acks}



## Packet Retransmission (NAKs) {#packet-naks}

## Packet Acknowledgment in SRT

## Bidirectional Transmission Queues

## ACKs, ACKACKs & Round Trip Time

## Loss List


# Encryption

TODO: Priority 3.

## Overview

## Definitions

## Encryption Process Walkthrough

## Messages

## Parameters

## Security Issues

## Implementation Notes


# Security Considerations

TODO Security

# IANA Considerations

TODO IANA

# Acknowledgments
{:numbered="false"}

TODO acknowledge.

--- back

# Packet Sequence List coding {#packet-seq-list-coding}

For any single packet sequence number,
it uses the original sequence number in the field. The first bit 
must start with "0".

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|                   Sequence Number                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #single-sequence-number title="single sequence numbers coding"}

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


