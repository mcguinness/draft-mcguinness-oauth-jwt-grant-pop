---
title: "OAuth 2.0 Key-Bound JWT Authorization Grants"
abbrev: "JWT Grant PoP"
category: std
ipr: trust200902

docname: draft-mcguinness-oauth-jwt-grant-pop-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - proof-of-possession
 - sender-constrained
 - DPoP
 - mutual TLS
 - token exchange
 - JWT authorization grant
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-jwt-grant-pop"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-jwt-grant-pop/draft-mcguinness-oauth-jwt-grant-pop.html"

author:
 -
    fullname: "Karl McGuinness"
    initials: "K."
    surname: "McGuinness"
    organization: "Independent"
    email: "public@karlmcguinness.com"

normative:
  RFC6749:
  RFC7519:
  RFC7521:
  RFC7523:
  RFC7638:
  RFC7800:
  RFC8414:
  RFC8693:
  RFC8705:
  RFC9449:
  RFC10070:
  IANA.oauth-parameters:

informative:
  RFC6755:
  RFC7516:
  RFC7591:
  RFC8725:
  RFC9201:
  RFC9700:
  I-D.ietf-oauth-identity-chaining:
  I-D.ietf-oauth-identity-assertion-authz-grant:
  I-D.parecki-oauth-jwt-dpop-grant:
  I-D.mcguinness-oauth-token-exchange-cnf:

--- abstract

This specification defines how an authorization server binds a JSON Web Token (JWT) authorization grant (RFC 7523) to a key or certificate held by the client, using OAuth 2.0 Demonstrating Proof of Possession (DPoP) or OAuth 2.0 Mutual-TLS, and how the authorization server that receives the grant verifies that binding. It profiles OAuth 2.0 Token Exchange so that a client can request a DPoP-bound grant and learn which binding was applied, and it defines a grant type that requires a DPoP-bound grant. It also lists the choices that a specification defining a particular JWT authorization grant makes to use this mechanism.

--- middle

# Introduction

A JWT authorization grant ({{Section 2.1 of RFC7523}}) lets a client obtain an access token from an authorization server by presenting a JWT issued by a party that the authorization server trusts. Identity and authorization chaining {{I-D.ietf-oauth-identity-chaining}} uses OAuth 2.0 Token Exchange {{RFC8693}} to obtain such a grant in one trust domain and present it in another, and the Identity Assertion JWT Authorization Grant {{I-D.ietf-oauth-identity-assertion-authz-grant}} profiles that pattern for deployments that share a single sign-on identity provider. {{Section 5.5 of I-D.ietf-oauth-identity-chaining}} notes that "The authorization grant obtained from the Token Exchange process is a bearer token": any party that obtains the grant can redeem it until it expires. The mitigations listed there do not include binding the grant to a key.

OAuth 2.0 Demonstrating Proof of Possession (DPoP) {{RFC9449}} and OAuth 2.0 Mutual-TLS {{RFC8705}} bind the access token, and in some cases the refresh token, issued in a token response to a key or certificate held by the client ({{Section 5 of RFC9449}}; {{Section 3 of RFC8705}} and {{Section 4 of RFC8705}}). They do not define:

* how an authorization server binds a JWT authorization grant that Token Exchange issues, for which the `token_type` response parameter is `N_A` ({{Section 2.2.1 of RFC8693}});
* how an authorization server that receives a JWT authorization grant containing a `cnf` claim {{RFC7800}} verifies proof of possession; or
* how a client learns which binding was applied when it cannot read the issued token, for example because the token is opaque or is encrypted {{RFC7516}} for its consumer.

This specification defines both halves of the binding. {{issuance}} profiles Token Exchange so that a client can request a DPoP-bound grant, an issuing authorization server can bind a grant to the client's TLS client certificate, and the client learns which binding was applied. {{redemption}} profiles the JWT authorization grant so that the authorization server that receives a bound grant verifies the binding, however the grant was issued. {{metadata}} defines how an authorization server advertises that it verifies DPoP-bound grants and requires them from a client. {{profiles}} lists the choices that a specification defining a particular JWT authorization grant makes to use this mechanism.

This specification defines no new confirmation method or error code. It uses the `jkt` confirmation method ({{Section 6.1 of RFC9449}}) and the `x5t#S256` confirmation method ({{Section 3.1 of RFC8705}}).

## Relationship to Other Documents

This specification replaces {{I-D.parecki-oauth-jwt-dpop-grant}} and {{I-D.mcguinness-oauth-token-exchange-cnf}}. It retains the first's `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type ({{jwt-dpop}}) and extends that document from DPoP to mutual TLS. Unlike that document, which rejects a DPoP-bound grant presented with the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type, this specification verifies the binding of a grant presented with either grant type ({{grant-type-substitution}}). It incorporates the second's Token Exchange response parameter ({{cnf-param}}).

{{RFC9201}} registers `cnf` as a token response parameter for the Authentication and Authorization for Constrained Environments (ACE) framework, with the syntax and semantics of the `cnf` claim ({{Section 3.1 of RFC7800}}) for JSON-based interactions. {{cnf-param}} defines the use of that parameter in a Token Exchange response.

This specification generalizes the sender-constraining rules that the Identity Assertion JWT Authorization Grant {{I-D.ietf-oauth-identity-assertion-authz-grant}} defines for its grant. That grant is described as a profile of this specification, with the differences between the two documents, in {{profile-examples}}.

## Conventions and Terminology

{::boilerplate bcp14-tagged}

This specification uses the terms "access token", "authorization grant", "authorization server", "client", "refresh token", and "token endpoint" defined by {{RFC6749}}, the terms "JSON Web Token (JWT)" and "claim" defined by {{RFC7519}}, and the term "DPoP proof" defined by {{RFC9449}}. It also defines the following terms:

JWT authorization grant:
: A JWT used as an authorization grant as defined in {{Section 2.1 of RFC7523}}.

Issuing authorization server:
: The authorization server that issues a JWT authorization grant.

Consuming authorization server:
: The authorization server to which a client presents a JWT authorization grant in an access token request ({{Section 4.1 of RFC7521}}).

Bound grant:
: A JWT authorization grant that contains a `cnf` claim ({{Section 3.1 of RFC7800}}).

Confirmation member:
: A member of a `cnf` claim or of the `cnf` response parameter, such as `jkt` ({{Section 6.1 of RFC9449}}) or `x5t#S256` ({{Section 3.1 of RFC8705}}).

Binding request:
: A Token Exchange request that contains a DPoP proof ({{binding-request}}).

# Protocol Overview {#overview}

