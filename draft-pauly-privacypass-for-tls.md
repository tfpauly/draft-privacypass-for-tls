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
handshake, like in L7 loadbalancers with heavy-weight protocol conversions
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

# Overview

Clients can include Privacy Pass tokens as part of the TLS Client Hello
via the `privacy_pass_token` extension. This is described in {{token_extension}}.
Clients MAY be configured to always present tokens when performing a TLS
handshake with a particular server. However, in general, clients SHOULD NOT
automatically include Privacy Pass tokens; without an explicit challenge,
clients won't know the relevant token type or issuer to use.

Servers can request tokens by adding the `privacy_pass_challenge` extension
to a TLS Hello Retry Request. This is described in {{challenge_extension}}.
Servers that want to receive Privacy Pass tokens as a way to enforce DoS
protection SHOULD send challenges to clients when these clients would
otherwise be blocked or rate-limited in some fashion.

# Requesting Privacy Pass Tokens {#challenge_extension}

In order to request that a client sends a Privacy Pass token, a
server can send a Hello Retry Request ({{TLS13, Section
4.1.4}}) that includes a `privacy_pass_challenge` extension.

The `privacy_pass_challenge` extension has the following format:

~~~
      struct {
          opaque challenge<1..2^16-1>;
          opaque token_key<0..2^16-1>;
      } PrivacyPassChallenge;
~~~

The fields are defined as follows:

- `challenge` contains a `TokenChallenge` structure, as defined in
  {{PPAUTH, Section 2.1.1}}.
- `token_key` contains a public key for use with the issuance protocol,
  where applicable. This is equivalent to the `token-key` parameter used
  in HTTP authentication challenges discussed in {{PPAUTH, Section 2.1.1}}.
  The `token_key` may be empty (have a zero length), in which case clients
  are expected to fetch the token key for a particular issuer name in
  another way.

If a client does not include the token in the Client Hello (or subsequent Client
Hello after being challenged), the server MAY reject the request or apply
rate-limiting.

Clients SHOULD apply some form of consistency check on the token challenge
to avoid (malicious) anonymity set partitioning by the server; see {{Section 6.2
of PPARCH}} for more details.

Servers sending challenges can use a non-empty `redemption_context`
in order to bind the token challenge to a particular context (such as the client
IP address, or a time window) to aid in token replay prevention.
Servers MAY combine sending `privacy_pass_challenge` extensions with
a `cookie` extension ({{TLS13, Section 4.2.2}}). For example, servers
that cannot statefully persist the token challenge presented to the
client in the `privacy_pass_challenge` extension can use the `cookie`
extension to encode this challenge.

# Presenting Privacy Pass Tokens in Encrypted Client Hello {#token_extension}

Clients can include Privacy Pass tokens in TLS handshakes using the
`privacy_pass_token` extension. This extension MUST be sent in the Inner Client Hello,
using {{!ECH}}. If ECH is not supported, clients SHOULD NOT use Privacy
Pass tokens in TLS in order to avoid adding more tracking entropy visible on
the wire, and making it easier to trivially replay tokens to a server.

The `privacy_pass_token` extension has the following format:

~~~
      struct {
          opaque token<1..2^16-1>;
      } PrivacyPassToken;
~~~

The `token` field uses the `Token` structure defined in {{PPAUTH, Section 2.1.1}}.

Tokens are generally presented after receiving a challenge, but a client MAY
include a token without having received a challenge if it has other out-of-band
configuration to do so.

## Handling Inability to Present Tokens

Servers need to be able to detect when clients are unable to present a token after
receiving a challenge. A client might be unable to present tokens because it
has reached a token rate limit, because it does not have a way to generate tokens
for the required token issuer, or simply because it does not support this
specification.

The RECOMMENDED approach to handle such cases is for the server to include a
`cookie` extension ({{TLS13, Section 4.2.2}}) along with the challenge, and
for clients to retry the handshake including the `cookie` extension, but
not including the `privacy_pass_token` extension. Servers can then assume
that the client received the challenge and was unable to generate a valid
token. The policy for what servers do in such cases will be specific
to the overall use case, and beyond the scope of this document.

# Applicable Token Types {#applicable-types}

This document is defined such that any Privacy Pass token type would be possible
to use in the TLS handshake. However, different token types will have different
properties for latency, replay protection, and privacy.

Ideally, deployments can use token types that allow for unique redemption contexts
(to prevent replay attacks) that also do not require communicating with a token
attester or issuer for each token creation (thus improving latency, and not
creating new activity that can be used to fingerprint clients). Some proposed
token types like {{?ARC=I-D.yun-privacypass-arc}} and
{{?BBS=I-D.ladd-privacypass-bbs}} have these properties.

# Security Considerations

Servers redeeming Privacy Pass tokens in TLS handshakes need to take care to
avoid replay attacks. Using a fresh redemption context in the challenge ensures
that tokens are equally fresh and unique.

As discussed in {{applicable-types}}, token issuance types that don't require
clients talking to an issuance server with a new network request for every token
generation will have better properties for privacy, since the client won't make
a new request after each TLS handshake challenge.

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
