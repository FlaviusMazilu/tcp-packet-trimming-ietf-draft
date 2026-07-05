---
title: "TCP Packet Trimming Extension"
abbrev: "TCP Trimming"
category: std
stand_alone: false
ipr: trust200902
pi:
  toc: true
  sortrefs: false
  symrefs: true

docname: draft-mazilu-tcpm-packet-trimming-latest
submissiontype: IETF
date:
consensus: true
v: 3
area: Transport
workgroup: tcpm Working Group

author:
 -
    fullname: Flavius Romeo Mazilu
    initials: F.
    surname: Mazilu
    organization: UPB
    email: flavius.mazilu@stud.acs.upb.ro
 -
    fullname: Valentin Ionut Bobaru
    initials: V.
    surname: Bobaru
    organization: UPB
    email: valentin.bobaru@stud.acs.upb.ro
 -
    fullname: Costin Raiciu
    initials: C.
    surname: Raiciu
    organization: UPB
    email: costin.raiciu@upb.ro

normative:
  RFC2119:
  RFC8174:
  RFC9293:
  RFC2474:
  RFC5681:

informative:
  RFC2018:
  RFC2475:
  RFC2883:
  RFC3168:
  RFC5961:
  RFC6298:
  RFC6582:
  RFC6675:
  RFC6994:
  RFC8985:
  UEC-SPEC:
    target: https://ultraethernet.org/
    title: Ultra Ethernet Consortium Specification 1.0
    author:
      -
        org: Ultra Ethernet Consortium
    date: 2025-06

...

--- abstract

This document specifies a TCP extension that enables TCP endpoints to use packet trimming information when it is available. When switch buffers exceed a threshold, rather than silently dropping a packet, the switch trims the payload and forwards the header. This allows the destination to issue a deterministic Negative Acknowledgment (NACK), enabling faster, more deterministic loss recovery.


--- middle

# Introduction

TCP's congestion control relies primarily on packet loss as a signal of congestion, or Explicit Congestion Notification (ECN) {{RFC3168}} as a proactive measure. The accuracy and speed of these signals are vital for a mechanism to adapt quickly and utilize available bandwidth efficiently.

It is widely recognized that falling back to a Retransmission Timeout (RTO) {{RFC6298}} is undesirable. RTOs are overly conservative, drastically reduce the congestion window to one segment, and impose a high latency penalty (traditionally bounded to a 1-second minimum, though modern implementations often use around 300 ms).

For these reasons, fast retransmit {{RFC5681}} is always preferred, as it recovers from loss much faster. The duplicate acknowledgment (DupAck) heuristic was the first method introduced to achieve this, but it suffers in environments with high packet reordering, short flows, or when packets at the end of a flight are lost (tail loss). RACK-TLP {{RFC8985}} represents a more modern approach, utilizing a Tail Loss Probe (TLP) for tail losses and replacing sequence-based DupAck counting with time-based inferences derived from per-segment transmit timestamps.

However, because these approaches are fundamentally heuristics, they can fail to detect lost packets in time (forcing an RTO) or trigger false retransmissions, particularly in networks with significant packet reordering.

Packet Trimming offers a deterministic alternative. Originally introduced in the Ultra Ethernet Consortium's specifications {{UEC-SPEC}}, this approach is gaining support in modern network switch hardware. When switch buffers get full or exceed a certain threshold (as in Active Queue Management), rather than silently dropping the packet, the switch trims the payload. It retains only the bytes up to the transport header and forwards this "trimmed" packet to the destination.

By using Differentiated Services Code Point (DSCP) markings {{RFC2474}}, the network can inform the destination of the trimming event. The destination then issues a Negative Acknowledgment (NACK) carrying the specific sequence number of the affected packet, eliminating the need for the data sender to rely on heuristics to determine what to retransmit.

This document defines the TCP signaling and endpoint behavior needed to use this information. It does not specify how a network is configured for packet trimming, which packets a switch chooses to trim, or how DSCP values are provisioned inside an administrative domain. That network-side configuration is outside the scope of this document; deployments can refer to the Ultra Ethernet Consortium specifications {{UEC-SPEC}}.


# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.


# TCP Extensions for Packet Trimming

All additional state and signaling required between the two communicating hosts are carried as TCP options.


## Capability Negotiation (Trimming Permitted) {#capability-negotiation-trimming-permitted}