~~~ ascii-art
+--------+                                 +---------------+
|        |--(1) Token Exchange request --->|    Issuing    |
|        |     and DPoP proof for key K    | Authorization |
|        |                                 |    Server     |
|        |<-(2) grant with cnf for K ------|               |
|        |     and cnf response parameter  +---------------+
| Client |
|        |                                 +---------------+
|        |--(3) jwt-dpop request --------->|   Consuming   |
|        |     with grant and DPoP proof   | Authorization |
|        |     for key K                   |    Server     |
|        |<-(4) access token --------------|               |
|        |     (normally bound to K)       |               |
+--------+                                 +---------------+
~~~
{: title="Issuing and Redeeming a DPoP-Bound Grant"}

1. The client sends a Token Exchange request with a DPoP proof for its key K ({{binding-request}}).
2. The issuing authorization server issues a grant whose `cnf` claim identifies K ({{bind}}) and returns the binding in the `cnf` response parameter ({{cnf-param}}), which the client checks ({{client-processing}}).
3. The client presents the grant to the consuming authorization server with a DPoP proof for K ({{token-request}}).
4. The consuming authorization server verifies the proof against the grant's `cnf` claim ({{processing}}) and issues an access token, normally bound to K ({{token-response}}).

The client sends a separate DPoP proof to each authorization server, signed with the same key, because the `htu` claim of a DPoP proof identifies the endpoint that receives it ({{Section 4.2 of RFC9449}}). With mutual TLS, the client's TLS client certificate takes the place of K, the client proves possession in the TLS handshake instead of with a DPoP proof, and the client redeems the grant with the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type.

# Issuing a Bound Grant with Token Exchange {#issuance}

This section applies when an issuing authorization server issues a JWT authorization grant in response to a Token Exchange request ({{Section 2.1 of RFC8693}}), except for {{cnf-param}} and {{client-processing}}, which apply to any token that Token Exchange issues. The issuing authorization server determines from the `requested_token_type`, `audience`, and `resource` parameters and from its policy whether the issued token is a JWT authorization grant. An issuing authorization server that supports this specification supports at least one of the `jkt` and `x5t#S256` confirmation methods, and implements {{cnf-param}}.

## Binding Request {#binding-request}

A client requests a bound grant by including a DPoP proof ({{Section 4 of RFC9449}}) in the DPoP header field of a Token Exchange request. This is the signal that {{Section 5 of RFC9449}} defines for requesting a DPoP-bound access token, applied here to a token that is not an access token. The client signs the DPoP proof with the key that it will use to redeem the grant ({{token-request}}).

The issuing authorization server MUST validate the DPoP proof as described in {{Section 4.3 of RFC9449}}, and responds to an invalid proof or a missing nonce as described in {{errors}}.

If an issuing authorization server that supports the `jkt` confirmation method receives a binding request with a valid DPoP proof, it MUST either bind the issued grant to the public key of the DPoP proof ({{bind}}) or reject the request ({{errors}}). It MUST NOT issue an unbound grant, or a grant bound to another key or certificate, in response to a binding request, including when the client also presents a TLS client certificate.

If the `subject_token` or `actor_token` is bound to a key or certificate, such as a refresh token issued to a public client ({{Section 5 of RFC9449}}), the issuing authorization server MUST reject the request with the `invalid_request` error code ({{Section 2.2.2 of RFC8693}}) unless the request proves possession of that key or certificate, with a DPoP proof signed with the key or a mutual-TLS handshake using the certificate, or the requesting client authenticates and is an intended audience of the token other than the client to which the token was issued, such as a resource server that exchanges an access token it received ({{Section 2.3 of RFC8693}}). The issuing authorization server determines the audience from the token's `aud` claim or from its own records of the token. That a token was issued to a party other than the requesting client does not exempt the request from this check. A grant issued in response to a binding request that proves possession of a DPoP-bound `subject_token` is therefore bound to the same key.

When it rejects a request for a JWT authorization grant, the issuing authorization server MUST NOT treat the `subject_token` or `actor_token` as used, for example by rotating a refresh token, so that the client can retry, such as after a `use_dpop_nonce` error response.

A TLS client certificate is not a binding request, because a client presents one for mutual-TLS client authentication ({{Section 2 of RFC8705}}) without necessarily intending to bind the grant. If a request contains no DPoP proof and the client presented a TLS client certificate in the mutual-TLS handshake of the connection that carries the request, an issuing authorization server that supports the `x5t#S256` confirmation method MAY bind the grant to that certificate ({{bind}}). It SHOULD NOT do so unless the profile of the grant requires certificate binding or the client's registration or configuration at the issuing authorization server indicates that the client presents that certificate when it redeems the grant, because a client that does not support this specification neither learns of the binding nor presents the certificate to the consuming authorization server. The `tls_client_certificate_bound_access_tokens` client metadata parameter ({{Section 3.4 of RFC8705}}) indicates an intention to use certificate-bound access tokens, not certificate-bound grants, and is not such an indication.

An issuing authorization server binds a grant only as described in this section. This specification defines no request parameter by which a client requires a bound grant; a client that requires one enforces the requirement when it processes the response ({{client-processing}}).

## Binding the Grant {#bind}

To bind a grant, the issuing authorization server includes in it a `cnf` claim ({{Section 3.1 of RFC7800}}) that contains exactly one confirmation member:

* for a key binding, `jkt` ({{Section 6.1 of RFC9449}}), containing the JWK SHA-256 Thumbprint {{RFC7638}} of the client's public key, which for a binding request is the key in the `jwk` JOSE Header Parameter of the DPoP proof; or
* for a certificate binding, `x5t#S256` ({{Section 3.1 of RFC8705}}), containing the SHA-256 thumbprint of the DER encoding of the client's certificate, which for a Token Exchange request is the TLS client certificate.

This is narrower than {{Section 3.1 of RFC7800}}, which requires only that a `cnf` claim "represent only a single proof-of-possession key". The issuing authorization server MUST NOT bind a grant to a key or certificate whose possession the client has not proven in the request for the grant.

The issuing authorization server MUST NOT bind a grant using a confirmation method unless it has determined that the consuming authorization server identified by the grant's `aud` claim verifies that confirmation method as described in {{processing}} (see {{non-enforcing-consumers}}). It makes this determination from configuration, from an established trust relationship, or, for the `jkt` confirmation method, from the consuming authorization server's metadata ({{as-metadata}}). If the issuing authorization server cannot bind the grant for the consuming authorization server in response to a binding request, it MUST reject the request with the `invalid_target` error code ({{errors}}).

