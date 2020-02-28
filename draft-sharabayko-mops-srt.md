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

## Timestamp Based Packet Delivery

<!--
TODO:
* Introduction
* Section with notations plus put TsbPD in here, check how to make the first reference
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
-->

The initial intent of the SRT Timestamp Based Packet Delivery (TsbPD) feature was 
to reproduce the output of the encoding engine at the input of the decoding one.

It's designed in a way that SRT receiver, using the timestamp of the SRT data packet header,
delivers packets to the decoder at the same pace they were provided to the SRT sender by an encoder.
Basically, the sender timestamp in the received packet is adjusted to the receiver’s local time
(compensating for time drift or different time zone) before releasing the packet to the application.
<!-- ms - microseconds -->
<!-- How the delay is configured -->
Packets can be withheld by the SRT receiver for a configured receiver delay (ms).
Higher delay can accommodate a larger uniform packet drop rate or larger packet burst drop.
Packets received after their “play time” are dropped (!!! link to Too Late Packet Drop section, ? packets are dropped only if the option eneabled).

The packet timestamp (in microseconds) is relative to the SRT connection creation time (!!! link).
<!-- Understand this part an rewrite a bit -->
The original UDT code uses the packet sending time to stamp the packet. This is inappropriate for
the TsbPD feature since a new time (current sending time) is used for retransmitted packets,
putting them out of order when inserted at their proper place in the stream. Packets are
inserted based on the sequence number in the header field.
The origin time (in microseconds) of the packet is already sampled when a packet is first
submitted by the application to the SRT sender. The TsbPD feature uses this time to stamp the
packet for first transmission and any subsequent re-transmission. This timestamp and the
configured latency (!!! link) control the recovery buffer size and the instant that packets are delivered at
the destination.

<!-- TODO -->
Picture which reflects the idea:
1. Comment regarding initial RTT and that RTT may vary in time
2. Comment regarding relatively small sending delay, plus reflect receiving delay
3. Change SRT documentation in order to reflect the formula.
4. Under the picture explain all the notations: Packet Time, Packet Sending Time, Packet Receiving Time, Packet Delivery Time
<!-- TODO ends -->

### Packet Delivery Time

<!-- Packet delivery time is the time point, estimated by the receiver, when a packet should be given (delivered) to the upstream application (via `srt_recvmsg(...)`). -->

The calculation of packet delivery time (PktTsbpdTime) is performed upon receiving a data packet according to the following formula:

~~~
PktTsbpdTime = TsbpdTimeBase + PKT_TIMESTAMP + TsbpdDelay + Drift
~~~

where
TsbpdTimeBase is the base time which reflects or used by receiver to ... (write, but it is not a time difference), in microseconds,
<!-- `TsbPdTimeBase` is the base time difference between local clock of the receiver, and the clock used by the sender to timestamp packets being sent. A unit of measurement is microseconds. 
the base time difference between sender's and receiver's clock
Question:
time base - what's this https://www.lingvolive.com/en-us/translate/en-ru/time%20base
time base or base time -->
PKT_TIMESTAMP is the data packet timestamp, in microseconds,
TsbpdDelay is the receiver's buffer delay (?latency), in milliseconds (?),
Drift is the time drift (write later).

///
NB: The default live mode settings set SRTO_RCVLATENCY to 120 ms! The buffer mode settings set SRTO_RCVLATENCY to 0.
The time that should elapse since the moment when the packet was sent and the moment when it's delivered to the receiver application in the receiving function. This time should be a buffer time large enough to cover the time spent for sending, unexpectedly extended RTT time, and the time needed to retransmit the lost UDP packet. The effective latency value will be the maximum of this options' value and the value of SRTO_PEERLATENCY set by the peer side. This option in pre-1.3.0 version is available only as SRTO_LATENCY.

#### TSBPD Base Time Calculation

The initial value of TSBPD base time (TsbpdTimeBase) is calculated at the moment of second handshake request is received as follows

~~~
TsbpdTimeBase = T_NOW - HSREQ_TIMESTAMP
~~~

where
T_NOW is the current time at the receiver clocks,
HSREQ_TIMESTAMP is the handshake packet timestamp, in microseconds.

<!-- This value should roughly correspond to the one-way delay \(**~RTT/2**\). - It's initial RTT -->

During the transmission process, this value may be adjusted due to the following reasons:
<!-- TODO -->
Time Base may change because of 2 reasons:
1. 32-bit integer is not enough (link to SRT packet structure).
2. Drift algorithm.


## Drift Management (provide formula and leave for later)

Start with this 
https://srtlab.github.io/srt-cookbook/protocol/tsbpd/drift-management/

The drift tracer algorithm is designed to capture the fluctuations in time between sender and receiver and adjust the value of TSBPD base time (TsbpdTimeBase) whenever necessary. 
<!-- (due to clock inaccuracy) -->


