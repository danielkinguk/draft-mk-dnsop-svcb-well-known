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
  - DNS-AID
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
    org: Lancaster University
    email: "d.king@lancaster.ac.uk"
normative:
  RFC2119:
  RFC8174:
  RFC3986:
  RFC5234:
  RFC7405:
  RFC8615:
  RFC9460:
informative:
  RFC9727:
  I-D.mozleywilliams-dnsop-dnsaid:

--- abstract

This document proposes the "well-known" Service Parameter Key (SvcParamKey)
for SVCB and HTTPS resource records. It carries one or more well-known URI
paths, for example the path to an agent's capability descriptor. It defines
the parameter format and requests its registration with IANA.

--- middle

# Introduction

DNS for AI Discovery (DNS-AID) {{I-D.mozleywilliams-dnsop-dnsaid}} uses SVCB
records {{RFC9460}} to publish agent endpoints. It also uses a "well-known"
parameter to give the path to a capability descriptor.

This draft provides a separate definition of that parameter. It carries
full paths under "/.well-known/" {{RFC8615}}, including sub-paths where
the well-known registration allows them. Each service binding can advertise
its own paths. The application defines the resource format and meaning.

[Discussion Points]

1\. Encoding and retrieval rules still need agreement, followed by HTTPBIS review before a renewed allocation request.

2\. Full paths or suffixes? DNS-AID -02 Figure 1 uses
"agent-card.json" and Section 7.1 assumes the prefix. Figure 3 uses
"/not-well-known/other-card.json". This draft uses full well-known paths, as in the registration request. These will also need to agree.

3\. The dns-aid-core implementation accepts suffixes and other paths.
Should we retain that, or require full well-known paths?

## Requirements Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals,
as shown here.

# The "well-known" SvcParamKey {#param}

The "well-known" SvcParamKey advertises one or more well-known URI paths
{{RFC8615}}. A client MAY retrieve a resource at an advertised path. The
list may not include every resource available at the service.

## Wire Format {#wire}

The wire-format value for the "well-known" SvcParamKey consists of one or more path entries concatenated without separators between entries. Each entry is encoded as:

* a 2-octet unsigned length in network byte order, giving the path length
  in octets; then

* the path itself.

[Discussion Points]

4\. Is a two-octet length suitable? Check against existing
DNS-AID implementations.

5\. The dns-aid-core implementation uses a single string at key65409.
Any concerns with the proposed list and length encoding?

## Presentation Format {#presentation}

~~~
well-known="path[,path]*"
~~~

The value is a comma-separated list of one or more paths, using the
character-string and list escaping rules of Section 2.1 and Appendix A.1
of {{RFC9460}}. Empty items and whitespace around separators are not allowed.

Each decoded path MUST match this ABNF {{RFC5234}} {{RFC7405}}:

~~~
wk-path = %s"/.well-known/" segment-nz *( "/" segment )
~~~

The segment-nz and segment rules are from Section 3.3 of {{RFC3986}}.
Sub-paths are permitted where the well-known registration allows them
{{RFC8615}}. The value carries paths only; see {{resolving}} for the origin.

Non-ASCII characters MUST be percent-encoded. Invalid percent escapes and
segments equal to "." or "..", including their percent-encoded forms,
MUST be rejected. Other percent-encoded octets MUST be preserved without
decoding or normalisation.

Each decoded path is encoded as a length-prefixed entry ({{wire}}).
Conversion in either direction MUST preserve path octets and list order,
using the RFC 9460 escaping rules for presentation output.

## Examples {#examples}

The examples below use the requested mnemonic for illustration; no
allocation is implied, and the well-known names shown are not registered by
this document.

[Discussion Points]

6\. Check the DNS-AID ALPN examples. Are "a2a" and "mcp" TLS
ALPN identifiers, or labels for protocols over HTTP?

Should we use HTTP ALPN values here if that is clearer?

~~~
agent-name.example.com. 3600 IN SVCB 1 . (
    alpn="a2a,h2"
    well-known="/.well-known/agent-card.json"
)
agent-name.example.com. 3600 IN SVCB 1 host.provider.example (
    alpn="mcp,h2,h3"
    well-known="/.well-known/agent-card.json,/.well-known/api-catalog"
)
~~~

