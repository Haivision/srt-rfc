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

SRT packets are transmitted in UDP packets.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ ---
|            SrcPort            |            DstPort            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ UDP Header
|              Len              |            ChkSum             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ ---
|                                                               |
+                          SRT Packet                           +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

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
: Packet Type Flag. The control packet must set this flag as "1".
  The data packet must set this flag as "0".

Timestamp (32 bits):
: The time stamp of the packet in microseconds.
  The value is relative to the time the SRT connection was established.
  This is relative time value starting from when the connection is established.
  Depending on the transmission mode, the field stores the packet sent time or
  the packet origin time.

Destination Socket ID (32 bits):
: A fixed-width field providing the socket ID to which a packet should be dispatched.
  The field may have the special value "0" when the packet is a connection request.


## Data Packets

The structure of the SRT data packet is the following.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|                    Packet Sequence Number                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| PP|O|KK |R|                   Message Number                  |
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
: Packet Position. This field indicates the position of data packet in the message.
  The value "10b" means the first packet of the message, "00b" - a packet in the middle,
  "01b" is the last packet. If a single data packet forms the whole message,
  the value is "11b".

O (1 bit):
: Order Flag. Indicates whether the message should be delivered by the receiver
  in order (1) or not (0).
  Note. In file transfer mode this a message with O=0 that is sent later 
  (but reassembled before an earlier message which may be incomplete due to packet loss)
  is allowed to be delivered immediately, without waiting for the earlier message to be completed.
  In Live Transmission Mode the only valid value is "1".

KK (2 bits):
: Encryption Flag. The flag bits indicate whether or not data is encrypted.
  The value "00b" means data is not encrypted, "01b" - data is encrypted with even key, 
  "11b" - data is encrypted with odd key.

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


The types of SRT control packets are shown as following. The value, "0x7ffff", is
reserved for user-defined type.

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

The CIF of handshake control packet is revealed as the following.

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

#### Handshake Extension

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