If both endpoints support packet trimming, they SHOULD set the DSCP_TRIMMABLE codepoint on all data packets. However, to prevent a scenario where a sender marks packets as trimmable but the receiver does not understand how to process trimmed packets or issue NACKs, capability negotiation is required.

A new TCP option, Trimming Permitted, MUST be exchanged during the three-way handshake. Only if both the SYN and SYN+ACK packets carry this option will the subsequent data packets be considered trimmable. This option MUST NOT be set on non-SYN packets. It is similar in this regard to the SACK Permitted option {{RFC2018}}.

TCP Trimming Permitted Option Format:

~~~~ ascii-art
    +----------+----------+
    | Kind=TBD | Length=2 |
    +----------+----------+
~~~~
{: #fig-trimming-permitted-option}


## Trimming NACK Option {#trimming-nack-option}

When congestion occurs at a network switch along the path, the switch trims the payload of trimmable packets, updates the IP header to DSCP_TRIMMED, and forwards the header to the data receiver.

Upon receiving a packet, the data receiver inspects the DSCP field. If the packet is marked as DSCP_TRIMMED, the receiver MUST immediately issue a new TCP packet with no data payload, carrying the Trimming NACK option. This option contains the sequence number of the trimmed packet. The receiver MAY include other options in this packet (e.g., SACK blocks {{RFC2018}}); there is not any requirement for NACK to be the only option set.

Trimming NACK Option Format:

~~~~ ascii-art
 Kind: TBD
 Length: 6 bytes
 Payload: 4 bytes (NACK sequence number)
    +-----------+---------+------------+
    | Kind= TBD | Length=6| Payload 4B |
    +-----------+---------+------------+
~~~~
{: #fig-trimming-nack-option-format}


## Basic Trimming Packet Flow

The following diagram illustrates the flow of packets when congestion occurs and a packet is trimmed:

~~~~ ascii-art
Event  TCP DATA SENDER                     TCP DATA RECEIVER
_____  ____________________________________________________________
  1.   [Sends Full Packet]        -->   [Receives Header Only]
        Format: IP|TCP|Payload              Format: IP|TCP
        DSCP:   DSCP_TRIMMABLE              DSCP:   DSCP_TRIMMED
        SEQ:    X                           SEQ:    X

                                          (Receiver detects Payload
                                          removal via DSCP change)

                                  <--    [Sends NACK Signal]
                                            Format: IP|TCP
                                            Options: NACK X

  2.   Receive NACK X
        (Triggers instant recovery
        of original payload)

        Retransmit segment X      -->    [Receives Full Packet]
        Format: IP|TCP|Payload              Format: IP|TCP|Payload
        DSCP:   DSCP_TRIMMABLE              DSCP:   DSCP_TRIMMABLE
        SEQ:    X                           SEQ:    X
~~~~
{: #fig-basic-trimming-packet-flow}


# Data Receiver Behavior upon Receiving a Trimmed Packet

## Bypassing TCP Checksum Validation {#bypassing-tcp-checksum-validation}

When a network switch experiences congestion, it trims the payload of a packet rather than dropping it entirely. In this scenario, the switch updates the relevant IP header fields: Total Length, Time to Live, Differentiated Services Code Point (DSCP), and IP Checksum. However, to avoid the high computational cost of parsing transport-layer headers, the switch does not update the TCP header {{RFC9293}}. Consequently, the TCP checksum remains calculated against the original, full payload, rendering it invalid for the trimmed packet.

For this reason, when a TCP data receiver identifies a trimmed packet, specifically, by observing the DSCP_TRIMMED codepoint in the IP header, it MUST skip the standard TCP checksum verification.

While bypassing checksum validation carries inherent risks, a trimmed packet is processed exclusively for the information contained within its TCP header: identifying the connection tuple and the sequence number. The receiver MUST NOT process or deliver any payload data from this packet to the application layer. The potential failure modes of an undetected header corruption in this case are minimal:

* Corrupted Port Numbers: If the source or destination port is corrupted, the receiver may fail to find a matching connection socket. The packet is silently discarded. If a false match occurs, the data receiver will issue a NACK for an unexpected sequence number, which the data sender will ignore, provided the sequence number does not match any unacknowledged packet. If the sender does falsely retransmit, the spurious retransmission will be mitigated by the receiver issuing a Duplicate Selective Acknowledgment (DSACK) {{RFC2883}}, allowing the sender to revert any unwarranted congestion window reductions.
* Corrupted Sequence Number: If the sequence number is corrupted, the receiver will issue a NACK for an incorrect sequence. As above, this results either in an ignored NACK or a spurious retransmission.


## Ignoring Remaining Payload {#ignoring-remaining-payload}

Network switches may not trim the payload precisely at the TCP header boundary, as doing so requires parsing variable-length TCP header to locate the data offset. Therefore, a trimmed packet may still contain a partial payload fragment.

Because the TCP checksum is invalid and the integrity of this data cannot be verified, the data receiver MUST NOT process or acknowledge any data payload remaining in the trimmed packet.


## Generating the NACK Packet

Upon processing a valid trimmed packet, the data receiver SHOULD immediately generate and transmit a new TCP packet to the data sender. This packet MUST NOT contain any payload data. It MUST include the NACK TCP option.

The NACK option payload is 4 bytes in length and MUST contain the sequence number extracted from the received trimmed packet.


# Data Sender Behavior upon Receiving a NACK

## Identifying and Retransmitting the Lost Segment {#identifying-and-retransmitting-the-lost-segment}

Upon receiving a TCP packet containing a NACK option, the data sender should extract the sequence number from the NACK option payload and locate the corresponding unacknowledged data segment in its retransmission queue.

Due to mechanisms such as TCP Segmentation Offload (TSO) or middlebox fragmentation, the sequence number reported in the NACK might not match the exact starting sequence number of a segment in the data sender's queue. The data sender should identify the segment where the sequence space covers the NACKed sequence number (i.e., Segment.SEQ <= NACK.SEQ < Segment.SEQ + Segment.LEN).

Once the appropriate segment is identified, the data sender MUST mark it as lost and SHOULD retransmit a SMSS starting from NACK sequence number immediately, subject to congestion window and flow control limits. If no unacknowledged segment covers the sequence number provided in the NACK (e.g., the segment has already been acknowledged or the NACK was spurious), the data sender MUST ignore the NACK and MUST NOT proceed to [](#entering-fast-recovery).


## Entering Fast Recovery {#entering-fast-recovery}

If the sender successfully identifies and marks a segment as lost based on a NACK, it MUST enter Fast Recovery (as defined in {{RFC5681}}), provided it is not already in a recovery phase.

A trimmed packet is an explicit signal of network congestion and packet loss. The data sender should anticipate that other packets within the same flight may have also been trimmed or dropped, which will be subsequently detected via additional NACKs, Duplicate ACKs, or RACK-TLP {{RFC8985}} timer expirations. By entering Fast Recovery, the data sender ensures that the congestion window is appropriately updated just once for the entire loss episode, conforming to the principle that losses, regardless of the detection mechanism, constitute a unified indication of congestion.


# DSCP Marking Strategy {#dscp-marking-strategy}

This specification introduces the use of specific DSCP values to manage trimming behavior:

* DSCP_TRIMMABLE: Applied by the sender to data packets eligible for trimming.
* DSCP_TRIMMED: Applied by a network switch after removing a packet's payload.

Ideally, network switches should utilize at least two queues per port: a standard queue for DSCP_TRIMMABLE and regular traffic, and a high-priority queue for DSCP_TRIMMED and other control packets.

Return Path DSCP for NACKs:

A critical consideration is the DSCP value applied to the return NACK packets. These should ideally bypass standard data low priority queue to reach the sender with minimal latency.

* Alternative 1 (DSCP_TRIMMED): NACKs could use DSCP_TRIMMED. However, this requires demultiplexing at the sender to differentiate incoming NACKs from inbound data packets that were trimmed on the reverse path.
* Alternative 2 (DSCP_CONTROL): The optimal approach, echoing Ultra Ethernet Transport protocols, is to map control packets to a dedicated DSCP_CONTROL codepoint. NACKs, pure ACKs, and FINs can utilize this to achieve improved latency.
* Alternative 3 (DSCP_TRIMMABLE): NACKs are sent with standard data priority. This reduces the number of required DSCP values but sacrifices the latency advantage needed for rapid fast-recovery.


# Interaction with Existing Loss Detection Mechanisms {#interaction-with-existing-loss-detection-mechanisms}

In a fully trimming-enabled network topology where every switch is capable of packet trimming, heuristic mechanisms like DupAck and RACK-TLP MAY be disabled. Packet Trimming naturally handles high degrees of packet reordering without generating the false positive retransmissions common to DupAck heuristics.

However, over the public Internet, it should not be assumed that every middlebox supports trimming. Consequently, DupAck and RACK-TLP MUST NOT be disabled. Packet Trimming does not hinder their functionality and can work cooperatively alongside these mechanisms.


## Interaction with RACK-TLP

Upon receiving a NACK, the data sender enters Fast Retransmit, retransmits the specified packet, and transitions to Fast Recovery. In this state, RACK-TLP continues to monitor for lost packets. RACK-TLP will not falsely retransmit the trimmed packet again because its internal RACK timer (based on rack.RTT + rack.reo_wnd) will not have expired for the newly retransmitted segment.

Scenario A: Mixed Drop and Trim Events

If a flight contains both dropped and trimmed packets, NACKs and SACKs cooperatively recover the flight:

~~~~ ascii-art
Event  TCP DATA SENDER                        TCP DATA RECEIVER
 _____  _________________________________________________________
   1.   Send P0, P1, P2, P3          -->
        [P1 dropped by network]
        [P2 trimmed by network]
                                     <--   Receive P0, ACK 0
                                     <--   Receive P2 (Trimmed),
                                                NACK 2, ACK 0
                                     <--   Receive P3, ACK 0, SACK 3

   2.   Receive ACK 0
        Receive NACK 2, ACK 0
        (NACK triggers immediate
         retransmission of P2)
        Retransmit P2                -->
                                     <--          Receive P2, ACK 0,
                                                     SACK 2, 3

   3.   Receive ACK 0, SACK 2, 3
        RACK-TLP timer expires
        [Logic: Now() - P1_ts > RTT + reo_wnd?]
        (Condition = YES)
        Retransmit P1                -->
                                     <--          Receive P1, ACK 3
~~~~
{: #fig-mixed-drop-and-trim-events}

Scenario B: Delayed NACK / Preemptive SACK

If a SACK arrives before a NACK, and the rack.RTT + rack.reo_wnd threshold has not been met, no retransmission occurs. When the NACK subsequently arrives, P2 is instantly recovered. The succeeding ACK+SACK for P2 provides the necessary time-lapse evidence for RACK to securely declare P1 lost and trigger its retransmission. This improves the standard RACK-TLP behavior which would otherwise, in this scenario, wait 2 RTTs for sending a TLP probe. By having the NACK trigger a retransmission which would bring more SACK information does the same thing as a TLP probe would do but faster, in only 1 RTT.

~~~~ ascii-art
Event  TCP DATA SENDER                           TCP DATA RECEIVER
 _____  ____________________________________________________________
   1.   Send P0, P1, P2, P3          -->
        [P1 dropped by network]
        [P2 trimmed by network]
                                     <--   Receive P0, ACK 0
                                     <--   Receive P2 (Trimmed),
                                                ACK 0, NACK 2
                                     <--   Receive P3, ACK 0, SACK 3

   2.   Receive ACK 0
        Receive ACK 0, SACK 3
        [No retransmission yet:
         Thresholds not met]

   3.   Receive NACK 2, ACK 0
        (Immediate recovery of P2)
        Retransmit P2                -->
                                     <--    Receive P2, ACK 0, SACK 2

   4.   Receive ACK 0, SACK 2
        (RACK: Since a later packet
         P2 has been delivered, P1
         is marked lost by time)
        Retransmit P1                -->
                                     <--      Receive P1, ACK 3
                                            (All data now contiguous)
~~~~
{: #fig-delayed-nack-preemptive-sack}

(Implementation Note: To prevent redundant retransmissions in edge cases where a NACK arrives late but RACK has already retransmitted the segment, implementations MAY enforce a rule where NACK-driven retransmissions are ignored if the segment was already retransmitted within the last min_RTT.)


## Interaction with DupAck (NewReno / SACK)

Regardless of whether the connection utilizes NewReno {{RFC6582}} or SACK-based recovery {{RFC6675}}, both standards mandate that a packet is retransmitted only once per Fast Recovery phase. Therefore, there is no risk of a DupAck retransmitting the exact same packet after a NACK redundantly.

In both approaches based on DupAck, multiple different packets can be retransmitted in the same Fast Recovery phase. This is useful for us because both DupAck and NACKs have the chance to retransmit different packets.

Scenario C: NACK and Partial ACKs (NewReno Style)

If P2 is trimmed and P3 is dropped, the NACK triggers the recovery of P2. The DupAcks associated with the trimmed packet are effectively absorbed because P2 is already being handled. When the Partial ACK confirming P2 arrives, NewReno {{RFC6582}} uses it to infer the loss of P3 and triggers the subsequent retransmission.

~~~~ ascii-art
Event  TCP DATA SENDER                            TCP DATA RECEIVER
 _____  ___________________________________________________________

  1.   Send P1, P2, P3, P4          -->
       [P2 trimmed]
       [P3 dropped by network]
                                    <--     Receive P1, ACK 1
                                    <--     Receive P2(Trimmed),
                                                ACK 1, NACK 2
                                    <--     Receive P4, ACK 1

  2.   Receive ACK 1
       Receive ACK 1, NACK 2
       NACK: Retransmit P2          -->
       Receive ACK 1
       (DupAck heuristic ignores P2
        as it is already recovering)
                                    <--     Receive P2, ACK 2
                                              [PARTIAL ACK:
                                               P3 still missing]

  3.   Receive ACK 2
       (Partial ACK triggers NewReno)
       Retransmit P3                -->
                                    <--     Receive P3, ACK 4
~~~~
{: #fig-nack-and-partial-acks}

Scenario D: SACK Scoreboard Updates

NACKs actively aid in updating the SACK scoreboard faster. In a complex loss event, the NACK allows for the immediate retransmission of the trimmed packet, while standard SACK processing {{RFC6675}} seamlessly handles the standard drops.

~~~~ ascii-art
Event  TCP DATA SENDER                            TCP DATA RECEIVER
 _____  ___________________________________________________________
  1.   Send P0, P1, P2, P3, P4      -->
       [P1 trimmed, P3 dropped]

  2.                                <--    Receive P0, ACK 0
                                    <--    Receive P1 (Trimmed),
                                                 ACK 0, NACK 1

                                    <--    Receive P2, ACK 0, SACK 2
                                    <--  Receive P4, ACK 0, SACK 2, 4

  3.   Receive ACK 0
       Receive ACK 0, NACK 1
       Fast Retransmit P1           -->

  4.   Receive ACK 0, SACK 2
       [SACK ignores P1; already sent]

  5.   Receive ACK 0, SACK 2, 4
       (Scoreboard indicates P3 loss)
       Retransmit P3                -->

  6.                                <--    Receive P1, ACK 2, SACK 4
                                              (P0-P2 now contiguous)

  7.                                <--    Receive P3, ACK 4
                                        (All segments accounted for)
~~~~
{: #fig-sack-scoreboard-updates}


# Congestion Control Considerations

A trimmed packet is an explicit and authoritative indication of network congestion: a switch removed the payload precisely because a queue exceeded its threshold. A NACK therefore conveys both a loss to be repaired and a congestion signal. A trimming sender MUST treat the receipt of a NACK as an indication of congestion and reduce its sending rate, as it would for a segment loss detected by other means {{RFC5681}}.

The precise magnitude of this reduction is left as future work. Because trimming reports loss on a per-segment basis, a sender learns the exact number of segments lost in a flight rather than merely inferring that at least one was lost, which may permit a more accurate response than a single multiplicative decrease per loss episode. Defining and evaluating such a response is an open item; this document requires only that the reaction to a NACK be no more aggressive than the standard response to an equivalent inferred loss {{RFC5681}}.

A retransmission triggered by a NACK is ordinary TCP data: it MUST remain subject to the congestion window and to the count of packets in flight, and MUST NOT bypass these limits. If the congestion window does not currently allow the retransmission, the retransmission is sent once the window permits.

Because a NACK is generated only after a switch has actually removed a payload, the corresponding loss is never spurious; a sender SHOULD NOT undo a congestion-window reduction caused by a NACK. Reductions caused by inference-based mechanisms that remain enabled follow their own rules; see [](#interaction-with-existing-loss-detection-mechanisms).

# IANA Considerations

## TCP Options

This document defines two new TCP options that require assignment from the "TCP Option Kind Numbers" registry {{RFC9293}}. IANA is requested to assign two values:

| Kind | Length | Meaning            | Reference     |
|------|--------|--------------------|---------------|
| TBD1 | 2      | Trimming Permitted | This document |
| TBD2 | 6      | Trimming NACK      | This document |

The option formats are specified in [](#capability-negotiation-trimming-permitted) and [](#trimming-nack-option). Prior to the assignment of permanent Kind values, experimental deployments can carry these options using the shared experimental option Kinds and an Experiment Identifier as described in {{RFC6994}}.

## Differentiated Services Codepoints

This document does not request any new assignments from the Differentiated Services Field Codepoints registry. The trimmable, trimmed, and control service classes used in this document (see [](#dscp-marking-strategy)) are realized with existing Differentiated Services codepoints {{RFC2474}} selected by local or domain policy. The names DSCP_TRIMMABLE, DSCP_TRIMMED, and DSCP_CONTROL are placeholders for those locally configured codepoints and do not denote new codepoint assignments.


# Security Considerations

The mechanisms in this document add a new control signal, the NACK, and rely on Differentiated Services markings to convey trimming events. Packet trimming is primarily intended for deployment within a single administrative domain, such as a data center, where the path is largely trusted. The analysis below also notes the residual risks that apply when traffic crosses a less trusted path.

## Forged NACK Injection

An attacker who can inject a TCP segment carrying a forged Trimming NACK option for an existing connection can attempt to force the sender to retransmit a segment, wasting bandwidth and provoking an unwarranted reduction of the congestion window. Several factors bound this attack:

* A NACK is acted upon only when it arrives on a segment that passes the connection's normal demultiplexing checks. An off-path attacker must therefore guess the connection's four-tuple, as for any spoofed TCP segment.
* As required in [](#identifying-and-retransmitting-the-lost-segment), the sender MUST ignore a NACK whose sequence number does not correspond to an outstanding unacknowledged segment. A forged NACK is effective only for an in-window sequence number, which confines the attack to the same blind in-window space analyzed in {{RFC5961}}.
* A retransmission provoked by a forged NACK carries only data the receiver legitimately expects; it cannot cause the receiver to accept incorrect data. The impact is limited to wasted bandwidth and an unnecessary congestion-window reduction.

To bound the congestion impact, the response to a NACK SHOULD be no larger than the response to an equivalent loss inferred by other means, so that a forged NACK is no more damaging than a forged duplicate acknowledgment. Implementations MAY additionally rate-limit the retransmissions and congestion reactions driven by NACKs.

## Risks of Bypassing the TCP Checksum

As described in [](#bypassing-tcp-checksum-validation), a receiver skips TCP checksum validation for a trimmed packet, because the checksum was computed over the original payload and is necessarily invalid once the payload has been removed. This weakens the end-to-end integrity check for the header of a trimmed packet, so corruption of those header fields may pass undetected.

The consequences are bounded by the restricted use the receiver makes of a trimmed packet. The receiver reads only the connection four-tuple and the sequence number, and MUST NOT deliver any payload from a trimmed packet to the application (see [](#ignoring-remaining-payload)). Undetected corruption therefore cannot deliver wrong data to the application; at worst it yields a NACK for the wrong connection or sequence number. A NACK that matches no connection is discarded, and a NACK with a wrong sequence number is bounded exactly as a forged NACK is: it is ignored unless it falls within the outstanding window, and even then it can only cause a spurious retransmission, which the receiver disambiguates with a Duplicate SACK {{RFC2883}}.

## DSCP Remarking and Priority Abuse

This specification places control traffic, including NACKs, and trimmed packets into higher-priority service classes than ordinary trimmable data (see [](#dscp-marking-strategy)). Where traffic can cross a trust boundary, this raises two DSCP-related concerns:

* Priority theft: an attacker that sets the control or trimmed-class codepoint on its own traffic could obtain preferential queueing, claiming an unfair share of capacity or degrading the latency of legitimate control traffic.
* Trimming evasion: an attacker that clears the trimmable codepoint on its flows could render them ineligible for trimming, escaping the congestion response that trimming imposes and competing unfairly with conforming flows.

Both are instances of the general Differentiated Services trust model {{RFC2474}} {{RFC2475}}: DSCP markings arriving from outside a trust boundary cannot be relied upon. A trimming-enabled domain MUST treat its boundary as a trust boundary and police or re-mark the DSCP of ingress traffic, so that the trimmable, trimmed, and control codepoints carry their intended meaning only within the domain that enforces them.


--- back