The api-catalog path is defined in {{RFC9727}}.

The first record advertises one path at the owner name. The second, with a
hosted TargetName, advertises two paths at that host. Their "well-known"
SvcParamValues are, respectively, the first entry below and both entries
concatenated:

~~~
00 1c | ASCII("/.well-known/agent-card.json")
00 18 | ASCII("/.well-known/api-catalog")
~~~

The path lengths are 28 and 24 octets; the values are 30 and 56 octets.
The following single-path value contains a literal comma:

~~~
well-known="/.well-known/foo/a\\,b"
~~~

Character-string decoding produces "/.well-known/foo/a\,b"; list decoding
produces "/.well-known/foo/a,b". Its wire value is:

~~~
00 14 | ASCII("/.well-known/foo/a,b")
~~~

# Interactions with Other SvcParamKeys

The "well-known" key MAY appear alongside "alpn", "ipv4hint", "ipv6hint",
and "port". See {{resolving}} for the use of the port.

[Discussion Points]

7\. Does DNS-AID use "port" for the descriptor URI too? For
HTTPS records it changes the connection port, not the origin.

8\. Descriptor retrieval in dns-aid-core currently uses port 443.
Should it use the SVCB port?

9\. If both "cap" and "well-known" are present, which takes precedence?

The "well-known" key MUST be ignored when it appears in an AliasMode SVCB
record.

Each ServiceMode record carries its own "well-known" value.

If "well-known" is listed in "mandatory", a client that does not implement
it MUST skip the record (Section 8 of {{RFC9460}}). This does not require
the client to retrieve every path.

# Resolving and Retrieving Advertised Paths {#resolving}

The origin against which an advertised path is resolved is not yet agreed.
For HTTPS records, Section 9 of {{RFC9460}} keeps the origin of the URI
being resolved. For SVCB records, the candidate is an "https" origin at
the TargetName (or the owner name when TargetName is "."), using the
"port" key or 443.

[Discussion Points]

10\. Original agent name or TargetName for the descriptor?
Also check TLSA lookup: DNS-AID -02 Section 3.2 uses TargetName;
Section 6.2 uses `_443._tcp.<owner>`.
Need to cover aliases, hosted agents and other ports/transports.

11\. Can the descriptor use a different host or port from the agent service?

12\. With several paths, which resource does "cap-sha256" cover?
Does DNS-AID need just one descriptor path for now?

# Security Considerations

The security and privacy considerations of {{RFC9460}} and {{RFC8615}}
apply. The "well-known" parameter is conveyed in the DNS and is subject to
the same integrity guarantees as the enclosing SVCB record. 

Publishers SHOULD sign records carrying this key with DNSSEC, as DNS-AID recommends,
so that a validating client can detect alteration.

A client MUST NOT relax endpoint authentication, destination restrictions,
or redirect policy because a path was advertised in the DNS.

[Discussion Points]

13\. If TargetName changes the origin, how do we know that host
is authorised to represent the agent?

14\. Its certificate alone does not establish this. Need to cover operation without DNSSEC validation.

15\. Should DNS-AID use the same authorisation rules for hosted descriptors
as for off-domain catalogue pointers?

Forged or hostile advertisements can induce unwanted requests or disclose
client interests. Implementations SHOULD bound retrieval work rather than
automatically fetch every advertised path. Publishing paths can also
disclose information about a service; publishers SHOULD keep advertisements
compact.

DNSSEC can authenticate publication of a path; it does not establish that
the resource at that path, or its contents, are trustworthy. Consumers MUST
treat an advertised path as metadata, not as a trust signal.

# IANA Considerations

IANA is requested to add the following entry to the "Service Parameter Keys
(SvcParamKeys)" registry, in the "DNS Service Bindings (SVCB)" registry
group, per the procedure defined in Section 14.3 of {{RFC9460}}:

| Number | Name | Meaning | Format Reference | Change Controller |
| --- | --- | --- | --- | --- |
| TBD | well-known | One or more well-known URI paths available at the service endpoint | (This document) | IETF |

This request does not register individual well-known URI suffixes. Testing
before allocation can use the private-use SvcParamKey range of {{RFC9460}}.

[Discussion Points]

16\. Update DNS-AID Section 7.1 to reference this definition once agreed.

--- back
