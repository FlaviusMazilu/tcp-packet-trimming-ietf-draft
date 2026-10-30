diff --git a/draft-mazilu-tcpm-packet-trimming.md b/draft-mazilu-tcpm-packet-trimming.md
index 9ccd69e..259943e 100644
--- a/draft-mazilu-tcpm-packet-trimming.md
+++ b/draft-mazilu-tcpm-packet-trimming.md
@@ -49,19 +49,23 @@ informative:
   RFC2475:
   RFC2883:
   RFC3168:
+  RFC5925:
   RFC5961:
   RFC6298:
   RFC6582:
   RFC6675:
   RFC6994:
+  RFC7323:
+  RFC7413:
+  RFC8684:
   RFC8985:
   UEC-SPEC:
     target: https://ultraethernet.org/
-    title: Ultra Ethernet Consortium Specification 1.0
+    title: Ultra Ethernet Consortium Specification 1.1
     author:
       -
         org: Ultra Ethernet Consortium
-    date: 2025-06
+    date: 2026-10
 
 ...
 
@@ -96,40 +100,70 @@ The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "S
 
 # TCP Extensions for Packet Trimming
 
-All additional state and signaling required between the two communicating hosts are carried as TCP options.
+All additional state and signaling required between the two communicating hosts are carried as TCP options. This document defines a single TCP option, the TCP Trimming option, that is used for both capability negotiation and NACK signaling. The meaning of the option is inferred from its length and from the phase of the connection in which it appears.
+
+
+## TCP Trimming Option
+
+The TCP Trimming option uses a single TCP option Kind. Two lengths are defined:
+
+* Length=7: Trimming Permitted, exchanged only during the three-way handshake. It carries a DSCP profile that the two endpoints use to negotiate the DSCP values used for trimming.
+* Length=6: Trimming NACK, exchanged only on packets sent after the connection is established. It carries the sequence number of the trimmed packet.
+
+A receiver MUST ignore a TCP Trimming option whose length is not 6 or 7. A receiver MUST ignore a Length=7 option on a non-SYN packet and MUST ignore a Length=6 option on a SYN packet.
 
 
 ## Capability Negotiation (Trimming Permitted) {#capability-negotiation-trimming-permitted}
 
-If both endpoints support packet trimming, they SHOULD set the DSCP_TRIMMABLE codepoint on all data packets. However, to prevent a scenario where a sender marks packets as trimmable but the receiver does not understand how to process trimmed packets or issue NACKs, capability negotiation is required.
+If both endpoints support packet trimming, they SHOULD set the DSCP_TRIMMABLE codepoint on data packets once negotiation is complete. To prevent a scenario where a sender marks packets as trimmable but the receiver does not recognize the corresponding trimmed or control codepoints, capability negotiation is required.
+
+A TCP Trimming option with Length=7, Trimming Permitted, MUST be exchanged during the three-way handshake. Only if both the SYN and SYN+ACK packets carry an identical Trimming Permitted option will subsequent data packets be considered trimmable. This option MUST NOT be set on non-SYN packets.
 
