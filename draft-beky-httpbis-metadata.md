---
title: "METADATA frame for HTTP/2 and HTTP/3"
abbrev: "METADATA"
docname: draft-beky-httpbis-metadata-latest
date: {DATE}
category: std

ipr: trust200902
area: "Applications and Real-Time"
workgroup: "HTTP"
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: "B. Béky"
    name: "Bence Béky"
    organization: "Google LLC"
    email: bnc@google.com

    ins: "B. Roy"
    name: "Biren Roy"
    organization: "Google LLC"
    email: birenroy@google.com

normative:

informative:


--- abstract

This document describes a mechanism to send meta information over HTTP/2 and
HTTP/3 connections.


--- middle

# Introduction

HTTP/2 and HTTP/3 connections are capable of transporting multiple HTTP
messages, which are composed of field sections and bodies.  This document
described a mechanism to convey additional information about HTTP messages or
the entire connection, in a way that does not change HTTP semantics, over the
same connection.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# METADATA frame

Both HTTP/2 and HTTP/3 specifications allow the protocol to be extended.

TODO https://httpwg.org/specs/rfc7540.html#extensibility Section 5.5
TODO https://www.rfc-editor.org/rfc/rfc9114.html#name-extensions-to-http-3
Section 9

This document defines a new frame type: METADATA.  The payload of a METADATA
frame is an encoded list of key-value pairs.  METADATA frames do not change
HTTP semantics.

## METADATA HTTP/2 frame

The type of METADATA HTTP/2 frame is 0x4d.

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

  : END_METADATA (0x04): When set, the END_METADATA flag indicates that this
  frame contains the entire encoded list of key-value pairs and is not followed
  by any CONTINUATION frames.

  A METADATA frame without the END_METADATA flag set MUST be followed by a
  CONTINUATION frame for the same stream. A receiver MUST treat the receipt of
  any other type of frame or a frame on a different stream as a connection error
  (Section 5.4.1) of type PROTOCOL_ERROR.  In this case, the CONTINUATION frame
  or frames will carry the remaining of the encoded key-value pairs instead of
  an encoded header block.  The CONTINUATION frame with the last fragment of the
  encoded key-value pairs has the END_HEADERS frag set to indicate the
  completion of the encoded key-value pairs.  TODO: find a succinct name for
  encoded key-value pairs that clearly reflects that these are not headers.

METADATA frames are allowed on any stream.  METADATA frames on stream 0 carry
information pertaining to the whole connections.  METADATA frames on any other
stream are associated with the exchange carried by that stream.

METADATA frames can be sent on a stream in any state.  METADATA frames do not
alter the state of a stream.

METADATA frames obey the maximum frame size set by SETTINGS_MAX_FRAME_SIZE.

METADATA frames are not subject to flow control.

The payload of METADATA frames is a list of key-value pairs encoded using HPACK
instructions.  An endpoint MUST not use any HPACK instructions that change the
dynamic table.

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

The payload of METADATA frames is a list of key-value pairs encoded using QPACK
representations.  An endpoint MUST not use any QPACK representations that
reference the dynamic table.  Therefore the Required Insert Count is be zero,
and decoding METADATA frame payloads do not elicit instructions on the QPACK decoder
stream.

# Negotiating METADATA

This document defines a new HTTP/2 setting identifier, SETTINGS_ENABLE_METADATA,
with value 0x4d44.  It also defines a new HTTP/3 setting identifier,
SETTINGS_ENABLE_METADATA, with value 0x4d44.  

An endpoint that supports METADATA frames SHOULD advertise that by sending
SETTINGS_ENABLE_METADATA with value 1 on each connection.  A value of 0
indicates that the endpoint does not support METADATA frames.  A value other
than 0 or 1 MUST not be sent.  For HTTP/2, SETTINGS_ENABLE_METADATA MUST not be
sent in any SETTINGS frame other than the first one.

An endpoint MAY send METADATA frames before it learns that the peer supports
them.  For example, a proxy might chose to forward METADATA frames, or it might
chose to buffer them, before it receives a SETTINGS frame.  An endpoint SHOULD
NOT send METADATA frames after it learns that the peer does not support them.

# Security Considerations

TODO Security


# IANA Considerations

TODO IANA: two frame types and two setting identifiers.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
