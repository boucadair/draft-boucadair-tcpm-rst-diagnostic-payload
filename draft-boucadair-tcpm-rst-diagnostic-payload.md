---
title: "TCP RST Diagnostic Payload"
abbrev: "RST Diagnostic Payload"
category: std

docname: draft-boucadair-tcpm-rst-diagnostic-payload-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: ""
workgroup: "TCP Maintenance and Minor Extensions"
keyword:
 - Service diagnostic


author:
 -
    fullname: Mohamed Boucadair
    organization: Orange
    email: mohamed.boucadair@orange.com

 -
    fullname: Tirumaleswar Reddy
    organization: Nokia
    country: India
    email: kondtir@gmail.com

 -
    fullname: Jason Xing
    organization: Tencent
    email: kerneljasonxing@gmail.com


contributor:

normative:

informative:
  IANA-TCP:
    title: Transmission Control Protocol (TCP) Parameters
    date: false
    target: https://www.iana.org/assignments/tcp-parameters/tcp-parameters.xhtml#

  Private-Enterprise-Numbers:
    title: Private Enterprise Numbers
    date: 4 May 2020
    target: https://www.iana.org/assignments/enterprise-numbers

--- abstract

   This document specifies a diagnostic payload format returned in TCP
   RST segments.  Such payloads are used to share with an endpoint the
   reasons for which a TCP connection has been reset.  Sharing this
   information is meant to ease diagnostic and troubleshooting.

--- middle

