---
title: The SRT Protocol
abbrev: SRT
docname: draft-sharabayko-mops-srt-00
category: std

ipr: trust200902
area: opsarea
workgroup: MOPS
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
    ins: "M.A. Sharabayko"
    name: "Maria Sharabayko"
    organization: "Haivision Network Video, GmbH"
    email: msharabayko@haivision.com
 -
    ins: "J. Dube"
    name: "Jean Dube"
    organization: "Haivision"
    email: jdube@haivision.com
 -
    ins: "JS. Kim"
    name: "Jeongseok Kim"
    organization: "SK Telecom Co., Ltd."
    email: jeongseok.kim@sk.com
 -
    ins: "JW. Kim"
    name: "Joonwoong Kim"
    organization: "SK Telecom Co., Ltd."
    email: joonwoong.kim@sk.com

normative:
  RFC2119:
  RFC0768:

informative:
  RFC8174:
  RFC8216:
  RFC3031:
  RFC3394:
  RFC3547:
  RFC3711:
  RFC3830:
  RFC8312:
  RFC4987:
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
  SRTTO:
    title: SRT Protocol Technical Overview
    author:
      - 
        name: Jean Dube
      - 
        name: Steve Matthews
    date: December, 2019
  RTMP:
    target: https://www.adobe.com/devnet/rtmp.html
    title: Real-Time Messaging Protocol
    date: none
  ISO23009:
    title: Information technology — Dynamic adaptive streaming over HTTP (DASH)
    author:
      org: ISO
    date: none
    seriesinfo:
      "ISO/IEC": 23009:2019
  ISO13818-1:
    title: >
      Information technology 
      — Generic coding of moving pictures and associated audio information:
      Systems
    author:
      org: ISO
    date: none
    seriesinfo:
      "ISO/IEC": 13818-1
  I-D.ietf-quic-http:
  I-D.ietf-quic-transport:
  H.265:
    title: "H.265 : High efficiency video coding"
    author:
      org: International Telecommunications Union
    date: 2019
    seriesinfo:
      ITU-T: "Recommendation H.265"
  VP9:
    target: https://www.webmproject.org/vp9
    title: VP9 Video Codec
    author:
      org: WebM 
    date: none
  AV1:
    target: https://aomediacodec.github.io/av1-spec/av1-spec.pdf
    title: AV1 Bitstream & Decoding Process Specification
    author:
      -
        name: Peter de Rivaz
        org: Argon Design Ltd
      -
        name: Jack Haughton
        org: Argon Design Ltd
    date: none
  SRTSRC:
    target: https://github.com/Haivision/srt
    title: SRT fully functional reference implementation
    date: none

--- abstract

This document specifies Secure Reliable Transport (SRT) protocol. 
SRT is a user-level protocol over User Datagram Protocol and provides 
reliability and security optimized for low latency live video streaming, 
as well as generic bulk data transfer. For this, SRT introduces control
packet extension, improved flow control, enhanced congestion control
and a mechanism for data encryption.

--- middle

# Introduction

## Motivation 

The demand for live video streaming has been increasing steadily for many years. With 
the emergence of cloud technologies, many video processing pipeline components have 
transitioned from on-premises appliances to software running on cloud instances. While 
real-time streaming over TCP-based protocols like RTMP{{RTMP}} is possible at low bitrates and 
on a small scale, the exponential growth of the streaming market has created a need for 
more powerful solutions. 

To improve scalability on the delivery side, content delivery networks (CDNs) at one 
point transitioned to segmentation-based technologies like HLS (HTTP Live Streaming){{RFC8216}}
and DASH (Dynamic Adaptive Streaming over HTTP){{ISO23009}}. This move increased the end-to-end 
latency of live streaming to over 30 seconds, which makes it unattractive for many 
use cases. Over time, the industry optimized these delivery methods, bringing the 
latency down to 3 seconds.  

While the delivery side scaled up, improvements to video transcoding became a necessity. 
Viewers watch video streams on a variety of different devices, connected over different 
types of networks. Since upload bandwidth from on-premises locations is often limited, 
video transcoding moved to the cloud. 

RTMP became the de facto standard for contribution over the public Internet. But there 
are limitations for the payload to be transmitted, since RTMP as a media specific 
protocol only supports two audio channels and a restricted set of audio and video codecs, 
lacking support for newer formats such as HEVC{{H.265}}, VP9{{VP9}}, or AV1{{AV1}}.

Since RTMP, HLS and DASH rely on TCP, these protocols can only guarantee acceptable 
reliability over connections with low RTTs, and can not use the bandwidth of network 
connections to their full extent due to limitations imposed by congestion control. 
Notably, QUIC{{I-D.ietf-quic-transport}} has been designed to address these problems with HTTP-based delivery 
protocols in HTTP/3{{I-D.ietf-quic-http}}. Like QUIC, SRT{{SRTSRC}} uses UDP instead of the TCP transport protocol,
but assures more reliable delivery using Automatic Repeat Request (ARQ), packet acknowledgments,
end-to-end latency management, etc.

## Secure Reliable Transport Protocol 

Low latency video transmissions across reliable (usually local) IP based networks 
typically take the form of MPEG-TS{{ISO13818-1}} unicast or multicast streams using the UDP/RTP 
protocol, where any packet loss can be mitigated by enabling forward error correction 
(FEC). Achieving the same low latency between sites in different cities, countries or 
even continents is more challenging. While it is possible with satellite links or 
dedicated MPLS{{RFC3031}} networks, these are expensive solutions. The use of public Internet 
connectivity, while less expensive, imposes significant bandwidth overhead to achieve 
the necessary level of packet loss recovery. Introducing selective packet retransmission 
(reliable UDP) to recover from packet loss removes those limitations.  

Derived from the UDP-based Data Transfer protocol{{GHG04b}} (UDT), SRT is a user-level protocol 
that retains most of the core concepts and mechanisms while introducing several 
refinements and enhancements, including control packet modifications, improved flow 
control for handling live streaming, enhanced congestion control, and a mechanism for 
encrypting packets.

SRT is a transport protocol that enables the secure, reliable transport of data across 
unpredictable networks, such as the Internet. While any data type can be transferred 
via SRT, it is ideal for low latency (sub-second) video streaming. SRT provides 
improved bandwidth utilization compared to RTMP, allowing much higher 
contribution bitrates over long distance connections. 

As packets are streamed from source to destination, SRT detects and adapts to the 
real-time network conditions between the two endpoints, and helps compensate for 
jitter and bandwidth fluctuations due to congestion over noisy networks. Its error 
recovery mechanism minimizes the packet loss typical of Internet connections.

To achieve low latency streaming, SRT had to address timing issues. The characteristics 
of a stream from a source network are completely changed by transmission over the public 
Internet, which introduces delays, jitter, and packet loss. This, in turn, leads to 
problems with decoding, as the audio and video decoders do not receive packets at the 
expected times. The use of large buffers helps, but latency is increased. 
SRT includes a mechanism to keep a constant end-to-end latency, thus recreating
the signal characteristics on the receiver side, and reducing the need for buffering.

Like TCP, SRT employs a listener/caller model. The data flow is bi-directional and 
independent of the connection initiation - either the sender or receiver can operate 
as listener or caller to initiate a connection. The protocol provides an internal 
multiplexing mechanism, allowing multiple SRT connections to share the same UDP port, 
providing access control functionality to identify the caller on the listener side. 

Supporting forward error correction (FEC) and selective packet retransmission (ARQ), 
SRT provides the flexibility to use either of the two mechanisms or both combined, 
allowing for use cases ranging from the lowest possible latency to the highest possible 
reliability. 

