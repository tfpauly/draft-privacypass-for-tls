---
title: "Including Privacy Pass Tokens in TLS Handshakes"
abbrev: "Privacy Pass for TLS"
category: info

docname: draft-pauly-privacypass-for-tls-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: ""
workgroup: "Privacy Pass"
venue:
  group: "Privacy Pass"
  type: ""
  mail: "privacy-pass@ietf.org"
  github: "tfpauly/draft-privacypass-for-tls"

author:
 -
    fullname: Tommy Pauly
    organization: Apple
    email: tpauly@apple.com
 -
    fullname: Scott Hendrickson
    organization: Google
    email: scott@shendrickson.com

normative:

informative:


--- abstract

This document defines a mechanism for TLS servers to request, and TLS clients to
provide, Privacy Pass tokens as part of the Encrypted Client Hello in the TLS
handshake. This creates a way to add support for anonymous attestation and
rate-limiting to servers that are enforcing denial-of-service protections as
part of processing TLS handshakes.


--- middle

# Introduction

Privacy Pass Tokens {{!PPARCH=RFC9576}} are cryptographic authentication messages
that can be used to verify properties of a network entity, such as proving that a
client passed some attestation check, without being linkable to other tokens
or revealing identities.

{{!PPAUTH=RFC9576}} defines how Privacy Pass Tokens can be requested by HTTP servers
(via an authentication challenge) and provided by HTTP clients. This is useful
for providing privacy-preserving authentication or attestation in HTTP workflows.
However, Privacy Pass Tokens can also be used in other contexts and protocols.
For example, {{?I-D.sawant-eap-ppt}} defines how to include tokens in EAP
(Extensible Authentication Protocol) exchanges.

Some server deployments enforce rate-limiting on TLS {{!TLS13=RFC8446}}
handshakes to prevent denial-of-service (DoS) attacks, particularly by rate
limiting the number of connections allowed from individual client IP addresses
or IP address subnets. This is common in scenarios where the cost of handling a
terminated TLS connection is significantly higher than handling the initial
handshake, like in L7 loadbalancers with heavy-weight protocool conversions
after termination. 

This enforcement can particularly impact cases where many clients are using a
particular IP subnet due to using a privacy-preserving proxy (some examples are
described in {{?PRIVACYPARTITIONING=RFC9614}}). For such cases, even if clients
are able to provide Privacy Pass Tokens or similar proofs at the HTTP layer,
their connections might be denied or rate-limiting during TLS session
establishment.

In order to signal that clients meet certain criteria (rate-limiting, etc),
without disclosing individual client identities or pseudonyms, this document
defines a way to include Privacy Pass Tokens within the TLS handshake.
Specifically, these tokens are sent within the TLS Encrypted Client Hello
{{!ECH=I-D.ietf-tls-esni}}. This  prevents network observers from being able to
directly observe the tokens, while still allowing the TLS server to observe the
token early in the handshake.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Requesting Privacy Pass Tokens


Servers may require that the Excrypted Client Hello contains the
privacy_pass_challenge. If the privacypass deployment requires context to sign
over, servers should implement a Hello Retry Request {{!TLS13HHR=RFC8446 Section
4.1.4}} that includes a privacy_pass_challenge.

If a client does not include the token in the Client Hello (or subsequent Client
Hello after being challenged), the server may reject the requst.


Section Should Cover

- (done) Clients won't always include tokens
- (done) Servers send challenges in HRR, similar to cookie
   - Needs example
- Should have a context to sign over
  - Needs example


# Presenting Privacy Pass Tokens in Encrypted Client Hello

Clients will convert the privacy_pass_challenge (possibly from handling the
Hello Retry request) into a privacy_pass_token as described in {{!RFC 9577,
Section 2.1.3}}. Clients will provide the now finalized privacy_pass_token to
the server in the client hello. Clients MUST use {{!ECH}} to pass privacy pass
tokens. If ECH is not supported, clients or deployments should not use Privacy
Pass tokens in TLS due to potentially adding more tracking entropy visible on
the wire.


TODO

- (done) Clients include the token inside encrypted client hello
- (done) If no ECH, no use of privacy pass


# Security Considerations

TODO Security

- Preventing replay attacks
- Privacy/timing attacks

# IANA Considerations

## Update of the TLS ExtensionType Registry

IANA is requested to create the following entries in the existing registry for
ExtensionType (defined in {{TLS13}}):

1. privacy_pass_challenge(0xfd00), with the "TLS 1.3" column values set to "HRR",
   "DTLS-Only" column set to "N", "Recommended" column set to "Yes".
1. privacy_pass_token(TBD), with "TLS 1.3" column values set to
   "CH", "DTLS-Only" column set to "N", and "Recommended" column set
   to "Yes", and the "Comment" column set to "Only appears in inner CH."

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
