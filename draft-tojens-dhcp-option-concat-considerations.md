---
title: "DHCP Option Concatenation Considerations"
docname: draft-tojens-dhcp-option-concat-considerations
category: info
updates: 3396
area: INT
workgroup: dhc
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
v: 3
keyword:
 - concatenation
 - DHCP

author:
 -
    name: Tommy Jensen
    organization: Microsoft
    email: tojens.ietf@gmail.com
 -
    name: Milan Justel
    organization: Microsoft
    email: milanjustel@microsoft.com

normative: TODO: the refs

informative: TODO: the refs

--- abstract

DHCP has a length limit of 255 on individual options because of its one-byte
length field for options. To accommodate longer options, splitting option data
across multiple instances of the same Option Type is defined by {{!RFC3396}}. 
However, this mechanism is required to be supported for all options. This leads
to real-world implementations in the years since the RFC was published to
deviate from these requirements to avoid breaking basic functionality. This
document updates RFC 3396 to be more flexible regarding when DHCP agents are
required to concatenate options to reflect deployement experiences.


--- middle

# Introduction

{{!RFC3396}} defines how DHCP agents are to split and concatenate option data
within an DHCP message. This has proven to be valuable as more DHCP options are
defined that require support for concatenation as their data can exceed the 255
octet limit for options. Examples include {{?RFC4702}}, {{?RFC6731}} (as a MAY),
{{?RFC7291}}, {{?RFC8572}}, {{?RFC8973}}, and {{?RFC9463}}.

However, the way that {{!RFC3396}} defined concatenation is not the way it is
supported by major DHCP agent implementations today. New or existing
implementors of DHCP will find real-world behavior will differ from the
documented standard. This document updates {{!RFC3396}} to clarify how option
concatenation works in practice along with why it needs to differ from the
previous standard.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Previous Definition of Concatenation

{{!RFC3396}}  defines the terms "concatenation-requiring" and
"non-concatenation-requiring" to describe DHCP Options. While at multiple points
in the document it requires implementors to handle concatenation of Options
defined as concatenation-requiring, it also contains the following text which
effectively requires implementors to handle concatenation of either Option type:

  "However, an implementation which
  supports any concatenation-requiring option MUST be capable of
  concatenating received options for both concatenation-requiring and
  non-concatenation-requiring options."

# Implementation Challenges {challenges}

In combination with the only permitted use of duplicate instances of an option
type in the same DHCP message being option data splitting that then requires
concatenation, this means that there is no way to recover from real-world
DHCP deployment mistakes that can otherwise be handled under Jon Postel's
Robustness Principle {{?RFC791}}.

For example, if a DHCP server sends two instances of an option type with fixed
length, such as Option 51 (IP Address Lease Time) {{?RFC2132}}, concatenating
these into an eight-octet payload will result in a protocol violation: Option 51
has to be four octets long exactly. In common deployments of DHCP agents today,
we observe that this situation is handled by choosing one of the option
instances that has the correct length to accept as the Option 51 value and
ignoring the other instances. If implementations decided instead to strictly
adhere to always concatenating multiple instances of the same option type, this
would entirely block IPv4 network connectivity for the network stack.

# What Concatenation Intends to Achieve

DHCP option concatenation is intended to allow option data to exceed the 255
octet limit imposed by its single-octet length field. This can also be used for
splitting options across DHCP message section boundaries when the Overload
Option indicates that the sname field, file field, or both also contain option
data.

# Changes to How Options are Split

To support the intention of option concatenation without causing the challenges
described in {{challenges}}, this document updates {{!RFC3396}} to limit
concatenation to concatenation-requiring options. DHCP agents SHOULD NOT provide
multiple instances of an option type unless that option type is defined as
concatenation-requiring. To split non-concatenation-requiring options is
out-of-spec behavior that leads to implementation-specific message processing.

# Changes to How Options are Concatenated

When DHCP agents receive messages with split options that are concatenation-
required options, they MUST concatenate the duplicate concatenation-required
options as described in {{!RFC3396}}.

If DHCP agents sending messages never split non-concatenation-requiring options,
no further guidance would be needed. However, real-world deployments have seen
out-of-spec behavior that clients may wish to be defensive against and liberal
in parsing. Therefore, when DHCP agents receive messages with split options that
are not concatenation-requiring options, they MAY make best-effort attempts to
interpret the message or fail processing entirely as a protocol error. This is
implementation-specific, though some reasonable suggestions are broken down in
this section based on four types of situations involving non-concatenation-
requiring options still being split (even though they should not be).

## Duplicates of Options with Defined Concatenation Behavior

Some non-concatenation-requiring options may still define how they are to be
processed when DHCP agents receive multiple instances of the option type. In
this case, DHCP agents SHOULD follow the guidance defined by the option's
standard.

## Duplicates of Fixed-length Options

Fixed-length options are options that only allow a single length value, such as
Option 51 Lease Time which can only be four octets long. DHCP agents receiving
messages with more than one instance of fixed-length non-concatenation-requiring
options MAY choose to attempt processing one of the instances if it has the
correct length.

## Duplicates of Multiple-of-fixed-length Options

Multiple-of-fixed-length options are options that are lists of fixed-length
elements, such as Option 6 DNS Name servers which MUST be a multiple of four
octets long. DHCP agents receiving messages with more than one instance of
multiple-of-fixed-length options MAY choose to attempt processing one of the
instances if it has a correct length.

It MAY also choose to concatenate the options if the length of the concatenated
option data is a correct length (which may indicate a need to split a long list
because the total length is longer than 255 octets). DHCP agents MAY choose to
only do this if the length of the concatenated option data is greater than 255
octets if it wants to reduce how permissive it is.

## Duplicates of Arbitrary-length Options

Arbitrary-length options are options that may have any non-zero octet length,
such as Option 114 Captive-Portal {{?RFC8910}}. DHCP agents receiving messages
with more than one instance of arbitrary-length options MAY concatenate or
choose one instance to process. It MAY choose to do some validation of the
content that would result from each approach, meaning if the data only makes
sense in the context of the option's definition, it could use that to decide
which approach to take.

# Security Considerations

This document changes the conditions under which a DHCP agent accepts or rejects
option data. The only way this might reduce a DHCP agent's security posture
would be if it would have previously refused to process data from the network
that it will now process. However, this was always possible for an attacker
crafting DHCP messages. Any attacker capable of creating a malformed message
could instead craft a well-formed message (which would be processed in the
same way before and after this document). Therefore, this document does not
introduce any additional security considerations beyond the previous
definitions of DHCP and option concatenation.


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

Thank you to Dieter Siegmund, Stuart Cheshire, and Ted Lemon for their comments and suggestions.