When the issued token is a bound grant, the issuing authorization server MUST set the `token_type` response parameter to `N_A` ({{Section 2.2.1 of RFC8693}}). The requirement in {{Section 5 of RFC9449}} to return a `token_type` of `DPoP` applies to a DPoP-bound access token, not to a grant.

The other contents of the grant, including the `aud` claim, are defined by {{RFC7523}} as updated by {{RFC10070}} and by the profile of the grant ({{profiles}}).

## Confirmation Response Parameter {#cnf-param}

This specification defines the use of the `cnf` token response parameter, registered by {{RFC9201}}, in a Token Exchange response ({{Section 2.2.1 of RFC8693}}):

`cnf`:
: REQUIRED if the authorization server bound the token in the `access_token` response parameter to a key or certificate; otherwise, it MUST NOT be present. A JSON object containing exactly one confirmation member, which identifies that key or certificate. Its syntax and member semantics are those of the `cnf` claim ({{Section 3.1 of RFC7800}}).

The parameter describes only the token in the `access_token` response parameter, whatever its `issued_token_type`, and not a `refresh_token` in the same response. It applies to any token that Token Exchange issues, including an opaque or encrypted token whose `cnf` claim the client cannot read, and an access token for which the `token_type` response parameter also indicates the binding.

When the issued token is a JWT that contains a `cnf` claim, the parameter MUST identify the same key or certificate as that claim ({{cnf-comparison}}). The authorization server SHOULD use the same confirmation member in the parameter as in the claim, and SHOULD use a thumbprint member, such as `jkt` or `x5t#S256`, rather than a `jwk` member ({{privacy}}).

A client that does not support this specification ignores the parameter, as it ignores any unrecognized response parameter ({{Section 5.1 of RFC6749}}).

### Comparing Confirmation Values {#cnf-comparison}

This specification compares two confirmation values, such as the `cnf` response parameter and a `cnf` claim, or a confirmation member and the client's own key or certificate, by confirmation method and value rather than by byte representation:

* JSON member order and insignificant whitespace are ignored.
* A thumbprint member, such as `jkt` or `x5t#S256`, is compared as the base64url-encoded string it contains, character by character.
* A public key in a `jwk` member is compared with a `jkt` member, or with the client's own public key, by computing JWK SHA-256 Thumbprints {{RFC7638}}.
* Members of different confirmation methods, other than a `jwk` and `jkt` pair, do not match.

## Client Processing of the Response {#client-processing}

A client that sent a binding request, or that presented a TLS client certificate and requires a bound token, processes the Token Exchange response as follows:

1. If the `cnf` parameter is present and is not a JSON object that contains exactly one confirmation member, the client MUST discard the issued token.
2. The client determines the binding from the `cnf` parameter and, if it can read the issued token, from the token's `cnf` claim. If neither is present, the issued token is not bound, and a client that requires a bound token MUST discard it.
3. The client MUST verify that the `cnf` claim, if readable, contains exactly one confirmation member, and that the confirmation member of the `cnf` parameter, if present, and of the `cnf` claim, if readable, each match its own key or certificate ({{cnf-comparison}}): after a binding request, the public key of its DPoP proof; otherwise, its TLS client certificate. If any of these checks fails, the client MUST discard the issued token. A confirmation member that the client does not recognize does not match.

The client determines the binding from the `cnf` parameter and claim, not from the `token_type` response parameter. It cannot distinguish an authorization server that does not support this specification from one that failed to apply the binding; the steps above treat both the same way.

A client that receives the `invalid_target` error code in response to a binding request, and does not require a bound grant, MAY repeat the request without a DPoP proof.

A client that did not send a binding request but receives a `cnf` parameter holds a bound token. If the client holds the identified key or certificate, such as the certificate it used for mutual-TLS client authentication, it presents the token with the corresponding proof. Otherwise, it cannot use the token and SHOULD treat the response as an error.

## Other Issuance Interfaces {#other-issuance}

A JWT authorization grant can also be issued through an interface other than Token Exchange, such as a workload platform's token service. {{bind}} applies to a bound grant issued through such an interface, except for the requirements on the Token Exchange response, with the request for the grant being the request that the interface defines. How the client requests the binding and learns of it is defined by that interface. {{redemption}} applies however the grant was issued.

# Redeeming a Bound Grant {#redemption}

## Access Token Request {#token-request}

The client presents a bound grant in an access token request ({{Section 4.1 of RFC7521}}) and proves possession of the key or certificate identified by the grant's `cnf` claim:

* For a `jkt` member, the client MUST include a DPoP proof ({{Section 4 of RFC9449}}) signed with that key in the DPoP header field of the request, and uses the `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type ({{jwt-dpop}}) or the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type ({{Section 2.1 of RFC7523}}). The client SHOULD use the `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type if the consuming authorization server advertises it ({{as-metadata}}).
* For an `x5t#S256` member, the client MUST send the request, with the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type, over a mutual-TLS connection that uses that certificate ({{Section 3 of RFC8705}}), to the token endpoint or to its mutual-TLS endpoint alias ({{Section 5 of RFC8705}}).

Whether the client also authenticates is determined by the consuming authorization server's policy and the profile of the grant ({{client-auth}}).

### DPoP-Bound Grant Type {#jwt-dpop}

To use the `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type, the client uses the following parameters, in addition to the DPoP header field:

`grant_type`:
: REQUIRED. The value `urn:ietf:params:oauth:grant-type:jwt-dpop`.

`assertion`:
: REQUIRED. A single JWT authorization grant whose `cnf` claim contains a `jkt` member.

`scope`:
: OPTIONAL. The requested scope, as defined in {{Section 4.1 of RFC7521}}.

The consuming authorization server processes the request as defined in {{processing}}, and MUST also reject it with the `invalid_grant` error code if the grant has no `cnf` claim or if the confirmation member of its `cnf` claim is not `jkt`.

## Processing Rules {#processing}

A consuming authorization server validates a JWT authorization grant as defined in {{Section 3 of RFC7523}}, as updated by {{Section 4 of RFC10070}}, and by the profile of the grant. If the grant contains a `cnf` claim, a consuming authorization server that supports this specification MUST also ensure the following, whichever grant type the request uses, and MUST otherwise reject the request with the indicated error code ({{errors}}):

1. The `cnf` claim is a JSON object that contains exactly one confirmation member; that member is `jkt`, `x5t#S256`, or a member for which the profile of the grant defines a proof-of-possession check; and the consuming authorization server supports its confirmation method; otherwise, `invalid_grant`. This is an application requirement in the sense of {{Section 3.1 of RFC7800}}, which otherwise requires implementations to ignore confirmation members they do not understand.
2. For a `jkt` member:
   * the request contains a DPoP proof; otherwise, `invalid_grant`;
   * the DPoP proof is valid as described in {{Section 4.3 of RFC9449}}; otherwise, `invalid_dpop_proof`, or `use_dpop_nonce` as described in {{Section 8 of RFC9449}}; and
   * the JWK SHA-256 Thumbprint {{RFC7638}} of the public key in the `jwk` JOSE Header Parameter of the DPoP proof equals the `jkt` value; otherwise, `invalid_grant`.
