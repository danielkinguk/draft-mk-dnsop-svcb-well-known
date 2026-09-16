---
title: "An SVCB Service Parameter for Well-Known URI Paths"
abbrev: "SVCB well-known SvcParamKey"
category: std
docname: draft-mk-dnsop-svcb-well-known-00
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Individual Submission"
keyword:
  - SVCB
  - HTTPS
  - well-known
  - service parameter
venue:
  group: "Individual Submission"
  type: "Individual"
  mail: "dnsop@ietf.org"
author:
  -
    name: Jim Mozley
    org: Infoblox, Inc.
    email: "jmozley@infoblox.com"
  -
    name: Daniel King
    org: Old Dog Consulting
    email: "daniel@olddog.co.uk"
normative:
  RFC2119:
  RFC8174:
  RFC3986:
  RFC5234:
  RFC8615:
  RFC9460:
informative:
  RFC4033:
  RFC9461:
  RFC9727:
  I-D.mozleywilliams-dnsop-dnsaid:
  A2A:
    title: Agent2Agent (A2A) Protocol Specification
    author:
      - org: Linux Foundation
    target: https://a2a-protocol.org/latest/specification/
    date: false

--- abstract

This document defines the "well-known" Service Parameter Key (SvcParamKey)
for SVCB and HTTPS resource records. It carries one or more well-known URI
suffixes, identifying resources under "/.well-known/" that are available
at the service endpoint. It specifies the presentation and wire formats
and requests registration of the key with IANA.

--- middle

# Introduction

SVCB and HTTPS resource records {{RFC9460}} let a client learn connection
parameters for a service before connecting, such as the protocols it
supports and alternative endpoints. They do not say which resources are
available once connected.

Well-known URIs {{RFC8615}} give machine-readable metadata a conventional
location under "/.well-known/". The IANA "Well-Known URIs" registry holds
many such suffixes, for example "api-catalog" {{RFC9727}} and
"agent-card.json", registered by the Linux Foundation for the Agent2Agent
protocol {{A2A}}. A client that already knows the applicable suffix can
construct the URI itself, but it cannot learn from the DNS which of the
registered resources a particular service offers.

The "well-known" SvcParamKey closes that gap. A service binding advertises
the well-known URI suffixes available at its endpoint, so a client can
retrieve the resource it needs directly after resolution, without probing.
Each record carries its own list, so a hosted endpoint or an alternative
protocol can advertise different resources.

The parameter is defined here as a general primitive so that other 
specifications can reference it.

o DNS for AI Discovery {{I-D.mozleywilliams-dnsop-dnsaid}} is one user of
this parameter, advertising the location of an agent's capability
descriptor.

o [editors note] Authors will add other in progress work. 

## Requirements Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals,
as shown here.

# The "well-known" SvcParamKey {#param}

The "well-known" SvcParamKey advertises one or more well-known URI
suffixes {{RFC8615}}, each naming a resource under "/.well-known/" at the
service endpoint. A client MAY retrieve any advertised resource. The list
need not include every well-known resource available at the endpoint.

## Wire Format {#wire}

The wire-format SvcParamValue consists of one or more entries concatenated
without separators. Each entry is:

* a 2-octet unsigned length in network byte order, giving the length of
  the suffix in octets; then

* the suffix itself.

A decoder reads entries until the SvcParamValue is exhausted. A value that
ends inside an entry, or that contains a zero-length entry, does not have
the expected format, so the RR is malformed (Section 2.2 of {{RFC9460}}).

## Presentation Format {#presentation}

~~~
well-known="suffix[,suffix]*"
~~~

The value is a comma-separated list of one or more suffixes, using the
character-string and list escaping rules of Section 2.1 and Appendix A.1
of {{RFC9460}}. Empty items and whitespace around separators are not
allowed.

Each decoded suffix MUST match this ABNF {{RFC5234}}:

~~~
wk-suffix = segment-nz *( "/" segment )
~~~

The segment-nz and segment rules are from Section 3.3 of {{RFC3986}}. A
suffix does not begin with "/", does not include the "/.well-known/"
prefix, and carries no query or fragment component. A suffix may extend
below a registered well-known name where that registration allows it.

Non-ASCII characters MUST be percent-encoded. Invalid percent escapes and
segments equal to "." or "..", including their percent-encoded forms,
MUST be rejected. Other percent-encoded octets MUST be preserved without
decoding or normalisation. A value containing a suffix that fails these
checks does not have the expected format, so the RR is malformed.

Each decoded suffix is encoded as one length-prefixed entry ({{wire}}).
Conversion in either direction MUST preserve suffix octets and list order,
using the {{RFC9460}} escaping rules for presentation output.

## Examples {#examples}

The examples use the requested key name for illustration; no allocation is
implied.

