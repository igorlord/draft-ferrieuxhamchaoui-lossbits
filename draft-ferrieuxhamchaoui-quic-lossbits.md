---
title: Packet Loss Signaling for Encrypted Protocols
abbrev: loss-bits
docname: draft-ferrieuxhamchaoui-quic-lossbits-latest
date: {DATE}
category: info

ipr: trust200902
area: Loss Signaling
workgroup: QUIC
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
    email: ilubashe@akamai.com
  -
    ins: D. Tikhonov
    role: editor
    name: Dmitri Tikhonov
    org: LiteSpeed Technologies
    email: dtikhonov@litespeedtech.com

normative:
  QUIC-TRANSPORT: I-D.ietf-quic-transport
  TRANSPORT-ENCRYPT: I-D.ietf-tsvwg-transport-encrypt
  QUIC-TLS: I-D.ietf-quic-tls

informative:
  GREASE: I-D.ietf-tls-grease
  DATAGRAM: I-D.pauly-quic-datagram
  LOSSBITS: I-D.ferrieuxhamchaoui-tsvwg-lossbits

--- abstract

This draft adapts the general technique described in
draft-ferrieuxhamchaoui-tsvwg-lossbits for QUIC using reserved bits in QUIC v1
short header.  It describes a method that employs two bits to allow endpoints to
signal packet loss in a way that can be used by network devices to measure and
locate the source of the loss. It further describes a way to negotiate this
functionality as an extension in QUIC v1.


--- middle

# Introduction

Packet loss is a hard and pervasive problem of day-to-day network operation.
Proactively detecting, measuring, and locating it is crucial to maintaining high
QoS and timely resolution of crippling end-to-end throughput issues. To this
effect, in a TCP-dominated world, network operators have been heavily relying on
information present in the clear in TCP headers: sequence and acknowledgment
numbers, and SACK when enabled.  These allow for quantitative estimation of
packet loss by passive on-path observation. Additionally, the lossy segment
(upstream or downstream from the observation point) can be quickly identified by
moving the passive observer around.

With QUIC, the equivalent transport headers are encrypted and passive packet
loss observation is not possible, as described in {{TRANSPORT-ENCRYPT}}.

Measuring TCP loss between similar endpoints cannot be relied upon to
evaluate QUIC loss.  QUIC could be routed by the network differently and
the fraction of Internet traffic delivered using QUIC is increasing every
year.  It is imperative to measure packet loss experienced by QUIC users
directly.

Since explicit path signals are preferred by {{!RFC8558}}, two explicit loss
bits in the clear portion of short headers are used to signal packet loss to
on-path network devices.

# Notational Conventions    {#conventions}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in {{!RFC2119}}.

# Loss Bits

The draft introduces two bits that are to be present in packets with a short
header.  Therefore, only loss of short header packets is reported using loss
bits.  Whenever this specification refers to packets, it is referring only to
packets with short headers.

* Q: The "sQuare signal" bit is toggled every N outgoing packets as explained
  below in {{squarebit}}.

* L: The "Loss event" bit is set to 0 or 1 according to the Unreported Loss
  counter, as explained below in {{lossbit}}.

Each endpoint maintains appropriate counters independently and separately for
each connection 4-tuple and Destination Connection ID.  Whenever this
specification refers to connections, it is referring to packets sharing the same
4-tuple and Destination Connection ID.  A "QUIC connection", however, refers to
connections in the traditional QUIC sense.


## Setting the sQuare Signal Bit on Outgoing Packets {#squarebit}

The sQuare Value is initialized to the Initial Q Value (0 or 1) and is reflected
in the Q bit of every outgoing packet. The sQuare value is inverted after
sending every N packets (a Q run). Hence, Q Period is 2*N. The Q bit represents
"packet color" as defined by {{?RFC8321}}.

Observation points can estimate upstream losses by counting the number of
packets during one period of the square signal, as described in {{usage}}.

### Q Run Length Selection

The sender is expected to choose N (Q run length) based on the expected amount
of loss and reordering on the path.  The choice of N strikes a compromise -- the
observation could become too unreliable in case of packet reordering and/or
severe loss if N is too small, while short connections may not yield a useful
upstream loss measurement if N is too large (see {{upstreamloss}}).

The value of N MUST be at least 64 and be a power of 2. This requirement allows
an Observer to infer the Q run length by observing one period of the square
signal. It also allows the Observer to identify flows that set the loss bits to
arbitrary values (see {{ossification}}).

If the sender does not have sufficient information to make an informed decision
about Q run length, the sender SHOULD use N=64, since this value has been
extensively tried in large-scale field tests and yielded good results.
Alternatively, the sender MAY also choose a random N for each connection,
increasing the chances of using a Q run length that gives the best signal for
some connections.

The sender MUST keep the value of N constant for a given connection.  The sender
can change the value of N during a QUIC connection by switching to a new
Destination Connection ID, if one is available.


## Setting the Loss Event Bit on Outgoing Packets {#lossbit}