3. For an `x5t#S256` member:
   * the request was received on a TLS connection in whose handshake the client presented a certificate and proved possession of its private key; otherwise, `invalid_grant`; and
   * the SHA-256 thumbprint of the DER encoding of that certificate equals the `x5t#S256` value ({{Section 3.1 of RFC8705}}); otherwise, `invalid_grant`.

For an `x5t#S256` member, the consuming authorization server verifies possession of the certificate's private key. It need not authenticate the client with the certificate or trust the certificate's issuer ({{Section 4 of RFC8705}}). When TLS is terminated before the consuming authorization server, how the certificate is conveyed to it is outside the scope of this specification, as it is in {{Section 6.5 of RFC8705}}. Client authentication, when present, does not take the place of these checks.

If the profile of the grant requires binding ({{profiles}}) and the grant has no `cnf` claim, the consuming authorization server MUST reject the request with the `invalid_grant` error code.

A consuming authorization server that detects reuse of a grant, for example by its `jti` claim ({{Section 4.1.7 of RFC7519}}), MUST NOT record the grant as used when it rejects the request ({{single-use}}).

A grant without a `cnf` claim, presented with the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type, is otherwise processed as defined in {{RFC7523}}. {{Section 5 of RFC9449}} and {{Section 3 of RFC8705}} apply to the access token request whether or not the grant is bound, except as {{token-response}} restricts the binding of tokens issued for a bound grant.

## Access Token Response {#token-response}

When the grant is bound, the consuming authorization server SHOULD bind the access token it issues, and indicate the binding as defined in {{Section 5 of RFC9449}} or {{Section 3 of RFC8705}}. For a bound grant, it MUST NOT bind the access token, or a refresh token that it issues, to a key or certificate other than the following:

* if the consuming authorization server supports DPoP and the request contains a valid DPoP proof ({{Section 4.3 of RFC9449}}), the public key of that proof, which for a `jkt`-bound grant is the grant's key ({{processing}}); or
* otherwise, the certificate that the grant's `x5t#S256` member identifies.

A TLS client certificate presented with a `jkt`-bound grant does not determine the binding, even when the client authenticates with it, because the request does not show that the holder of the certificate holds the grant's key ({{proof-replay}}). A DPoP proof presented with an `x5t#S256`-bound grant does determine the binding: the mutual-TLS handshake shows that the request comes from the holder of the grant's certificate, and the DPoP proof is the request for a DPoP-bound access token defined in {{Section 5 of RFC9449}}.

{{binding-outcomes}} summarizes these rules for a request that the consuming authorization server accepts. The access token can instead be unbound; a refresh token is always bound as shown ({{refresh}}). A consuming authorization server that does not support DPoP treats the last row as the second. For an unbound grant, {{Section 5 of RFC9449}} and {{Section 3 of RFC8705}} apply unchanged.

