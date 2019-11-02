---
title: Packet Loss Signaling for Encrypted Protocols
abbrev: loss-bits
docname: draft-ferrieuxhamchaoui-quic-lossbits-latest
date: {DATE}
category: info

ipr: trust200902
area: Loss Signaling
workgroup: TSVWG
keyword: Internet-Draft

stand_alone: yes
pi:
  rfcedstyle: yes
  toc: yes
  tocindent: yes
  sortrefs: yes
  symrefs: yes
  strict: yes
  comments: yes
  compact: yes
  inline: yes
  text-list-symbols: -o*+

author:
  -
    ins: A. Ferrieux
    role: editor
    name: Alexandre Ferrieux
    org: Orange Labs
    email: alexandre.ferrieux@orange.com
  -
    ins: I. Hamchaoui
    role: editor
    name: Isabelle Hamchaoui
    org: Orange Labs
    email: isabelle.hamchaoui@orange.com
  -
    ins: I. Lubashev
    role: editor
    name: Igor Lubashev
    org: Akamai Technologies
    street: 150 Broadway
    city: Cambridge
    code: 02142
    region: MA
    country: USA
    email: ilubashe@akamai.com

normative:
  IP: RFC0791
  IPv6: RFC8200

informative:
  QUIC-TRANSPORT: I-D.ietf-quic-transport
  TRANSPORT-ENCRYPT: I-D.ietf-tsvwg-transport-encrypt
  GREASE: I-D.ietf-tls-grease
  LOSSBITS: I-D.ferrieuxhamchaoui-tsvwg-lossbits

--- abstract

This draft adapts the general technique described in
draft-ferrieuxhamchaoui-tsvwg-lossbits ({{LOSSBITS}}) for QUIC using reserved
bits in QUIC v1 header.  It describes a method that employs two bits to allow
endpoints to signal packet loss in a way that can be used by network devices to
measure and locate the source of the loss.


--- middle

# Introduction

Packet loss is a hard and pervasive problem of day-to-day network operation, and
proactively detecting, measuring, and locating it is crucial to maintaining high
QoS and timely resolution of crippling end-to-end throughput issues. To this
effect, in a TCP-dominated world, network operators have been heavily relying on
information present in the clear in TCP headers: sequence and acknowledgment
numbers, and SACK when enabled.  These allow for quantitative estimation of
packet loss by passive on-path observation. Additionally, the lossy segment
(upstream or downstream from the observation point) can be quickly identified by
moving the passive observer around.

With QUIC, the equivalent transport headers are encrypted and passive packet
loss observation is not possible, as described in {{TRANSPORT-ENCRYPT}}.

QUIC could be routed by the network differently and the fraction of Internet
traffic delivered using QUIC is increasing every year. Therefore, is it
imperative to measure packet loss experienced by QUIC users directly instead of
relying on measuring TCP loss between similar endpoints.

Since explicit path signals are preferred by {{!RFC8558}}, this document
proposes adding two explicit loss bits to the clear portion of short headers to
restore network operators' ability to maintain high QoS for QUIC users.

# Notational Conventions    {#conventions}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in {{!RFC2119}}.

# Loss Bits

The proposal introduces two bits that are to be present in packets with a short
header.  Therefore, only loss of short header packets is reported using loss
bits.  Whenever this specification refers to packets, it is referring only to
packets with short headers.

* Q: The "sQuare signal" bit is toggled every N outgoing packets as explained
  below in {{squarebit}}.

* L: The "Loss event" bit is set to 0 or 1 according to the Unreported Loss
  counter, as explained below in {{lossbit}}.

Each endpoint maintains appropriate counters independently and separately for
each connection 4-tuple and destination Connection ID.


## Setting the sQuare Bit on Outgoing Packets {#squarebit}

The sQuare Value is initialized to the Initial Q Value (0 or 1) and is reflected
in the Q bit of every outgoing packet.  The sQuare value is inverted after
sending every N packets (Q Period is 2*N), where N is a parameter of the method,
discussed below.

Observation points can estimate the upstream losses by counting the number of
packets during a half period of the square signal, as described in {{usage}}.

## Setting the Loss Event Bit on Outgoing Packets {#lossbit}

The Unreported Loss counter is initialized to 0, and the L bit of every outgoing
packet indicates whether the Unreported Loss counter is positive (L=1 if the
counter is positive, and L=0 otherwise).

The value of the Unreported Loss counter is decremented every time a packet with
L=1 is sent.

The value of the Unreported Loss counter is incremented for every packet that
the protocol declares lost, using QUIC's existing loss detection machinery.

Observation points can estimate the end-to-end loss, as determined by the
upstream endpoint's loss detection machinery, by counting packets in this
direction with a L bit equal to 1, as described in {{usage}}.

# Using the Loss Bits for Passive Loss Measurement {#usage}

There are three sources of observable loss:

* _upstream loss_ - loss between the sender and the observation point
  ({{upstreamloss}})

* _downstream loss_ - loss between the observation point and the destination
  ({{downstreamloss}})

* _observer loss_ - loss by the observer itself that does not cause downstream
  loss ({{observerloss}})

The upstream and downstream loss together constitute _end-to-end loss_
({{endtoendloss}}).

The Q and L bits allow detection and measurement of the types of loss listed
above.


## End-To-End Loss    {#endtoendloss}

The Loss Event bit allows an observer to calculate the end-to-end loss rate by
counting packets with L bit value of 0 and 1 for a given connection. The
end-to-end loss rate is the fraction of packets with L=1.