-A new TCP option, Trimming Permitted, MUST be exchanged during the three-way handshake. Only if both the SYN and SYN+ACK packets carry this option will the subsequent data packets be considered trimmable. This option MUST NOT be set on non-SYN packets. It is similar in this regard to the SACK Permitted option {{RFC2018}}.
+The Trimming Permitted option carries a DSCP profile that contains the six DSCP values defined in [](#dscp-marking-strategy). The profile allows each endpoint to verify that the peer uses the same codepoints, avoiding silent failures caused by mismatched local DSCP configuration.
 
 TCP Trimming Permitted Option Format:
 
 ~~~~ ascii-art
-    +----------+----------+
-    | Kind=TBD | Length=2 |
-    +----------+----------+
+    +----------+----------+---------------------------+
+    | Kind=TBD | Length=7 |   DSCP Profile (5 bytes)  |
+    +----------+----------+---------------------------+
 ~~~~
 {: #fig-trimming-permitted-option}
 
+The DSCP Profile is a 36-bit big-endian value, padded with 4 reserved bits in the most-significant nibble of the first profile byte. The six DSCP values are packed as follows:
+
+~~~~ c
+profile = ((DSCP_TRIMMABLE       & 0x3F) << 30) |
+          ((DSCP_TRIMMED         & 0x3F) << 24) |
+          ((DSCP_CONTROL         & 0x3F) << 18) |
+          ((DSCP_NO_TRIM         & 0x3F) << 12) |
+          ((DSCP_TRIMMABLE_RETX  & 0x3F) <<  6) |
+          ((DSCP_TRIMMED_LASTHOP & 0x3F) <<  0);
+/* profile is written as 5 bytes in network byte order */
+~~~~
+
+If the local configuration does not provide an explicit value for an optional DSCP category, the endpoint infers it as follows:
+
+* DSCP_CONTROL defaults to DSCP_TRIMMABLE.
+* DSCP_NO_TRIM defaults to 0 (CS0/DF).
+* DSCP_TRIMMABLE_RETX defaults to DSCP_TRIMMABLE.
+* DSCP_TRIMMED_LASTHOP defaults to DSCP_TRIMMED.
+
+Trimming is enabled only when the Trimming Permitted option received from the peer exactly matches the option sent by the endpoint. If the options differ in any bit, both endpoints MUST disable trimming for the connection.
+
 
 ## Trimming NACK Option {#trimming-nack-option}
 
-When congestion occurs at a network switch along the path, the switch trims the payload of trimmable packets, updates the IP header to DSCP_TRIMMED, and forwards the header to the data receiver.
+When congestion occurs at a network switch along the path, the switch trims the payload of packets marked with a DSCP_TRIMMABLE value. The switch updates the IP header to the corresponding DSCP_TRIMMED or DSCP_TRIMMED_LASTHOP value according to the domain's configured mapping between trimmable and trimmed codepoints, and forwards the header to the data receiver. The handshake ensures that both endpoints agree on the values used in this mapping.
 
-Upon receiving a packet, the data receiver inspects the DSCP field. If the packet is marked as DSCP_TRIMMED, the receiver MUST immediately issue a new TCP packet with no data payload, carrying the Trimming NACK option. This option contains the sequence number of the trimmed packet. The receiver MAY include other options in this packet (e.g., SACK blocks {{RFC2018}}); there is not any requirement for NACK to be the only option set.
+Upon receiving a packet, the data receiver inspects the DSCP field. If the packet is marked with the exact negotiated DSCP_TRIMMED or DSCP_TRIMMED_LASTHOP value, the receiver MUST immediately issue a new TCP packet with no data payload, carrying a TCP Trimming option with Length=6 (Trimming NACK). This option contains the sequence number of the trimmed packet. The receiver MAY include other options in this packet (e.g., SACK blocks {{RFC2018}}); there is not any requirement for NACK to be the only option set.
 
 Trimming NACK Option Format:
 
 ~~~~ ascii-art
- Kind: TBD
- Length: 6 bytes
- Payload: 4 bytes (NACK sequence number)
-    +-----------+---------+------------+
-    | Kind= TBD | Length=6| Payload 4B |
-    +-----------+---------+------------+
+    +----------+----------+-------------------------+
+    | Kind=TBD | Length=6 | NACK Sequence (4 bytes) |
+    +----------+----------+-------------------------+
 ~~~~
 {: #fig-trimming-nack-option-format}
 
@@ -174,9 +208,11 @@ Switches may not trim the payload precisely at the TCP header boundary, as doing
 
 # Data Receiver Behavior upon Receiving a Trimmed Packet
 
+An endpoint MUST recognize only the exact DSCP values negotiated in the Trimming Permitted option. A packet whose DSCP field does not exactly match one of the negotiated values MUST be processed as ordinary TCP traffic, even if the DSCP value would belong to a trimming-related category in another deployment.
+
 ## Bypassing TCP Checksum Validation {#bypassing-tcp-checksum-validation}
 
-When a TCP data receiver identifies a trimmed packet by observing the DSCP_TRIMMED codepoint in the IP header, it MUST skip the standard TCP checksum verification for that packet, because trimming removes payload bytes that were covered by the original checksum. The receiver MUST process the packet only according to the trimmed-packet rules in this section; it MUST NOT treat the packet as ordinary TCP data.
+When a TCP data receiver identifies a trimmed packet by observing the exact negotiated DSCP_TRIMMED or DSCP_TRIMMED_LASTHOP value in the IP header, it MUST skip the standard TCP checksum verification for that packet, because trimming removes payload bytes that were covered by the original checksum. The receiver MUST process the packet only according to the trimmed-packet rules in this section; it MUST NOT treat the packet as ordinary TCP data.
 
 The checksum effects of trimming are described in [](#effects-of-trimming-on-tcp-checksum-and-payload). Security considerations for bypassing checksum validation are discussed in [](#risks-of-bypassing-the-tcp-checksum).
 
@@ -188,13 +224,15 @@ A trimmed packet may still contain payload bytes after the TCP header. The data
 
 ## Generating the NACK Packet
 
-Upon processing a valid trimmed packet, the data receiver SHOULD immediately generate and transmit a new TCP packet to the data sender. This packet MUST NOT contain any payload data. It MUST include the NACK TCP option.
+Upon processing a valid trimmed packet, the data receiver SHOULD immediately generate and transmit a new TCP packet to the data sender. This packet MUST NOT contain any payload data. It MUST include the TCP Trimming option with Length=6 (Trimming NACK).
 
 The NACK option payload is 4 bytes in length and MUST contain the sequence number extracted from the received trimmed packet.
 
 
 # Data Sender Behavior upon Receiving a NACK
 
+The data sender MUST mark trimmable data packets with the exact negotiated DSCP_TRIMMABLE value (or DSCP_TRIMMABLE_RETX when used). NACKs SHOULD arrive with the exact negotiated DSCP_CONTROL value.
+
 ## Identifying and Retransmitting the Lost Segment {#identifying-and-retransmitting-the-lost-segment}
 
 Upon receiving a TCP packet containing a NACK option, the data sender should extract the sequence number from the NACK option payload and locate the corresponding unacknowledged data segment in its retransmission queue.
@@ -213,22 +251,45 @@ A trimmed packet is an explicit signal of network congestion and packet loss. Th
 
 # DSCP Marking Strategy {#dscp-marking-strategy}
 
-This specification introduces the use of specific DSCP values to manage trimming behavior:
+This specification uses six DSCP categories, following the naming and semantics defined in the Ultra Ethernet Consortium specifications {{UEC-SPEC}}:
 
 * DSCP_TRIMMABLE: Applied by the sender to data packets eligible for trimming.
 * DSCP_TRIMMED: Applied by a network switch after removing a packet's payload.
+* DSCP_CONTROL: MAY be applied by endpoints to control traffic, including NACKs.
+* DSCP_NO_TRIM: MAY be applied to packets that must not be trimmed.
+* DSCP_TRIMMABLE_RETX: MAY be applied to retransmitted data packets to reduce the probability that they are trimmed.
+* DSCP_TRIMMED_LASTHOP: MAY be applied by the last-hop switch when trimming occurs at the final network hop.
+
+DSCP_TRIMMABLE and DSCP_TRIMMED are required for trimming. The other four categories are optional; when they are not configured, the defaults specified in [](#capability-negotiation-trimming-permitted) apply. All six values are negotiated using the Trimming Permitted option. Endpoints and switches MUST use only the exact negotiated values when marking or recognizing trimming-related traffic. The endpoint MUST recognize every DSCP value in the negotiated profile, including DSCP_TRIMMED_LASTHOP, because any switch along the path may set it even if the endpoint itself does not originate packets with that codepoint. A packet carrying any other DSCP value MUST NOT be treated as trimmable, trimmed, or trimming control traffic.
 
 Ideally, network switches should utilize at least two queues per port: a standard queue for DSCP_TRIMMABLE and regular traffic, and a high-priority queue for DSCP_TRIMMED and other control packets.
 
 Return Path DSCP for NACKs:
 
-A critical consideration is the DSCP value applied to the return NACK packets. These should ideally bypass standard data low priority queue to reach the sender with minimal latency.
+A critical consideration is the DSCP value applied to the return NACK packets. These should ideally bypass the standard data low-priority queue to reach the sender with minimal latency.
 
 * Alternative 1 (DSCP_TRIMMED): NACKs could use DSCP_TRIMMED. However, this requires demultiplexing at the sender to differentiate incoming NACKs from inbound data packets that were trimmed on the reverse path.
 * Alternative 2 (DSCP_CONTROL): The optimal approach, echoing Ultra Ethernet Transport protocols, is to map control packets to a dedicated DSCP_CONTROL codepoint. NACKs, pure ACKs, and FINs can utilize this to achieve improved latency.
 * Alternative 3 (DSCP_TRIMMABLE): NACKs are sent with standard data priority. This reduces the number of required DSCP values but sacrifices the latency advantage needed for rapid fast-recovery.
 
 
+# Interactions with Other TCP Extensions
+
+## TCP Fast Open
+
+The Trimming Permitted option MAY be carried in SYN and SYN+ACK packets that also carry TCP Fast Open {{RFC7413}} data or cookie requests. Because trimming is negotiated during the three-way handshake, SYN and SYN+ACK packets are sent before the endpoints have agreed to use trimming; therefore, any data carried in a SYN or SYN+ACK MUST use the connection's default DSCP and MUST NOT be marked DSCP_TRIMMABLE.
+
+
+## TCP Authentication Option (TCP-AO)
+
+When TCP-AO {{RFC5925}} is used, a trimmed data packet cannot be authenticated because the switch removes payload bytes that are covered by the TCP-AO authentication tag. An attacker that can inject DSCP_TRIMMED packets toward a receiver may cause the receiver to emit NACKs to a spoofed source. Those NACKs may themselves be authenticated by TCP-AO, so the victim source receives apparently valid NACKs and reduces its sending rate. Because the trigger packet could not be authenticated, TCP-AO does not protect against this attack. Implementations that require full TCP-AO authentication of every data packet MUST NOT use packet trimming, or MUST rate-limit reactions to NACKs.
+
+
+## Multipath TCP (MPTCP)
+
+When MPTCP {{RFC8684}} is used, the TCP Trimming option and the DSCP profile are negotiated independently on each subflow. A trimming event on one subflow affects only the sequence space and congestion state of that subflow; it does not imply loss on other subflows. An MPTCP sender MUST treat a NACK received on a subflow as a loss signal for that subflow only and MUST NOT use it to infer loss across the whole MPTCP connection.
+
+
 # Interaction with Existing Loss Detection Mechanisms {#interaction-with-existing-loss-detection-mechanisms}
 
 In a fully trimming-enabled network topology where every switch is capable of packet trimming, heuristic mechanisms like DupAck and RACK-TLP MAY be disabled. Packet Trimming naturally handles high degrees of packet reordering without generating the false positive retransmissions common to DupAck heuristics.
@@ -309,6 +370,7 @@ Event  TCP DATA SENDER                           TCP DATA RECEIVER
 
 (Implementation Note: To prevent redundant retransmissions in edge cases where a NACK arrives late but RACK has already retransmitted the segment, implementations MAY enforce a rule where NACK-driven retransmissions are ignored if the segment was already retransmitted within the last min_RTT.)
 
+
 # Congestion Control Considerations
 
 A trimmed packet is an explicit and authoritative indication of network congestion: a switch removed the payload precisely because a queue exceeded its threshold. A NACK therefore conveys both a loss to be repaired and a congestion signal. A trimming sender MUST treat the receipt of a NACK as an indication of congestion and reduce its sending rate, as it would for a segment loss detected by other means {{RFC5681}}.
@@ -319,22 +381,23 @@ A retransmission triggered by a NACK is ordinary TCP data: it MUST remain subjec
 
 Because a NACK is generated only after a switch has actually removed a payload, the corresponding loss is never spurious; a sender SHOULD NOT undo a congestion-window reduction caused by a NACK. Reductions caused by inference-based mechanisms that remain enabled follow their own rules; see [](#interaction-with-existing-loss-detection-mechanisms).
 
+
 # IANA Considerations
 
 ## TCP Options
 
-This document defines two new TCP options that require assignment from the "TCP Option Kind Numbers" registry {{RFC9293}}. IANA is requested to assign two values:
+This document defines one new TCP option that requires assignment from the "TCP Option Kind Numbers" registry {{RFC9293}}. IANA is requested to assign one value:
 
 | Kind | Length | Meaning            | Reference     |
 |------|--------|--------------------|---------------|
-| TBD1 | 2      | Trimming Permitted | This document |
-| TBD2 | 6      | Trimming NACK      | This document |
+| TBD  | 7      | Trimming Permitted | This document |
+| TBD  | 6      | Trimming NACK      | This document |
 
-The option formats are specified in [](#capability-negotiation-trimming-permitted) and [](#trimming-nack-option). Prior to the assignment of permanent Kind values, experimental deployments can carry these options using the shared experimental option Kinds and an Experiment Identifier as described in {{RFC6994}}.
+The option formats are specified in [](#capability-negotiation-trimming-permitted) and [](#trimming-nack-option). Prior to the assignment of a permanent Kind value, experimental deployments can carry these options using the shared experimental option Kinds and an Experiment Identifier as described in {{RFC6994}}.
 
 ## Differentiated Services Codepoints
 
-This document does not request any new assignments from the Differentiated Services Field Codepoints registry. The trimmable, trimmed, and control service classes used in this document (see [](#dscp-marking-strategy)) are realized with existing Differentiated Services codepoints {{RFC2474}} selected by local or domain policy. The names DSCP_TRIMMABLE, DSCP_TRIMMED, and DSCP_CONTROL are placeholders for those locally configured codepoints and do not denote new codepoint assignments.
+This document does not request any new assignments from the Differentiated Services Field Codepoints registry. The DSCP categories used in this document (see [](#dscp-marking-strategy)) are realized with existing Differentiated Services codepoints {{RFC2474}} selected by local or domain policy. The names DSCP_TRIMMABLE, DSCP_TRIMMED, DSCP_CONTROL, DSCP_NO_TRIM, DSCP_TRIMMABLE_RETX, and DSCP_TRIMMED_LASTHOP are placeholders for those locally configured codepoints and do not denote new codepoint assignments.
 
 
 # Security Considerations
@@ -351,7 +414,11 @@ An attacker who can inject a TCP segment carrying a forged Trimming NACK option
 
 To bound the congestion impact, the response to a NACK SHOULD be no larger than the response to an equivalent loss inferred by other means, so that a forged NACK is no more damaging than a forged duplicate acknowledgment. Implementations MAY additionally rate-limit the retransmissions and congestion reactions driven by NACKs.
 
-## Risks of Bypassing the TCP Checksum
+## TCP-AO and Trimmed Packets
+
+When TCP-AO {{RFC5925}} is in use, a trimmed data packet can no longer be authenticated because the authentication tag covers payload bytes that the switch has removed. An attacker that can inject DSCP_TRIMMED packets toward a receiver may cause the receiver to emit NACKs to a spoofed source. Those NACKs may be authenticated by TCP-AO, so the victim source receives apparently valid NACKs and reduces its sending rate. Thus, trimming weakens TCP-AO's protection against spoofed loss signals. Deployments that require full TCP-AO authentication for every data packet MUST NOT use packet trimming, or MUST treat the congestion reaction to NACKs as untrusted and rate-limit it.
+
+## Risks of Bypassing the TCP Checksum {#risks-of-bypassing-the-tcp-checksum}
 
 As described in [](#bypassing-tcp-checksum-validation), a receiver skips TCP checksum validation for a trimmed packet. This weakens the end-to-end integrity check for the header of that packet, so corruption of those header fields may pass undetected. The consequences are bounded by the restricted use the receiver makes of a trimmed packet. The receiver reads only the connection four-tuple and the sequence number, and MUST NOT deliver any payload from a trimmed packet to the application (see [](#ignoring-remaining-payload)).
 