| Grant's confirmation member | Proof in the request | Binding of issued tokens |
|---|---|---|
| `jkt` | DPoP proof, with or without a TLS client certificate | Grant's key |
| `x5t#S256` | TLS client certificate only | Grant's certificate |
| `x5t#S256` | TLS client certificate and DPoP proof | Key of the DPoP proof |
{: #binding-outcomes title="Binding of Tokens Issued for a Bound Grant"}

## Refresh Tokens {#refresh}

If the consuming authorization server issues a refresh token for a bound grant, it MUST bind the refresh token as described in {{token-response}}, whether or not the client authenticates. This binding is in addition to the binding to the client that {{Section 10.4 of RFC6749}} requires. It is stricter than {{Section 5 of RFC9449}}, under which a refresh token issued to a confidential client is not bound to the DPoP key, and than the recommendation in {{Section 4 of RFC8705}}, because a refresh token bound only to the client would let a party that replays a captured grant and DPoP proof while authenticating as its own client obtain new access tokens without the grant's key ({{proof-replay}}).

When a client presents a refresh token bound under this section, the consuming authorization server MUST verify its binding as items 2 and 3 of {{processing}} verify the binding of a grant, with the same error codes. It MUST apply {{token-response}} and this section to the access token and to any refresh token that it issues in response, with the presented refresh token's key or certificate in place of the grant's. Every replacement refresh token is therefore bound under the same rules as the first, whether or not the client authenticates. At a consuming authorization server that supports DPoP, a valid DPoP proof presented with a certificate-bound refresh token determines the binding of the issued tokens ({{binding-outcomes}}); the binding of the presented refresh token does not change.

The response does not always show which binding was applied to a refresh token. For example, a consuming authorization server that does not support DPoP binds the refresh token to the certificate even when the request contained a DPoP proof ({{binding-outcomes}}), and returns a `token_type` of `Bearer`. The client therefore presents a refresh token issued for a bound grant with each proof of possession that it presented in the request in which it obtained that refresh token: a DPoP proof for the same key, a mutual-TLS connection that uses the same certificate, or both.

## Bound Grant in Place of Client Authentication {#client-auth}

Client authentication is optional for a JWT authorization grant ({{Section 4.1 of RFC7521}}, {{Section 3.1 of RFC7523}}). A profile ({{profiles}}) can allow a consuming authorization server to accept a bound grant from a client that does not authenticate, with proof of possession of the grant's key or certificate taking the place of client authentication.

Proof of possession does not by itself identify a client ({{Section 3 of RFC9449}}). The consuming authorization server relies instead on the issuing authorization server, which authenticates the client in the request in which it verifies proof of possession and identifies that client in the grant. A profile that allows a bound grant in place of client authentication specifies the items listed in item 4 of {{profiles}}.

Where a profile allows a bound grant in place of client authentication, a consuming authorization server MUST reject a request that has neither client authentication nor a bound grant with the `invalid_client` error code ({{Section 5.2 of RFC6749}}). This error code takes precedence over the `invalid_grant` error code for an unbound grant ({{processing}}, {{jwt-dpop}}).

# Error Responses {#errors}

An authorization server returns errors as defined in {{Section 5.2 of RFC6749}}, with the HTTP status code 400 (Bad Request) except where that section permits 401 (Unauthorized) for `invalid_client`. This specification defines no new error codes. It uses the following error codes:

`invalid_dpop_proof`:
: At either authorization server, the DPoP proof is invalid ({{Section 5 of RFC9449}}).

`use_dpop_nonce`:
: At either authorization server, the authorization server requires a nonce in the DPoP proof, or the nonce is missing or not current. The response includes a DPoP-Nonce header field ({{Section 8 of RFC9449}}). The authorization server MUST use this error code, and not `invalid_grant` or `invalid_request`, for these conditions, including when the DPoP proof is needed to verify a bound grant or a bound `subject_token`.

`invalid_request`:
: At the issuing authorization server, the `subject_token` or `actor_token` is bound to a key or certificate whose possession the request does not prove, and the requesting client is not an authenticated intended audience of that token other than the client to which the token was issued ({{binding-request}}, {{Section 2.2.2 of RFC8693}}).

`invalid_target`:
: At the issuing authorization server, the authorization server cannot honor a binding request for the consuming authorization server ({{bind}}, {{Section 2.2.2 of RFC8693}}).

`invalid_grant`:
: At the consuming authorization server, the grant's `cnf` claim is not a JSON object, does not contain exactly one confirmation member, or contains one without a proof-of-possession check; the request carries no proof of possession for a bound grant or for a refresh token issued for one, or a proof for another key or certificate; the profile requires binding and the grant is unbound; a `urn:ietf:params:oauth:grant-type:jwt-dpop` request carries a grant without a `jkt` member; or the grant lacks the `typ` value that its profile defines ({{processing}}, {{jwt-dpop}}, {{refresh}}, {{Section 4.1.1 of RFC7521}}).

`invalid_client`:
: At the consuming authorization server, where a profile allows a bound grant in place of client authentication, the request has neither client authentication nor a bound grant ({{client-auth}}).

`unauthorized_client`:
: At the consuming authorization server, the client is not registered for the grant type of the request ({{client-metadata}}).

# Metadata {#metadata}

## Authorization Server Metadata {#as-metadata}

A consuming authorization server that supports the `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type SHOULD include it in the `grant_types_supported` parameter of its authorization server metadata ({{Section 2 of RFC8414}}). Because a consuming authorization server that supports that grant type supports this specification, an issuing authorization server MAY rely on that value to determine that the consuming authorization server verifies the `jkt` confirmation method ({{bind}}), in metadata that it retrieved and validated as described in {{Section 3 of RFC8414}} for one of the following issuer identifiers:

* the grant's `aud` value; or
* an issuer identifier that the issuing authorization server's configuration associates with the grant's `aud` value, for example when that value is a token endpoint URL ({{Section 4 of RFC10070}}), provided that the `token_endpoint` value of the metadata equals the `aud` value.

The `token_endpoint` value of metadata does not by itself establish that association, because only the `issuer` value is validated ({{Section 3.3 of RFC8414}}, {{Section 1 of RFC10070}}).

Support for DPoP-bound or certificate-bound access tokens, as indicated by the `dpop_signing_alg_values_supported` ({{Section 5.1 of RFC9449}}) or `tls_client_certificate_bound_access_tokens` ({{Section 3.3 of RFC8705}}) authorization server metadata parameters, does not indicate that an authorization server verifies bound grants.

## Client Registration Metadata {#client-metadata}

A consuming authorization server can require a client to present DPoP-bound grants by registering the client with a `grant_types` client metadata value ({{Section 2 of RFC7591}}) that includes `urn:ietf:params:oauth:grant-type:jwt-dpop` and does not include `urn:ietf:params:oauth:grant-type:jwt-bearer`. The consuming authorization server then rejects a request from that client that uses the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type with the `unauthorized_client` error code ({{Section 5.2 of RFC6749}}), and {{jwt-dpop}} rejects an unbound grant. When a bound grant takes the place of client authentication ({{client-auth}}), the client is the one that the profile identifies from the grant.

The `dpop_bound_access_tokens` client metadata parameter ({{Section 5.2 of RFC9449}}) requires a DPoP proof in each token request, but does not require the grant to be bound.

# Profiling This Specification {#profiles}

A specification that defines a JWT authorization grant, such as {{I-D.ietf-oauth-identity-assertion-authz-grant}}, uses this specification by stating:

1. The value of the grant's `typ` JOSE Header Parameter ({{Section 3.11 of RFC8725}}) and, if the grant is issued through Token Exchange, its `requested_token_type` value.
2. Which confirmation methods an issuing authorization server may apply, which a consuming authorization server must verify, which are mandatory to implement, and whether the `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type is mandatory to implement.
3. Whether binding is optional, recommended, or required, and whether that varies by consuming authorization server.
4. Whether a consuming authorization server may accept a bound grant in place of client authentication ({{client-auth}}). If so: the claim in which the issuing authorization server identifies the client; how that identity is tied to the key or certificate; what the issuing authorization server verifies before it binds the grant; and how the consuming authorization server maps that identity to its own client registration and policy.
5. Any error codes or processing beyond those in {{errors}} and {{processing}}.

A profile that permits a bound grant in place of client authentication MUST require the issuing authorization server to authenticate the client in the same request in which it verifies proof of possession of the key or certificate.

When a profile does not address an item above, binding is optional, either confirmation method may be used, a consuming authorization server verifies any `cnf` claim it receives ({{processing}}), and a bound grant does not take the place of client authentication: whether the client authenticates is the consuming authorization server's policy ({{Section 3.1 of RFC7523}}), and {{client-auth}} does not apply.

# Examples {#examples}

The examples in this section are non-normative. They use the public key from the examples in {{RFC9449}}, whose JWK SHA-256 Thumbprint is `0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I`. Request bodies are shown with extra line breaks and without URL encoding, and tokens are truncated, for display purposes only.

## Token Exchange with a Binding Request

The following is an example of a Token Exchange request, from a client that authenticates with a client secret, for a grant for the consuming authorization server `https://as.example/`:

~~~ http-message
POST /token HTTP/1.1
Host: issuer.example
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6...

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:jwt
&audience=https://as.example/
&subject_token=eyJraWQiOiJzMTZ0cVNtODhwREo4VGZCXzdrSEtQ...
&subject_token_type=urn:ietf:params:oauth:token-type:id_token
~~~
{: title="Token Exchange Request with a DPoP Proof"}

The decoded JOSE header of the DPoP proof in that request is:

~~~ json
{
  "typ": "dpop+jwt",
  "alg": "ES256",
  "jwk": {
    "kty": "EC",
    "x": "l8tFrhx-34tV3hRICRDY9zCkDlpBhF42UQUfWVAWBFs",
    "y": "9VE4jf_Ok_o64zbTTlcuNJajHmt6v9TDVrU0CdvGRDA",
    "crv": "P-256"
  }
}
~~~
{: title="DPoP Proof JOSE Header"}

Its decoded claims are:

~~~ json
{
  "jti": "e1j3V_bKic8-LAEB",
  "htm": "POST",
  "htu": "https://issuer.example/token",
  "iat": 1730000000
}
~~~
{: title="DPoP Proof Claims for the Issuing Authorization Server"}

The issuing authorization server binds the grant and returns the binding in the `cnf` response parameter:

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0...",
  "issued_token_type": "urn:ietf:params:oauth:token-type:jwt",
  "token_type": "N_A",
  "expires_in": 300,
  "cnf": {
    "jkt": "0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I"
  }
}
~~~
{: title="Token Exchange Response with the cnf Parameter"}