The simplifying assumption here is that upstream loss affects packets with L=0
and L=1 equally. This may be a simplification, if some loss is caused by
tail-drop in a network device. If the sender congestion controller reduces the
packet send rate after loss, there may be a sufficient delay before sending
packets with L=1 that they have a greater chance of arriving at the observer.


## Upstream Loss   {#upstreamloss}

Blocks of N (half of Q Period) consecutive packets are sent with the same value
of the Q bit, followed by another block of N packets with inverted value of the
Q bit. Hence, knowing the value of N, an on-path observer can estimate the
amount of loss after observing at least N packets. The upstream loss rate is one
minus the average number of packets in a block of packets with the same Q value
divided by N.

The observer needs to be able to tolerate packet reordering that can blur the
edges of the square signal.

The observer also needs to differentiate packets as belonging to different
connections, since they use independent counters.

The choice of N strikes a compromise: the observation could become too
unreliable in case of packet reordering and loss if N is too small; and when N
is too large, short connections may not yield a useful upstream loss
measurement.

To leave some room for adaptation, we only constrain the sender to select an N
that is (1) constant for a given connection and (2) equal to a power of two.
The latter allows on-path observers to derive N after a few periods.  It is thus
also acceptable for a simple implementation to choose a global constant; N=64
has been extensively tried in large-scale field tests and yielded good results.

## Correlating End-to-End and Upstream Loss    {#losscorrelation}

Upstream loss is calculated by observing the actual packets that did not suffer
the upstream loss.  End-to-end loss, however, is calculated by observing
subsequent packets after the sender's protocol detected the loss.  Hence,
end-to-end loss is generally observed with a delay of between 1 RTT (loss
declared due to multiple duplicate acknowledgments) and 1 RTO (loss declared due
to a timeout) relative to the upstream loss.

The connection RTT can sometimes be estimated by timing protocol handshake
messages. This RTT estimate can be greatly improved by observing a dedicated
protocol mechanism for conveying RTT information, such as the Latency Spin bit
of {{QUIC-TRANSPORT}}.

Whenever the observer needs to perform a computation that uses both upstream and
end-to-end loss rate measurements, it SHOULD use upstream loss rate leading the
end-to-end loss rate by approximately 1 RTT. If the observer is unable to
estimate RTT of the connection, it should accumulate loss measurements over time
periods of at least 4 times the typical RTT for the observed connections.

If the calculated upstream loss rate exceeds the end-to-end loss rate calculated
in {{endtoendloss}}, then either the Q Period is too short for the amount of
packet reordering or there is observer loss, described in {{observerloss}}. If
this happens, the observer SHOULD adjust the calculated upstream loss rate to
match end-to-end loss rate.

## Downstream Loss   {#downstreamloss}

Because downstream loss affects only those packets that did not suffer upstream
loss, the end-to-end loss rate (`e`) relates to the upstream loss rate (`u`) and
downstream loss rate (`d`) as `(1-u)(1-d)=1-e`. Hence, `d=(e-u)/(1-u)`.

## Observer Loss   {#observerloss}

A typical deployment of a passive observation system includes a network tap
device that mirrors network packets of interest to a device that performs
analysis and measurement on the mirrored packets. The observer loss is the loss
that occurs on the mirror path.

Observer loss affects upstream loss rate measurement since it causes the
observer to account for fewer packets in a block of identical Q bit values (see
{{upstreamloss)}).  The end-to-end loss rate measurement, however, is unaffected
by the observer loss, since it is a measurement of the fraction of packets with
the set L bit value, and the observer loss would affect all packets equally (see
{{endtoendloss}}).

The need to adjust the upstream loss rate down to match end-to-end loss rate as
described in {{losscorrelation}} is a strong indication of the observer loss,
whose magnitude is between the amount of such adjustment and the entirety of the
upstream loss measured in {{upstreamloss}}.

# Ossification Considerations

Accurate loss information is not critical to the operation of any protocol,
though its presence for a sufficient number of connections is important for the
operation of the networks.

The loss bits are amenable to "greasing" described in {{GREASE}}, if the
protocol designers are not ready to dedicate (and ossify) bits used for loss
reporting to this function.  The greasing could be accomplished similarly to the
Latency Spin bit greasing in {{QUIC-TRANSPORT}}.  Namely, implementations could
decide that a fraction of connections should not encode loss information in the
loss bits and, instead, the bits would be set to arbitrary values. The observers
would need to be ready to ignore connections with loss information more
resembling noise than the expected signal.

# Security Considerations

Passive loss observation has been a part of the network operations for a long
time, so exposing loss information to the network does not add new security
concerns.

# Privacy Considerations

Guarding user's privacy is an important goal for modern protocols and protocol
extensions per {{?RFC7285}}.  While an explicit loss signal -- a preferred way
to share loss information per {{!RFC8558}} -- helps to minimize unintentional
exposure of additional information, implementations of loss reporting must
ensure that loss information does not compromise protocol's privacy goals.

For example, {{QUIC-TRANSPORT}} allows changing Connection IDs in the middle of
a connection to reduce the likelihood of a passive observer linking old and new
subflows to the same device. A QUIC implementation would need to reset all
counters when it changes Connection ID used for outgoing packets.  It would also
need to avoid incrementing Unreported Loss counter for loss of packets sent with
a different Connection ID.

# IANA Considerations

This document makes no request of IANA.

# Acknowledgments

The sQuare Bit was originally specified by Kazuho Oku in early proposals for
loss measurement, and is an instance of the "alternate marking" as defined in
{{!RFC8321}}.