SRT maintains the ability for fast file transfers introduced in UDT, and adds support 
for AES encryption. 

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
+-+-+-+-+-+-+-+-+-+-+-+-+- SRT Header +-+-+-+-+-+-+-+-+-+-+-+-+-+
|F|        (Field meaning depends on the packet type)           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          (Field meaning depends on the packet type)           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Destination Socket ID                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+- CIF -+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                        Packet Contents                        |
|                  (depends on the packet type)                 +
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
+-+-+-+-+-+-+-+-+-+-+-+-+- SRT Header +-+-+-+-+-+-+-+-+-+-+-+-+-+
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
{: #srtdatapacket title="Data packet structure"}

Packet Sequence Number (31 bits):
: The sequential number of the data packet.

PP (2 bits):
: Packet Position Flag. This field indicates the position of the data packet in the message.
  The value "10b" (binary) means the first packet of the message. "00b" indicates a packet 
  in the middle. "01b" designates the last packet. If a single data packet forms the whole 
  message, the value is "11b".

O (1 bit):
: Order Flag. Indicates whether the message should be delivered by the receiver in order (1) 
  or not (0). Certain restrictions apply depending on the data transmission mode used 
  ({{data-transmission-mode}}).

KK (2 bits):
: Key-based Encryption Flag. The flag bits indicate whether or not data is encrypted.
  The value "00b" (binary) means data is not encrypted. "01b" indicates that data is
  encrypted with an even key, and "10b" is used for odd key encryption. Refer to {{encryption}}.
  The value "11b" is only used in control packets.

R (1 bit):
: Retransmitted Packet Flag. This flag is clear when a packet is transmitted the first time.
  The flag is set to "1" when a packet is retransmitted.

Message Number (26 bits):
: The sequential number of consecutive data packets that form a message (see PP field).

Data (variable length):
: The payload of the data packet. The length of the data is the remaining length of 
  the UDP packet.

## Control Packets

An SRT control packet has the following structure.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+- SRT Header +-+-+-+-+-+-+-+-+-+-+-+-+-+
|1|         Control Type        |            Subtype            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Type-specific Information                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Destination Socket ID                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+- CIF -+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                   Control Information Field                   +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #controlpacket title="Control packet structure"}

Control Type (15 bits):
: Control Packet Type. The use of these bits is determined
  by the control packet type definition. See {{srt-ctrl-pkt-type-table}}.

Subtype (16 bits):
: This field specifies an additional subtype for specific packets.
  See {{srt-ctrl-pkt-type-table}}.

Type-specific Information (32 bits):
: The use of this field depends on the particular control
  packet type. Handshake packets do not use this field.

Control Information Field (variable length):
: The use of this field is defined by the Control Type field of the control packet.

The types of SRT control packets are shown in {{srt-ctrl-pkt-type-table}}.
The value "0x7FFF" is reserved for a user-defined type.

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

Handshake control packets (Control Type = 0x0000) are used to exchange peer configurations,
to agree on connection parameters, and to establish a connection.

The Control Information Field (CIF) of a handshake control packet is shown 
in {{handshake-packet-structure}}.

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
|                 Maximum Transmission Unit Size                |
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
{: #handshake-packet-structure title="Handshake packet structure"}

Version (32 bits):
: A base protocol version number. Currently used values are 4 and 5.
  Values greater than 5 are reserved for future use.

Encryption Field (16 bits):
: Block cipher family and block size. The values of this field are
  described in {{handshake-encr-fld}}.

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
  
  (1) If the handshake control packet is the INDUCTION message, this field is 
  sent back by the Listener. 
  (2) In the case of a CONCLUSION message, this field value should contain a combination 
  of Extension Type values. 

  For more details, see {{caller-listener-handshake}}.

 | Bitmask    | Flag |
 | ---------- | :---------------: |
 | 0x00000001 | HSREQ             |
 | 0x00000002 | KMREQ             |
 | 0x00000004 | CONFIG            |
{: #hs-ext-flags title="Handshake Extension Flags"}

Initial Packet Sequence Number (32 bits):
: The sequence number of the very first data packet to be sent.

Maximum Transmission Unit Size (32 bits):
: This value is typically set to 1500, which is the default Maximum Transmission Unit (MTU) 
  size for Ethernet, but can be less.

Maximum Flow Window Size (32 bits):
: The value of this field is the maximum number of data packets allowed to be "in flight"  
  (i.e. the number of sent packets for which an ACK control packet has not yet been received).

Handshake Type (32 bits):
: This field indicates the handshake packet type.
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
: This field holds the ID of the source SRT socket from which a handshake packet is issued.

SYN Cookie (32 bits):
: Randomized value for processing a handshake. The value of this field is specified
  by the handshake message type. See {{handshake-messages}}.

Peer IP Address (128 bits):
: IPv4 or IPv6 address of the packet's sender. The value consists of four 32-bit fields.
  In the case of IPv4 addresses, fields 2, 3 and 4 are filled with zeroes.

Extension Type (16 bits):
: The value of this field is used to process an integrated handshake.
  There are two extensions: Handshake Extension Message ({{handshake-extension-msg}})
  and Key Material Exchange ({{key-material-exchange}}).
  Each extension can have a pair of request and response types.

 | Value   | Extension Type       | HS Extension Flag |
 | ------- | :------------------: | :---------------: |
 |       1 | SRT_CMD_HSREQ        | HSREQ             |
 |       2 | SRT_CMD_HSRSP        | HSREQ             |
 |       3 | SRT_CMD_KMREQ        | KMREQ             |
 |       4 | SRT_CMD_KMRSP        | KMREQ             |
 |       5 | SRT_CMD_SID          | CONFIG            |
 |       6 | SRT_CMD_CONGESTION   | CONFIG            |
 |       7 | SRT_CMD_FILTER       | CONFIG            |
 |       8 | SRT_CMD_GROUP        | CONFIG            |
{: #handshake-ext-type title="Handshake Extension Type values"}

Extension Length (16 bits):
: The length of the Extension Contents field in four-byte blocks.

Extension Contents (variable length):
: The payload of the extension.

#### Handshake Extension Message {#handshake-extension-msg}

In a Handshake Extension, the value of the Extension Field of the
handshake control packet is defined as 1 for a Handshake Extension request (SRT_CMD_HSREQ in {{handshake-ext-type}}),
and 2 for a Handshake Extension response (SRT_CMD_HSRSP in {{handshake-ext-type}}).

The Extension Contents field of a Handshake Extension Message is structured as follows:

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
: SRT library version MUST be formed as major * 0x10000 + minor * 0x100 + patch.

SRT Flags (32 bits):
: SRT configuration flags (see {{hs-ext-msg-flags}}).

Receiver TSBPD Delay (16 bits):
: TimeStamp-Based Packet Delivery (TSBPD) Delay of the receiver. Refer to {{tsbpd}}.

Sender TSBPD Delay (16 bits):
: TSBPD of the sender. Refer to {{tsbpd}}.

##### Handshake Extension Message Flags {#hs-ext-msg-flags}

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
{: #hs-ext-msg-flags-tbl title="Handshake Extension Message Flags"}

- TSBPDSND flag defines if the TSBPD mechanism ({{tsbpd}}) will be used for sending.

- TSBPDRCV flag defines if the TSBPD mechanism ({{tsbpd}}) will be used for receiving. 

- CRYPT flag MUST be set. It is a legacy flag that indicates the party understands 
KK field of the SRT Packet ({{srtdatapacket}}).

- TLPKTDROP flag should be set if too-late packet drop mechanism will be used during transmission.
See {{too-late-packet-drop}}.

- PERIODICNAK flag set indicates the peer will send periodic NAK packets. See {{packet-naks}}.

- REXMITFLG flag MUST be set. It is a legacy flag that indicates the peer understands the R field
of the SRT DATA Packet ({{srtdatapacket}}).

- STREAM flag identifies the transmission mode ({{data-transmission-mode}}) to be used in the connection.
If the flag is set the buffer mode ({{transmission-mode-buffer}}) will be used.
Otherwise, message mode ({{transmission-mode-msg}}) is to be used.

- PACKET_FILTER flag indicates if the peer supports packet filter.

#### Key Material Exchange {#key-material-exchange}

The Key Material Exchange portion of a Handshake packet has both request and response type 
extensions. The value of a request is 3, and the response value is 4.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|S|  V  |   PT  |              Sign             |    Resv   | KK|
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

S ( ): 1 bit. Value: {0}  
: This is a fixed-width field that is a remnant from the header of a previous design.

Version (V): 3 bits. Value: {1}  
: This is a fixed-width field that indicates the SRT version:
  - 1: initial version

Packet Type (PT): 4 bits. Value: {2}  
: This is a fixed-width field that indicates the Packet Type:

  - 0: Reserved
  - 1: Media Stream Message (MSmsg)
  - 2: Keying Material Message (KMmsg)
  - 7: Reserved to discriminate MPEG-TS packet (0x47=sync byte)

Signature (Sign): 16 bits. Value: {0x2029}  
: This is a fixed-width field that contains the signature ‘HAI‘ encoded as a 
  PnP Vendor ID ({{PNPID}}) (in big-endian order)

Reserved (Resv): 6 bits. Value: {0}  
: This is a fixed-width field reserved for flag extension or other usage.

Key-based Data Encryption (KK): 2 bits.
: This is a fixed-width field that indicates whether or not data is encrypted:

  - 00b: not encrypted (data packets only)
  - 01b: even key
  - 10b: odd key
  - 11b: even and odd keys

Key Encryption Key Index (KEKI): 32 bits. Value: {0}
: This is a fixed-width field for specifying the KEK index (big-endian order)

  - 0: Default stream associated key (stream/system default)
  - 1..255: Reserved for manually indexed keys

Cipher ( ): 8 bits. Value: {0..2}
: This is a fixed-width field for specifying encryption cipher and mode:

  - 0: None or KEKI indexed crypto context
  - 1: AES-ECB (not supported in SRT)
  - 2: AES-CTR {{SP800-38A}}

Authentication (Auth): 8 bits. Value: {0}  
: This is a fixed-width field for specifying a message authentication code algorithm:

  - 0: None or KEKI indexed crypto context

Stream Encapsulation (SE): 8 bits. Value: {2}  
: This is a fixed-width field for describing the stream encapsulation:

  - 0: Unspecified or KEKI indexed crypto context
  - 1: MPEG-TS/UDP
  - 2: MPEG-TS/SRT

Reserved (Resv1): 8 bits. Value: {0}  
: This is a fixed-width field reserved for future use.

Reserved (Resv2): 16 bits. Value: {0}  
: This is a fixed-width field reserved for future use.

Slen/4 ( ): 4 bits. Value: {0..255}  
: This is a fixed-width field for specifying salt length in bytes divided by 4.
  Can be zero if no salt/IV present

Klen/4 ( ): 8 bits. Value: {4,6,8}  
: This is a fixed-width field for specifying SEK length in bytes divided by 4.
  Size of one key even if two keys present.

Salt (Slen): Slen*8 bits. Value: { }  
: This is a variable-width field for specifying a salt key 

Wrap ( ): (64+n * Klen * 8) bits. Value: { }
: This is a variable-width field for specifying Wrapped key(s), where n = 1 or 2

  NOTE 1: n = (KK + 1)/2
  NOTE 2: size in bytes = (((KK+1/2) * Klen) + 8)

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                  Integrity Check Vector (ICV)                 +
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
  
### Keep-Alive {#ctrl-pkt-keepalive}

Keep-Alive control packets are sent after a certain timeout from the last time
any packet (Control or Data) was sent. The purpose of this control packet is to 
notify the peer to keep the connection open when no data exchange is taking place.

The default timeout for a Keep-Alive packet to be sent is 1 second.

An SRT Keep-Alive packet is formatted as follows:

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+- SRT Header +-+-+-+-+-+-+-+-+-+-+-+-+-+
|1|         Control Type        |            Reserved           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Type-specific Information                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Time Stamp                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Destination Socket ID                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+- CIF -+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           CIF (none)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #keepalive-structure title="Keep-Alive structure"}

Packet Type ( ): 1 bit. Value: 1
: The type value of a Keep-Alive control packet is "1".

Control Type ( ): 15 bits.  Value: KEEPALIVE{1}
: This is a fixed-width field used to indicate message type 

Reserved ( ): 16 bits.  Value: ???
: This is a fixed-width field reserved for future use. 

Type-specific Information: 
: This field is reserved for future definition.

Time Stamp (TS): 32 bits.  Value: ???
: This is a fixed-width field usually containing the time (in microseconds) when a 
   packet was sent, although the real interpretation may vary depending on the type.

Destination Socket ID (DestSockID): 32 bits.  Value: ???
: This is a fixed-width field providing the socket ID to which a packet should be 
   dispatched, although it may have the special value 0 when the packet is a 
   connection request.
   
Control Information Field (CIF): n bits. Value: {none}  
: This field must not appear in Keep-Alive control packets.

### ACK (Acknowledgment) {#ctrl-pkt-ack}

Acknowledgment control packets are used to provide delivery status of data packets.
These packets may also carry some additional information from the receiver like
RTT, bandwidth, receiving speed, etc. The CIF portion of the ACK control packet is 
expanded as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+- SRT Header +-+-+-+-+-+-+-+-+-+-+-+-+-+
|1|        Control Type         |           Reserved            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Type-specific Information                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Time Stamp                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Destination Socket ID                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+- CIF -+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
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
: RTT value (in microseconds) estimated by the receiver based on the previous ACK-ACKACK
packet exchange.

RTT variance (32 bits):
: The variance of the RTT estimation (in microseconds).

Available Buffer Size (32 bits):
: Available size of the receiver's buffer (in packets).

Packets Receiving Rate (32 bits):
: The rate at which packets are being received (in packets per second).

Estimated Link Capacity (32 bits):
: Estimated bandwidth of the link (in packets per second).

Receiving Rate (32 bits):
: Estimated receiving rate (in bytes per second).

There are several types of ACK packets:

- A Full ACK control packet is sent every 10 ms and has all the fields 
  of {{ack-control-packet}}.
- A Lite ACK control packet includes only the Last Acknowledged Packet Sequence Number 
  field. The Type-specific Information field should be set to 0.
- A Small ACK includes the fields up to and including the Available Buffer Size field. 
  The Type-specific Information field should be set to 0.

The sender only acknowledges the receipt of Full ACK packets (see ACKACK Section {{ctrl-pkt-ackack}}).

The Lite ACK and Small ACK packets are used in cases when the receiver should acknowledge
received data packets more often than every 10 ms. This is usually needed at high data rates.
It is up to the receiver to decide the condition and the type of ACK packet to send (Lite or Small).
The recommendation is to send a Lite ACK for every 64 packets received.

### NAK (Loss Report) {#ctrl-pkt-nak}

Negative acknowledgment (NAK) control packets are used to signal failed data packet 
deliveries. The receiver notifies the sender about lost data packets by sending a NAK 
packet that contains a list of sequence numbers for those lost packets.

An SRT NAK packet is formatted as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+- SRT Header +-+-+-+-+-+-+-+-+-+-+-+-+-+
|1|        Control Type         |           Reserved            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Type-specific Information                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Time Stamp                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Destination Socket ID                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|                 Lost packet sequence number                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1|                    List of lost packets                     |
+-+-+-+-+-+-+-+-+-+-+-+- CIF (Loss List) -+-+-+-+-+-+-+-+-+-+-+-+ 
|0|                            Up to                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|                 Lost packet sequence number                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #nak-control-packet title="NAK control packet"}

Control Type: 
: The type value of a NAK control packet is "3".

Type-specific Information: 
: This field is reserved for future definition.

Control Information Field (CIF):
: A single value or a list of lost packets sequence numbers. See packet sequence number 
coding in {{packet-seq-list-coding}}.

### Shutdown {#ctrl-pkt-shutdown}

Shutdown control packets are used to initiate the closing of an SRT connection.

An SRT SHUTDOWN Control packet is formatted as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+- SRT Header +-+-+-+-+-+-+-+-+-+-+-+-+-+
|1|        Control Type         |           Reserved            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Type-specific Information                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Time Stamp                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Destination Socket ID                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+- CIF -+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              None                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #shutdown-control-packet title="SHUTDOWN control packet"}

Control Type: 
: The type value of Shutdown control packet is "5".

Type-specific Information: 
: This field is reserved for future definition.

Control Information Field:
: This field must not appear in shutdown control packets.

### ACKACK {#ctrl-pkt-ackack}

ACKACK control packets are sent to acknowledge the reception of a Full ACK, and are used 
in the calculation of RTT by the receiver.

An SRT ACKACK Control packet is formatted as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+- SRT Header +-+-+-+-+-+-+-+-+-+-+-+-+-+
|1|        Control Type         |           Reserved            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Type-specific Information                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Time Stamp                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Destination Socket ID                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+- CIF -+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              None                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #ackack-control-packet title="ACKACK control packet"}

Control Type:
: The type value of ACKACK control packet is "6".

Type-specific Information: 
: ACK Sequence Number. This field is used for the sequence number of
  the ACK packet being acknowledged.

Control Information Field:
: This field must not appear in ACKACK control packets.

# SRT Data Transmission and Control

This section describes key concepts related to the handling of control
and data packets during the transmission process.

After the handshake and exchange of capabilities is completed, packet data 
can be sent and received over the established connection. To fully utilize 
the features of low latency and error recovery provided by SRT, the sender 
and receiver MUST handle control packets, timers, and buffers for the connection
as specified in this section.

## Stream Multiplexing

Multiple SRT sockets may share the same UDP socket so that the packets
received to this UDP socket will be correctly dispatched to those
SRT sockets they are currently destined.

During the handshake, the parties exchange their SRT Socket IDs.
These IDs are then used in the Destination Socket ID field of
every control and data packet (see {{packet-structure}}).

## Data Transmission Modes {#data-transmission-mode}

SRT has been mainly created for Live Streaming and therefore its main and
default transmission mode is "live". SRT supports, however, the modes that
the original UDT library supported, that is, buffer and message transmission.

### Message Mode {#transmission-mode-msg}

When the STREAM flag of the handshake Extension Message
{{handshake-extension-msg}} is set to 0, the protocol operates
in Message mode, characterized as follows:

- Every packet has its own Packet Sequence Number.
- One or several consecutive SRT Data packets can form a message.
- All the packets belonging to the same message have a similar message number set
      in the Message Number field.

The first packet of a message has the first bit of the Packet Position Flags ({{data-pkt}})
set to 1. The last packet of the message has the second bit of the Packet Position Flags
set to 1. Thus, a PP equal to "11b" indicates a packet that forms the whole message.
A PP equal to "00b" indicates a packet that belongs to the inner part of the message.

The concept of the message in SRT comes from UDT ({{GHG04b}}). In this mode a single 
sending instruction passes exactly one piece of data that has boundaries (a message). 
This message may span across multiple UDP packets (and multiple SRT data packets). The 
only size limitation is that it shall fit as a whole in the buffers of the sender and the 
receiver. Although internally all operations (e.g. ACK, NAK) on data packets are performed 
independently, an application MUST send and receive the whole message. Until the message 
is complete (all packets are received) the application will not be allowed to read it.

When the Order Flag of a Data packet is set to 1, this imposes a sequential reading order
on messages. An Order Flag set to 0 allows an application to read messages that are 
already fully available, before any preceding messages that may have some packets missing.

### Live Mode {#transmission-mode-live}

Live mode is a special type of message mode where only data packets
with their PP field set to "11b" are allowed.

Additionally Timestamp Based Packet Delivery (TSBPD) ({{tsbpd}}) and
Too-Late Packet Drop ({{too-late-packet-drop}}) mechanisms are used in this mode.

### Buffer Mode {#transmission-mode-buffer}

Buffer mode is negotiated during the Handshake by setting the STREAM flag
of the handshake Extension Message Flags to 1.

In this mode consecutive packets form one continuous stream that can be read, with 
portions of any size.

## Handshake Messages {#handshake-messages}

SRT is a connection-oriented protocol. It embraces the concepts of "connection"
and "session". The UDP system protocol is used by SRT for sending data
and control packets.

An SRT connection is characterized by the fact that it is:

- first engaged by a handshake process;

- maintained as long as any packets are being exchanged in a timely manner;

- considered closed when a party receives the appropriate close command from
  its peer (connection closed by the foreign host), or when it receives no
  packets at all for some predefined time (connection broken on timeout).

SRT supports two connection configurations:

1. Caller-Listener, where one side waits for the other to initiate a connection
2. Rendezvous, where both sides attempt to initiate a connection

The handshake is performed between two parties: "Initiator" and "Responder":

- Initiator starts the extended SRT handshake process and sends appropriate
  SRT extended handshake requests.

- Responder expects the SRT extended handshake requests to be sent by the
  Initiator and sends SRT extended handshake responses back.

There are two basic types of SRT handshake extensions that are exchanged
in the handshake:

- Handshake Extension Message exchanges the basic SRT information;
- Key Material Exchange exchanges the wrapped stream encryption key (used only if
  encryption is requested).
- Stream ID extension exchanges some stream-specific information that can be used
  by the application to identify the incoming stream connection.

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

1. After starting the connection: WAVEAHAND.
2. After receiving the above message from the peer: CONCLUSION.
3. After receiving the above message from the peer: AGREEMENT.

When a connection process has failed before either party can send the CONCLUSION handshake, 
the Handshake Type field will contain the appropriate error value for the rejected 
connection. See the list of error codes in {{hs-rej-reason}}.

 | Code | Error            | Description                             |
 | ---- | ---------------- | --------------------------------------- |
 | 1000 | REJ_UNKNOWN      | Unknown reason                          |
 | 1001 | REJ_SYSTEM       | System function error                   |
 | 1002 | REJ_PEER         | Rejected by peer                        |
 | 1003 | REJ_RESOURCE     | Resource allocation problem             |
 | 1004 | REJ_ROGUE        | incorrect data in handshake             |
 | 1005 | REJ_BACKLOG      | listener's backlog exceeded             |
 | 1006 | REJ_IPE          | internal program error                  |
 | 1007 | REJ_CLOSE        | socket is closing                       |
 | 1008 | REJ_VERSION      | peer is older version than agent's min  |
 | 1009 | REJ_RDVCOOKIE    | rendezvous cookie collision             |
 | 1010 | REJ_BADSECRET    | wrong password                          |
 | 1011 | REJ_UNSECURE     | password required or unexpected         |
 | 1012 | REJ_MESSAGEAPI   | Stream flag collision                   |
 | 1013 | REJ_CONGESTION   | incompatible congestion-controller type |
 | 1014 | REJ_FILTER       | incompatible packet filter              |
 | 1015 | REJ_GROUP        | incompatible group                      |
{: #hs-rej-reason title="Handshake Rejection Reason Codes"}

The specification of the cipher family and block size is decided by the data Sender.
When the transmission is bidirectional, this value must be agreed upon at the outset
because when both are set the Responder wins. For Caller-Listener connections it is
reasonable to set this value on the Listener only. In the case of Rendezvous the only
reasonable approach is to decide upon the correct value from the different sources and to 
set it on both parties (note that **AES-128** is the default).

### Caller-Listener Handshake {#caller-listener-handshake}

This section describes the handshaking process where a Listener is
waiting for an incoming Handshake request on a bound UDP port from a Caller.
The process has two phases: induction and conclusion.

#### The Induction Phase

The INDUCTION phase serves only to set a cookie on the Listener so that it
doesn't allocate resources, thus mitigating a potential DoS attack that might be
perpetrated by flooding the Listener with handshake commands.

The Caller begins by sending the INDUCTION handshake, which contains the following
(significant) fields:

- Version: must always be 4
- Encryption Field: 0
- Extension Field: 2
- Handshake Type: INDUCTION
- SRT Socket ID: SRT Socket ID of the Caller
- SYN Cookie: 0

The Destination Socket ID of the SRT packet header in this message is 0, which is
interpreted as a connection request.

The handshake version number is set to 4 in this initial handshake.
This is due to the initial design of SRT that was to be compliant with the UDT
protocol ({{GHG04b}}) on which it is based.

The Listener responds with the following:

- Version: 5
- Encryption Field: Advertised cipher family and block size.
- Extension Field: SRT magic code 0x4A17
- Handshake Type: INDUCTION
- SRT Socket ID: Socket ID of the Listener
- SYN Cookie: a cookie that is crafted based on host, port and current time
  with 1 minute accuracy to avoid SYN flooding attack {{RFC4987}}

At this point the Listener still does not know if the Caller is SRT or UDT,
and it responds with the same set of values regardless of whether the Caller is
SRT or UDT.

If the party is SRT, it does interpret the values in Version and Extension Field.
If it receives the value 5 in Version, it understands that it comes from an SRT
party, so it knows that it should prepare the proper handshake messages
phase. It also checks the following:

- whether the Extension Flags contains the magic value 0x4A17; otherwise the
  connection is rejected. This is a contingency for the case where someone who,
  in an attempt to extend UDT independently, increases the Version value to 5
  and tries to test it against SRT.

- whether the Encryption Flags contain a non-zero
  value, which is interpreted as an advertised cipher family and block size.

A legacy UDT party completely ignores the values reported in Version and Handshake Type.  
It is, however, interested in the SYN Cookie value, as this must be passed to the next 
phase. It does interpret these fields, but only in the "conclusion" message.

#### The Conclusion Phase

Once the Caller gets the SYN cookie from the Listener, it sends the CONCLUSION handshake
to the Listener.

The following values are set by the compliant caller:

- Version: 5
- Handshake Type: CONCLUSION
- SRT Socket ID: Socket ID of the Caller
- SYN Cookie: the cookie previously received in the induction phase

The Destination Socket ID in this message is the
socket ID that was previously received in the induction phase in the SRT Socket ID field
of the handshake structure.

- Encryption Flags: advertised cipher family and block size.
- Extension Flags: A set of flags that define the extensions provided in the handshake.

The Listener responds with the same values shown above, without the cookie (which
is not needed here), as well as the extensions for HS Version 5 (which will probably be
exactly the same).

There is not any "negotiation" here. If the values passed in the
handshake are in any way not acceptable by the other side, the connection will
be rejected. The only case when the Listener can have precedence over the Caller
is the advertised Cipher Family and Block Size ({{handshake-encr-fld}})
in the Encryption Field of the Handshake.

The value for latency is always agreed to be the greater of those reported
by each party.

### Rendezvous Handshake

The Rendezvous process uses a state machine. It is slightly
different from UDT Rendezvous handshake {{GHG04b}},
although it is still based on the same message request types.

Both parties start with WAVEAHAND and use the Version value of 5.
Legacy Version 4 clients do not look at the Version value,
whereas Version 5 clients can detect version 5.
The parties only continue with the Version 5 Rendezvous process when Version is set to 5
for both. Otherwise the process continues exclusively according to Version 4 rules {{GHG04b}}.

With Version 5 Rendezvous, both parties create a cookie for a process called the
"cookie contest". This is necessary for the assignment of Initiator and Responder
roles. Each party generates a cookie value (a 32-bit number) based on the host,
port, and current time with 1 minute accuracy. This value is scrambled using
an MD5 sum calculation. The cookie values are then compared with one another.

Since it is impossible to have two sockets on the same machine bound to the same NIC
and port and operating independently, it is virtually impossible that the
parties will generate identical cookies. However, this situation may occur if an
application tries to "connect to itself" - that is, either connects to a local
IP address, when the socket is bound to INADDR_ANY, or to the same IP address to
which the socket was bound. If the cookies are identical (for any reason), the
connection will not be made until new, unique cookies are generated (after a
delay of up to one minute). In the case of an application "connecting to itself",
the cookies will always be identical, and so the connection will never be established.

When one party's cookie value is greater than its peer's, it wins the cookie
contest and becomes Initiator (the other party becomes the Responder).

At this point there are two possible "handshake flows":
serial and parallel.

#### Serial Handshake Flow

In the serial handshake flow, one party is always first, and the other follows.
That is, while both parties are repeatedly sending WAVEAHAND messages, at
some point one party - let's say Alice - will find she has received a
WAVEAHAND message before she can send her next one, so she sends a
CONCLUSION message in response. Meantime, Bob (Alice's peer) has missed
Alice's WAVEAHAND messages, so that Alice's CONCLUSION is the first message
Bob has received from her.

This process can be described easily as a series of exchanges between the first
and following parties (Alice and Bob, respectively):

1. Initially, both parties are in the waving state. Alice sends a handshake
   message to Bob:
   - Version: 5
   - Type: Extension field: 0, Encryption field: advertised "PBKEYLEN".
   - Handshake Type: WAVEAHAND
   - SRT Socket ID: Alice's socket ID
   - SYN Cookie: Created based on host/port and current time.

While Alice does not yet know if she is sending this message to
a Version 4 or Version 5 peer, the values from these fields would not be interpreted by
the Version 4 peer when the Handshake Type is WAVEAHAND.

2. Bob receives Alice's WAVEAHAND message, switches to the "attention"
   state. Since Bob now knows Alice's cookie, he performs a "cookie contest"
   (compares both cookie values). If Bob's cookie is greater than Alice's, he will
   become the Initiator. Otherwise, he will become the Responder.

The resolution of the Handshake Role
(Initiator or Responder) is essential for further processing.

Then Bob responds:

- Version: 5
- Extension field: appropriate flags if Initiator, otherwise 0
- Encryption field: advertised PBKEYLEN
- Handshake Type: CONCLUSION

If Bob is the Initiator and encryption is on, he will use either his
own cipher family and block size or the one received from Alice (if she has advertised
those values).

3. Alice receives Bob's CONCLUSION message. While at this point she also
   performs the "cookie contest", the outcome will be the same. She switches to the
   "fine" state, and sends:
   - Version: 5
   - Appropriate extension flags and encryption flags
   - Handshake Type: CONCLUSION

Both parties always send extension flags at this point, which will
contain HSREQ if the message comes from an Initiator, or
HSRSP if it comes from a Responder. If the Initiator has received a
previous message from the Responder containing an advertised cipher family and block size in the
encryption flags field, it will be used as the key length
for key generation sent next in the KMREQ extension.

4. Bob receives Alice's CONCLUSION message, and then does one of the
   following (depending on Bob's role):
   - If Bob is the Initiator (Alice's message contains HSRSP), he:
     - switches to the "connected" state
     - sends Alice a message with Handshake Type AGREEMENT, but containing
       no SRT extensions (Extension Flags field should be 0)

   - If Bob is the Responder (Alice's message contains HSREQ), he:
     - switches to "initiated" state
     - sends Alice a message with Handshake Type CONCLUSION that also contains
       extensions with HSRSP
        - awaits a confirmation from Alice that she is also connected (preferably
          by AGREEMENT message)

5. Alice receives the above message, enters into the "connected" state, and
   then does one of the following (depending on Alice's role):
    - If Alice is the Initiator (received CONCLUSION with HSRSP),
      she sends Bob a message with Handshake Type = AGREEMENT.
    - If Alice is the Responder, the received message has Handshake Type AGREEMENT
      and in response she does nothing.

6. At this point, if Bob was Initiator, he is connected already. If he was a
   Responder, he should receive the above AGREEMENT message, after which he
   switches to the "connected" state. In the case where the UDP packet with the
   agreement message gets lost, Bob will still enter the "connected" state once
   he receives anything else from Alice. If Bob is going to send, however, he
   has to continue sending the same CONCLUSION until he gets the confirmation
   from Alice.

#### Parallel Handshake Flow

The chances of the parallel handshake flow are very low, but still it may
occur if the handshake messages with WAVEAHAND are sent and received by both peers
at precisely the same time.

The resulting flow is very much like Bob's behavior in the serial handshake flow,
but for both parties. Alice and Bob will go through the same state transitions:

    Waving -> Attention -> Initiated -> Connected

In the Attention state they know each other's cookies, so they can assign
roles. In contrast to serial flows,
which are mostly based on request-response cycles, here everything
happens completely asynchronously: the state switches upon reception
of a particular handshake message with appropriate contents (the
Initiator must attach the HSREQ extension, and Responder must attach the
`HSRSP` extension).

Here's how the parallel handshake flow works, based on roles:

Initiator:

1. Waving
   - Receives WAVEAHAND message
   - Switches to Attention
   - Sends CONCLUSION + HSREQ
2. Attention
   - Receives CONCLUSION message, which:
     - contains no extensions:
       - switches to Initiated, still sends CONCLUSION + HSREQ
     - contains `HSRSP` extension:
       - switches to Connected, sends AGREEMENT
3. Initiated
   - Receives CONCLUSION message, which:
     - Contains no extensions:
       - REMAINS IN THIS STATE, still sends CONCLUSION + HSREQ
     - contains `HSRSP` extension:
       - switches to Connected, sends AGREEMENT
4. Connected
   - May receive CONCLUSION and respond with AGREEMENT, but normally
     by now it should already have received payload packets.

Responder:

1. Waving
   - Receives WAVEAHAND message
   - Switches to Attention
   - Sends CONCLUSION message (with no extensions)
2. Attention
   - Receives CONCLUSION message with HSREQ.
     This message might contain no extensions, in which case the party 
     shall simply send the empty CONCLUSION message, as before, and remain 
     in this state.
   - Switches to Initiated and sends CONCLUSION message with HSRSP
3. Initiated
   - Receives:
     - CONCLUSION message with HSREQ
       - responds with CONCLUSION with HSRSP and remains in this state
     - AGREEMENT message
       - responds with AGREEMENT and switches to Connected
     - Payload packet
       - responds with AGREEMENT and switches to Connected
4. Connected
   - Is not expecting to receive any handshake messages anymore. The
     AGREEMENT message is always sent only once or per every final
     CONCLUSION message.

Note that any of these packets may be missing, and the sending party will
never become aware. The missing packet problem is resolved this way:

1. If the Responder misses the CONCLUSION + HSREQ message, it simply
   continues sending empty CONCLUSION messages. Only upon reception of
   CONCLUSION + HSREQ does it respond with CONCLUSION + HSRSP.

2. If the Initiator misses the CONCLUSION + HSRSP response from the
   Responder, it continues sending CONCLUSION + HSREQ. The Responder must
   always respond with CONCLUSION + HSRSP when the Initiator sends
   CONCLUSION + HSREQ, even if it has already received and interpreted it.

3. When the Initiator switches to the Connected state it responds with a
   AGREEMENT message, which may be missed by the Responder. Nonetheless, the
   Initiator may start sending data packets because it considers itself
   connected - it does not know that the Responder has not yet switched
   to the Connected state. Therefore it is exceptionally allowed that when
   the Responder is in the Initiated state and receives a data packet
   (or any control packet that is normally sent only between connected
   parties) over this connection, it may switch to the Connected
   state just as if it had received a AGREEMENT message.

4. If the the Initiator has already switched to the Connected state it will not
   bother the Responder with any more handshake messages. But the Responder may be
   completely unaware of that (having missed the AGREEMENT message from the
   Initiator). Therefore it does not exit the connecting state, which means that it
   continues sending CONCLUSION + HSRSP messages until it receives any
   packet that will make it switch to the Connected state (normally
   AGREEMENT). Only then does it exit the connecting state and the
   application can start transmission.

## SRT Buffer Latency {#srt-latency}

The SRT sender and receiver have buffers to store packets.

On the sender, latency is the time that SRT holds a packet to give it a chance to be
delivered successfully while maintaining the rate of the sender at the receiver. If an 
acknowledgment (ACK) is missing or late for more than the configured latency, the packet 
is dropped from the sender buffer. A packet can be retransmitted as long as it remains
in the buffer for the duration of the latency window. On the receiver, packets are 
delivered to an application from a buffer after the latency interval has passed. This 
helps to recover from potential packet losses. See sections {{tsbpd}}, 
{{too-late-packet-drop}} for details.

Latency is a value (specified in milliseconds) that can cover the time to transmit 
hundreds or even thousands of packets at high bitrate. Latency can be thought of as a 
window that slides over time, during which a number of activities take place, such as 
the reporting of acknowledged packets (ACKs) ({{packet-acks}}) and unacknowledged 
packets (NAKs)({{packet-naks}}).

Latency is configured through the exchange of capabilities during the extended handshake 
process between initiator and responder. The Handshake Extension Message ({{handshake-extension-msg}}) 
has TSBPD delay information (in milliseconds) from the SRT receiver and sender. The 
latency for a connection will be established as the maximum value of latencies proposed 
by the initiator and responder.

## Timestamp Based Packet Delivery {#tsbpd}

The goal of the SRT Timestamp Based Packet Delivery (TSBPD) mechanism is to reproduce 
the output of the sending application (e.g., encoder) at the input of the receiving 
application (e.g., decoder) in live data transmission mode (see {{data-transmission-mode}}). 
It attempts to reproduce the timing of packets committed by the sending application to 
the SRT sender. This allows packets to be scheduled for delivery by the SRT receiver, 
making them ready to be read by the receiving application (see {{fig-latency-points}}).

The SRT receiver, using the timestamp of the SRT data packet header, delivers packets to 
a receiving application with a fixed minimum delay from the time the packet was scheduled 
for sending on the SRT sender side. Basically, the sender timestamp in the received packet
is adjusted to the receiver’s local time (compensating for the time drift or different 
time zones) before releasing the packet to the application. Packets can be withheld by 
the SRT receiver for a configured receiver delay. A higher delay can accommodate a larger 
uniform packet drop rate, or a larger packet burst drop. Packets received after their 
"play time" are dropped if the Too-Late Packet Drop feature is enabled 
(see {{too-late-packet-drop}}).

The packet timestamp (in microseconds) is relative to the SRT connection creation time. 
Packets are inserted based on the sequence number in the header field. The origin time 
(in microseconds) of the packet is already sampled when a packet is first submitted by 
the application to the SRT sender. The TSBPD feature uses this time to stamp the packet 
for first transmission and any subsequent retransmission. This timestamp and the 
configured SRT latency ({{srt-latency}}) control the recovery buffer size and the 
instant that packets are delivered at the destination (the aforementioned "play time" 
which is decided by adding the timestamp to the configured latency).

It is worth mentioning that the use of the packet sending time to stamp the packets is
inappropriate for the TSBPD feature, since a new time (current sending time) is used for 
retransmitted packets, putting them out of order when inserted at their proper place 
in the stream. 

{{fig-latency-points}} illustrates the key latency points during the packet transmission 
with the TSBPD feature enabled.

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

The main packet states shown in {{fig-latency-points}} are the following:

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

It is worth noting that TsbpdDelay limits the number of packet retransmissions to a certain extent
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
of a packet and ends once the packet with timestamp within (30, 60) seconds interval
is delivered (read from the buffer). The updated value of TsbpdTimeBase will be recalculated as follows:

~~~
TsbpdTimeBase = TsbpdTimeBase + MAX_TIMESTAMP + 1
~~~

2. By drift tracer. See {{drift-management}} for details.


## Too-Late Packet Drop {#too-late-packet-drop}

The Too-Late Packet Drop (TLPKTDROP) mechanism allows the sender to drop packets that 
have no chance to be delivered in time, and allows the receiver to skip missing packets 
that have not been delivered in time. The timeout of dropping a packet is based on the 
TSBPD mechanism (see {{tsbpd}}).

In the SRT, when Too-Late Packet Drop is enabled, and a packet timestamp is older than 
125% of the SRT latency, it is considered too late to be delivered and may be dropped
by the sender. However, the sender keeps packets for at least 1 second in case the
SRT latency is not enough for a large RTT (that is, if 125% of the SRT latency is less 
than 1 second).

When enabled on the receiver, the receiver drops packets that have not been delivered 
or retransmitted in time, and delivers the subsequent packets to the application when 
it is their time to play.

In pseudo-code, the algorithm of reading from the receiver buffer is
the following:

    <CODE BEGINS>
    pos = 0;  /* Current receiver buffer position */
    i = 0;    /* Position of the next available in the receiver buffer 
                 packet relatively to the current buffer position pos */

    while(True) {
        // Get the position i of the next available packet
        // in the receiver buffer
        i = next_avail();
        // Calculate packet delivery time PktTsbpdTime
        // for the next available packet
        PktTsbpdTime = delivery_time(i);

        if T_NOW < PktTsbpdTime:
            continue;

        Drop packets which buffer position number is less than i;

        Deliver packet with the buffer position i;

        pos = i + 1;
    }
    <CODE ENDS>

where T_NOW is the current time according to the receiver clock.

The TLPKTDROP mechanism can be turned off to always ensure a clean
delivery. However, a lost packet can simply pause a delivery for some
longer, potentially undefined time, and cause even worse tearing
for the player. Setting higher SRT latency will help much more in the
case when TLPKTDROP causes packet drops too often.


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
packet is received it is not given right away to the application.
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
determine, for example, if it is time to deliver a certain packet
right away. The timestamps in the packets themselves are just
references to the beginning of the session. When a packet is received
(with a timestamp from the sender), the receiver makes a reference to
the beginning of the session to recalculate its timestamp. The start
time is derived from the local time at the moment that the session is
connected. A packet timestamp equals "now" minus "StartTime", where
the latter is the point in time when the socket was created.

## Acknowledgement and Lost Packet Handling

To enable the Automatic Repeat reQuest of data packet retransmissions, a sender stores
all sent data packets in its buffer. 

The SRT receiver periodically sends acknowledgments (ACKs) for the
received data packets so that the SRT sender can remove the
acknowledged packets from its buffer ({{packet-acks}}). Once the acknowledged packets are
removed, their retransmission is no longer possible and presumably not needed.

Upon receiving the full acknowledgment (ACK) control packet, the SRT sender should acknowledge
its reception to the receiver by sending an ACKACK control packet with the sequence number
of the full ACK packet being acknowledged.

The SRT receiver also sends NAK control packets to notify the sender about the missing packets ({{packet-naks}}).
The sending of a NAK packet can be triggered immediately after a gap in sequence numbers of
data packets is detected. In addition, a Periodic NAK report mechanism can be used to 
send NAK reports periodically. The NAK packet in that case will list all the packets that 
the receiver considers being lost up to the moment the Periodic NAK report is sent.

Upon reception of the NAK packet, the SRT sender prioritizes retransmissions of lost packets over the regular data
packets to be transmitted for the first time.

The retransmission of the missing packet is repeated until the receiver acknowledges its receipt,
or if both peers agree to drop this packet (see {{too-late-packet-drop}}).

### Packet Acknowledgement (ACKs, ACKACKs) {#packet-acks}

At certain intervals (see below), the SRT receiver sends an acknowledgment (ACK) that
causes the acknowledged packets to be removed from the SRT sender's buffer.

An ACK control packet contains the sequence number of the packet immediately
following the latest in the list of received packets. Where no packet loss has
occurred up to the packet with sequence number n, an ACK would include the sequence number (n + 1).

An ACK (from a receiver) will trigger the transmission of an ACKACK (by the sender), with almost no delay.
The time it takes for an ACK to be sent and an ACKACK to be received is the RTT.
The ACKACK tells the receiver to stop sending the ACK position because the sender already
knows it. Otherwise, ACKs (with outdated information) would continue to be sent regularly.
Similarly, if the sender does not receive an ACK, it does not stop transmitting.

There are two conditions for sending an acknowledgment. A full ACK is based on a timer of 10
milliseconds (the ACK period). For high bit rate transmissions, a "light ACK" can be sent, which is an ACK
for a sequence of packets. In a 10 milliseconds interval, there are often so many packets being sent and
received that the ACK position on the sender does not advance quickly enough. To mitigate this,
after 64 packets (even if the ACK period has not fully elapsed) the receiver sends a light ACK.
A light ACK is a shorter ACK (SRT header  and one 32-bit field). It does not trigger an ACKACK.


When a receiver encounters the situation where the next packet to be played was not
successfully received from the sender, it will "skip" this packet (see {{too-late-packet-drop}})
and send a fake ACK. To the sender, this fake ACK is a real ACK, and so it just behaves as if the packet had been received.
This facilitates the synchronization between SRT sender and receiver. The fact that a packet was
skipped remains unknown by the sender. Skipped packets are recorded in the statistics on the
SRT receiver.

### Packet Retransmission (NAKs) {#packet-naks}

The SRT receiver sends NAK control packets to notify the sender about the missing packets.
The NAK packet sending can be triggered immediately after a gap in sequence numbers of
data packets is detected. 

Upon reception of the NAK packet, the SRT sender prioritizes retransmissions of lost packets over the regular data
packets to be transmitted for the first time.

The SRT sender maintains a list of lost packets (loss list) that is built from NAK reports. When
scheduling packet transmission, it looks to see if a packet in the loss list has priority and sends it if so.
Otherwise, it sends the next packet scheduled for the first transmission list.
Note that when a packet is transmitted, it stays in the buffer in case it is not received by the SRT receiver.

NAK packets are processed to fill in the loss list. As the latency window advances and packets are
dropped from the sending queue, a check is performed to see if any of the dropped or resent
packets are in the loss list, to determine if they can be removed from there as well so that they
are not retransmitted unnecessarily.

There is a counter for the packets that are resent. If there is no ACK
for a packet, it will stay in the loss list and can be resent more than
once. Packets in the loss list are prioritized.

If packets in the loss list continue to block the send queue, at some point this will cause the
send queue to fill. When the send queue is full, the sender will begin to drop packets without
even sending them the first time. An encoder (or other application) may continue to provide
packets, but there's no place for them, so they will end up being thrown away.

This condition where packets are unsent does not happen often. There is a maximum number of
packets held in the send buffer based on the configured latency. Older packets that have no
chance to be retransmitted and played in time are dropped, making room for newer real-time
packets produced by the sending application. See sections {{tsbpd}}, {{too-late-packet-drop}} for details.

In addition to the regular NAKs, the Periodic NAK report mechanism can be used to send NAK reports periodically.
The NAK packet in that case will have all the packets that the receiver considers being lost
at the time of sending the Periodic NAK report.

An ACKACK tells the receiver to stop sending the ACK position because the sender already
knows it. Otherwise, ACKs (with outdated information) would continue to be sent regularly.

An ACK serves as a ping, with a corresponding ACKACK pong, to measure RTT. The time it
takes for an ACK to be sent and an ACKACK to be received is the RTT. Each ACK has a number.
A corresponding ACKACK has that same number. The receiver keeps a list of all ACKs in a
queue to match them. Unlike a full ACK, which contains the current RTT and several other
values in the CIF, a light ACK just contains the sequence number. All control messages
are sent directly and processed upon reception, but ACKACK processing time is negligible
(the time this takes is included in the round-trip time).

## Bidirectional Transmission Queues

Once an SRT connection is established, both peers can send data packets simultaneously.

## Round Trip Time Estimation

The round-trip time is estimated during the transmission of SRT data packets
based on the time difference between the ACK packet is sent and the
corresponding ACKACK is received by the data receiver.

## Congestion Control

SRT provides certain mechanisms for the sender to get some feedback
from the receiving side through the ACK packets ({{ctrl-pkt-ack}}).
Every 10 ms the sender receives the latest values of RTT and RTT variance,
Available Buffer Size, Packets Receiving Rate and Estimated Link Capacity.
Upon reception of the NAK packet ({{ctrl-pkt-nak}}) the sender can detect
packet losses during the transmission.
These mechanisms provide a solid background for various congestion control
algorithms.

Given that SRT can operate in live and file transfer modes, there are two possible groups
of congestion control algorithms.

### Live Congestion Control (LiveCC)

For live transmission mode ({{transmission-mode-live}}) the congestion control algorithm
does not need to control the sending pace of the data packets, as the sending timing
is provided by the live input source. It is mainly a reactor on network events. Certain
limitations on the minimal inter-sending time of consecutive packets can be applied in order
to avoid congestion during fluctuations of the source bitrate.
LiveCC determines the minimum time between consecutive packets are sent.
Also, it is allowed to drop those packets that can not be delivered in time.

On the receiving side, LiveCC may decide when an ACK is needed prior to ACK timeout.
It also determines the interval of periodic NAK packets to be reported to the sender.

### File Transfer Congestion Control (FileCC)

For file transfer, any known File Congestion Control algorithms, like CUBIC {{RFC8312}}, can apply,
including the congestion control mechanism proposed in UDT {{GHG04b}}.
The UDT congestion control relies on the available link capacity, packet loss reports (NAK)
and packet acknowledgments (ACKs).
It then slows down the output of packets as needed by adjusting the packet sending pace.
In periods of congestion, it can block the main stream and focus on the lost packets.

The algorithm consists of three states: slow start, congestion avoidance and slow down.
Starting with no information, the congestion control module probes the network to
determine the available bandwidth and the sending pace for the desired operation mode: congestion avoidance.
In this state, if there is no congestion detected via Loss reports, the sending pace is gradually increased.
On the other side, if congestion is detected in the network the state is changed to slow down and
the sending pace is decreased to reduce packet loss.

#### Slow Start

Slow start state has a limit on the number of data packets that can be sent before
acknowledgment of those segments is received. File CC starts with 16 DATA packets (CWND_SIZE=16)
sent with minimum possible interval.

CWND_SIZE is increased by the difference between the acknowledged sequence number since last ACK processing
and the sequence number being acknowledged:
 
~~~
CWND_SIZE += LAST_ACK_SEQNO - ACK_SEQNO
~~~

When the congestion window size exceeds its maximum set value (MAX_CWND_SIZE), slow start period ends. Note
that this congestion maximum window size is equal to the flow window size (FLOW_WND_SIZE). If a delivery
rate was reported by the receiver the inter sending interval is set as:

~~~
PKT_SND_PERIOD = 1000000.0 / DELIVERY_RATE
~~~

If delivery rate has not been reported, the sending period is set to:

~~~
PKT_SND_PERIOD = CWND_SIZE / (RTT + RC_INTERVAL)
~~~

#### Congestion Avoidance
At the end of the slow start state, the congestion avoidance state begins. If the estimated bandwidth (B) is
smaller than the actual sending rate (1/PKT_SND_PERIOD), the increase (inc) is equal to the inverse of MSS.

If not the increase is computed as:

~~~
inc = max(10 ^ ceil(log10( B * MSS * 8 ) * Beta / MSS, 1/MSS)
~~~

With beta being a constant set to 1.5 * 10^(-6). The variable inc is capped at 1/MSS.

The new sending period is then calculated as:

~~~
PKT_SND_PERIOD = (PKT_SND_PERIOD * RC_INTERVAL) / (PKT_SND_PERIOD * inc + RC_INTERVAL)
~~~

#### Slow Down

Upon the arrival of a NACK a congestion period is started if the lost packet sequence in the NACK
is bigger than the largest sequence number sent so far (LastDecSeq).

The decreasing sending rate is computed with the following variables:

* AvgNAKNum: The average number of NACKs. Initial value: 1.
* NAKCount: Number of NACKs received in the current congestion period. Initial value: 1.
* DecCount: Number of times the sending period has been increased in the former congestion period.
Initial value: 0.
* DecRandom: Uniformly distributed random number to avoid global synchronization. Ranges from 1 to AvgNAKNum.

When a new congestion period is started the previously enumerated variables are reset and the
packet sending period is increased as:

~~~
PKT_SND_PERIOD = PKT_SND_PERIOD * 1.125
~~~

If the following statement is true:

~~~
DecCount <= 5 and NAKCount == DecCount * DecRandom
~~~

The PKT_SND_PERIOD and DecCount are increased: 

~~~
PKT_SND_PERIOD = PKT_SND_PERIOD * 1.125
DecCount += 1 
~~~

# Encryption {#encryption}

This section describes the encryption mechanism that protects the payload of SRT streams.
Based on standard cryptographic algorithms, the mechanism allows an efficient stream cipher
with a key establishment method.

##Overview
AES in counter mode (AES-CTR) is used with a short-lived key to encrypt the media stream.
This cipher is suitable for random access of a continuous stream, content protection
(used by HDCP 2.0), and strong confidentiality when the counter is managed properly.
SRT implements encryption using AES-CTR mode, effectively using encryption to decrypt
an encrypted packet.

<!-- TODO: [[reference for AES-CTR, HDCP 2.0??]] -->

<!-- # TODO: [[Figure]] -->

SRT encrypts the media stream at the Transmission Payload level
(UDP payload of MPEG-TS/UDP encapsulation, which is about 7 MPEG-TS packets of 188 bytes each).
A small packet header is required to keep the synchronization of the cipher counter
between the encoder and the decoders. No padding is required by counter mode ciphers.

What is encrypted is a 128-bit counter, which is then used to do an XOR with each block
of 128 bits in a packet, resulting in the ciphertext to transmit (this is the XOR technique 
for counter mode).

The counter for AES-CTR is the size of the cipher's block, i.e. 128 bits. It is derived from
a 128-bit sequence consisting of (1) a block counter in the least significant 16 bits, 
which counts the blocks in a packet, (2) a packet index - based on the packet sequence number
in the UDT header - in the next 32 bits, and (3) eighty bits that are zeroes.
The upper 112 bits of this sequence are XORed with an initialization vector (IV)
to produce a unique counter for each crypto block. Note that the block counter is unique for 
the duration of the session. The same counter cannot be used twice.

This key used for encryption is called the "Stream Encrypting Key" (SEK), which is used for
2^25 packets.
The short-lived SEK is generated by the sender using a pseudo-random number generator (PRNG),
and transmitted within the stream (KM Tx Period), wrapped with another longer-term key,
the Key Encrypting Key (KEK), using a known AES key wrap protocol.

<!-- TODO: [[Figure]] -->

For connection-oriented transport such as SRT, there is no need to periodically transmit
the short-lived key since no party can join the stream at any time. The keying material
is transmitted within the connection handshake packets, and for a short period
when rekeying occurs.

The KEK is derived from a secret shared (e.g. passphrase) between the sender and the receiver.
The shared secret provides access to the stream key, which provides access to the protected
media stream. The distribution and management of the secret is more flexible than the stream
encrypting key. The KEK has to be at least as long as the SEK.

The KEK is generated by a password-based key generation function (PBKDF2)<!-- [PCKS5] -->,
which uses the passphrase, a number of iterations (2048), a keyed-hash (HMAC-SHA1), and
a length KEYLEN. The PBKDF2 function hashes the passphrase to make a long string,
by repetition or padding. The number of iterations is based on how much time can be given to
the process without it becoming disruptive.

<!-- TODO: PCKS5 reference, HMAC-SHA1 reference -->

The KEK is used to generate a wrap {{RFC3394}} that is put in a key material (KM) message
to send to the receiver. The KM message contains the key length,
the salt (one of the arguments provided to the PBKDF2 function), the protocol being used
(AES-256 in this case) and the AES counter (which will eventually change).

On the other side, the receiver attempts to decode the wrap to obtain the key. In the protocol
for the wrap there is a padding, which is a known template, so the receiver knows from the KM
that it has the right KEK to decode the SEK.
The SEK (generated and transmitted by the sender) is random, and cannot be known in advance.
The KEK formula is calculated on both sides, with the difference that the receiver gets
the key length (KEYLEN) from the sender via the key material (KM).
It is the sender who decides on the configured length. The receiver obtains it from
the material sent by the sender. This is why the length is not configured on a decoder,
thus avoiding the need to match.

The receiver returns the same KM message to show that it has the same information as the sender,
and that the encoded material will be decrypted. If the receiver does not return this status,
this means that any encrypted packets from the sender will be lost. Even if they are transmitted
successfully, the receiver will be unable to decrypt them, and so will be dropped.
In the UDT design, even if one peer could not decrypt, this did not mean that others should not 
receive the information. In UDT, the key exchange is a one way process, where the key is
regularly injected into a stream so that it can be received at any time. But with SRT the connection
is established in the beginning, so the key exchange happens then.
It cannot be done just anywhere in the stream.

The short lived SEK is regenerated for cryptographic reasons when enough packets have been
encrypted with it by implementation of either side. One example implementation has default
value of 2^25, with a pre-announcement period of 4000 packets (i.e. a new key is generated,
wrapped, and sent at 2^25 minus 4000 packets). Even and odd keys are alternated this way.
The packets with the earlier key (we will call it key #1, or "odd") will continue to be sent.
The receiver will receive the new key #2 (even), then decrypt and unwrap it.
The receiver will reply to the sender if it is able to understand.
Once the sender gets to the 2^25 packet using the odd key (key #1), it will then start to
send packets with the even key (key #2). It knows that the receiver has what it needs to
decrypt that even key (key #2). This happens transparently, from one packet to the next.
At 2^25 plus 4000 packets the first key will be decommissioned automatically in this example
implementation.

<!-- TODO: [[Refreshing rate is not a parameter of the protocol, but a parameter for the current implementation. This is not crticial for interoperability, so this part was rephrased as an example implementation.]] -->

<!-- TODO: [[Figure]] -->

The keys live in parallel for a certain period of time. A bit before and a bit after the given
refresh point (in the previous example, the 2^25) there are two keys that exist in parallel,
in case there is retransmission. It is possible for packets with the older key to arrive
at the receiver a bit late. Each packet contains a description of which key it requires,
so the receiver will still have the ability to decrypt it.

During handshake process, keying materials piggyback on the second round-trip,
including the response. The handshake initiator sends the key length,
and then the key material exchange happens on the last trip.

<!-- TODO: [[Figure]] -->

##Definitions for encryption mechanism

###Ciphers (AES-CTR)

<!-- TODO: format of item title? -->

The payload is encrypted with a cipher in counter mode (AES-CTR).
The AES counter mode is one of the only cipher modes suitable for continuous stream encryption
that permits decryption from any point, without access to start of the stream (random access),
and for the same reason tolerates packet lost.
The Electronic Code Book (ECB) mode also has these characteristics but does not provide serious
confidentiality and is not recommended in cryptography.

Integrity protection might eventually be achieved with an Authenticated Encryption with
Associated Data (AEAD) cipher such as AES-CCM or AES-GCM.
These cipher modes remove the need of a separate message authentication protocol such as SHA-2.

<!-- TODO: [[not sure if this definition is necessary. But if it is to be included, references are needed for ECB, AEAD, AES-GCM, SHA-2..]] -->

### Media Stream Message (MSmsg)

The Media Stream message is formed from the SRT media stream (data) packets with some elements of
the SRT header used for the cryptography. An SRT header already carries a 32-bit packet sequence
number that is used for the cipher's counter (ctr), with 2 bits taken from the header's message number
(which is thereby reduced to 27 bits) for the encryption key (odd/even) indicator.
Note that the message number field is used when the message is larger than the MTU are send,
which is not the case for MPEG-TS/SRT so the reduction to 27 bits is without consequence.

<!-- TODO: [[reference for MPEG-TS?]] -->
<!-- TODO: [[message number is reduced to 26 bits in the end..because of 'R' retransmission flag]] -->

### Keying Material

For each stream, the sender generates a Stream Encrypting Key (SEK) and a Salt.
For the initial implementation and for most envisioned scenarios where no separate
authentication algorithm is used for message integrity, the SEK is used directly
to encrypt the media stream. The Initial Vector (IV) for the counter is derived from the Salt only.
In other scenarios, the SEK can be used along with the Salt as a key generating material
to produce distinct encryption, authentication, and salt keys.

### Stream Encrypting Key (SEK)

The Stream Encrypting Key (SEK) is pseudo-random and different for each stream.
It must be 128, 192, or 256 bits long for the AES-CTR ciphers.
It is non-persistent and relatively short lived.
In a typical scenario the SEK is expected to last, in cryptographic terms,
around 37 days for a 31-bit counter (2^31 packets / 667 packets/second).

The SEK is regenerated every time a stream starts and does not survive system reboot.
It must be discarded before 2^31 packets are encrypted (31-bit packet index) according
to an implementation example of SRT and replaced seamlessly using an odd/even
key mechanism described below.
Reusing an IV with the same key on different clear text is a known catastrophic issue
of counter mode ciphers. By regenerating the SEK each time a stream starts we remove
the need for elaborate management of the IV to ensure uniqueness.

### Initialization Vector (IV)

The IV (also called "nonce" in the AES-CTR context) is a 112-bit random number.
For the initial implementation of SRT and for most envisioned scenarios
where no separate authentication algorithm is used for message integrity (Auth=0),
the IV is derived from the salt only.

IV = MSB(112, Salt):
: Most significant 112 bits of the salt.

### Counter (ctr)

The counter for AES-CTR is the size of the cipher's block, i.e. 128 bits.
It is based on a block counter in the least significant 16 bits, for counting blocks
of a packet, a packet index in the next 32 bits, and 80 zeroed bits.
The upper 112 bits are XORed with the IV to produce a unique counter for each crypto block.

<!-- TODO: [[Figure]] -->

The block counter (bctr) is incremented for each cipher block while producing the key stream.
The packet index is incremented for each packet submitted to the cipher.
The IV is derived from the Salt provided with the Keying Material.

### Keying Material message (KMmsg)

The SEK and a salt are transported in-stream, in a Keying Material message (KMmsg),
implemented as a custom SRT control packet, wrapped with a longer term Key Encrypting Key (KEK)
using AES key wrap {{RFC3394}}.

The connection-oriented SRT KM control packet is transmitted at the start of the connection,
before any data packet. In most cases, if the control packet is not lost, the receiver
is able to decrypt starting with the first packet. Otherwise, the initial packets are dropped
(or stored for later decryption) until the KM control packet is received
(the SRT control packet is retransmitted until acknowledged by the receiver).

### Odd/Even Stream Encrypting Key (oSEK/eSEK)

To ensure seamless rekeying for cryptographic (counter exhausted) or access control reasons,
SRT employs a two-key mechanism, similar to the one used with DVB systems.
The two keys are identified as the odd key and the even key (oSEK/eSEK).
An odd/even flag in the SRT data header tells which key is in use. 
The next key to be used is transmitted in advance to the receivers in an SRT control packet.
When rekeying occurs, the SRT data header odd/even flag flips, and the receiver already has
the new key in hand to continue decrypting the stream without missing a packet.

<!-- TODO: [[reference for DVB system?]] -->

<!-- TODO: [[Figure]] -->


### Key Encrypting Key (KEK)

The Key Encrypting Key (KEK) is used by the sender to wrap the SEK, and by the receiver to
unwrap it and then decrypt the stream. The KEK must be at least the size of the key it protects,
the SEK. The KEK is derived from a shared secret, a pre-shared password by default.

The KEK is derived with the PBKDF2 <!--{{PCKS5}} --> derivation function with the stream
Salt and the shared secret for input.
Each stream then uses a unique KEK to encrypt its Keying Material.
A compromised KEK does not compromise other streams protected with the same shared secret
(but a compromised shared secret compromises all streams protected with KEK derived from it).
Late derivation of the KEK using stream Salt also permits the generation
of a KEK of the proper size, based on the size of the key it protects.

The shared secret can be pre-shared; password derived PKCS5;
distributed using a proprietary mechanism, or by using a standard key distribution mechanism
such as GDOI {{RFC3547}} or MIKEY {{RFC3830}}.

The cryptographic usage limit of the KEK is 248 wraps (AESKW) which means virtual infinity
at the expected SEK rekeying rate (90,000 years to rekey 100 keys every second).

## Encryption Process Walkthrough

### Stream configuration

In the simplest stream protection form, a stream is configured with a password
(preferably a passphrase) to enable encryption. The sender decides to protect or not.
On the receiver, a configured a passphrase is used only if the incoming media stream is encrypted.
A receiver knows that a received stream is encrypted and how (cipher/key length),
based on the received stream itself - not because it has a password configured.

In the absence of security policies, operators and admins can perform this task.
The only other parameter to set in the initial implementation of SRT, which uses only AES-CTR,
is the key size: 128, 192, or 256.

A system-wide stream protection passphrase (and other crypto parameters) can also be set
by a Security Officer. On the sender, a stream can be protected by only enabling encryption
(using a system-wide passphrase derived key).
On the receiver, no specific setting is required if the decoder detects encryption
from the received stream and has access to the system-wide passphrase to decrypt it.

### Keying Material Generation

#### SEK, Salt, and KEK

For each protected stream, the encoder generates a random SEK and Salt.
In the case of a pre-shared password derived key, it derives the KEK from 
the pre-shared secret and the 64 least significant bits of the Salt.

~~~~~~~~~~~
SEK = PRNG(Klen)
Salt = PRNG(128)
KEK = PBKDF2(password, LSB(64,Salt), Iter, Klen)
~~~~~~~~~~~

PRNG is a pseudo-random number generator, Klen is the key length (128, 192, 256), 
PBKDF2 is the Password Based Key Derivation Function 2.0 <!-- [PKCS5] -->,
and Iter is the iteration count (2048).
For proper protection, the KEK must be at least as long as the key it protects.

#### KMmsg

The encoder assembles the Keying Material message once for the life of the SEK and transmits
it periodically. It may be assembled twice if the 2nd key (odd and even) is later added
and transmitted in the same message for seamless rekeying (SEK = eSEK || oSEK).

~~~~~~~~~~~
Wrap = AESkw(KEK, SEK)
KMmsg = Header || CryptoInfo || Wrap
~~~~~~~~~~~

#### Encryption and Transmission

<!-- TODO: [[Figure]] -->
<!-- TODO: [[This figure seems somehow duplicated before... Need to sort it out and keep one instance only. Maybe first instance?]] -->

#### KMmsg Transmission (KM Tx Period)

The sender transmits the KMmsg periodically, for example, every second.

#### Key Stream Generation

When a co-processor is used for cryptography the key stream can be generated in parallel,
in advance of the media stream. The initial counter is generated from the IV and the Salt.

~~~~~~~~~~~
IV = MSB(112, Salt)
ctr = (IV112 || 016) XOR (index32 || 016)
KeyStream[index] = AES(SEK, ctr) || AES(SEK, ctr+1) || ... || AES(SEK, ctr+n-1)
index = index + 1
~~~~~~~~~~~

'n' is the number of 128 bits block in the payload.

#### Media Stream Message Generation
The MPEG-TS/UDP payload is XORed with the Key stream:

~~~~~~~~~~~
MSmsg[index] = Header || index || (KeyStream[index] XOR payload[index])
~~~~~~~~~~~

####Reception and Decryption

<!-- TODO: [[Figure]] -->

#### Detect Encryption
The receiver of a stream can detect from the message whether or not the stream is encrypted.

#### Regenerate the KEK, SEK and IV

If the receiver does not have the Keying Material,
it waits for a KMmsg to extract the Salt and Wrap.
With the pre-shared password and the Salt it regenerates the KEK and
then unwrap the SEK. It checks the Wrap integrity from the Integrity Check Vector.

~~~~~~~~~~~
KEK = PBKDF2(password, Salt, Iter, Klen)
ICV || SEK = AESkw(KEK, Wrap)
~~~~~~~~~~~

#### Generate Key Stream
With the next stream message (MSmsg) the receiver grabs the packet counter and odd/even
key flag from the clear text header and prepares the cipher to generate a key stream for some
packets in advance.

#### Counter Preset
As an optimization, while waiting for the first KMmsg, the decoder can grab the packet counter
and odd/even flag from the stream message (MSmsg) received before the KMmsg and know
what the packet counter of the next MSmsg should be, then prepares the cipher earlier.

#### Early Frames vs. Latency
Where the key stream is generated in any arrangement where a crypto processor can process
the key stream in parallel, there is a choice between displaying the earlier frames received or
achieving low latency.

When a decoder has prepared the cipher to generate the key stream, it has the choice to (1) store
the received stream packets until the key stream is ready, and then display the earliest received
frame with some latency (the lag between the first packet buffered and the first packet
decrypted and displayed), or (2) drop the early packets until the key stream is ready, and then
achieve lower latency. It is assumed here that the crypto engine can thereafter generate the key
stream as fast as it receives MSmsg.

While it would be possible to buffer the received stream packets before receiving the KMmsg,
there is no guarantee that the odd or even key used to encrypt these messages is the one
encrypted in the KMmsg received after, especially if backward secrecy is achieved (where a new
receiver cannot decrypt an earlier stream).

## Messages

### Data Message Header

<!-- TODO: [[Figure]] -->

<!-- TODO: [[Table]] -->

### SRT Control Message Header

<!-- TODO: [[Figure]] -->

<!-- TODO: [[Table]] -->

### SRT Keying Material Control Message (KMmsg)
The Keying Material message provides the information to decrypt the Media Stream message
payload.

<!-- TODO: [[Figure]] -->
<!-- TODO: [[Table]] -->

<!-- TODO: [[Reference needed for AES-CCM, AES-GCM... it's already there so need to include in reference section.]] -->

#### KEKI

The KEK index (KEKI) tells which KEK was used to wrap (and optionally authenticate) the SEK(s).
With a key management system, the KEKI is used as one of the parameters
(possibly with the IP address and port of the stream) to retrieve a cryptographic context.
In the absence of a key management system, the value 0 is used to indicate the default key
of the current stream.

If it is required to seamlessly change the KEK for good security practice (key lifetime elapsed),
to preserve backward/forward secrecy when a new receiver joins/leaves,
for a security breach (compromised KEK or device),
or if there are multiple sensitive streams to protect (e.g. secret, sensitive, public, etc.),
then the system needs to manage multiple keys and this fields tells which one to use.

#### Auth

In the eventuality that a separate authentication protocol such as SHA2 is used 
for message integrity, the SAK (Stream Authentication Key) may be derived
from the SEK instead of being carried in the KMmsg.
The SEK then becomes a Key Generating Key and the Salt a master salt from
which the stream encryption, authentication, and salt keys are derived.
This is the method used for SRTP {{RFC3711}}.

#### Salt

This field complements the keying material.
It can carry a nonce; salting key; Initialization Vector, or be used as a generator for them.

#### Wrap

Once KMmsg is unwrapped, the following fields are revealed.

<!-- [[Figure]] -->
<!-- [[Table]] -->

The same KMmsg can carry two keys only if they share the same KEKI and same length.
If there is a security parameter change with SEK rekeying (key length or new KEKI),
then a different KMmsg must be used to carry the odd and even key.

Note that the same Salt is reused on SEK rekeying.
This does not break the nonce requirement of the counter mode that is used only once for a given key,
and preserves the password-based derived KEK of this stream.

<!-- [[Need to check if it is okay or not...the same salt for SEK and KEK or...the same salt for SEK and next SEK??]] -->

##### ICV

This field is used to detect if the keys were unwrapped properly.
If the KEK in hand is invalid, validation fails and unwrapped keys are discarded.

##### xSEK

This field identifies an odd or even SEK.
If only one key is present, the bit set in the Flags field tells which one it is.
If both keys are present, then this field is eSEK (even key) and the next one is the odd key.

##### oSEK

This field is present only when the message carries the two SEKs.
This may be only for a short period before re-keying the stream for a new receiver
to decrypt the current stream, and for all members to prepare for seamless re-keying.

## Parameters
<!-- TODO: [[Table]] -->
<!-- TODO: [[Now some or all of these parameters are implementation specific, so need to be removed from the draft. Maybe some of them should be fixed value for the current version of SRT???]] -->

# Security Considerations

SRT supports confidentiality of user data using stream ciphering based on AES.
Session keys for ciphering are delivered through control packets during handshake,
with the protection by Key Encryption Key, which is generated by a sender and receiver
with pre-shared secret such as passphrase. As in UDT, careful uses of SYN Cookies
may help to deter denial of service attacks. Appropriate security policy including
key size, key refresh period, as well as passphrase should be managed by security
officers, which is out of scope of the present document.

# IANA Considerations

This document makes no requests of the IANA.

# Contributors
{:numbered="false"}

This specification is heavily based on the SRT Protocol Technical Overview {{SRTTO}}
written by Jean Dube and Steve Matthews.

In alphabetical order, the contributors to the pre-IETF SRT project and
specification at Haivision are: Marc Cymontkowski, Roman Diouskine,
Jean Dube, Mikolaj Malecki, Steve Matthews, Maria Sharabayko,
Maxim Sharabayko, Adam Yellen.

The contributors to this specification at SK Telecom
are Jeongseok Kim and Joonwoong Kim.

We cannot list all the contributors to the open-sourced implementation of SRT on GitHub.
But we appreciate the help, contribution, integrations and feedback of the
SRT and SRT Alliances community.

# Acknowledgments
{:numbered="false"}

The basis of the SRT protocol and its implementation was
the UDP-based Data Transfer Protocol {{GHG04b}}.
The authors thank Yunhong Gu and Robert Grossman,
the authors of the UDP-based Data Transfer Protocol {{GHG04b}}.

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
{: #single-sequence-number title="Single sequence numbers coding"}

For any consecutive packet sequence numbers that the difference between
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
{: #list-sequence-numbers title="List of sequence numbers coding"}