The claims of the issued grant are:

~~~ json
{
  "iss": "https://issuer.example/",
  "sub": "U019488227",
  "aud": "https://as.example/",
  "iat": 1730000000,
  "exp": 1730000300,
  "jti": "9e43f81b64a33f20116179",
  "cnf": {
    "jkt": "0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I"
  }
}
~~~
{: title="Claims of a DPoP-Bound Grant"}

The client verifies that the `jkt` value in the `cnf` response parameter equals the JWK SHA-256 Thumbprint of its DPoP public key.

## Redeeming a DPoP-Bound Grant

The following is an example of an access token request that presents the grant with the `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type and a DPoP proof for the same key. The `htu` claim of this DPoP proof is `https://as.example/token`. The client does not authenticate, which the consuming authorization server's policy permits.

~~~ http-message
POST /token HTTP/1.1
Host: as.example
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6...

grant_type=urn:ietf:params:oauth:grant-type:jwt-dpop
&assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0...
~~~
{: title="Access Token Request with a DPoP-Bound Grant"}

If the consuming authorization server requires a DPoP nonce ({{Section 8 of RFC9449}}), it rejects the request without recording the grant as used ({{processing}}):

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
DPoP-Nonce: eyJ7S_zG.eyJH0-Z.HX4w-7v

{
  "error": "use_dpop_nonce",
  "error_description": "Nonce required in DPoP proof"
}
~~~
{: title="Nonce Challenge for a DPoP-Bound Grant"}

The client repeats the request with the same grant and a new DPoP proof whose `nonce` claim contains the supplied value. The consuming authorization server verifies that DPoP proof against the grant's `cnf` claim and issues a DPoP-bound access token:

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "Kz~8mXK1EalYznwH-LC-1fBAo.4Ljp~zsPE_NeO.gxU",
  "token_type": "DPoP",
  "expires_in": 300
}
~~~
{: title="Access Token Response"}

If the response also contained a refresh token, the client would present it with a DPoP proof for the same key ({{refresh}}).

If the request carries no DPoP proof, the consuming authorization server rejects it:

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
  "error": "invalid_grant",
  "error_description": "Proof of possession required for this grant"
}
~~~
{: title="Error Response for a Missing DPoP Proof"}

## Certificate-Bound Grant {#cert-example}

The following is an example of a Token Exchange response in which the issuing authorization server binds the grant to the TLS client certificate that the client used for mutual-TLS client authentication. The request contained no DPoP proof, and the issuing authorization server's configuration for the client indicates that the client redeems grants over mutual TLS with that certificate ({{binding-request}}).

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0...",
  "issued_token_type": "urn:ietf:params:oauth:token-type:jwt",
  "token_type": "N_A",
  "expires_in": 300,
  "cnf": {
    "x5t#S256": "bwcK0esc3ACC3DB2Y5_lESsXE8o9ltc05O89jdN-dg0"
  }
}
~~~
{: title="Token Exchange Response for a Certificate-Bound Grant"}

The client redeems the grant with the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type over a mutual-TLS connection that uses the same certificate. If the client also includes a DPoP proof in that request, the consuming authorization server binds the access token to the key of the DPoP proof and returns a `token_type` of `DPoP` ({{binding-outcomes}}).

## Refreshing a Certificate-Bound Token with a DPoP Proof

The following is an example of a refresh request for a refresh token that the consuming authorization server bound to the certificate when the client redeemed the grant in {{cert-example}} without a DPoP proof. The client sends the request over a mutual-TLS connection that uses that certificate, which it also uses for mutual-TLS client authentication ({{Section 2 of RFC8705}}), and adds a DPoP proof:

~~~ http-message
POST /token HTTP/1.1
Host: as.example
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6...

grant_type=refresh_token
&client_id=s6BhdRkqt
&refresh_token=Q..Zkm29lexi8VnWg2zPW1x-tgGad0Ibc3s3EwM_Ni4-g
~~~
{: title="Refresh Request with a Certificate-Bound Refresh Token"}

The consuming authorization server verifies the certificate against the binding of the refresh token, which applies although the client authenticates, and binds the new access token and the replacement refresh token to the key of the DPoP proof ({{refresh}}):

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "Rk3Lq0x_bVz9Ty2Wd8nPe4Hs1Uj6Yc7Ma5Gf",
  "token_type": "DPoP",
  "expires_in": 300,
  "refresh_token": "4LTC8lb0acc6Oy4esc1Nk9BWC0imAwH-hPRPxchW2gs"
}
~~~
{: title="Refresh Response with DPoP-Bound Tokens"}

The replacement refresh token is bound to the key of the DPoP proof, and the presented refresh token remains bound to the certificate. Because the client presented both proofs in the request in which it obtained the replacement refresh token, it presents both when it uses that refresh token: a DPoP proof for the same key, over a mutual-TLS connection that uses the same certificate. The consuming authorization server verifies the DPoP proof.

## Binding Request Rejected

The following is an example of an error response from an issuing authorization server that has not determined that the consuming authorization server verifies DPoP-bound grants:

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
  "error": "invalid_target",
  "error_description": "Audience does not accept bound grants"
}
~~~
{: title="Error Response for a Binding Request That Cannot Be Honored"}

# Security Considerations {#security}

The security considerations of {{RFC7521}}, {{RFC7523}}, {{RFC8693}}, {{RFC9449}}, and {{RFC8705}} apply to this specification.

## Non-Enforcing Consuming Authorization Servers {#non-enforcing-consumers}

{{Section 3.1 of RFC7800}} requires implementations to ignore confirmation members they do not understand, and {{RFC7523}} does not process the `cnf` claim. A consuming authorization server that does not support this specification therefore accepts a bound grant as a bearer grant, and a party that obtains the grant can redeem it without the key. The client's check of the `cnf` response parameter does not detect this. {{bind}} therefore forbids an issuing authorization server to bind a grant for a consuming authorization server that it has not determined verifies the binding.

