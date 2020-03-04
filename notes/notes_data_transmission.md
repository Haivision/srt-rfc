# Notes - Data Transmission and Control Section

## Timestamp Based Packet Delivery {#tsbpd}

1. Insert link to SRT connection creation time, SRT latency.

2. Reflect three main ideas of this section:
    * After handshakes exchange - end-to-end delay is fixed
    * It's worth noting that TsbpdDelay limits the number of packet retransmissions to a certain extent making impossible to retransmit packets endlessly. This is important for live data transmission.
    * Each packet has timestamp (when it was ready to be sent), receiver uses this timestamp to calculate delivery time: send time + end-to-end delay = read time

#### TSBPD Time Base Calculation {#tsbpd-time-base}

1. Some introductory notes - not necessary to add

Timestamps are relative to the connection. The transmission is not based on an absolute time.
The scheduled execution time is based on a real clock time. A time base is maintained to
convert each packet‘s timestamp to local clock time. Packets are offset from the sender‘s
StartTime. Any time reference based on local StartTime is maintained, taking into account RTT,
time zone and drift caused by the sum of truncated nanoseconds and other measurements.

2. TSBPD wrapping period: add pseudo-code.

3. Deleted

It is designed in a way that SRT receiver, using the timestamp of the SRT data packet header,
delivers packets to a receiving application at the same pace they were provided to the SRT sender by a sending applicaton.

4. Rewrite introduction

This
The goal of the SRT Timestamp Based Packet Delivery (TSBPD) mechanism
is to reproduce the output of the sending application (e.g., encoder)
at the input of the receiving one (e.g., decoder) in live data
transmission mode (see {{data-transmission-mode}}). In terms of SRT,
it means to reproduce the timing of packets commited by the sending
application to the SRT sender at the timing packets are scheduled
for the delivery by the SRT receiver and ready to be read by the
receiving application (see {{fig-latency-points}}).

This
The packet timestamp (in microseconds) is relative to the SRT connection creation time.
**It is worth mentioning that the use of the packet sending time to stamp the packets is
inappropriate for TSBPD feature since a new time (current sending time) is used for retransmitted packets,
putting them out of order when inserted at their proper place in the stream. Packets are
inserted based on the sequence number in the header field.** The origin time (in microseconds)
of the packet is already sampled when a packet is first submitted by the application to the SRT sender.
The TSBPD feature uses this time to stamp the packet for first transmission and any subsequent retransmission.
This timestamp and the configured SRT latency control the recovery buffer size and the instant that packets
are delivered at the destination.

## Too-Late Packet Drop {#too-late-packet-drop}

1. Rephrase this part

Implementation related, delete implementation details in future:
In the SRT sender, when Too-Late Packet Drop is enabled, a packet is
considered too late to be delivered and may be dropped by the sending
application if its timestamp is older than 125% of the SRT latency.
However, the sender keeps packets for at least 1 second in case the
SRT latency is not enough for a large RTT (that is, if 125% of the
SRT latency is less than 1 second).

2. Pictures and tables to illustrate pseudo-code

This table should be corrected, but as an example

| s @ Dst | SrcTime (PKT_TIMESTAMP) | SND Clock   | Time Base    | RCV Clock   | SRT Latency | Drift | Packet Delivery Time |   |
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

3. According to Maxim, this additional delay is implementation related

From API

NB: The default live mode settings set SRTO_SNDDROPDELAY to 0. The buffer mode settings set SRTO_SNDDROPDELAY to -1.

[SET] - Sets an extra delay before TLPKTDROP is triggered on the data sender. TLPKTDROP discards packets reported as lost if it is already too late to send them (the receiver would discard them even if received). The total delay before TLPKTDROP is triggered consists of the LATENCY (SRTO_PEERLATENCY), plus SRTO_SNDDROPDELAY, plus 2 * the ACK interval (default ACK interval is 10ms). The minimum total delay is 1 second. A value of -1 discards packet drop. SRTO_SNDDROPDELAY extends the tolerance for retransmitting packets at the expense of more likely retransmitting them uselessly. To be effective, it must have a value greater than 1000 - SRTO_PEERLATENCY.

4. Note regarding acknowledgment process for dropped packets.

TODO: It also sends a fake ACK message to the sender. - Should we write about this?

## Drift Management {#drift-management}

1. Add detailed description of the algorithm with pseudo-code.
https://srtlab.github.io/srt-cookbook/protocol/tsbpd/drift-management/

2. Some parts of the current drift description can be moved into introduction or other sections of TSBPD as the introductory words. Here we can leave only the idea and pseudo-code.

E.g., this part

The receiver uses local time to be able to schedule events — to
determine, for example, if it's time to deliver a certain packet
right away. The timestamps in the packets themselves are just
references to the beginning of the session. When a packet is received
(with a timestamp from the sender), the receiver makes a reference to
the beginning of the session to recalculate its timestamp. The start
time is derived from the local time at the moment that the session is
connected. A packet timestamp equals "now" minus "StartTime", where
zthe latter is the point in time when the socket was created.

3. Note regarding drift tracer: The current algorithm does not take into account RTT variations, but we are going to improve this.

Assuming that the link latency is constant (RTT=const), the only cause of the drift fluctuations should be clock inaccuracy.

## SRT Buffer Latency

1. Describe the difference between sender buffer latency and receiver buffer latency which is configered in SRT settings and is TsbpdDelay at the same time.

## Acknowledgement and Lost Packet Handling

This section should be reworked completely.

1. From the very beginning we should list the types of ack packets.

We should say that there are two types of packets: data and control. Then there several types of control packets: list them -> maybe it's better to do at the very beginning of Data Transmission section.

And here we say that ack packets are control packets. And there are several types of ack packets, list them.

- ACK (full ACK, light ACK)
- ACKACK
- NAK
- Periodic NAK

2. Later we should have the algorithms of working with each type of ack like

On receipt of ACK do: 1, 2, etc.
Plus the timing of receiving and processing ack-s.

3. Tell what's inside ack-s: RTT, BW, Rcv speed estimations, etc.

Each ACK has a number. A corresponding ACKACK has that same number.
The receiver keeps a list of all ACKs in a queue to match them. Unlike a full ACK,
which contains the current RTT and several other values in the CIF,
a light ACK just contains the sequence number. All control messages are sent directly and
processed upon reception, but ACKACK processing time is negligible (the time this takes
is included in the round-trip time).

4. How the estimations are obtained: bw, rcv speed, RTT - separate section with the algorithm description.

5. Move the bold part to TLPKTDROP, TSBPD regarding 1 second latency in sender buffer

This condition where packets are unsent doesn't happen often. There is a maximum number of
packets held in the send buffer based on the configured latency. Older packets that have no
chance to be retransmitted and played in time are dropped, making room for newer real-time
packets produced by the sending application. See sections {{tsbpd}}, {{too-late-packet-drop}} for details.
**A minimum of one second is applied before
dropping the packet when low latency is configured. This one-second limit derives from the
behavior of MPEG I-frames with SRT used as transport. I-frames are very large (typically 8 times
larger than other packets), and consequently take more time to transmit. They can be too large
to keep in the latency window, and can cause packets to be dropped from the queue. To
prevent this, SRT imposes a minimum of one second (or the latency value) before dropping a
packet. This allows for large I-frames when using small latency values.**