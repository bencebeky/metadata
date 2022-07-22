---
title: "METADATA frame for HTTP/2 and HTTP/3"
abbrev: "METADATA"
docname: draft-beky-httpbis-metadata-latest
date:
category: std
submissiontype: IETF
number:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "HTTP"
keyword:
  - http
  - metadata
venue:
  group: "HTTP"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "bencebeky/metadata"
  latest: "https://bencebeky.github.io/metadata/draft-beky-httpbis-metadata.html"
author:
 -
    ins: "B. Béky"
    name: "Bence Béky"
    organization: "Google LLC"
    email: bnc@google.com
 -
    ins: "B. Roy"
    name: "Biren Roy"
    organization: "Google LLC"
    email: birenroy@google.com
normative:
  H2:
    =: RFC9113
    display: HTTP/2
  H3:
    =: RFC9114
    display: HTTP/3


--- abstract

This document describes a mechanism to send meta information over HTTP/2
and HTTP/3 connections that refers to either the
entire connection or a specific stream without changing the semantics of the
HTTP messages.  This mechanism can be used, for example, to gather information
for accounting or logging purposes.


--- middle

# Introduction

HTTP/2 {{H2}} and HTTP/3 {{H3}} connections are capable of transporting multiple HTTP
messages, which are composed of field sections and bodies.  This document
describes a mechanism to convey additional information about HTTP messages or
the entire connection, in a way that does not change HTTP semantics, over the
same connection.  For instance, an endpoint may wish to convey the CPU cost or
other loadbalancing information for a particular HTTP message, or perhaps
certain statistics for a particular HTTP message or for the connection as a
whole.  Applications may wish to provide such information without affecting HTTP
messages themselves. These are some non-exhaustive examples of use cases that
may be well served by the METADATA frame.

METADATA frames convey information to the next hop; they are explicitly not designed
as an end-to-end mechanism.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# METADATA frame

Both HTTP/2 and HTTP/3 specifications allow the protocol to be extended, see
{{Section 5.5 of H2}} and {{Section 9 of H3}}.

This document defines a new frame type: METADATA.

The METADATA frame can be used to transmit a metadata block, which is an encoded
list of key-value pairs. Each key and value is a sequence of bytes with no
restriction on the allowed values. The encoded block is packaged as the payload
of one or more frames.

An endpoint MAY transmit multiple metadata blocks on the same stream.

METADATA frames do not change HTTP semantics.

## METADATA HTTP/2 frame

The type of the METADATA HTTP/2 frame is 0x4d.

~~~~~~~~~~ ascii-art
METADATA HTTP/2 Frame {
  Length (24),
  Type (8) = 0x4d,

  Flags (8),

  Reserved(1),
  Stream Identifier (31),

  Encoded key-value pairs (..),
}
~~~~~~~~~~
{: title="METADATA HTTP/2 frame"}

The METADATA frame defines the following flag:

**END_METADATA (0x04)**:
  : When set, the END_METADATA flag indicates that this frame ends the logical
  metadata block.

  : A METADATA frame without the END_METADATA flag set MUST be followed by a
  another METADATA frame on the same stream.  However, METADATA frames MAY be
  interleaved with non-METADATA frames on the same stream, or frames of any type
  on different streams.

METADATA frames are allowed on any stream.  METADATA frames on stream 0 carry
information pertaining to the whole connection.  METADATA frames on any other
stream are associated with the exchange carried by that stream.

METADATA frames do not alter the state of a stream.  METADATA frames MUST NOT
be sent on a stream in the "closed" or "half closed (local)" state.  An endpoint
that receives METADATA for a stream in the “idle” state MAY choose to retain
the payload for a period of time, under the assumption that the stream will soon
transition to the “open” state.

A metadata block is the concatenation of the payloads of a sequence of one or
more METADATA frames, only the last of which has the END_METADATA flag set.   If
the transfer of the last metadata block cannot be completed due to the stream or
connection being closed before a METADATA frame with the END_METADATA flag, then
the incomplete metadata block SHOULD be discarded.  This SHOULD NOT affect
processing of previous metadata blocks on the same stream or connection.

METADATA frames obey the maximum frame size set by SETTINGS_MAX_FRAME_SIZE.

METADATA frames are not subject to flow control.

The metadata block of an HTTP/2 METADATA frame is encoded using HPACK
representations ({{!HPACK=RFC7541}}).  An endpoint MUST NOT use any HPACK
representations that change the dynamic table inside METADATA frames; any
METADATA frame with such representations SHOULD be treated as a connection
error.

## METADATA HTTP/3 frame

The type of METADATA HTTP/3 frame is 0x4d.