The Unreported Loss counter is initialized to 0, and the L bit of every outgoing
packet indicates whether the Unreported Loss counter is positive (L=1 if the
counter is positive, and L=0 otherwise).  The value of the Unreported Loss
counter is decremented every time a packet with L=1 is sent.

The value of the Unreported Loss counter is incremented for every packet that
the protocol declares lost, using QUIC's existing loss detection machinery. If
the implementation is able to rescind the loss determination later, a positive
Unreported Loss counter MAY be decremented due to the rescission, but it SHOULD
NOT become negative.

This loss signaling is similar to loss signaling in {{?RFC7713}}, except the
Loss Event bit is reporting the exact number of lost packets, whereas the Echo
Loss bit in {{?RFC7713}} is reporting an approximate number of lost bytes.

Observation points can estimate the end-to-end loss, as determined by the
upstream endpoint, by counting packets in this direction with the L bit equal
to 1, as described in {{usage}}.


# Using Loss Bits for Passive Loss Measurement {#usage}

There are three sources of observable loss:

* _upstream loss_ - loss between the sender and the observation point
  ({{upstreamloss}})

* _downstream loss_ - loss between the observation point and the destination
  ({{downstreamloss}})

* _observer loss_ - loss by the observer itself that does not cause downstream
  loss ({{observerloss}})

The upstream and downstream loss together constitute _end-to-end loss_
({{endtoendloss}}).

The Q and L bits allow detection and measurement of all these types of loss.


## End-To-End Loss    {#endtoendloss}

The Loss Event bit allows an observer to calculate the end-to-end loss rate by
counting packets with the L bit value of 0 and 1 for a given connection. The
end-to-end loss rate is the fraction of packets with L=1.

The assumption here is that upstream loss affects packets with L=0 and L=1
equally.  If some loss is caused by tail-drop in a network device, this may
be a simplification.  If the sender congestion controller reduces the
packet send rate after loss, there may be a sufficient delay before sending
packets with L=1 that they have a greater chance of arriving at the observer.


## Upstream Loss   {#upstreamloss}

Blocks of N (Q run length) consecutive packets are sent with the same value of
the Q bit, followed by another block of N packets with an inverted value of the
Q bit. Hence, knowing the value of N, an on-path observer can estimate the
amount of loss after observing at least N packets. The upstream loss rate (`u`)
is one minus the average number of packets in a block of packets with the same Q
value (`p`) divided by N (`u=1-avg(p)/N`).

The observer needs to be able to tolerate packet reordering that can blur the
edges of the square signal.

The observer needs to differentiate packets as belonging to different
connections, since they use independent counters.


## Correlating End-to-End and Upstream Loss    {#losscorrelation}

Upstream loss is calculated by observing packets that did not suffer the
upstream loss.  End-to-end loss, however, is calculated by observing subsequent
packets after the sender's protocol detected the loss.  Hence, end-to-end loss
is generally observed with a delay of between 1 RTT (loss declared due to
multiple duplicate acknowledgments) and 1 RTO (loss declared due to a timeout)
relative to the upstream loss.

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
in {{endtoendloss}}, then either the Q run length is too short for the amount of
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

Observer loss affects upstream loss rate measurement, since it causes the
observer to account for fewer packets in a block of identical Q bit values (see
{{upstreamloss)}).  The end-to-end loss rate measurement, however, is unaffected
by the observer loss, since it is a measurement of the fraction of packets with
the set L bit value, and the observer loss would affect all packets equally (see
{{endtoendloss}}).

The need to adjust the upstream loss rate down to match end-to-end loss rate as
described in {{losscorrelation}} is a strong indication of the observer loss,
whose magnitude is between the amount of such adjustment and the entirety of the
upstream loss measured in {{upstreamloss}}. Alternatively, a high apparent
upstream loss rate could be an indication of significant reordering, possibly
due to packets belonging to a single connection being multiplexed over several
upstream paths with different latency characteristics.


# QUIC v1 Implementation

## Transport Parameter  {#tp}

The use of the loss bits is negotiated using a transport parameter:

loss_bits (0x1057):

: The loss bits transport parameter is an integer value, encoded as a
  variable-length integer, that can be set to 0 or 1 indicating the level of
  loss bits support.

When loss_bits parameter is present, the peer is allowed to use reserved bits in
the short packet header as loss bits if the peer sends loss_bits=1.

When loss_bits is set to 1, the sender will use reserved bits as loss bits if
the peer includes the loss_bits transport parameter.

A client MUST NOT use remembered value of loss_bits for 0-RTT connections.


## Short Packet Header  {#shortheader}

When sending loss bits has been negotiated, the reserved (R) bits are replaced
by the loss (Q and L) bits in the short packet header (see Section 17.3 of
{{QUIC-TRANSPORT}}).

~~~
    0 1 2 3 4 5 6 7
   +-+-+-+-+-+-+-+-+
   |0|1|S|Q|L|K|P P|
   +-+-+-+-+-+-+-+-+
