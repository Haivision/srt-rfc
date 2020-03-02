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
    organization: "Haivision Network Video, GmbH"
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

This document specifies Secure Reliable Transport (SRT) protocol. 
SRT is a user-level protocol over User Datagram Protocol and provides 
reliability and security optimized for low latency live video streaming, 
as well as generic bulk data transfer. For this, SRT introduces control
packet extension, improved flow control, enhanced congestion control
and a mechanism for data encryption.

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
data packet and control packet.
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
  Depending on the transmission mode ({{data-transmission-mode}}),
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
  encrypted with even key, and "11b" is used for odd key encryption. Refer to {{encryption}}.

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
  by the control packet type definition. See {{srt-ctrl-pkt-type-table}}.

Subtype (16 bits):
: This field specifies additional subtype of specific packets.
  See {{srt-ctrl-pkt-type-table}}.

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
  described om {{handshake-encr-fld}}.

 | Value | Cipher family and block size |
 | ----- | :--------------------------: |
 |     0 | No Encryption                |
 |     2 | AES-128                      |
 |     3 | AES-192                      |
 |     4 | AES-256                      |
{: #handshake-encr-fld title="Handshake Encryption Field Values"}

Extension Field (16 bits):
: This field is message specific extension related to Handshake Type field.
  The value must be set to 0 except for the following cases.
  If the handshake control packet is the INDUCTION message, this field is 
  sent back by the Listener. In case of the CONCLUSION message, this field value 
  should contain a combination of Extension Type value. For more details, see
  {{caller-listener-handshake}}.

Initial Packet Sequence Number (32 bits):
: The sequence number for the very first data packet to be sent.

Maximum Transmission Unit Size (32 bits):
: The value is typically 1500 to follow the default size of MTU (Maximum Transmission Unit)
  in Ethernet, but can be less.

Maximum Flow Window Size (32 bits):
: The value of this field is the maximum number of data packets allowed
  to be "in flight" which means the number of sent packets
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
: The field holds the source SRT socket ID from which the handshake packet is issued.

SYN Cookie (32 bits):
: Randomized value for processing handshake. The value of this field is specified
  by handshake message type. See {{handshake-messages}}.

Peer IP Address (128 bits):
: The sender's IPv4 or IPv6 IP address. The value consists of four 32-bit fields.

Extension Type (16 bits):
: The value of this field is used to process integrated handshake.
  There are two basic extensions: Handshake Extension Message ({{handshake-extension-msg}})
  and Key Material Exchange ({{key-material-exchange}}).
  Each extension can have a pair of request and response types.

Extension Length (16 bits):
: The length of Extension Contents.

Extension Contents (variable length):
: The payload of the extension.

#### Handshake Extension Message {#handshake-extension-msg}

In Handshake Extension, the value of the Extension Field of the
handshake control packet is defined as 1 for Handshake Extension Request,
and 2 for Handshake Extension Response.

The Extension Contents of the Extension Message is the following.

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
{: #handshake-extension-msg-structure title="Handshake Extension Message structure"}

SRT Version (32 bits):
: SRT library version.

SRT Flags (32 bits):
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
{: #keymaterial-extension-structure title="Key Material Extension structure"}

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
{: #unwrapped-key-structure title="Unwrapped key structure"}


### Keep Alive {#ctrl-pkt-keepalive}

Keep-alive control packets are sent after a certain timeout from the last time
any packet (Control or Data) was sent. The purpose of this control packet
is to notify the peer to keep the connection live in case when
no data exchange is taking place.

The default timeout for keepalive packet to be sent is 1 second.

Control Type: 
: The type value of keep-alive control packet is "1".

Type-specific Information: 
: This field is reserved for future definition.

Control Information Field:
: This field must not appear in keep-alive control packets.


### ACK (Acknowledgement) {#ctrl-pkt-ack}

Acknowledgement control packets are used to provide delivery status of data packets.
These packets may also carry some additional information from the receiver like
RTT, bandwidth, receiving speed, etc. The CIF of the ACK control packet is expanded
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

Type-specific Information (32 bits):
: The time-specific Information ({{controlpacket}}) of the ACK packet stores the
  sequential number of the full ACK packet starting from 1.

Last Acknowledged Packet Sequence Number (32 bits):
: The sequence number of the last acknowledged data packet +1.

RTT (32 bits):
: RTT value estimated by the receiver based on the ACK-ACKACK packets exchange.

RTT variance (32 bits):
: The variance of the RTT estimation.

Available Buffer Size (32 bits):
: Available size of the receiver's buffer in packets.

Packets Receiving Rate (32 bits):
: The receiving rate of the packets in packets / second.

Estimated Link Capacity (32 bits):
: Estimated bandwidth of the link in packets per second.

Receiving Rate (32 bits):
: Estimated receiving rate in bytes per second.

There are several types of ACK packets.
The full ACK control packet is sent every 10 ms and has all the fields of {{ack-control-packet}}.
The Lite ACK control packet includes only Last Acknowledged Packet Sequence Number field, and
the Type-specific Information field should be set to 0.
The Small ACK includes the fields up to and including the Available Buffer Size field.
The Type-specific Information field should be set to 0.

The sender acknowledges (see ACKACK) only the receival of Full ACK packets.

The Lite ACK and Small packets are used in cases when the receiver should acknowledge
received data packets more often than every 10 ms. This is usually needed at high data rates.
It is up to the receiver to decide the condition and the type of ACK packet to send (Lite or Small).
The recommendation is to send Lite ACK on every 64 received packets.

### NAK (Loss Report) {#ctrl-pkt-nak}

Negative acknowledgement control packets are used to signal failed
data packet deliveries. The receiver notifies the sender about lost data packets sending the NAK packets.
The NAK packet contains a list of sequence numbers of lost packets.

Control Type: 
: The type value of NAK control packet is "3".

Type-specific Information: 
: This field is reserved for future definition.

Control Information Field:
: A single value or a list of lost packets sequence numbers. See packet sequence number coding in
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

ACKACK control packets are used to acknowledge the reception of the Full ACK.
Furthermore, these packets are used in the calculation of RTT by the receiver.

Control Type: 
: The type value of ACKACK control packet is "6".

Type-specific Information: 
: ACK Sequence Number. This field is used for the sequence number of
  the ACK packet being acknowledged.

Control Information Field:
: This field must not appear in ACKACK control packets.

# SRT Data Transmission and Control

After handshakes and exchanges of capability information, packet data 
can be sent and received over the established connection. To fully utilize 
the features of low latency and error recovery provided by SRT, the sender 
and receiver MUST handle control packets, timers and buffers for the connection
as specified in this section.

## Stream Multiplexing


## Data Transmission Modes {#data-transmission-mode}

In file transfer mode this a message with O=0 that is sent later 
(but reassembled before an earlier message which may be incomplete due to packet loss)
is allowed to be delivered immediately, without waiting for the earlier message to be completed.
In Live Transmission Mode the only valid value is "1".

### Message Mode

### Live mode

### Buffer mode

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

## Too-Late-Packet-Drop {#tl-pkt-drop}

Too-Late-Packet Drop allows the sender to drop packets that have no chance to be delivered in time.
In the SRT sender, when Too-Late Packet Drop is enabled, and a packet timestamp
is older than 125% of the SRT latency, it is considered too late to be delivered and may be dropped
by the sender. Packets of an IFrame tail can then be dropped before being delivered.
In the receiver, tail packets of a big I-Frame may be quite late and not held by the SRT receive buffer.
They pass through to the application. The receiver buffer depletes and there is no time left
for retransmission if missing packets are discovered. Missing packets are then skipped by the receiver.

## Acknowledgement and Lost Packet Handling

To enable the Automatic Repeat Request of data packet retransmissions, the sender stores
all sent data packets in its buffer.
The data receiver sends acknowledgement (ACK) for the received data packets so that the sender can remove
acknowledged packets from its buffer. After that retransmission of these acknowledged packets
are no longer possible and presumably not needed.

The sender should acknowledge the reception of the Full ACK control packet ({{ctrl-pkt-ack}})
by sending the ACKACK control packet ({{ctrl-pkt-ackack}}) with the sequence number
of the Full ACK packet being acknowledged.

The receiver sends NAK control packets to notify the sender about the missing packets.
The NAK packet sending can be triggered immediately after a gap in sequence numbers of
DATA packets is detected.
In addition to that, the Periodic NAK report mechanism can be used to send NAK reports periodically.
The NAK packet in that case will have all the packets that the receiver considers being lost
at the time of the Periodic NAK report.

Upon reception of the NAK packet, the sender prioritizes retransmissions of lost packets over the regular DATA
packets to be transmitted for the first time.

The retransmission of the missing packet is repeated until the receiver acknowledges its receival,
or if both peers agree to drop this packet (see {{tl-pkt-drop}}).

### Packet Acknowledgement (ACKs) {#packet-acks}

At certain intervals (see ACKs, ACKACKs & Round Trip Time), the receiver sends an ACK that
causes the acknowledged packets to be removed from the sender's buffer.
An ACK contains the sequence number of the packet immediately
following the latest of the previous packets that have been received. Where no packet loss has
occurred up to the packet with sequence number n, ACK would include the sequence number n + 1.
The ACK needs to acknowledged by ACKACK (see ACKACK), and if not the ACK will be retransmitted.
If the sender doesn't receive an ACK, it doesn't stop transmitting. There are two conditions
for sending an acknowledgement. A full ACK is based on a timer of 10 milliseconds (the ACK period).
For high bit rate transmissions, a “light ACK” can be sent, which is an ACK for
a sequence of packets. In a 10 milliseconds interval, there are often so many packets being sent
and received that the ACK position on the sender doesn't advance quickly enough.
To mitigate this, after 64 packets (even if the ACK period has not fully elapsed) the receiver
sends a light ACK. When a receiver encounters the situation where the next packet to be played was
not successfully received from the sender, it will “skip” this packet and send a fake ACK. To the
sender, this fake ACK is a real ACK, and so it just behaves as if the packet had been received.

This facilitates the synchronization between sender and receiver. The fact that a packet was
skipped remains unknown by the sender. Skipped packets are recorded in the statistics on the
receiver.

### Packet Retransmission (NAKs) {#packet-naks}

When a packet is received but the previous packets are not yet arrived in a receiver buffer,
if a certain amount of time is passed, NAKs for previous packets are sent to the sender.
If periodic NAK report is enabled in live mode, the lost packets list is sent periodically. 
The period is 4 * RTT + RTTVar + SYN, but this could be reduced (e.g. by half)
when a certain condition is met.  

The sender maintains a list of lost packets (loss list) that is built from NAK reports. When
scheduling to transmit, it looks to see if a packet in the loss list has priority, and will send it.
Otherwise, it will send the next packet in the sender buffer. Note that when a packet is transmitted,
it stays in the buffer in case it is not received.
NAK packets are processed to fill the loss list. As the latency window advances and packets are
dropped from the sender buffer, a check is performed to see if any of the dropped or resent
packets are in the loss list, to determine if they can be removed from there as well so that they
are not retransmitted unnecessarily.

What the sender sees is the NAKs that it has received. There is a counter for the packets that
are resent. If there is no ACK for a packet, it will stay in the loss list and can be resent more than
once. Packets in the loss list are prioritized.
If packets in the loss list continue to block the send queue, at some point this will cause the
send queue to fill. When the send queue is full, the sender will begin to drop packets without
even sending them the first time. An encoder (or other application) may continue to provide
packets, but there's no place for them, so they will end up being thrown away.

This condition where packets are unsent doesn't happen often. There is a maximum number of
packets held in the send buffer based on the configured latency. Older packets that have no
chance to be retransmitted and played in time are dropped, making room for newer real-time
packets produced by the sending application. A minimum of one second is applied before
dropping the packet when low latency is configured. This one-second limit derives from the
behavior of MPEG I-frames with SRT used as transport. I-frames are very large (typically 8 times
larger than other packets), and consequently take more time to transmit. They can be too large
to keep in the latency window, and can cause packets to be dropped from the queue. To
prevent this, SRT imposes a minimum of one second (or the latency value) before dropping a
packet. This allows for large I-frames when using small latency values.

### Packet Acknowledgment in SRT

The ACKACK tells the receiver to stop sending the ACK position because the sender already
knows it. Otherwise, ACKs (with outdated information) would continue to be sent regularly.
An ACK serves as a ping, with a corresponding ACKACK pong, to measure RTT.
The time it takes for an ACK to be sent and an ACKACK to be received is the RTT.
Each ACK has a number. A corresponding ACKACK has that same number.
The receiver keeps a list of all ACKs in a queue to match them. Unlike a full ACK,
which contains the current RTT and several other values in the CIF,
a light ACK just contains the sequence number. All control messages are sent directly and
processed upon reception, but ACKACK processing time is negligible (the time this takes
is included in the round-trip time).

### Bidirectional Transmission Queues

### ACKs, ACKACKs & Round Trip Time

### Loss List


# Encryption {#encryption}

TODO: Priority 3.

## Overview

## Definitions

## Encryption Process Walkthrough

## Messages

## Parameters

## Security Issues

## Implementation Notes


# Security Considerations

SRT support confidentiality of user data using stream ciphering based on AES,
as specified in {{encryption}}. Session key for ciphering is delivered control
packet during handshake, with the protection by Key encryption key,
which is generated by a sender and receiver with pre-shared secret such as passphrase.
Appropriate security policy including key size, key refresh period,
as well as passphrase should be managed by security officers,
which is out of scope of the present document.
Other security considerations related to ciphering scheme and key exchanges
are described in {{encryption}}.

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


