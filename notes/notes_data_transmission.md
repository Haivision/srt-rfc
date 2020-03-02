# Notes - Data Transmission and Control Section

#### TSBPD Time Base Calculation {#tsbpd-time-base}

1. Some introductory notes - not necessary to add

<!-- Timestamps are relative to the connection. The transmission is not based on an absolute time.
The scheduled execution time is based on a real clock time. A time base is maintained to
convert each packet‘s timestamp to local clock time. Packets are offset from the sender‘s
StartTime. Any time reference based on local StartTime is maintained, taking into account RTT,
time zone and drift caused by the sum of truncated nanoseconds and other measurements. -->

2. TSBPD wrapping period: add pseudo-code.