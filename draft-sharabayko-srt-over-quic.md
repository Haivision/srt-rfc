---
title: Tunnelling SRT over QUIC
abbrev: SRTQ
docname: draft-sharabayko-srt-over-quic-00
category: info

ipr: trust200902
area:
workgroup: Network Working Group
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
    ins: "T. Add"
    name: "To Add"

normative:
  RFC7323:
  RFC9000:
  RFC2119:
  RFC8174:
  GHG04b:
    title: Experiences in Design and Implementation of a High Performance Transport Protocol
    author:
      - 
        name: Yunhong Gu
      - 
        name: Xinwei Hong
      - 
        name: Robert L. Grossman
    date: December, 2004
    seriesinfo:
      DOI: 10.1109/SC.2004.24

informative:
  SRTRFC:
    target: https://datatracker.ietf.org/doc/draft-sharabayko-srt/
    title: The SRT Protocol
    author:
      - 
        name: Maxim Sharabayko
      - 
        name: Maria Sharabayko
      - 
        name: Jean Dube
      - 
        name: Jeongseok Kim
      - 
        name: Joonwoong Kim
    date: December, 2019

  QUIC-DATAGRAM:
    target: https://datatracker.ietf.org/doc/draft-ietf-quic-datagram/
    title: An Unreliable Datagram Extension to QUIC
    author:
      -
        name: T. Pauly
      -
        name: E. Kinnear
      -
        name: D. Schinazi
  
  SRTSRC:
    target: https://github.com/Haivision/srt
    title: SRT fully functional reference implementation
    date: none

--- abstract

This document presents an approach to tunnel SRT live streaming over QUIC datagrams.

QUIC{{RFC9000}} is a UDP-based transport protocol providing TLS encryption, stream multiplexing,
connection migration. It was designed to provide a faster alternative to the TCP protocol {{RFC7323}}.

An Unreliable Datagram Extension to QUIC {{QUIC-DATAGRAM}} adds support for sending and receiving
unreliable datagrams over a QUIC connection.

SRT{{SRTRFC}} is a UDP-based transport protocol. In its live streaming configuration
it provides end-to-end latency-aware mechanisms for packet loss recovery.
A lost packet can be dropped if it is too late to recover it.

Datagram Extension to QUIC could be used as an underlying transport instead of UDP.
This way QUIC would provide TLS level security, connection migration, potentially multi-path support.
It would be easy for existing network facilities to process, route and load balance the unified QUIC traffic.
SRT on its side would provide end-to-end latency tracking and latency-aware loss recovery.

--- middle

# Introduction

## SRT for Low Latency

The Secure Reliable Transport (SRT) protocol {{SRTRFC}} is a connection-based transport protocol
that enables the secure, reliable transport of data across 
unpredictable networks, such as the Internet. While any data type can be transferred 
via SRT, it is ideal for low latency (sub-second) video streaming. SRT provides 
improved bandwidth utilization compared to RTMP, allowing much higher 
contribution bitrates over long distance connections.

To achieve low latency streaming, SRT had to address timing issues. The characteristics 
of a stream from a source network are completely changed by transmission over the public 
Internet, which introduces delays, jitter, and packet loss. This, in turn, leads to 
problems with decoding, as the audio and video decoders do not receive packets at the 
expected times. The use of large buffers helps, but latency is increased. 
SRT includes a mechanism to keep a constant end-to-end latency, thus recreating
the signal characteristics on the receiver side, and reducing the need for buffering.

SRT employs a listener (server) / caller (client) model. The data flow is bi-directional and 
independent of the connection initiation - either the sender or receiver can operate 
as listener or caller to initiate a connection. The protocol provides an internal 
multiplexing mechanism, allowing multiple SRT connections to share the same UDP port, 
providing access control functionality to identify the caller on the listener side. 

Supporting forward error correction (FEC) and selective packet retransmission (ARQ), 
SRT provides the flexibility to use either of the two mechanisms or both combined, 
allowing for use cases ranging from the lowest possible latency to the highest possible 
reliability. 

SRT also allows fast file transfers, and adds support for AES encryption.

## QUIC for Universal Transport

The QUIC transport protocol {{RFC9000}} is a connection-based transport protocol built on top of UDP.
It provides workflow similar to TCP, but for modern fast networks.

TODO: Fill this section.

Lower connection times, faster delivery, ARQ, CC, etc.