~~~

sQuare Signal Bit (Q):

: The fourth most significant bit (0x10) is the sQuare signal bit, set as
  described in {{squarebit}}.

Loss Event Bit (L):
: The fifth most significant bit (0x08) is the Loss event bit, set as
  described in {{lossbit}}.


## Header Protection

Unlike the reserved (R) bits, the loss (Q and L) bits are not protected.  When
sending loss bits has been negotiated, the first byte of the header protection
mask used to protect short packet headers has its five most significant bits
masked out instead of three.

The algorithm specified in Section 5.4.1 of {{QUIC-TLS}} changes as
follows:

~~~
   else:
      # Short header: 3 bits masked
      packet[0] ^= mask[0] & 0x07
~~~


# Ossification Considerations  {#ossification}

Accurate loss reporting signal is not critical for the operation QUIC protocol,
though its presence in a sufficient number of connections is important for the
operation of networks.

The loss bits are amenable to "greasing" described in {{GREASE}} and MUST be
greased.  The greasing should be accomplished similarly to the Latency Spin bit
greasing in {{QUIC-TRANSPORT}}.  Namely, implementations MUST NOT include
loss_bits transport parameter for a random selection of at least one in every 16
QUIC connections.

It is possible to observe packet reordering near the edge of the square signal.
A middle box might observe the signal and try to fix packet reordering that it
can identify, though only a small fraction of reordering can be fixed using this
method.  Latency spin bit signal edge can be used for the same purpose.


# Security Considerations

In the absence of packet loss, the Q bit signal does not provide any information
that cannot be observed by simply counting packets transiting a network
path. The L bit signal discloses internal state of the protocol's loss detection
machinery, but this state can often be gleamed by timing packets and observing
congestion controller response. Hence, loss bits do not provide a viable new
mechanism to attack QUIC data integrity and secrecy.

## Optimistic ACK Attack

A defense against an Optimistic ACK Attack {{QUIC-TRANSPORT}} involves a
sender randomly skipping packet numbers to detect a receiver acknowledging
packet numbers that have never been received. The Q bit signal may inform the
attacker which packet numbers were skipped on purpose and which had
been actually lost (and are, therefore, safe for the attacker to
acknowledge). To use the Q bit for this purpose, the attacker must first
receive at least an entire Q run of packets, which renders the attack
ineffective against a delay-sensitive congestion controller.

For QUIC v1 connections, if the attacker can make its peer transmit data using a
single large stream, examining offsets in STREAM frames can reveal whether
packet number skips are deliberate. In that case, the Q bit signal provides no
new information (but it does save the attacker the need to remove packet
protection). However, an endpoint that communicates using {{DATAGRAM}} and uses
a loss-based congestion controller MAY shorten the current Q run by the number
of skipped packets. For example, skipping a single packet number will invert the
sQuare signal one outgoing packet sooner.


# Privacy Considerations

To minimize unintentional exposure of information, loss bits provide an explicit
loss signal -- a preferred way to share information per {{!RFC8558}}.

{{QUIC-TRANSPORT}} allows changing connection IDs in the middle of a QUIC
connection to reduce the likelihood of a passive observer linking old and new
subflows to the same device. Hence, a QUIC implementation would need to reset
all counters when it changes connection ID used for outgoing packets.  It would
also need to avoid incrementing Unreported Loss counter for loss of packets sent
with a different connection ID.

Accurate loss information allows identification and correlation of network
conditions upstream and downstream of the observer. This could be a powerful
tool to identify connections that attempt to hide their origin networks, if the
adversary is able to affect network conditions in those origin networks.
Similar information can be obtained by packet timing and inferring congestion
controller response to network events, but loss information provides a clearer
signal.

Implementations MUST allow administrators of clients and servers to disable loss
reporting either globally or per QUIC connection.  Additionally, as described in
{{ossification}}, loss reporting MUST be disabled for a certain fraction of all
QUIC connections.


# IANA Considerations

This document registers a new value in the QUIC Transport Parameter Registry:

Value: 0x1057 (if this document is approved)

Parameter Name: loss_bits

Specification: Indicates that the endpoint supports loss bits. An endpoint that
   advertises this transport parameter can receive loss bits. An endpoint that
   advertises this transport parameter with value 1 can also send loss bits.


# Change Log

## Since version 02

* Add QUIC v1 negotiation: transport parameter, header protection change, IANA
  Considerations
* Add Optimistic ACK Attack Defense to Security Considerations
* Expand Privacy Considerations
* Clarify Q run length selection


## Since version 01
* Add reference to RFC7713

## Since version 00
* Rewrote to base this draft on {{LOSSBITS}}


# Acknowledgments

The sQuare signal bit was originally specified by Kazuho Oku in early proposals
for loss measurement and is an instance of the "alternate marking" as defined in
{{?RFC8321}}.

Many thanks to Christian Huitema for pointing out the interaction of Q bit and
Optimistic ACK Attack defence.