#### Key Material Exchange

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
RTT, bandwidth, receiving speed, etc.

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
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+----
|            Last Acknowledged Packet Sequence Number           | Lite ACK
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+----
|                              RTT                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          RTT variance                         | Small Ack
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Available Buffer Size                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+----
|                     Packets Receiving Rate                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Estimated Link Capacity                   | Full ACK
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Receiving Rate                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+----
~~~
{: #ack-control-packet title="ACK control packet"}


Control Type: 
: The type value of ACK control packet is "2".

Subtype: 
: The type value of ACK control packet is "0".

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

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1|         Control Type        |            Subtype            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       ACK Sequence Number                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Destination Socket ID                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
+                             None                              +
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

Control Type: 
: The type value of ACKACK control packet is "6".

ACK Sequence Number: 
: This field stores the sequence number of the ACK packet acknowledged.


# SRT Data Transmission and Control

TODO: Priority 2.

## SRT Buffer Latency

The sender and receiver have buffers to store packets.
On the sender, latency is the time that SRT holds a packet to give it a chance to be
delivered successfully while maintaining the rate of the sender at the receiver.
The effect of latency is minimal on the sender, where it is used in the context of
dropping packets if an ACK is missing or late. It‘s much clearer on the receiver side.

Latency is a value specified in milliseconds, which can cover hundreds or even thousands
of packets at high bitrate. Latency can be thought of as a window that slides over time,
during which a number of activities take place.

## Timestamp Based Packet Delivery {#tsbpd}

<!--
TODO:
* Introduction
* Section with notations plus put TsbPD in here, check how to make the first reference: TSBPD, TLPKTDROP, SRT, round-trip time (RTT)
* Why TsbPD? Why not TSBPD?
* Encoder/Decoder - not 100% correct terminology here, cause it can be gateway, etc. -> sending/receiving application
* It's designed and used in live mode
* SRT connection creation time - there should be some reference
* This timestamp and the configured latency control - ther should be a reference to latency
* There should be earlier a section describing which data transmissions methods we have: live, file, message (also file)
as per https://github.com/Haivision/srt/blob/master/docs/API.md
* Do not forget about my notes from notebook - 3 main ideas behind TSBPD
* Renaming: PktTsbpdTime -> PktDeliveryTime, TsbPd -> Tsbpd in all places, TsbpdTimeBase -> TsbpdBaseTime, check everything
* Variables names: PKT_TIMESTAMP vs PktTimestamp
* Concept of caller, listener, rendevous
* Concept of receiver and sender
* Names for diagrams and tables
-->

The initial intent of the SRT Timestamp Based Packet Delivery (TSBPD) feature is
to reproduce the output of the sending application at the input of the receiving one.

It is designed in a way that SRT receiver, using the timestamp of the SRT data packet header,
delivers packets to a receiving application at the same pace they were provided to the SRT sender by a sending applicaton.
Basically, the sender timestamp in the received packet is adjusted to the receiver’s local time
(compensating for the time drift or different time zone) before releasing the packet to the application.
Packets can be withheld by the SRT receiver for a configured receiver delay.
Higher delay can accommodate a larger uniform packet drop rate or larger packet burst drop.
Packets received after their "play time" are dropped if Too-Late Packet Drop feature is enabled (see {{too-late-packet-drop}}).

<!-- TODO: Insert link to SRT connection creation time, SRT latency -->

The packet timestamp (in microseconds) is relative to the SRT connection creation time.
It is worth mentioning that the use of the packet sending time to stamp the packets is
inappropriate for TSBPD feature since a new time (current sending time) is used for retransmitted packets,
putting them out of order when inserted at their proper place in the stream. Packets are
inserted based on the sequence number in the header field. The origin time (in microseconds)
of the packet is already sampled when a packet is first submitted by the application to the SRT sender.
The TSBPD feature uses this time to stamp the packet for first transmission and any subsequent retransmission.
This timestamp and the configured SRT latency control the recovery buffer size and the instant that packets
are delivered at the destination.

{{fig-latency-points}} illustrates the key latency points during the packet transmission with TSBPD feature enabled.

~~~
                |  Sending  |                   |                         |
                |   Delay   |      ~RTT/2       |       SRT Latency       |
                |<--------->|<----------------->|<----------------------->|
                |           |                   |                         |
                |           |                   |                         |
                |           |                   |                         |
      _____ Ready to       Sent              Received                 Ready to be
     /      be sent         |                   |                     delivered
  Packet        |           |                   |                         |
  State         |           |                   |                         |
                |           |                   |                         |
                |           |                   |                         |
                ----------------------------------------------------------------->
                                                                                Time
~~~
{: #fig-latency-points title="Key Latency Points during the Packet Transmission"}

The main packet states shown at in {{fig-latency-points}} are the following:

- "Ready to be sent": the packet is commited by the sending application, stamped and ready to be sent;
- "Sent": the packet is passed to the UDP socket and sent;
- "Received": the packet is received and read from the UDP socket;
- "Ready to be delivered": the packet is ready to be delivered to the upstream (receiving) application,
  delivered and read by the application (see {{packet-delivery-time}}).

It is worth noting that the round-trip time (RTT) of an SRT link may vary in time. However the actual
end-to-end latency on the link will be approximately equal to (RTT_0/2 + SRT Latency),
where RTT_0 is the initial value of the round-trip time obtained during the SRT handshake
exchange (the value of the round-trip time once the SRT connection has been established).

The value of the sending delay is relatively small and evaluated in microseconds,
in contrast to RTT/2 and SRT latency which are usually measured in milliseconds.

### Packet Delivery Time {#packet-delivery-time}

Packet delivery time is the moment, estimated by the receiver, when a packet should be delivered
to the upstream application. The calculation of packet delivery time (PktTsbpdTime) is performed
upon receiving a data packet according to the following formula:

~~~
PktTsbpdTime = TsbpdTimeBase + PKT_TIMESTAMP + TsbpdDelay + Drift
~~~

where

- TsbpdTimeBase is the time base that reflects the time difference between local clock of the receiver
  and the clock used by the sender to timestamp packets being sent (see {{tsbpd-time-base}});
- PKT_TIMESTAMP is the data packet timestamp, in microseconds;
- TsbpdDelay is the receiver’s buffer delay (or receiver’s buffer latency, or SRT Latency).
  This is the time, in milliseconds, that SRT holds a packet from the moment it has been received till the time it
  should be delivered to the upstream application;
- Drift is the time drift used to adjust the fluctuations between sender and receiver clock, in microseconds.

SRT Latency (TsbpdDelay) should be a buffer time large enough to cover the unexpectedly
extended RTT time, and the time needed to retransmit the lost packet. The value of minimum TsbpdDelay
is negotiated during the SRT handshake exchange and is equal to 120 milliseconds. The recommended
value of TsbpdDelay is 3-4 times RTT.

#### TSBPD Time Base Calculation {#tsbpd-time-base}

The initial value of TSBPD time base (TsbpdTimeBase) is calculated at the moment of
the second handshake request is received as follows:

~~~
TsbpdTimeBase = T_NOW - HSREQ_TIMESTAMP
~~~

where T_NOW is the current time according to the receiver clock;
HSREQ_TIMESTAMP is the handshake packet timestamp, in microseconds.

<!-- TODO: RTT_O - right? -->
The value of TsbpdTimeBase is approximately equal to the initial one-way delay of the link RTT_0/2,
where RTT_0 is the value of the round-trip time once the SRT connection has been established.

During the transmission process, the value of TSBPD time base may be adjusted in two cases:

1. During the TSBPD wrapping period that happens every 01:11:35 hours. This time corresponds
to the maximum timestamp value of a packet (MAX_TIMESTAMP). MAX_TIMESTAMP is equal
to 0xFFFFFFFF, or maximum 32-bit unsigned integer value, in microseconds (TODO: see Link).
The TSBPD wrapping period starts 30 seconds before reaching the maximum timestamp value
of a packet and ends once the packet with timestamp within [30, 60] seconds interval
is received. The updated value of TsbpdTimeBase will be recalculated as follows:

~~~
TsbpdTimeBase = TsbpdTimeBase + MAX_TIMESTAMP + 1
~~~

<!-- TODO: Link to the packet structure to Packet Timestamp field -->
<!-- TODO: Check the sentense with starting and ending period of TSBPD wrapping period -->

2. By drift tracer. See {{drift-management}} for details.


## Drift Management {#drift-management}

<!-- TODO: Resolved questions -->

When the sender enters "connected" status it tells the application
there is a socket interface that is transmitter-ready. At this point
the application can start sending data packets. It adds packets to
the SRT sender's buffer at a certain input rate, from which they are
transmitted to the receiver at scheduled times.

A synchronized time is required to keep proper sender/receiver buffer
levels, taking into account the time zone and round-trip time (up to
2 seconds for satellite links). **Considering addition/subtraction
round-off, and possibly unsynchronized system times, an agreed-upon
time base drifts by a few microseconds every minute.** The drift may
accumulate over many days to a point where the sender or receiver
buffers will overflow or deplete, seriously affecting the quality
of the video. SRT has a time management mechanism to compensate
for this drift.

When a packet is received, SRT determines the difference between the
time it was expected and its timestamp. The timestamp is calculated on
the receiver side. The RTT tells the receiver how much time it was
supposed to take. SRT maintains a reference between the time at the
leading edge of the send buffer's latency window and the corresponding
time on the receiver (the present time). **This allows conversion to real
time to be able to schedule events, based on a local time reference.**

The receiver samples time drift data and periodically calculates a
packet timestamp correction factor, which is applied to each data
packet received by adjusting the inter-packet interval. When a
packet is received it isn't given right away to the application.
As time advances, the receiver knows the expected time for any
missing or dropped packet, and can use this information to fill
any "holes" in the receive queue with another packet
(see {{tsbpd}}).

It is worth noting that the period of sampling time drift data is based
on a number of packets rather than time duration to ensure enough
samples, independently of the media stream packet rate. The effect of
network jitter on the estimated time drift is attenuated by using a
large number of samples. **The time drift being very slow (affecting a
stream only after many hours), a fast reaction is not necessary.**

The receiver uses local time to be able to schedule events — to
determine, for example, if it's time to deliver a certain packet
right away. The timestamps in the packets themselves are just
references to the beginning of the session. When a packet is received
(with a timestamp from the sender), the receiver makes a reference to
the beginning of the session to recalculate its timestamp. The start
time is derived from the local time at the moment that the session is
connected. A packet timestamp equals "now" minus "StartTime", where
zthe latter is the point in time when the socket was created.


## Too-Late Packet Drop {#too-late-packet-drop}

Too-Late Packet Drop (TLPKTDROP) mechanism allows the sender to drop
packets that have no chance to be delivered in time and the receiver
to skip missing packets that have not been delivered in time.

In the SRT sender, when Too-Late Packet Drop is enabled, a packet is
considered too late to be delivered and may be dropped by the sending
application if its timestamp is older than 125% of the SRT latency.
However, the sender keeps packets for at least 1 second in case the
SRT latency is not enough for a large RTT (that is, if 125% of the
SRT latency is less than 1 second).

When enabled on the receiver, the receiver drops packets that have not
been delivered or retransmitted in time and delivers the subsequent
packets to the application when their time-to-play has come.

<!-- TODO: It also sends a fake ACK message to the sender. - Should we write about this? -->

In pseudo-code, the algorithm of reading from the receiver buffer is
the following:

    pos = 0;  /* Current receiver buffer position */
    i = 0;    /* Position of the next available in the receiver buffer 
                 packet relatively to the current buffer position pos */

    while(True) {
        Get the position of the next available in receiver buffer packet i;
        Calculate packet delivery time for the next available packet PktTsbpdTime;

        if T_NOW < PktTsbpdTime:
            continue;

        Drop packets which buffer position number is less than i;

        Deliver packet with the buffer position i;

        pos = i + 1;
    }

where T_NOW is the current time according to the receiver clock.

The TLPKTDROP mechanism can be turned off to always ensure a clean
delivery. However, a lost packet can simply pause a delivery for some
longer, potentially undefined time, and cause even worse tearing
for the player. Setting higher SRT latency will help much more in the
case when TLPKTDROP causes packet drops too often.


## Packet Acknowledgement (ACKs)



## Packet Retransmission (NAKs)

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