# Terms and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}}
when, and only when, they appear in all capitals, as shown here.

SRT:
: The Secure Reliable Transport protocol.

# Use Cases for SRT over QUIC

SRT itself is very close to QUIC and provides similar transport mechanisms.
However, the main focus of SRT is made on low-latency live contribution and distribution.
QUIC is supported by CDN companies. A lot of facilities know how to handle and route QUIC traffic.
QUIC provides certain security advantages (TLS, encrypting headers so that traffic is not distinguishable).

SRT tunneled over QUIC allows managing live delivery mechanisms (preserving end-to-end latency and dropping too late data).

It also comes at a cost of extra packet headers, sometimes duplicated with those of QUIC.

# Tunneling SRT over QUIC

The QUIC DATAGRAM frame is structured as follows:

~~~
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                        [Length (i)]                         ...
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                      Datagram Data (*)                      ...
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #quic-datagram-frame title="QUIC DATAGRAM Frame Format"}

Length
:  A variable-length integer specifying the length of the
   datagram in bytes.

Datagram Data
:  The bytes of the datagram to be delivered.


The structure of the SRT packet is shown in {{srt-packet}}. For the SRT over QUIC tunneling
the full SRT packet is placed inside the Datagram Data.

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
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                        Packet Contents                        |
|                  (depends on the packet type)                 +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #srt-packet title="SRT packet structure"}

F: 1 bit.
: Packet Type Flag. The control packet has this flag set to "1".
  The data packet has this flag set to "0".

Timestamp: 32 bits.
: The timestamp of the packet, in microseconds.
  The value is relative to the time the SRT connection was established.
  Depending on the transmission mode,
  the field stores the packet send time or the packet origin time.

Destination Socket ID: 32 bits.
: A fixed-width field providing the SRT socket ID to which a packet should be dispatched.
  The field may have the special value "0" when the packet is a connection request.
  
## Packet Integrity
  
SRT does not provide mechanisms neither to verify the integrity of packets,
nor to distinguish a packet from a continuous data stream.
SRT assumes that the underlying transport protocol delivers a single and undamaged packet to SRT.
Therefore, the underlying transport MUST provide a mechanism for SRT to send and receive
exactly one packet.

One SRT packet MUST be sent over exactly one QUIC Datagram frame.  
 
## Connection Establishment
  
QUIC has a fast and secure crypto handshake based on TLS.
A client connects to a server, and it can verify it based on the server certificate.
A new client connection takes 2 RTT to be established.
If a client connects to a known server, then it can try to establish a faster 0-RTT connection.

Once a QUIC connection is established QUIC datagrams can be sent in both directions.
SRT can use this QUIC datagram tunneling to establish one or many connections on its own.
Each SRT connection would take 2-RTT for handshaking.

## Bidirectional Transmission

Both QUIC and SRT allow bidirectional transmission of the payload over a single SRT connection.
Even with payload sent in one direction, some control packets are sent in the opposite direction over the same connection.
  
## Congestion Control

QUIC as a transport can apply congestion control. It should be noted however the specifics of live streaming compared
to file-based transmissions. The payload is not available right away, therefore regular bandwidth probing mechanisms
by increasing the sending rate would not work.

The current sending rate is provided by SRT, which in terms receives the payload from a live source and can add some overhead
when re-transmission of lost packets is performed.

However, QUIC can use congestion control to detect congestion and throughput bottlenecks, and refuse sending above the certain limit.
Then, SRT MUST handle such cases eventually dropping packets the loss of which it is not able to recover.

It would make sense for a QUIC connection to provide this throughput limitation value back to SRT. In that case SRT can use the number to
make clever and transport-aware retransmission decisions.

It is also possible for QUIC to not apply any congestion control, relying on SRT. However, SRT does not reduce the sending rate below the input rate.
If the bitrate of the original stream already exceeds network throughput, SRT would still try to deliver it, maintaining congestion and
eventually breaking SRT connection.

## Pacing

SRT uses ACK - ACKACK packet pair to measure RTT on the link, track latency and clock drift.
It also uses packet pair probing to estimate connection bandwidth.

Buffering and pacing SRT packet by QUIC SHOULD be done with awareness that these mechanisms of SRT would be interfered.


## Datagram vs H3 Datagram

As an alternative to QUIC Datagram extension it might be possible to consider the H3 Datagram version to be compatible with more existing load balancers.