~~~~~~~~~~ ascii-art
METADATA HTTP/3 Frame {
  Type (i) = 0x4d,
  Length (i),

  Encoded key-value pairs (..),
}
~~~~~~~~~~
{: title="METADATA HTTP/3 frame"}

METADATA frames are allowed on any stream that uses HTTP/3 frames.  METADATA
frames on the control stream carry information pertaining to the whole
connection.  METADATA frames on a request stream or a push stream are associated
with the exchange carried by that stream.

The metadata block of a HTTP/3 METADATA frame is encoded using QPACK
representations ({{!QPACK=RFC9204}}).  An endpoint MUST NOT use any QPACK representations that
reference the dynamic table inside METADATA frames; any METADATA frame with such
representations SHOULD be treated as a connection error.  Therefore the
Required Insert Count MUST be zero, and decoding METADATA frame payloads do not
elicit instructions on the QPACK decoder stream.

# Negotiating METADATA

This document defines a new HTTP/2 setting identifier, SETTINGS_ENABLE_METADATA,
with value 0x4d44.  It also defines a new HTTP/3 setting identifier,
SETTINGS_ENABLE_METADATA, with value 0x4d44.

An endpoint that supports METADATA frames SHOULD advertise that by sending
SETTINGS_ENABLE_METADATA with value 1 on each connection.  A value of 0
indicates that the endpoint does not support METADATA frames.  A value other
than 0 or 1 MUST NOT be sent.  In HTTP/2, the initial value is 0; in HTTP/3,
the default value is 0.  For HTTP/2,
SETTINGS_ENABLE_METADATA MUST NOT be sent in any SETTINGS frame other than the
first one.

An endpoint MAY send METADATA frames before it learns that the peer supports
them.  For example, an HTTP intermediary might chose to forward METADATA frames, or it might
chose to buffer them, before it receives a SETTINGS frame.  An endpoint SHOULD
NOT send METADATA frames after it learns that the peer does not support them.

# Security Considerations

## Compression State Corruption

Since metadata blocks are encoded using HPACK or QPACK, they create the
possibility of changes to the compression state of a connection.  However,
METADATA frames are extension frames, and might be dropped by implementations or
intermediaries.  To avoid the problem of compression state desynchronization
between endpoints, HPACK or QPACK representations that change compression state
are disallowed.

## Denial-of-Service Considerations

Depending on the application, metadata blocks sent over HTTP/2 might be larger
than the negotiated SETTINGS_MAX_FRAME_SIZE.  To facilitate interoperability,
endpoints MUST respect the SETTINGS_MAX_FRAME_SIZE expressed by the peer when
encoding METADATA frames.

# IANA Considerations

## HTTP/2 Frame

This document adds an entry to the "HTTP/2 Frame Type" registry maintained at
<[](https://www.iana.org/assignments/http2-parameters/http2-parameters.xhtml)>
with the following parameters:

**Code:**
  : 0x4d

**Frame Type:**
  : METADATA

**Reference:**
  : This Document

## HTTP/2 Setting

This document adds an entry to the "HTTP/2 Settings" registry maintained at
<[](https://www.iana.org/assignments/http2-parameters/http2-parameters.xhtml)>
with the following parameters:

**Code:**
  : 0x4d44

**Name:**
  : SETTINGS_ENABLE_METADATA

**Initial Value:**
  : 0

**Reference:**
  : This Document

## HTTP/3 Frame

This document adds an entry to the "HTTP/3 Frame Types" registry maintained at
<[](https://www.iana.org/assignments/http3-parameters/http3-parameters.xhtml)>
with the following parameters:

**Value:**
  : 0x4d

**Frame Type:**
  : METADATA

**Status**:
  : provisional (permanent if this document is approved)

**Reference:**
  : This Document

**Change Controller**:
  : Bence Béky (IETF if this document is approved)

**Contact**:
  : bnc@google.com (HTTP_WG; HTTP working group; ietf-http-wg@w3.org if this document is approved)

**Notes**:
  : None

## HTTP/3 Setting

This document adds an entry to the "HTTP/3 Settings" registry maintained at
<[](https://www.iana.org/assignments/http3-parameters/http3-parameters.xhtml)>
with the following parameters:

**Value:**
  : 0x4d44

**Settings Name:**
  : SETTINGS_ENABLE_METADATA

**Default:**
  : 0

**Status**:
  : provisional (permanent if this document is approved)

**Reference:**
  : This Document

**Change Controller**:
  : Bence Béky (IETF if this document is approved)

**Contact**:
  : bnc@google.com (HTTP_WG; HTTP working group; ietf-http-wg@w3.org if this document is approved)

**Notes**:
  : None


--- back

# Acknowledgments
{:numbered="false"}

The authors would like to acknowledge Dianna Hu and Ian Swett for their
contributions to this document.
