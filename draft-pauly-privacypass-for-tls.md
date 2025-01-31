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

normative:

informative:


--- abstract

This document defines a mechanism for TLS servers to request, and TLS clients to provide,
Privacy Pass tokens as part of the Encrypted Client Hello in the TLS handshake. This
is a way to add support for anonymous attestation and rate-limiting to servers that
are enforcing denial-of-service protections as part of processing TLS handshakes.


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

Some server deployments enforce rate-limiting on TLS {{!TLS13=RFC8446}} handshakes
to prevent denial-of-service (DoS) attacks, particularly by rate limiting the
number of connections allowed from individual client IP addresses or IP address
subnets. This can particularly impact cases where many clients are using a
particular IP subnet due to using a privacy-preserving proxy (some examples
are described in {{?PRIVACYPARTITIONING=RFC9614}}). For such cases, even if clients
are able to provide Privacy Pass Tokens or similar proofs at the HTTP layer, their
connections might be denied or rate-limiting during TLS session establishment.

In order to provide an extra signal that clients are authentic and meet certain
criteria (rate-limiting, etc), without disclosing individual client identities
or pseudonyms, this document defines a way to include Privacy Pass Tokens within
the TLS handshake. Specifically, these tokens are sent within the TLS Encrypted
Client Hello {{!ECH=I-D.ietf-tls-esni}} to prevent network observers from being
able to directly observe the tokens.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Requesting Privacy Pass Tokens

TODO

- Clients won't always include tokens
- Servers send challenges in HRR, similar to cookie
- Should have a context to sign over


# Presenting Privacy Pass Tokens in Encrypted Client Hello

TODO

- Clients include the token inside encrypted client hello
- If no ECH, no use of privacy pass


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