~~~
svc.example.com. 3600 IN HTTPS 1 . (
    alpn="h2,h3"
    well-known="api-catalog"
)
agent.example.com. 3600 IN SVCB 1 host.provider.example (
    alpn="h2,h3"
    well-known="agent-card.json,api-catalog"
)
~~~

The first record advertises one resource at the owner name,
"https://svc.example.com/.well-known/api-catalog". The second, with a
hosted TargetName, advertises two resources at that host. Their wire-format
SvcParamValues are, respectively, the first entry below and the second
and third entries concatenated:

~~~
00 0b | ASCII("api-catalog")
00 0f | ASCII("agent-card.json")
00 0b | ASCII("api-catalog")
~~~

The suffix lengths are 11 and 15 octets; the two values are 13 and 30
octets. The following single-suffix value contains a literal comma:

~~~
well-known="foo/a\\,b"
~~~

Character-string decoding produces "foo/a\,b"; list decoding produces
"foo/a,b". Its wire value is:

~~~
00 07 | ASCII("foo/a,b")
~~~

# Interactions with Other SvcParamKeys

The "well-known" key MAY appear alongside any other SvcParamKey. The
"alpn" key selects the protocol used to retrieve a resource; the "port"
key and the TargetName affect the origin ({{resolving}}).

Each ServiceMode record carries its own "well-known" value. The key MUST
be ignored when it appears in an AliasMode record.

If "well-known" is listed in "mandatory", a client that does not implement
it MUST skip the record (Section 8 of {{RFC9460}}). This does not oblige a
client to retrieve any advertised resource.

# Resolving and Retrieving Advertised Resources {#resolving}

A client forms the URI of an advertised resource by appending
"/.well-known/" and the suffix to the origin of the service.

For HTTPS records, the origin is that of the URI being resolved, as in
Section 9 of {{RFC9460}}; the "port" key changes the connection port, not
the origin. For SVCB records, the specification that maps the service to
SVCB defines the scheme and authority. Where it does not, the origin is
"https" at the TargetName (or the owner name when TargetName is "."),
using the "port" key or 443.

# Security Considerations

The security and privacy considerations of {{RFC9460}} and {{RFC8615}}
apply. The "well-known" parameter is conveyed in the DNS and has the same
integrity properties as the enclosing record.

The value is free text. It is deliberately not constrained to the suffixes
in the IANA "Well-Known URIs" registry: an enumerated encoding would
require DNS software to change each time a suffix is registered, and
neither authoritative servers nor resolvers are positioned to validate it.
A publisher, or an attacker able to alter records, can therefore advertise
any string. Clients MUST treat an advertised suffix as metadata, not as a
trust signal. A suffix that is well formed, or even registered, implies
nothing about whether a resource exists at that location or whether its
contents are trustworthy. A client MUST NOT relax endpoint authentication,
destination restrictions, or redirect policy because a suffix was
advertised in the DNS.

Publishers MAY sign records carrying this key with DNSSEC {{RFC4033}}.
DNSSEC lets a validating client confirm that the record was published by
the zone's authoritative source and was not altered in transit. It does
not validate the suffix or the resource it names.

Forged or hostile advertisements can induce unwanted requests or disclose
client interests. Implementations SHOULD bound retrieval work rather than
fetch every advertised resource. Publishing suffixes discloses information
about a service; publishers SHOULD keep advertisements compact.

# IANA Considerations

IANA is requested to add the following entry to the "Service Parameter Keys
(SvcParamKeys)" registry, in the "DNS Service Bindings (SVCB)" registry
group, per the procedure defined in Section 14.3 of {{RFC9460}}:

| Number | Name | Meaning | Format Reference | Change Controller |
| --- | --- | --- | --- | --- |
| TBD | well-known | One or more well-known URI suffixes available at the service endpoint | (This document) | IETF |

This request does not register individual well-known URI suffixes. Testing
before allocation can use the private-use SvcParamKey range of {{RFC9460}}.

# Open Issues
{:removeinrfc="true"}

This section lists questions on which the authors seek input, in
particular from the DNSOP and HTTPBIS working groups. It will be removed
before publication.

1. Suffix-only values. This document carries the suffix without the
   "/.well-known/" prefix, matching the IANA registry and saving octets.
   The original registration request and the current DNS-AID draft use
   full paths.

2. Origin for SVCB records. The default of an "https" origin at the
   TargetName is a proposal. Whether the owner name or the TargetName
   should be used, and how a client knows that a hosted TargetName is
   authorised to serve resources for the service, remain open.

3. Wire encoding. A list of length-prefixed entries is proposed, as used
   by other list-valued SvcParamKeys. An existing DNS-AID implementation
   carries a single string at a private-use key. Implementer input is
   welcome. The "dohpath" key {{RFC9461}} is the existing precedent for
   a path-valued parameter.

4. HTTPBIS review. The designated experts asked for consultation with
   HTTPBIS before a renewed allocation request.

--- back
