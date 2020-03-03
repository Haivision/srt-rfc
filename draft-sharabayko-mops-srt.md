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
  GHG04b:
    title: Experiences in Design and Implementation of a High Performance Transport Protocol
    author:
      - 
        name: Yunhong Gu
      - 
        name: Xinwei Hong
      - 
        name: Robert L. Grossman, 
    date: December, 2004
    seriesinfo:
      DOI: 10.1109/SC.2004.24
  PNPID:
    target: https://uefi.org/PNP_ACPI_Registry
    title: PNP ID AND ACPI ID REGISTRY
    date: none
  SP800-38A:
    title: Recommendation for Block Cipher Modes of Operation
    author:
      name: Morris Dworkin
      ins: M. Dworkin
    date: December, 2001
    
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

# Packet Structure {#packet-structure}

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


## Data Packets {#data-pkt}

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
|      Receiver TSBPD Delay     |       Sender TSBPD Delay      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #handshake-extension-msg-structure title="Handshake Extension Message structure"}

SRT Version (32 bits):
: SRT library version.

SRT Flags (32 bits):
: SRT configuration flags.

 | Bitmask    | Flag |
 | ---------- | :---------------: |
 | 0x00000001 | TSBPDSND          |
 | 0x00000002 | TSBPDRCV          |
 | 0x00000004 | CRYPT             |
 | 0x00000008 | TLPKTDROP         |
 | 0x00000010 | PERIODICNAK       |
 | 0x00000020 | REXMITFLG         |
 | 0x00000040 | STREAM            |
 | 0x00000080 | PACKET_FILTER     |
{: #hs-ext-msg-flags title="HS Extension Message Flags"}

Receiver TSBPD Delay (16 bits):
: TSBPD delay of the receiver. Refer to {{tsbpd}}.

Sender TSBPD Delay (16 bits):
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

S (1 bit):
: Start Bit. The value of this field sets to 0.

V (3 bits):
: SRT Version. The initial version value is 1.

PT (4 bits):
: Packet Type. The value of this field always sets to 2.
  The other values are reserved for future definition.

Sign (16 bits):
: Signature. This field is PnP Vendor ID({{PNPID}}) in big endian order.

Rev (4 bits):
: Reserved.

KK (2 bits):
: Encryption Flag. Refer to KK field in {{data-pkt}}.

KEKI (32 bits):
: Key Encryption Key Index in big endian order. The default stream associated
  key set to 0. The values from 1 to 255 are reserved for manually indexed keys.

Cipher (8 bits):
: Encryption Cipher and mode. 

 | Bitmask    | Flag |
 | ---------- | :---------------: |
 | 0x00       | None or KEK indexed crypto context |
 | 0x01       | AES-ECB (potentially for VF 2.0 compatible message) |
 | 0x02       | AES-CTR ({{SP800-38A}}) |
{: #hs-km-msg-flags title="HS Key Material Exchange Cipher mode"}
 
Auth (8 bits):
: Message authentication code algorithm. The default value of this field sets to 0 which
  means None or KEK indexed crypto context.

SE (8 bits):
: Stream Encapsulation. The value, 0, of this field means Unspecified or KEK indexed crypto
  context. If the stream is MPEG-TS over UDP, this field sets to 1. The value, 2, indicates
  that the stream is encapsulated as MPEG-TS over SRT.

SLen (4 bits):
: Salt length in bytes divided by 4. The value of this field can be 0 if no salt or initial
  vector presents.

KLen (4 bits):
: Stream encrypting key length in bytes divided by 4. The value is the size of one key even
  if two keys presents.

Salt (32 bits):
: Salt Key. The length of salt key is SLen * 4 * 8.

Wrapped Key (variable length):
: The length of this field is 64 + KLen * n * 4 * 8, n is the number of keys; 1 or 2.
  This field is expanded as the following.

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

ICV (64 bits): 
: 64-bit Integrity Check Vector(AES key wrap integrity).

xSEK (variable length):
: This field identifies an odd or even SEK. If both keys are present,
  then this field is eSEK (even key) and the next one is the odd key.
  The length of this field is calculated by KLen * 4 * 8.

oSEK (variable length):
: This field is present only when the message carries the two SEKs.
  
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

Multiple SRT socket may share one UDP socket and the packets received to this
UDP socket will be correctly dispatched to the SRT socket to which they are
currently destined.
During the handshake, the parties exchange their SRT Socket IDs.
These IDs are then used in the Destination Socket ID field of every control and data packet.

## Data Transmission Modes {#data-transmission-mode}

In file transfer mode this a message with O=0 that is sent later
(but reassembled before an earlier message which may be incomplete due to packet loss)
is allowed to be delivered immediately, without waiting for the earlier message to be completed.
In Live Transmission Mode the only valid value is "1".

### Message Mode {#transmission-mode-msg}

When the STREAM flag of the handshake Extension Message Flags is set to 0,
the protocol operates in the message mode.

Every packet has its own Packet Sequence Number.
One or several consecutive SRT Data packet can form a message.
In that case all the packets belonging to the same message have similar
message number set in the Message Number field.

The first packet of the message has the first bit of the Packet Position Flags ({{data-pkt}})
set to 1. The last packet of the message has the second bit of the Packet Position Flags
set to 1. Thus, PP equal to "11b" indicates a packet that forms the whole message.
The PP field equal to "00b" indicates a packet that belongs to the inner part of the message.

The concept of the message in SRT comes from UDT ({{GHG04b}}).
In this mode a single sending instruction passes exactly one piece of data
that has boundaries (a message). This message may span across multiple UDP packets
(and multiple SRT data packets). The only size limitation is that it shall fit as a whole
in the buffers of the sender and the receiver.
Although internally all the operations on data packets (ACK, NAK) are performed independently,
it is only allowed for an application to send and receive the whole message.
Until the message is complete (all packets are received) it will not be allowed to read it.

The Order Flag of the Data packet set to 1 restricts the reading order of the messages to be sequential.
While the Order Flag set to 0 allows to read those messages that are already fully available, before
preceding messages, that still have some packets missing.

### Live mode {#transmission-mode-live}

Live mode is a special case of the message mode where only data packets
with PP field set to "11b" are allowed.

Additionally Timestamp Based Packet Delivery (TSBPD) ({{tsbpd}}) and
Too-Late Packet Drop ({{too-late-packet-drop}}) mechanisms are used in this mode.

### Buffer mode {#transmission-mode-buffer}

Buffer mode is negotiated during the Handshake by setting the STREAM flag
of the handshake Extension Message Flags to 1.

In this mode the consecutive packets form one consecutive stream that can be read with
the portions of any size.

## Handshake Messages {#handshake-messages}

SRT is a connection protocol. It embraces the concepts of "connection"
and "session". The UDP system protocol is used by SRT for sending data
and control packets.

An SRT connection is characterized by the fact that it is:

- first engaged by a handshake process

- maintained as long as any packets are being exchanged in a timely manner

- considered closed when a party receives the appropriate close command from
  its peer (connection closed by the foreign host), or when it receives no
  packets at all for some predefined time (connection broken on timeout).

SRT supports two connection configurations:

1. Caller-Listener, where one side waits for the other to initiate a connection
2. Rendezvous, where both sides attempt to initiate a connection

The handshake is performed between two parties: "Initiator" and "Responder":

- Initiator starts the extended SRT handshake process and sends appropriate
  SRT extended handshake requests;

- Responder expects the SRT extended handshake requests to be sent by the
  Initiator and sends SRT extended handshake responses back.

There are two basic types of SRT handshake extensions that are exchanged
in the handshake:

- Handhshake Extension Message exchanges the basic SRT information;
- Key Material Exchange exchanges the wrapped stream encryption key (used only if
  encryption is requested).

The Initiator and Responder roles are assigned depending on the connection mode.

For Caller-Listener connections: the Caller is the Initiator,
the Listener is the Responder.
For Rendezvous connections: the Initiator and Responder roles are assigned
based on the initial data interchange during the handshake.

The Handshake Type field in the Handshake Structure (see {{handshake-packet-structure}}) 
indicates the handshake message type.

Caller-Listener handshake exchange has the following order of Handshake Types:

1. Caller to Listener: INDUCTION
2. Listener to Caller: INDUCTION (reports cookie)
3. Caller to Listener: CONCLUSION (uses previously returned cookie)
4. Listener to Caller: CONCLUSION (confirms connection established)

Rendezvous handshake exchange has the following order of Handshake Types:

1. After starting the connection: `URQ_WAVEAHAND`
2. After receiving the above message from the peer: `URQ_CONCLUSION`
3. After receiving the above message from the peer: `URQ_AGREEMENT`.

In case when the connection process has failed when the party was about to
send the CONCLUSION handshake, the Handshake Type field will contain appropriate
error value. See the list of error codes in {{hs-rej-reason}}.

 | Code | Rejection Reason | Description       |
 | ---- | :--------------: | :---------------: |
 | 1000 | REJ_UNKNOWN      | Unknown reason    |
 | 1001 | REJ_SYSTEM       | System function error       |
 | 1002 | REJ_PEER         | Rejected by peer            |
 | 1003 | REJ_RESOURCE     | Resource allocation problem |
 | 1004 | REJ_ROGUE        | incorrect data in handshake |
 | 1005 | REJ_BACKLOG      | listener's backlog exceeded |
 | 1006 | REJ_IPE          | internal program error |
 | 1007 | REJ_CLOSE        | socket is closing |
 | 1008 | REJ_VERSION      | peer is older version than agent's minimum set |
 | 1009 | REJ_RDVCOOKIE    | rendezvous cookie collision |
 | 1010 | REJ_BADSECRET    | wrong password |
 | 1011 | REJ_UNSECURE     | password required or unexpected |
 | 1012 | REJ_MESSAGEAPI   | Stream flag collision |
 | 1013 | REJ_CONGESTION   | incompatible congestion-controller type |
 | 1014 | REJ_FILTER       | incompatible packet filter |
 | 1015 | REJ_GROUP        | incompatible group |
{: #hs-rej-reason title="HS Rejection Reason Codes"}

The specification of `PBKEYLEN` is decided by the Sender. When the transmission 
is bidirectional, this value must be agreed upon at the outset because when both 
are set, the Responder wins. For Caller-Listener connections it is reasonable to 
set this value on the Listener only. In the case of Rendezvous the only reasonable 
approach is to decide upon the correct value from the different sources and to 
set it on both parties (note that **AES-128** is the default).

### Caller-Listener Handshake {#caller-listener-handshake}

This section describes the handshaking process where a Listener is
waiting for an incoming Handshake request on a bound UDP port from a Caller.
The process has two phases: induction and conclusion.

#### The Induction Phase

The Caller begins by sending an "induction" message, which contains the following
(significant) fields:

- Version: must always be 4
- Encryption Field: 0
- Extension Field: 2
- Handshake Type: INDUCTION
- SRT Socket ID: SRT Socket ID of the Caller
- SYN Cookie: 0

The Destination Socket ID of the SRT packet header in this message is 0, which is
interpreted as a connection request.

NOTE: The handshake version number is set to 4 in this initial handshake.
It is due to the initial design of SRT that was to be compliant with the UDT
protocol ({{GHG04b}}) it is based on.

NOTE: This phase serves only to set a cookie on the Listener so that it
doesn't allocate resources, thus mitigating a potential DOS attack that might be
perpetrated by flooding the Listener with handshake commands.

The Listener responds with the following:

- Version: 5
- Encryption Field: Advertised cipher family and block size.
- Extension Field: SRT magic code 0x4A17
- Handshake Type: INDUCTION
- ID: Socket ID of the HSv5 Listener
- SYN Cookie: a cookie that is crafted based on host, port and current time
  with 1 minute accuracy

NOTE: At this point the Listener still doesn't know if the Caller is SRT or UDT,
and it responds with the same set of values regardless of whether the Caller is
SRT or UDT.

If the party is SRT, it does interpret the values in Version and Type.
If it receives the value 5 in Version, it understands that it comes from an SRT
party, so it knows that it should prepare the proper handshake messages
phase. It also checks the following:

- whether the Extension Flags contains the magic value 0x4A17; otherwise the
  connection is rejected. This is a contingency for the case where someone who,
  in an attempt to extend UDT independently, increases the Version value to 5
  and tries to test it against SRT.

- whether the Encryption Flags contain a non-zero
  value, which is interpreted as an advertised cipher family and block size.

The legacy UDT party completely ignores the values reported in Version and
Handshake Type.  It is, however, interested in the SYN Cookie value, as this must be
passed to the next phase. It does interpret these fields, but only in the "conclusion" message.

### Rendezvous Handshake

Rendezvous handshake exchange has the following order of Handshake Types:

1. After starting the connection: `URQ_WAVEAHAND`
2. After receiving the above message from the peer: `URQ_CONCLUSION`
3. After receiving the above message from the peer: `URQ_AGREEMENT`.

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
between initiator and responder. The handshake extension ({{handshake-extension-msg}}) has 
receiver and sender TSBPD delay information in milliseconds ({{tsbpd}}). The maximum value of latencies
from initiator and responder will be established. 

## Timestamp Based Packet Delivery {#tsbpd}

The goal of the SRT Timestamp Based Packet Delivery (TSBPD) mechanism
is to reproduce the output of the sending application (e.g., encoder)
at the input of the receiving one (e.g., decoder) in live data
transmission mode (see {{data-transmission-mode}}). In terms of SRT,
it means to reproduce the timing of packets commited by the sending
application to the SRT sender at the timing packets are scheduled
for the delivery by the SRT receiver and ready to be read by the
receiving application (see {{fig-latency-points}}).

The SRT receiver, using the timestamp of the SRT data packet header,
delivers packets to a receiving application with a fixed minimum
delay from the time the packet was scheduled for sending on the SRT
sender side. Basically, the sender timestamp in the received packet
is adjusted to the receiver’s local (compensating for the time drift
or different time zone) before releasing the packet to the application.
Packets can be withheld by the SRT receiver for a configured receiver
delay. Higher delay can accommodate a larger uniform packet drop rate
or larger packet burst drop. Packets received after their "play time"
are dropped if Too-Late Packet Drop feature is enabled
(see {{too-late-packet-drop}}).

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
              |  Sending  |              |                   |
              |   Delay   |    ~RTT/2    |    SRT Latency    |
              |<--------->|<------------>|<----------------->|
              |           |              |                   |
              |           |              |                   |
              |           |              |                   |
    ___ Scheduled       Sent         Received           Scheduled
   /    for sending       |              |              for delivery
Packet        |           |              |                   |
State         |           |              |                   |
              |           |              |                   |
              |           |              |                   |
              ----------------------------------------------------->
                                                                Time
~~~
{: #fig-latency-points title="Key Latency Points during the Packet Transmission"}

The main packet states shown at in {{fig-latency-points}} are the following:

- "Scheduled for sending": the packet is committed by the sending application, stamped and ready to be sent;
- "Sent": the packet is passed to the UDP socket and sent;
- "Received": the packet is received and read from the UDP socket;
- "Scheduled for delivery": the packet is scheduled for the delivery
  and ready to be read by the receiving application.

It is worth noting that the round-trip time (RTT) of an SRT link may
vary in time. However the actual end-to-end latency on the link becomes
fixed and is approximately equal to (RTT_0/2 + SRT Latency) once the SRT handshake exchange happens,
where RTT_0 is the actual value of the round-trip time during the SRT handshake
exchange (the value of the round-trip time once the SRT connection has been established).

The value of sending delay depends on the hardware performance. Usually
it is relatively small (several microseconds) in contrast to RTT_0/2
and SRT latency which are measured in milliseconds.

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

It's worth noting that TsbpdDelay limits the number of packet retransmissions to a certain extent
making impossible to retransmit packets endlessly. This is important for live data transmission.

#### TSBPD Time Base Calculation {#tsbpd-time-base}

The initial value of TSBPD time base (TsbpdTimeBase) is calculated at the moment of
the second handshake request is received as follows:

~~~
TsbpdTimeBase = T_NOW - HSREQ_TIMESTAMP
~~~

where T_NOW is the current time according to the receiver clock;
HSREQ_TIMESTAMP is the handshake packet timestamp, in microseconds.

The value of TsbpdTimeBase is approximately equal to the initial one-way delay of the link RTT_0/2,
where RTT_0 is the actual value of the round-trip time during the SRT handshake
exchange.

During the transmission process, the value of TSBPD time base may be adjusted in two cases:

1. During the TSBPD wrapping period.

The TSBPD wrapping period happens every 01:11:35 hours. This time corresponds
to the maximum timestamp value of a packet (MAX_TIMESTAMP). MAX_TIMESTAMP is equal
to 0xFFFFFFFF, or the maximum value of 32-bit unsigned integer, in microseconds ({{packet-structure}}).
The TSBPD wrapping period starts 30 seconds before reaching the maximum timestamp value
of a packet and ends once the packet with timestamp within [30, 60] seconds interval
is delivered (read from the buffer). The updated value of TsbpdTimeBase will be recalculated as follows:

~~~
TsbpdTimeBase = TsbpdTimeBase + MAX_TIMESTAMP + 1
~~~

2. By drift tracer. See {{drift-management}} for details.


## Drift Management {#drift-management}

When the sender enters "connected" status it tells the application
there is a socket interface that is transmitter-ready. At this point
the application can start sending data packets. It adds packets to
the SRT sender's buffer at a certain input rate, from which they are
transmitted to the receiver at scheduled times.

A synchronized time is required to keep proper sender/receiver buffer
levels, taking into account the time zone and round-trip time (up to
2 seconds for satellite links). Considering addition/subtraction
round-off, and possibly unsynchronized system times, an agreed-upon
time base drifts by a few microseconds every minute. The drift may
accumulate over many days to a point where the sender or receiver
buffers will overflow or deplete, seriously affecting the quality
of the video. SRT has a time management mechanism to compensate
for this drift.

When a packet is received, SRT determines the difference between the
time it was expected and its timestamp. The timestamp is calculated on
the receiver side. The RTT tells the receiver how much time it was
supposed to take. SRT maintains a reference between the time at the
leading edge of the send buffer's latency window and the corresponding
time on the receiver (the present time). This allows to convert packet
timestamp to the local receiver time. Based on this time, various
events (packet delivery, etc.) can be scheduled.

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
large number of samples. The actual time drift being very slow (affecting a
stream only after many hours) does not require a fast reaction.

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
to skip missing packets that have not been delivered in time. The
timeout of dropping a packet is based on the TSBPD mechanism
(see {{tsbpd}}).

In the SRT sender, when Too-Late Packet Drop is enabled, a packet is
considered too late to be delivered and may be dropped by the sending
application if its timestamp is older than 125% of the SRT latency.
However, the sender keeps packets for at least 1 second in case the
SRT latency is not enough for a large RTT (that is, if 125% of the
SRT latency is less than 1 second).

When enabled on the receiver, the receiver drops packets that have not
been delivered or retransmitted in time and delivers the subsequent
packets to the application when their time-to-play has come.

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
or if both peers agree to drop this packet (see {{too-late-packet-drop}}).

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

Once an SRT connection is established, both peers can send data packets simultaneously.

### Round Trip Time Estimation

The round-trip time is estimated during the transmission of SRT data packets
based on the time difference between the ACK packet is sent and the
corresponding ACKACK is received by the data receiver.

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

This document makes no requests of the IANA.

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
