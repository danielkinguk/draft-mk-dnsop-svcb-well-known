# An SVCB Service Parameter for Well-Known URI Paths

Source for **draft-mk-dnsop-svcb-well-known**, an individual Internet-Draft
defining the `well-known` SvcParamKey for SVCB and HTTPS records
([RFC 9460]).

## What it defines

The `well-known` parameter advertises one or more full paths under
`/.well-known/` ([RFC 8615]), so a client can fetch a resource such as an
agent's capability descriptor directly from the service binding.

```
agent-name.example.com. 3600 IN SVCB 1 host.provider.example (
    alpn="h2,h3"
    well-known="/.well-known/agent-card.json,/.well-known/api-catalog"
)
```

The draft specifies the presentation format, the wire format, conversion
between them, interactions with other SvcParamKeys, and how a client
resolves an advertised path. The application defines what is at each path.

## Authors

- Jim Mozley, Infoblox, Inc.
- Daniel King, Old Dog Consulting

[RFC 9460]: https://www.rfc-editor.org/rfc/rfc9460
[RFC 8615]: https://www.rfc-editor.org/rfc/rfc8615
[dns-aid]: https://datatracker.ietf.org/doc/draft-mozleywilliams-dnsop-dnsaid/