Note regarding drift tracer: The current algorith does not take into account RTT variations, but we are going to improve this.
<!-- Assuming that the link latency is constant (RTT=const), the only cause of the drift fluctuations should be clock inaccuracy. -->


## Too-Late Packet Drop

<!-- ? Too-Late -> Too Late -->

Too-Late Packet Drop (TLPKTDROP) mechanism allows the sender to drop packets that have no chance
to be delivered in time and the receiver to skip missing packets that have not been delivered in time.

In the SRT sender, when Too-Late Packet Drop is enabled, a packet is
considered too late to be delivered and may be dropped by the sending application if its
timestamp is older than 125% of the SRT latency. However, the sender keeps packets at least
1000 milliseconds if SRT latency is lower than the specified value (SRT latency is not enough for large RTT).

When enabled on the receiver, it drops packets that have not been delivered or retransmitted in time and delivers
the subsequent packets to the application when their time-to-play has come. 
<!-- ??? It also sends a fake ACK message to the sender. -->

In pseudo-code the reading from receiver buffer 

// 1

both - seq numbers
NextScheduled = 1
<!-- NextAvailable = 1 -->

while (True) {
  read next available in the buffer packet NextAvailable;

  if NextScheduled == NextAvailable, then /* if next available in the buffer packet is equal to next scheduled for the delivery packet */
    calculate delivery time for the packet;
    sleep until the moment of packet is scheduled for delivery;
    deliver the packet;
    NextScheduled = NextScheduled + 1;
    continue

  if NextScheduled < NextAvailable, then /* if there are missing packets */
    calculate delivery time of the next available packet NextAvailableDeliveryTime;

    if T_NOW <= NextAvailableDeliveryTime:
      continue

}

// 2

NextScheduled = 1

while(True) {
  read next available in the buffer packet NextAvailable;
  calculate packet delivery time PacketDeliveryTime;

  if T_NOW <= PacketDeliveryTime

}






add Next Exp cplumn

| s @ Dst | SrcTime (PKT_TIMESTAMP) | SND Clocks   | Time Base    | RCV Clocks   | SRT Latency | Drift | Packet Delivery Time |   |
|---------|-------------------------|--------------|--------------|--------------|-------------|-------|----------------------|---|
| 1       | 20                      | 00:00:00,020 | 00:00:00,040 | 00:00:00,060 | 120         | 0     | 00:00:00,180         |   |
| 2       | 40                      | 00:00:00,040 | 00:00:00,040 | 00:00:00,080 | 120         | 0     | 00:00:00,200         |   |
| 5       | 100                     | 00:00:00,100 | 00:00:00,040 | 00:00:00,140 | 120         | 0     | 00:00:00,260         |   |
| 6       | 120                     | 00:00:00,120 | 00:00:00,040 | 00:00:00,160 | 120         | 0     | 00:00:00,280         |   |
| 7       | 140                     | 00:00:00,140 | 00:00:00,040 | 00:00:00,180 | 120         | 0     | 00:00:00,300         |   |
| 8       | 160                     | 00:00:00,160 | 00:00:00,040 | 00:00:00,200 | 120         | 0     | 00:00:00,320         |   |
| 3       | 60                      | 00:00:00,060 | 00:00:00,040 | 00:00:00,210 | 120         | 0     | 00:00:00,220         |   |
| 4       | 80                      | 00:00:00,080 | 00:00:00,040 | 00:00:00,212 | 120         | 0     | 00:00:00,240         |   |
| 9       | 180                     | 00:00:00,180 | 00:00:00,040 | 00:00:00,220 | 120         | 0     | 00:00:00,340         |   |
| 10      | 200                     | 00:00:00,200 | 00:00:00,040 | 00:00:00,240 | 120         | 0     | 00:00:00,360         |   |




///
SRTO_TLPKTDROP: When true (default), it will drop the packets that haven't been retransmitted on time, that is, before the next packet that is already received becomes ready to play. You can turn this off to always ensure a clean delivery. However, a lost packet can simply pause a delivery for some longer, potentially undefined time, and cause even worse tearing for the player. Setting higher latency will help much more in the case when TLPKTDROP causes packet drops too often.

///
NB: The default live mode settings set SRTO_SNDDROPDELAY to 0. The buffer mode settings set SRTO_SNDDROPDELAY to -1.

[SET] - Sets an extra delay before TLPKTDROP is triggered on the data sender. TLPKTDROP discards packets reported as lost if it is already too late to send them (the receiver would discard them even if received). The total delay before TLPKTDROP is triggered consists of the LATENCY (SRTO_PEERLATENCY), plus SRTO_SNDDROPDELAY, plus 2 * the ACK interval (default ACK interval is 10ms). The minimum total delay is 1 second. A value of -1 discards packet drop. SRTO_SNDDROPDELAY extends the tolerance for retransmitting packets at the expense of more likely retransmitting them uselessly. To be effective, it must have a value greater than 1000 - SRTO_PEERLATENCY.

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


