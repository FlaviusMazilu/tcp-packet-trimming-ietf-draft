# Summary of Changes — Draft v2

**Branch:** `draftv2`  
**Base:** `user/fmazilu/draft`  
**Commit:** `622fb83` — *Draft v2: single TCP Trimming option, DSCP profile negotiation, TCP-AO interactionn*

---

## Overview

This version replaces the two-option design (Trimming Permitted + Trimming NACK) with a single TCP option Kind whose meaning is inferred from its length and from the connection phase. It also adds DSCP-profile negotiation in the handshake, requires endpoints to recognize only the exact negotiated DSCP values, notes the possibility of extending trimming across administrative domains through DSCP translation, and clarifies interactions with TCP-AO, TCP Fast Open, MPTCP, and other TCP options.

```
draft-mazilu-tcpm-packet-trimming.md | 119 +++++++++++++++++++++++++++--------
 1 file changed, 93 insertions(+), 26 deletions(-)
```

---

## 1. Single TCP Trimming option

**Before:** Two separate TCP option Kinds were requested from IANA.

| Kind | Length | Meaning |
|------|--------|---------|
| TBD1 | 2 | Trimming Permitted |
| TBD2 | 6 | Trimming NACK |

**After:** One TCP option Kind with length-based discrimination.

| Length | Phase | Meaning |
|--------|-------|---------|
| 7 | SYN / SYN-ACK | Trimming Permitted (carries DSCP profile) |
| 6 | ESTABLISHED | Trimming NACK (carries 4-byte sequence number) |

A receiver MUST ignore any other length, MUST ignore Length=7 on non-SYN packets, and MUST ignore Length=6 on SYN packets.

---

## 2. DSCP profile negotiation

The Trimming Permitted option now carries a 5-byte packed DSCP profile containing six DSCP values:

1. `DSCP_TRIMMABLE`
2. `DSCP_TRIMMED`
3. `DSCP_CONTROL`
4. `DSCP_NO_TRIM`
5. `DSCP_TRIMMABLE_RETX`
6. `DSCP_TRIMMED_LASTHOP`

Packing uses 36 bits (6 bits per DSCP) plus 4 reserved bits in the most-significant nibble of the first profile byte:

```c
profile = ((DSCP_TRIMMABLE       & 0x3F) << 30) |
          ((DSCP_TRIMMED         & 0x3F) << 24) |
          ((DSCP_CONTROL         & 0x3F) << 18) |
          ((DSCP_NO_TRIM         & 0x3F) << 12) |
          ((DSCP_TRIMMABLE_RETX  & 0x3F) <<  6) |
          ((DSCP_TRIMMED_LASTHOP & 0x3F) <<  0);
/* profile is written as 5 bytes in network byte order */
```

### Inference of optional values

If the local system configuration does not provide an explicit value for an optional category, the endpoint infers it as follows:

* `DSCP_CONTROL` → `DSCP_TRIMMABLE`
* `DSCP_NO_TRIM` → 0 (CS0/DF)
* `DSCP_TRIMMABLE_RETX` → `DSCP_TRIMMABLE`
* `DSCP_TRIMMED_LASTHOP` → `DSCP_TRIMMED`

### Negotiation rule

Trimming is enabled **only if the Trimming Permitted option received from the peer exactly matches the option sent by the endpoint**. If the options differ in any bit, both endpoints MUST disable trimming for the connection.

---

## 3. Exact DSCP recognition

Endpoints MUST recognize only the exact DSCP values negotiated in the Trimming Permitted option. This rule is now explicit in:

* **Data Receiver Behavior:** A packet whose DSCP does not exactly match a negotiated value MUST be processed as ordinary TCP traffic, even if the value would be trimming-related in another deployment.
* **Checksum bypass:** The receiver only bypasses checksum validation when it observes the exact negotiated `DSCP_TRIMMED` or `DSCP_TRIMMED_LASTHOP` value.
* **NACK generation:** The receiver emits a NACK only when the packet is marked with the exact negotiated `DSCP_TRIMMED` or `DSCP_TRIMMED_LASTHOP` value.
* **DSCP Marking Strategy:** Endpoints and switches MUST use only the exact negotiated values when marking or recognizing trimming-related traffic.

---

## 4. Updated DSCP Marking Strategy

The DSCP Marking Strategy section now documents all six categories (previously only three). `DSCP_TRIMMABLE` and `DSCP_TRIMMED` are required; the other four (`DSCP_CONTROL`, `DSCP_NO_TRIM`, `DSCP_TRIMMABLE_RETX`, `DSCP_TRIMMED_LASTHOP`) are optional and use "MAY be applied" language. The original three-alternative discussion of return-path NACK DSCP is retained.

---

## 5. Inter-domain and Internet-scale extension

A new paragraph in the Introduction notes that the DSCP negotiation mechanism creates a foundation for extending packet trimming across multiple administrative domains, or even the public Internet, if domain-border devices translate DSCP values and rewrite the values carried in the Trimming Permitted option. Defining such translation behavior is explicitly left for future work.

---

## 6. New section: Interactions with Other TCP Extensions

### TCP Fast Open

Added a short note that the Trimming Permitted option MAY appear in SYN/SYN-ACK packets carrying TCP Fast Open data or cookies. Because those packets are sent before trimming is negotiated, any data in a SYN or SYN-ACK MUST use the default DSCP and MUST NOT be marked `DSCP_TRIMMABLE`.

### TCP Authentication Option (TCP-AO)

Added a new subsection explaining that a trimmed data packet cannot be authenticated because TCP-AO covers payload bytes the switch removes. An attacker injecting `DSCP_TRIMMED` packets toward a receiver can cause authenticated NACKs to be sent to a spoofed source, which then reduces its sending rate. Deployments requiring full TCP-AO authentication MUST NOT use trimming, or MUST rate-limit reactions to NACKs.

### Multipath TCP (MPTCP)

Added a subsection stating that trimming is negotiated per subflow and that a NACK received on one subflow MUST NOT be used to infer loss across the whole MPTCP connection.

### Other TCP Options and Extensions

Added a subsection noting that most TCP options are preserved in the trimmed header, while options that rely on payload integrity or authentication cannot be verified on trimmed packets.

---

## 7. Updated references

* Added RFC 5925 (TCP-AO), RFC 7323 (TCP Timestamps), RFC 7413 (TCP Fast Open), and RFC 8684 (MPTCP) to the informative references.
* Updated `UEC-SPEC` from *Ultra Ethernet Consortium Specification 1.0, 2025-06* to *Ultra Ethernet Consortium Specification 1.1, 2026-10*.

---

## 8. Updated IANA Considerations

**Before:** Two TCP option Kinds requested.

**After:** One TCP option Kind requested, with two defined lengths:

| Kind | Length | Meaning |
|------|--------|---------|
| TBD  | 7      | Trimming Permitted |
| TBD  | 6      | Trimming NACK |

The Differentiated Services Codepoints subsection now lists all six placeholder names.

---

## 9. Updated Security Considerations

* Renamed/reused the checksum-bypass section anchor (`{#risks-of-bypassing-the-tcp-checksum}`).
* Added a new subsection **"TCP-AO and Trimmed Packets"** describing the authenticated-NACK/spoofed-source attack scenario.

---

## Files changed

```
M draft-mazilu-tcpm-packet-trimming.md
```

No other files were modified.