# Introduction

   A TCP connection {{!RFC9293}} can be reset by a peer for various
   reasons, e.g., received data does not correspond to an active
   connection.  Also, a TCP connection can be reset by an on-path
   service function (e.g., Carrier Grade NAT (CGN) {{?RFC6888}}, NAT64
   {{?RFC6146}}, or firewall) for several reasons.  Typically, a Network
   Address Translator (NAT) function can generate an RST segment to
   notify an endpoint upon the expiry of the lifetime of the
   corresponding mapping entry or because an RST segment was received
   from a peer ({{Section 2.2 of ?RFC7857}}).

   A TCP connection can also be
   closed by a user or an application at any time.  However, the peer
   that receives an RST segment does not have any hint about the reason
   that led to terminating the connection.  Likewise, the application
   that relies upon such a TCP connection may not easily identify the
   reason for the connection closure.  Troubleshooting such events at
   the remote side of the connection that receives the RST segment may
   not be trivial.

   This document fills this void by specifying a format of the
   diagnostic payload that is returned in an RST segment.  Returning
   such data is consistent with the provision in {{Section 3.5.3 of  !RFC9293}} for RST segments, especially:

   {: quote}
   >"TCP implementations SHOULD allow a received RST segment to
   >  include data (SHLD-2)."

   This document does not change the conditions under which an RST
   segment is generated ({{Section 3.5.2 of !RFC9293}}).

   The generic procedure for processing an RST segment is specified in
   {{Section 3.5.3 of !RFC9293}}.  Only the deviations from that procedure
   to insert and validate a diagnostic payload is provided in {{payload}}.
   {{examples}} provides a set of examples to illustrate the use of TCP RST
   diagnostic payloads.

   This document specifies the format and the overall approach to ease
   maintaining the list of codes while allowing for adding new codes as
   needed in the future and accommodating any existing vendor-specific
   codes.  An initial version of error codes is available in Table 2.
   However, the authoritative source to retrieve the full list of error
   codes is the IANA-maintained registry ({{causes}}).

   Investigation based on some major CGN vendors revealed
   that RSTs with data are not discarded and are translated according to
   any matching mapping entry. Moreover, implementation and experimental validation in Linux are detailed in {{sec-validation}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

   This document makes use of the terms defined in {{Section 4 of !RFC9293}}.

#  RST Diagnostic Payload {#payload}

   The RST diagnostic payload MUST be encoded using Concise Binary
   Object Representation (CBOR) Sequence {{!RFC8742}}.  The Concise Data
   Definition Language (CDDL) {{!RFC8610}} for the diagnostic payload is
   shown in {{cddl}}.

~~~~ CDDL
; This defines an array, the elements of which are to be used
; in a CBOR Sequence. There is exactly one occurrence.
diagnostic-payload = [magic-cookie, reason]
; Magic cookie to identify a payload that follows this specification
magic-cookie = 12345
; Reset reason details:
reason= {
  ? reason-code: uint,
  ? pen:uint,
  ? reason-description: tstr,
}
; Map Keys
reason-code = 1
pen = 2
reason-description = 3
~~~~
{: #cddl title='Structure of the RST Diagnostic Payload'}

   The RST diagnostic payload comprises a magic cookie that is used to
   unambiguously identify an RST payload that follows this
   specification.  It MUST be set to the RFC number to be assigned to
   this document.

   > Note to the RFC Editor: Please replace "12345" with the RFC number
   > assigned to this document.

   All parameters in the reason component of an RST diagnostic payload
   are mapped to their CBOR key values as specified in {{cbor}}.  The
   description of these parameters is as follows:

   reason-code:
   :  This parameter takes a value from an available registry
      such as the "TCP Failure Causes" registry ({{causes}}).

   pen:
   :  Includes a Private Enterprise Number
      [Private-Enterprise-Numbers].  This parameter MAY be included when
      the reason code is not taken from the IANA-maintained registry
      ({{causes}}), but from a vendor-specific registry.

   reason-description:
   :  Includes a brief description of the reset reason
      encoded as UTF-8 {{!RFC3629}}.  This parameter MUST NOT be included
      if a reason code is supplied.  This parameter is useful only for
      reset reasons that are not yet registered or for application-
      specific reset reasons.

   At least one of "reason-code" and "reason-description" parameters
   MUST be included in an RST diagnostic payload.  The "pen" parameter
   MUST be omitted if a reason code from the IANA-maintained registry
   ({{causes}}) fits the reset case.

   Malformed RST diagnostic payload messages that include the magic
   cookie MUST be silently ignored by the receiver.

   A peer that receives a valid diagnostic payload may pass the reset
   reason information to the local application in addition to the
   information (MUST-12) described in {{Section 3.6 of !RFC9293}}.  That
   information may also be logged locally, unless a local policy
   specifies otherwise.  How the information is passed to an application
   and how it is stored locally is implementation-specific.

   Per {{Section 3.6 of !RFC9293}}, one or more RST segments can be sent
   to reset a connection.  Whether a TCP endpoint elects to send more
   than one RST with only a subset of them that include the diagnostic
   payload is implementation-specific.

#  Some Examples {#examples}

   To ease readability, the CBOR diagnostic notation ({{Section 8 of !RFC8949}})
   with the parameter names rather than their CBOR key values
   in {{cbor}} is used in Figures 3, 4, 5, and 6.

   {{fig-2}} depicts an example of an RST diagnostic payload that is
   generated to inform the peer that the TCP connection is reset because
   an ACK was received from that peer while the connection is still in
   the LISTEN state ({{Section 3.10.7.2 of ?RFC9293}}).

~~~~ CDDL
   19 3039 # unsigned(12345)
   A1    # map(1)
      01 # unsigned(1)
      02 # unsigned(2)
~~~~
{: #fig-2 title='Example of an RST Diagnostic Payload with Reason Code (CBOR Encoding)'}

   {{fig-3}} depicts the same RST diagnostic payload as the one shown in
   {{fig-2}} but following the CBOR diagnostic notation.

~~~~ CBOR
   [
     12345,
     {
       1: 2
     }
   ]
~~~~
{: #fig-3 title='Example of an RST Diagnostic Payload with Reason Code (Diagnostic Notation)'}

   {{fig-4}} shows an example of an RST diagnostic payload that includes
   a free description to report a case that is not covered by an
   appropriate code from the IANA-maintained registry ({{causes}}).

~~~~ CBOR
   [
     12345,
     {
       3: "brief human-readable description"
     }
   ]
~~~~
{: #fig-4 title='Example of an RST Diagnostic Payload with Reason Description (Diagnostic Notation)'}

   An RST diagnostic payload may also be sent by an on-path service
   function.  For example, the following diagnostic payload is returned
   by a NAT function upon expiry of the mapping entry to which the TCP
   connection is bound ({{fig-5}}).

~~~~ CBOR
   [
     12345,
     {
       1: 8
     }
   ]
~~~~
{: #fig-5 title='Example of an RST Diagnostic Payload to Report Connection Timeout (Diagnostic Notation)'}

   {{fig-6}} illustrates an RST diagnostic payload that is returned by a
   peer that resets a TCP connection for a reason code 1234 defined by a
   vendor with the private enterprise number 32473.

~~~~ CBOR
   [
     12345,
     {
       1: 1234,
       2: 32473
     }
   ]
~~~~
{: #fig-6 title='Example of an RST Diagnostic Payload to Report Vendor-Specific Reason Code (Diagnostic Notation)'}

   {{fig-6}} uses the Enterprise Number 32473 defined for documentation
   use {{?RFC5612}}.

# IANA Considerations

##  RST Diagnostic Payload CBOR Key Values {#cbor}

   IANA is requested to create a new registry titled "RST Diagnostic
   Payload CBOR Key Values" under the "Transmission Control Protocol
   (TCP) Parameters" registry group {{IANA-TCP}}.

   The key value MUST be an integer in the 1-255 range.

   The assignment policy for this registry is "IETF Review" ({{Section 4.8 of !RFC8126}}).

   The structure of this subregistry and the initial values are provided
   in {{sub}}.


| Parameter Name     | CBOR  Key| CBOR Major Type & Information| Reference    |
|:------------------:|:--------:|:----------------------------:|:------------:|
| reason-code        |   1      | 0 unsigned                   |[ThisDocument]|
| pen                |   2      | 0 unsigned                   |[ThisDocument]|
| reason-description |   3      | 3 text string                |[ThisDocument]|
{: #sub title='Initial CBOR Keys'}

##  New Registry for TCP Failure Causes {#causes}

   This document requests IANA to create a new registry entitled "TCP
   Failure Causes" under the "Transmission Control Protocol (TCP)
   Parameters" registry group [IANA-TCP].

   Values are taken from the 1-65535 range.

   The assignment policy for this registry is "Expert Review"
   ({{Section 4.5 of !RFC8126}}).

   The designated experts may approve registration once they checked
   that the new requested code is not covered by an existing code and if
   the provided reasoning to register the new code is acceptable.  A
   registration request may supply a pointer to a specification where
   that code is defined.  However, a registration may be accepted even
   if no permanent and readily available public specification is
   available.

   The registry is initially populated with the values listed in
   {{initial}}.

 | Value | Description                                                    | Specification (if available)               |
 |:-----:|:---------------------------------------------------------------|:-------------------------------------------|
 | 1     | Illegal Option                                                 | {{Section 3.1 of !RFC9293}}                |
 | 2     | Desynchronized state                                           | {{Section 3.5.1 of !RFC9293}}              |
 | 3     | New data is received after CLOSE is called                     | Sections 3.6.1 and  3.10.7.1 of {{!RFC9293}}  |
 | 4     | ABORT Process                                                  | {{Section 3.10.5 of !RFC9293}}             |
 | 5     | Unexpected ACK received by non-synchronized state connection   | {{Section 3.10.7 of !RFC9293}}             |
 | 6     | Unexpected SYN in the window                                   | {{Section 3.10.7 of !RFC9293}}             |
 | 7     | Unexpected security compartment                                | {{Section A.1 of !RFC9293}}                |
 | 8     | Malformed Message                                              | [ThisDocument]                             |
 | 9     | Not Authorized                                                 | [ThisDocument]                             |
 | 10    | Resource Exceeded                                              | [ThisDocument]                             |
 | 11    | Network Failure                                                | [ThisDocument]                             |
 | 12    | Reset received from he peer                                    | [ThisDocument]                             |
 | 13    | Destination Unreachable                                        | [ThisDocument]                             |
 | 14    | Connection Timeout                                             | [ThisDocument]                             |
 | 15    | Too much outstanding data                                      | {{Section 3.6 of !RFC8684}}                |
 | 16    | Unacceptable performance                                       | {{Section 3.6 of !RFC8684}}                |
 | 17    | Middlebox interference                                         | {{Section 3.6 of !RFC8684}}                |
 {: #initial title='Initial TCP Failure Causes'}

   Note that codes in the 8-14 range can be used by service functions (Carrier Grade NAT (CGN), firewall, proxy, etc.).

#  Security Considerations

   {{!RFC9293}} discusses TCP-related security considerations. In
   particular, RST-specific attacks and their mitigations are discussed
   in {{Section 3.10.7.3 of !RFC9293}}.

   In addition to these considerations, it is RECOMMENDED to control the
   size of acceptable diagnostic payload and keep it as brief as
   possible.  The RECOMMENDED acceptable maximum size of the RST
   diagnostic payload is 255 octets.

   Also, it is RECOMMENDED to avoid leaking privacy-related information
   as part of the diagnostic payload (e.g., including a description such
   as "user X resets explicitly the connection" is not recommended).
   The "reason-description" string, when present, MUST NOT include any
   private information that an observer would not otherwise have access
   to.

   The presence of vendor-specific reason codes (Section 3) may be used
   to fingerprint hosts.  Such a concern does not apply if the reason
   codes are taken from the IANA-maintained registry.  Implementers are,
   thus, encouraged to register new codes within IANA instead of
   maintaining specific registries.

   The reason description, when present, MUST NOT be displayed
   to end users but is intended to be consumed by applications. Such a description
   may carry a malicious message to mislead the end-user.

--- back

#  Implementation and Experimental Validation in Linux {#sec-validation}

Questions and concerns have been raised regarding whether RST with payload affects the normal termination of flows across different software platforms, operating systems, middleboxes, etc. Even though {{Section 3.5.3 of !RFC9293}} explicitly allows this behavior, a full implementation is needed to widely verify if unexpected cases can happen in the real world.

The overall design in Linux is to pre-allocate a large enough zeroed buffer, put a reset reason code in the first byte and sent it out to verify whether the RST with payload can be possibly declined by any equipment in between two sides and the other side successfully parses the RST with payload.

## Implementation

The following implementation is accomplished on top of Linux 6.16:

**Payload Attachment**:
: Allocate a 1000-byte data payload attached to all generated RST packets.

**Reason Code Encoding**:
: The first byte of the payload is used to store a predefined reset reason code that is listed in include/net/rstreason.h file, while the remainder of the payload is zero-padded. The reason code is generated by the existing mechanism called [TCP reset reasons](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d5115a55ffb52).

**Handling of Reset Types**:
: The implementation distinguishes between the two primary reset scenarios in `tcp_send_active_reset()` and `tcp_v4_send_reset()` respectively:

   + For an **Active Reset**, initiated proactively by the local system, the payload is placed in the linear area of the socket buffer (`sk_buff`).
   + For a **Passive Reset**, sent in response to an unexpected or invalid incoming packet, the payload is stored in the non-linear (paged) area of the `sk_buff`.

Complete patch is shown in {{patch}}.

~~~~
{::include-fold ./implem/patch.txt}
~~~~
{: #patch title='Complete Patch'}

## Experimental Validation

To ensure a thorough evaluation, a multi-layered experimental methodology was designed, progressing from basic functional checks to complex, real-world compatibility and stability tests. The whole implementation has been deployed in Tencent's production environment for almost six months.

### Functional Verification

The basic functionality test is using iperf or iperf3 to construct a normal termination senario. The `tcpdump` tool with `-X` option effectively helps to show the `[RST+]` flag and the 1000-byte payload, confirming that the kernel correctly generated and transmitted the augmented RST packets.

Two servers, designated as Client A and Server B. The test is conducted as following:

1.  Start the `iperf3` server on Server B (`iperf3 -s`).
2.  Initiate a connection from Client A to Server B (`iperf3 -c [IP_of_B]`).
3.  After the connection is established, one of the `iperf3` processes is terminated using the `kill` command, triggering the kernel to send an RST packet.
4.  Simultaneously, `tcpdump` is run on either host to capture the reset packet using the filter: `'tcp[tcpflags] & tcp-rst != 0' -X -nn -vv -S`.

### Compatibility Verification

**Hardwares and Kernels**:
: Tests were conducted on various Linux distributions (e.g., Ubuntu, CentOS) with different kernel versions. The physical hosts were equipped with a range of network interface cards (NICs), including Intel `i40e`, `ixgbe`, and Mellanox `mlx5`.

**Virtualization**:
: The mechanism was tested in a virtualized environment where the VM used a `virtio_net` driver and the host employed DPDK to redirect packets in the host.

**Middleboxes**:
: Tests were performed with Layer 4 (L4) and Layer 7 (L7) gateways placed between the client and server to verify correct packet parsing and forwarding.

**Wide Area Network (WAN)**:
: The setup was tested over long-haul international links to simulate complex conditions, including China-to-Singapore (RTT > 30ms) and China-to-Germany (RTT > 200ms).

In conclusion, across all complex environment tests, the RST packets with payloads were successfully received by the peer. No instances of packets being dropped or mishandled by intermediate middleboxes, gateways, or diverse hardware and software configurations were observed.

# Acknowledgments
{:numbered="false"}

   The "diagnostic payload" name is inspired by {{Section 5.5.2 of  ?RFC7252}}
  that was cited by Carsten Bormann in the tcpm mailing list.

   Thanks to Jon Shallow for the comments.  Thanks also to Li Jinghui
   for the discussion.