## Grant Type Substitution {#grant-type-substitution}

A separate grant type for bound grants does not by itself prevent substitution: a party holding a stolen bound grant can present it with the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type. This specification therefore requires a consuming authorization server that supports this specification to verify the binding of every JWT authorization grant that contains a `cnf` claim, whichever grant type the request uses ({{processing}}).

The `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type adds no protection against a stolen grant. It lets a consuming authorization server advertise that it verifies DPoP-bound grants ({{as-metadata}}) and refuse unbound grants from a client ({{client-metadata}}). {{I-D.parecki-oauth-jwt-dpop-grant}} rejects a DPoP-bound grant presented with the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type; verifying it instead keeps clients registered only for that grant type working.

A consuming authorization server that adopts this specification rejects a grant that it previously accepted if the grant's `cnf` claim contains more than one confirmation member, or a member without a proof-of-possession check.

## Informational Nature of the Response Parameter

The `cnf` response parameter is informational. The binding is enforced when the issued token is presented to its consumer, which verifies the token's `cnf` claim against the presenter's proof. A verified `cnf` parameter tells the client only that the authorization server applied the binding; it does not tell the client that the consumer enforces it ({{non-enforcing-consumers}}). The check protects against an authorization server that fails to apply a binding through error or misconfiguration, not against a malicious one, which controls the contents of the token.

The token endpoint requires TLS ({{Section 3.2 of RFC6749}}), which prevents an on-path attacker from removing or modifying the `cnf` parameter.

## Theft of the Subject Token

Binding the issued grant does not protect a bearer `subject_token` that is stolen before the exchange. An attacker that holds such a subject token, and can satisfy any client authentication that the issuing authorization server requires, can request a grant bound to the attacker's own key. A bound `subject_token` is protected by the check in {{binding-request}}, except from an authenticated client that is an intended audience of the token other than the client to which the token was issued, which can exchange it without the key. Protection of bearer subject tokens is outside the scope of this specification; see {{RFC9700}}.

## Proof of Possession and Client Identity

Proof of possession shows that the presenter holds the key or certificate, not which client the presenter is. Even at the issuing authorization server, a DPoP proof is not cryptographically bound to the client authentication that accompanies it ({{Section 11.11 of RFC9449}}). A consuming authorization server that accepts a bound grant in place of client authentication ({{client-auth}}) therefore depends on the issuing authorization server's authentication of the client and on the client's protection of its private key: any party that obtains the key can use a grant bound to it.

## DPoP Proof Replay {#proof-replay}

The DPoP proof replay protections of {{Section 11.1 of RFC9449}} and the authorization server-provided nonce of {{Section 8 of RFC9449}} apply at the token endpoints of both the issuing and the consuming authorization server. A DPoP proof sent to one cannot be replayed at the other, because the `htu` claim identifies the token endpoint ({{Section 4.3 of RFC9449}}).

A DPoP proof does not cover the grant it accompanies or the TLS connection that carries it. An attacker that captures a grant together with a DPoP proof for its key can replay both, over its own TLS connection, within the window in which the consuming authorization server accepts the proof ({{Section 11.1 of RFC9449}}). An access token bound to the grant's key is of no use to the attacker. An access token bound to a TLS client certificate that the attacker presents, or an unbound access token, would be; {{token-response}} therefore forbids the first. So would a refresh token bound only to a client as which the attacker authenticates, at the first refresh or a later one, which is why {{refresh}} binds every refresh token issued for a bound grant and every replacement. A consuming authorization server that issues unbound access tokens for `jkt`-bound grants SHOULD reject a DPoP proof whose `jti` value it has already accepted at its token endpoint within that window ({{Section 11.1 of RFC9449}}). A server-provided nonce ({{Section 8 of RFC9449}}) shortens the window but does not by itself detect reuse, because a client can use the same nonce more than once. Neither measure denies an unbound access token to an attacker whose request arrives before the client's; only binding the access token does.

## Exhaustion of Single-Use Grants {#single-use}

A consuming authorization server that limits a grant to a single use, as {{Section 5.5 of I-D.ietf-oauth-identity-chaining}} suggests, could mark a bound grant as used when a party without the key presents it, denying the legitimate client its redemption. It could also mark the grant as used when it rejects a request with the `use_dpop_nonce` error code, so that the client's retry with the nonce fails. {{processing}} prevents both by recording a grant as used only when the request succeeds. {{binding-request}} applies the same rule to a single-use `subject_token` at the issuing authorization server.

## Replay by the Key Holder

Binding limits who can redeem a grant, not how often. A client, or an attacker that has obtained the client's key, can redeem a bound grant until it expires unless the consuming authorization server limits it to a single use. Short grant lifetimes and single-use enforcement ({{Section 5.5 of I-D.ietf-oauth-identity-chaining}}) remain applicable to bound grants.

## Cross-JWT Confusion

A bound grant and a DPoP-bound JWT access token for the same client can carry the same `jkt` value. A party that receives one could present it where the other is expected. The `aud` claim of a grant identifies an authorization server ({{RFC10070}}), and an explicit `typ` value ({{Section 3.11 of RFC8725}}) lets a recipient reject a JWT of the wrong kind ({{Section 3.12 of RFC8725}}). When the profile of a grant defines a `typ` value, a consuming authorization server SHOULD reject a grant that does not carry it with the `invalid_grant` error code.

# Privacy Considerations {#privacy}

The privacy considerations of {{RFC9449}} and {{RFC8705}} apply to this specification.

A confirmation member is a stable identifier for the bound key or certificate. A client that uses the same key or certificate with several consuming authorization servers lets them correlate its requests. A client that needs to avoid such correlation uses a separate key for each consuming authorization server. This is not possible when the client exchanges a DPoP-bound `subject_token`, because the grant is then bound to that token's key ({{binding-request}}).

The `cnf` response parameter discloses the binding only to the client that made the request, which already holds the key or certificate. A thumbprint member discloses less than a `jwk` member and is sufficient for the client to verify the binding, which is why {{cnf-param}} recommends it.

# IANA Considerations {#iana}

## OAuth Parameters Registration

The `cnf` parameter is registered in the "OAuth Parameters" registry {{IANA.oauth-parameters}} with the usage location "token response" by {{RFC9201}}. This specification requests that IANA add this document as a reference for that entry:

Parameter name:
: `cnf`

Parameter usage location:
: token response

Change controller:
: IETF

Specification document(s):
: {{Section 5 of RFC9201}} and {{cnf-param}} of this document

## OAuth URI Registration

This specification requests registration of the following value in the "OAuth URI" registry {{IANA.oauth-parameters}} established by {{RFC6755}}:

URN:
: `urn:ietf:params:oauth:grant-type:jwt-dpop`

Common Name:
: DPoP-Bound JWT Authorization Grant

Change Controller:
: IETF

Specification Document(s):
: {{jwt-dpop}} of this document

--- back

# Profiles of This Specification {#profile-examples}

This appendix is informative. It shows how two specifications answer the items in {{profiles}}.

## Identity Assertion JWT Authorization Grant

{{I-D.ietf-oauth-identity-assertion-authz-grant}} would answer the items as follows:

1. The `typ` value is `oauth-id-jag+jwt`, and the `requested_token_type` value is `urn:ietf:params:oauth:token-type:id-jag`.
2. Both confirmation methods, `jkt` and `x5t#S256`, may be used. Neither is mandatory to implement, nor is the `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type.
3. Binding is optional.
4. A Resource Authorization Server may accept a bound grant in place of client authentication. The IdP Authorization Server authenticates the client in the Token Exchange request, binds the grant to the key or certificate for which that request carries proof of possession, and identifies the client in the grant's `client_id` claim.

Version -06 of that document differs from this specification in the following ways, which would need to be aligned:

* {{Section 9.11 of I-D.ietf-oauth-identity-assertion-authz-grant}} lets the IdP Authorization Server choose a certificate binding when a request carries both a DPoP proof and a TLS client certificate. This specification requires a grant issued in response to a binding request to be bound to the DPoP key ({{binding-request}}).
* {{Section 9.11.1.1 of I-D.ietf-oauth-identity-assertion-authz-grant}} and {{Section 9.11.2.1 of I-D.ietf-oauth-identity-assertion-authz-grant}} have the client read the grant's `cnf` claim to detect a downgrade. This specification adds the `cnf` response parameter, and also accepts the claim ({{client-processing}}).
* {{Section 9.11.2.2 of I-D.ietf-oauth-identity-assertion-authz-grant}} requires the Resource Authorization Server to validate the certificate per {{Section 2 of RFC8705}}. This specification requires only proof of possession of the certificate ({{processing}}).
* {{Section 9.11.1.2.4 of I-D.ietf-oauth-identity-assertion-authz-grant}} and {{Section 9.11.2.2 of I-D.ietf-oauth-identity-assertion-authz-grant}} define an `invalid_grant` error response when an unbound grant is presented for a resource that requires sender-constrained access tokens. This specification defines no error for that case ({{open-issues}}).
* {{Section 8 of I-D.ietf-oauth-identity-assertion-authz-grant}} requires a client that supports that profile to include `urn:ietf:params:oauth:grant-type:jwt-bearer` in its `grant_types`, which prevents a Resource Authorization Server from limiting the client to DPoP-bound grants ({{client-metadata}}).

## Identity and Authorization Chaining

{{I-D.ietf-oauth-identity-chaining}} defines no profile of this specification, so the defaults in {{profiles}} apply. Binding the grant adds a mitigation to those listed in {{Section 5.5 of I-D.ietf-oauth-identity-chaining}}.

# Open Issues {#open-issues}

\[Editor's note: this appendix lists questions for the working group. It will be removed before publication.\]

1. Grant type for certificate-bound grants. The `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type covers only DPoP-bound grants ({{jwt-dpop}}). Should certificate-bound grants have an equivalent grant type, or should one method-neutral grant type replace `urn:ietf:params:oauth:grant-type:jwt-dpop`, at the cost of breaking deployments that use it?
2. Unhonored binding request. This draft rejects a binding request that the issuing authorization server cannot honor for the consuming authorization server ({{bind}}) with the `invalid_target` error code. Because {{Section 2.2.2 of RFC8693}} also uses that code for an unacceptable audience, a client cannot tell whether a retry without a DPoP proof would succeed. Should this be `invalid_request` or a new error code instead? Alternatively, should the issuing authorization server issue an unbound grant without the `cnf` response parameter and let the client decide ({{client-processing}})?
3. Bound grants versus sender-constrained access tokens. A profile that requires bound grants is distinct from a consuming authorization server that requires sender-constrained access tokens for a resource: an unbound grant presented with a DPoP proof or over mutual TLS can still yield a sender-constrained access token ({{Section 5 of RFC9449}}, {{Section 3 of RFC8705}}). {{I-D.ietf-oauth-identity-assertion-authz-grant}} rejects an unbound grant in the second case when the request carries neither a DPoP proof nor a TLS client certificate ({{Section 9.11.1.2.4 of I-D.ietf-oauth-identity-assertion-authz-grant}}, {{Section 9.11.2.2 of I-D.ietf-oauth-identity-assertion-authz-grant}}). Should this specification define an error response for a request that carries no proof of possession when the resource requires a sender-constrained access token, or leave that to {{RFC9449}} and {{RFC8705}}?
4. Scope of the response parameter. {{RFC9201}} registers the `cnf` token response parameter for ACE. Should this document define its use for Token Exchange only, for every token response, or move it to a separate specification?
5. Discovery independent of grant type. `grant_types_supported` tells an issuing authorization server that a consuming authorization server verifies DPoP-bound grants ({{as-metadata}}). Nothing indicates that a consuming authorization server verifies certificate-bound grants, or verifies bound grants presented with the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type, and no client metadata indicates that a client redeems certificate-bound grants ({{binding-request}}). Should an authorization server metadata parameter list the confirmation methods that a consuming authorization server verifies for JWT authorization grants, and a client metadata parameter indicate that a client accepts certificate-bound grants, with `urn:ietf:params:oauth:grant-type:jwt-dpop` retained for registration policy?
6. Alignment with {{I-D.ietf-oauth-identity-assertion-authz-grant}}. The differences listed in {{profile-examples}} need to be resolved in one document or the other.

# Acknowledgments
{:numbered="false"}

This document combines and replaces {{I-D.parecki-oauth-jwt-dpop-grant}} and {{I-D.mcguinness-oauth-token-exchange-cnf}}, and generalizes the sender-constraining rules of {{I-D.ietf-oauth-identity-assertion-authz-grant}}. The author thanks the authors and reviewers of those documents.

# Document History
{:numbered="false"}

-00

* Initial version, combining {{I-D.parecki-oauth-jwt-dpop-grant}} and {{I-D.mcguinness-oauth-token-exchange-cnf}}.
* Retains the `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type and also verifies bound grants presented with `urn:ietf:params:oauth:grant-type:jwt-bearer`.
