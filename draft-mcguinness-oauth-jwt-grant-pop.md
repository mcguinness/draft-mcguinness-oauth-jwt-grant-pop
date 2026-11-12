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
  RFC7591:
  RFC7638:
  RFC7800:
  RFC8414:
  RFC8693:
  RFC8705:
  RFC9201:
  RFC9449:
  RFC10070:

informative:
  IANA.oauth-parameters:
  RFC6755:
  RFC7516:
  RFC8725:
  RFC9700:
  ID-CHAINING: I-D.ietf-oauth-identity-chaining
  ID-JAG: I-D.ietf-oauth-identity-assertion-authz-grant
  JWT-DPOP-GRANT: I-D.parecki-oauth-jwt-dpop-grant
  TE-CNF: I-D.mcguinness-oauth-token-exchange-cnf

--- abstract

This specification defines how an authorization server binds a JSON Web Token (JWT) authorization grant (RFC 7523) to a key or certificate held by the client, using OAuth 2.0 Demonstrating Proof of Possession (DPoP) or OAuth 2.0 Mutual-TLS, and how the authorization server that receives the grant verifies the binding. It profiles OAuth 2.0 Token Exchange so that a client can request a DPoP-bound grant and learn which binding was applied. It also defines a grant type that requires a key-bound grant and an authorization server metadata parameter that lists the confirmation methods that a server verifies in JWT authorization grants.

--- middle

# Introduction

A JWT authorization grant ({{Section 2.1 of RFC7523}}) lets a client obtain an access token from an authorization server by presenting a JWT issued by a party that the authorization server trusts. Identity and authorization chaining {{ID-CHAINING}} uses OAuth 2.0 Token Exchange {{RFC8693}} to obtain such a grant in one trust domain and present it in another, and the Identity Assertion JWT Authorization Grant {{ID-JAG}} profiles that pattern for deployments that share a single sign-on identity provider. Without a proof-of-possession binding, these grants are bearer tokens ({{Section 5.5 of ID-CHAINING}}): any party that obtains one can redeem it until it expires. The mitigations listed there do not include binding the grant to a key.

OAuth 2.0 Demonstrating Proof of Possession (DPoP) {{RFC9449}} and OAuth 2.0 Mutual-TLS {{RFC8705}} bind access tokens, and sometimes refresh tokens, to a key or certificate held by the client. They do not define:

* how an authorization server binds a JWT authorization grant that Token Exchange issues, for which the `token_type` response parameter is `N_A` ({{Section 2.2.1 of RFC8693}});
* how the authorization server that receives a grant with a `cnf` claim {{RFC7800}} verifies proof of possession; or
* how a client learns which binding was applied when it cannot read the issued token, for example because the token is opaque or encrypted for its consumer {{RFC7516}}.

This specification defines how a binding is applied and how it is verified:

* {{format}}: what a key-bound authorization grant contains;
* {{issuance}}: how a grant is bound when it is issued, and how the client learns the binding;
* {{redemption}}: how the resource authorization server verifies the binding, however the grant was issued;
* {{metadata}}: discovery and per-client enforcement; and
* {{profiles}}: the choices that a specification defining a particular JWT authorization grant makes.

This specification defines no new confirmation method. It uses the `jkt` ({{Section 6.1 of RFC9449}}) and `x5t#S256` ({{Section 3.1 of RFC8705}}) confirmation methods, and lets a resource authorization server also accept the `jwk` confirmation method ({{Section 3.2 of RFC7800}}).

## Relationship to Other Documents {#relationship}

This specification relates to other documents as follows:

* It replaces {{JWT-DPOP-GRANT}} and extends it from DPoP to mutual TLS. It generalizes the `jwt-dpop` grant type of that document into the method-neutral `jwt-pop` grant type ({{jwt-pop}}) and keeps `jwt-dpop` for existing clients; {{legacy-jwt-dpop}} lists the other differences.
* It replaces {{TE-CNF}}, whose Token Exchange response parameter it incorporates ({{cnf-param}}). That parameter is the `cnf` token response parameter that {{RFC9201}} registers for the Authentication and Authorization for Constrained Environments (ACE) framework.
* It generalizes the sender-constraining rules that the Identity Assertion JWT Authorization Grant {{ID-JAG}} defines for its grant, and {{profile-examples}} shows that grant as a profile of this specification and lists the conflicts between the two documents.

## Conventions and Terminology

{::boilerplate bcp14-tagged}

This specification uses the terms "access token", "authorization grant", "authorization server", "client", "refresh token", and "token endpoint" defined by {{RFC6749}}, the terms "JSON Web Token (JWT)" and "claim" defined by {{RFC7519}}, and the term "DPoP proof" defined by {{RFC9449}}. It also defines the following terms:

JWT authorization grant:
: A JWT used as an authorization grant as defined in {{Section 2.1 of RFC7523}}.

Issuing authorization server:
: The authorization server that issues a JWT authorization grant, such as the IdP Authorization Server of {{ID-JAG}}.

Resource authorization server:
: The authorization server to which a client presents a JWT authorization grant in an access token request ({{Section 4.1 of RFC7521}}), typically the authorization server for the resources that the client wants to access, such as the Resource Authorization Server of {{ID-JAG}}.

Key-bound authorization grant:
: A JWT authorization grant that contains a `cnf` claim ({{format}}). A DPoP-bound authorization grant has a `jkt` or `jwk` confirmation member; a certificate-bound authorization grant has an `x5t#S256` confirmation member.

Confirmation member:
: A member of a `cnf` claim or of the `cnf` response parameter, such as `jkt` or `x5t#S256`.

DPoP binding request:
: A Token Exchange request that contains a DPoP proof ({{te-request}}).

Permitted confirmation method:
: A confirmation method that the issuing authorization server supports and that both the profile of an authorization grant and the issuing authorization server's policy permit for that grant ({{issuer-processing}}).

This specification refers to grant types by the short names in {{grant-type-names}}.

| Short name | Grant type | Defined in |
|---|---|---|
| `jwt-bearer` | `urn:ietf:params:oauth:grant-type:jwt-bearer` | {{Section 2.1 of RFC7523}} |
| `jwt-pop` | `urn:ietf:params:oauth:grant-type:jwt-pop` | {{jwt-pop}} |
| `jwt-dpop` | `urn:ietf:params:oauth:grant-type:jwt-dpop` | {{legacy-jwt-dpop}} |
{: #grant-type-names title="Grant Type Short Names"}

The examples in this specification are non-normative. They use the public key from the examples in {{RFC9449}}, whose JWK SHA-256 Thumbprint is `0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I`, an issuing authorization server at `https://issuer.example/`, and a resource authorization server at `https://as.example/`. Request bodies are shown with extra line breaks and without URL encoding, and tokens are truncated, for display purposes only.

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
|        |--(3) jwt-pop request ---------->|   Resource    |
|        |     with grant and DPoP proof   | Authorization |
|        |     for key K                   |    Server     |
|        |<-(4) access token --------------|               |
|        |     (normally bound to K)       |               |
+--------+                                 +---------------+
~~~
{: title="Issuing and Redeeming a DPoP-Bound Authorization Grant"}

1. The client sends a Token Exchange request with a DPoP proof for its key K ({{te-request}}).
2. The issuing authorization server issues an authorization grant whose `cnf` claim identifies K ({{bind}}) and returns the binding in the `cnf` response parameter ({{cnf-param}}), which the client checks ({{client-processing}}).
3. The client presents the authorization grant to the resource authorization server with a DPoP proof for K ({{token-request}}).
4. The resource authorization server verifies the proof against the authorization grant's `cnf` claim ({{processing}}) and issues an access token, normally bound to K ({{token-response}}).

The client sends a separate DPoP proof to each authorization server, signed with the same key, because the `htu` claim of a DPoP proof identifies the endpoint that receives it ({{Section 4.2 of RFC9449}}). With mutual TLS, the client's TLS client certificate takes the place of K: the client proves possession in the TLS handshake instead of with a DPoP proof.

# Key-Bound JWT Authorization Grants {#format}

This section defines the `cnf` claim of a key-bound authorization grant and how confirmation values are compared.

## Confirmation Claim {#cnf-claim}

An issuing authorization server binds an authorization grant by including in it a `cnf` claim ({{Section 3.1 of RFC7800}}). The confirmation member of the claim identifies the key or certificate whose possession the client proves when it redeems the grant:

* `jkt` ({{Section 6.1 of RFC9449}}), in a DPoP-bound authorization grant, contains the JWK SHA-256 Thumbprint {{RFC7638}} of the client's public key.
* `x5t#S256` ({{Section 3.1 of RFC8705}}), in a certificate-bound authorization grant, contains the SHA-256 thumbprint of the DER encoding of the client's certificate.

The `cnf` claim of an authorization grant that an issuing authorization server binds MUST contain exactly one of these members. Requiring exactly one member is narrower than {{Section 3.1 of RFC7800}}, which requires only that a `cnf` claim "represent only a single proof-of-possession key".

A resource authorization server MAY also accept a `jwk` member ({{Section 3.2 of RFC7800}}) in a DPoP-bound authorization grant. It rejects a `cnf` claim unless the claim contains exactly one confirmation member and the resource authorization server supports that member's confirmation method ({{processing}}).

The following is an example of the claims of a DPoP-bound authorization grant:

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
{: #ex-grant title="Claims of a DPoP-Bound Authorization Grant"}

A certificate-bound authorization grant instead contains a `cnf` claim such as the following:

~~~ json
{
  "cnf": {
    "x5t#S256": "bwcK0esc3ACC3DB2Y5_lESsXE8o9ltc05O89jdN-dg0"
  }
}
~~~
{: title="Confirmation Claim of a Certificate-Bound Authorization Grant"}

## Comparing Confirmation Values {#cnf-comparison}

Two confirmation values, or a confirmation value and the client's own key or certificate, are compared by confirmation method and value, not by byte representation:

* A thumbprint member, such as `jkt` or `x5t#S256`, is compared as the base64url-encoded string it contains, character by character.
* A public key in a `jwk` member is compared with a `jkt` member, or with the client's own public key, by computing JWK SHA-256 Thumbprints {{RFC7638}}.
* Members of different confirmation methods, other than a `jwk` and `jkt` pair, do not match.

For example, the following `jwk` member matches a `jkt` member with the value `0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I`, the JWK SHA-256 Thumbprint of its key:

~~~ json
{
  "jwk": {
    "kty": "EC",
    "crv": "P-256",
    "x": "l8tFrhx-34tV3hRICRDY9zCkDlpBhF42UQUfWVAWBFs",
    "y": "9VE4jf_Ok_o64zbTTlcuNJajHmt6v9TDVrU0CdvGRDA"
  }
}
~~~
{: title="Confirmation Value with a jwk Member"}

# Issuing a Key-Bound Authorization Grant {#issuance}

This section covers a JWT authorization grant that an issuing authorization server issues in response to a Token Exchange request ({{Section 2.1 of RFC8693}}). {{cnf-param}} applies to any token that Token Exchange issues, and {{other-issuance}} covers other issuance interfaces.

An issued token is a JWT authorization grant if the issuing authorization server issues it as a JWT for the client to present to an authorization server as an authorization grant. The issuing authorization server decides this, and whether to issue the grant at all, from its policy, from its configuration for the `audience` and `resource` values, and from the `requested_token_type` value, which a profile can define.

## Token Exchange Request {#te-request}

A client requests a DPoP-bound authorization grant by including a DPoP proof ({{Section 4 of RFC9449}}) in the DPoP header field of a Token Exchange request, as it would for a DPoP-bound access token. Such a request is a DPoP binding request. The client signs the DPoP proof with the key that it will use to redeem the grant.

A TLS client certificate in a request does not ask for a certificate binding: the client may be presenting the certificate only for mutual-TLS client authentication ({{Section 2 of RFC8705}}). An issuing authorization server binds a grant to that certificate only as {{cert-binding}} describes.

If the `subject_token` or `actor_token` is bound to a key or certificate, the client MUST prove possession of that same key or certificate in the request, with a DPoP proof signed with the key or over a mutual-TLS connection that uses the certificate, unless the client is exempt ({{bound-subject}}). Two consequences follow:

* A request carries at most one DPoP proof ({{Section 4.3 of RFC9449}}). A request whose `subject_token` and `actor_token` are bound to different DPoP keys therefore proves possession of at most one of them, and is rejected unless the client is exempt for each token whose key the DPoP proof does not match.
* An authorization grant issued in response to a DPoP binding request is bound to the key of that request's DPoP proof, or unbound ({{binding-request}}). A key-bound grant therefore has the same key as each DPoP-bound `subject_token` or `actor_token` whose possession the request proves.

No request parameter lets a client require a key-bound authorization grant for an individual request. Binding is required when the profile of the grant, the issuing authorization server's policy, or the client's registration at the issuing authorization server requires it, and the issuing authorization server then binds the grant or rejects the request ({{required-binding}}). A client that always needs a key-bound grant can therefore request a grant whose profile requires binding, or arrange with the issuing authorization server for its registration to require binding; this specification defines no client metadata for that arrangement. A client that requires a key-bound grant also checks the response ({{client-processing}}).

The following is an example of a DPoP binding request for an authorization grant whose audience is the resource authorization server `https://as.example/`, from a client that authenticates with a client secret:

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
{: #ex-binding-request title="Token Exchange Request with a DPoP Proof"}

The decoded JOSE header and claims of the DPoP proof in that request are:

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

~~~ json
{
  "jti": "e1j3V_bKic8-LAEB",
  "htm": "POST",
  "htu": "https://issuer.example/token",
  "iat": 1730000000
}
~~~
{: title="DPoP Proof Claims for the Issuing Authorization Server"}

## Issuing Authorization Server Processing {#issuer-processing}

An issuing authorization server that supports this specification MUST support the `cnf` response parameter ({{cnf-param}}) and at least one of the `jkt` and `x5t#S256` confirmation methods, and MUST NOT bind an authorization grant other than as described in this section. When it rejects a request for a JWT authorization grant, the issuing authorization server MUST NOT treat the `subject_token` or `actor_token` as used, for example by rotating a refresh token ({{single-use}}).

For a Token Exchange request for a JWT authorization grant, the issuing authorization server MUST also ensure the following, and MUST otherwise reject the request with the indicated error code ({{errors}}):

1. Any DPoP proof that the request contains is valid as described in {{Section 4.3 of RFC9449}}; otherwise, `invalid_dpop_proof`, or `use_dpop_nonce` as described in {{Section 8 of RFC9449}}.
2. For each `subject_token` or `actor_token` that is bound to a key or certificate, the request proves possession of that key or certificate, or the requesting client is exempt ({{bound-subject}}); otherwise, `invalid_request` ({{Section 2.2.2 of RFC8693}}).
3. Binding is not required, or the issuing authorization server binds the authorization grant ({{binding-request}}, {{cert-binding}}); otherwise, the error code that {{required-binding}} gives.

When a request fails more than one of these checks, {{errors}} determines which error code the issuing authorization server returns. For example, a request without a DPoP proof that fails item 2, and for which {{required-binding}} gives `invalid_dpop_proof` under item 3, is rejected with `invalid_dpop_proof`.

Whether a confirmation method is a permitted confirmation method for an authorization grant does not depend on the proof of possession in the request. A permitted confirmation method does not by itself permit binding; {{binding-request}} and {{cert-binding}} define when the issuing authorization server binds the grant.

The issuing authorization server's policy can decline a confirmation method for an authorization grant whose `aud` claim identifies an authorization server that does not verify that method, as the issuing authorization server has determined from configuration, an established trust relationship, or metadata ({{as-metadata}}). That method is then not a permitted confirmation method for the grant, and {{binding-request}}, {{cert-binding}}, and {{required-binding}} apply as they do to a method that the issuing authorization server does not support.

The subsections below follow these checks in order: proof of possession of bound subject and actor tokens (item 2), selection of the binding, the error code when a required binding is not applied (item 3), and construction of a key-bound authorization grant.

### Bound Subject and Actor Tokens {#bound-subject}

Item 2 of {{issuer-processing}} applies to a `subject_token` or `actor_token` that is bound to a key or certificate, such as a refresh token issued to a public client ({{Section 5 of RFC9449}}). A request proves possession of that key or certificate with a DPoP proof signed with the key, or with a mutual-TLS handshake that uses the certificate. The requesting client is exempt if it authenticates and is an intended audience of the token, other than the client to which the token was issued, such as a resource server that exchanges an access token it received ({{Section 2.3 of RFC8693}}).

The issuing authorization server determines whether the requesting client is an intended audience of the token from the token's `aud` claim or its own records of the token, for example by associating a resource server's client registration with the resource identifiers that the `aud` claim contains.

The issuing authorization server applies item 2 to each bound token separately. {{te-request}} describes what this means for a request with two bound tokens, and for the key of the issued authorization grant.

### DPoP Binding {#binding-request}

In response to a DPoP binding request with a valid DPoP proof, the issuing authorization server MUST bind the issued authorization grant to the public key of the DPoP proof if `jkt` is a permitted confirmation method for the grant. Otherwise, it MUST issue an unbound grant, without the `cnf` response parameter, unless binding is required. It MUST NOT bind the grant to any other key or certificate, including when the client also presents a TLS client certificate.

The DPoP proof thus selects the grant's key. A TLS client certificate in the same request only authenticates the client or proves possession of a certificate-bound `subject_token` or `actor_token`. This has two benefits:

* A client can authenticate to the issuing authorization server with a certificate that it does not use elsewhere without receiving a grant bound to that certificate, which it might not be able to redeem.
* An attacker that replays a captured DPoP-bound `subject_token` or `actor_token` with its DPoP proof cannot obtain a grant bound to the attacker's own certificate ({{proof-replay}}).

A client that requires a key-bound authorization grant discards an unbound one. Issuing an unbound grant when `jkt` is not a permitted confirmation method keeps the exchange working for a client that sends a DPoP proof in every token request, and for a client that proves possession of a DPoP-bound `subject_token`.

### Certificate Binding {#cert-binding}

If a request contains no DPoP proof, arrives over a mutual-TLS connection in which the client presented a TLS client certificate, and `x5t#S256` is a permitted confirmation method for the authorization grant, the issuing authorization server MAY bind the grant to that certificate. It SHOULD NOT do so unless the profile of the grant requires certificate binding, or the client's registration or configuration at the issuing authorization server indicates that the client presents that certificate when it redeems the grant. Binding in any other case could give a client that does not support this specification a grant that it cannot redeem. Neither the `tls_client_certificate_bound_access_tokens` client metadata parameter ({{Section 3.4 of RFC8705}}), which concerns access tokens, nor a requirement for binding is such an indication.

The following is an example of a Token Exchange response for a certificate-bound authorization grant, issued to a client whose configuration at the issuing authorization server indicates that it redeems grants over mutual TLS with the certificate it used to authenticate:

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
{: #cert-example title="Token Exchange Response for a Certificate-Bound Authorization Grant"}

### Required Binding {#required-binding}

Binding is required for an authorization grant if the profile of the grant requires it, or if the issuing authorization server requires it by its policy (for example, for the grant's audience) or by the client's registration at the issuing authorization server. An issuing authorization server MAY require binding when the profile of the grant does not. A requirement for binding ensures that an issued grant is bound (item 3 of {{issuer-processing}}), not that each authorization server in the grant's audience verifies the binding; where that is needed, the profile, configuration, or the issuing authorization server's policy establishes it ({{non-enforcing-consumers}}).

When binding is required and the issuing authorization server does not bind the authorization grant, it rejects a Token Exchange request with the error code in {{required-binding-errors}} (item 3 of {{issuer-processing}}):

| Request contains | Permitted confirmation methods for the grant | Error code |
|---|---|---|
| No DPoP proof | Include `jkt` | `invalid_dpop_proof` |
| Neither a DPoP proof nor a TLS client certificate | Include `x5t#S256` but not `jkt`, and the issuing authorization server would bind the grant to the client's TLS client certificate, if the request had one, as {{cert-binding}} describes | `invalid_request` |
| Any other case | Any | `invalid_target` ({{Section 2.2.2 of RFC8693}}) |
{: #required-binding-errors title="Error Codes When a Required Binding Is Not Applied"}

The last row covers, for example, a request for a grant for which no confirmation method is permitted, and a request that contains a DPoP proof when `jkt` is not a permitted confirmation method for the grant ({{binding-request}}).

A client that receives `invalid_dpop_proof` under the first row can repeat the request with a DPoP proof, and one that receives `invalid_request` under the second row can repeat it over a mutual-TLS connection with its TLS client certificate. With `invalid_dpop_proof`, the issuing authorization server MAY include a DPoP-Nonce HTTP header field ({{Section 8.2 of RFC9449}}) so that the repeated request can include the nonce and need not first be rejected with `use_dpop_nonce`.

The following is an example of the error response to a Token Exchange request that contains no DPoP proof, when binding is required and `jkt` is a permitted confirmation method for the authorization grant:

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
DPoP-Nonce: eyJ7S_zG.eyJH0-Z.HX4w-7v

{
  "error": "invalid_dpop_proof",
  "error_description": "DPoP proof required to bind the grant"
}
~~~
{: title="Error Response for a Request without a Required DPoP Proof"}

The following is an example of the `invalid_target` error response to a DPoP binding request when binding is required and `jkt` is not a permitted confirmation method for the grant:

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
  "error": "invalid_target",
  "error_description": "DPoP binding is not permitted for this grant"
}
~~~
{: title="Error Response for a DPoP Binding Request That Cannot Be Honored"}

### Binding the Authorization Grant {#bind}

To bind an authorization grant, the issuing authorization server includes a `cnf` claim ({{format}}) whose confirmation member identifies the client's key or certificate: for a DPoP binding request, the public key in the `jwk` JOSE Header Parameter of the DPoP proof; for a certificate binding, the TLS client certificate that the client presented. The issuing authorization server MUST NOT bind a grant to a key or certificate whose possession the client has not proven in the request for the grant.

The issuing authorization server does not need to determine, before binding, that the resource authorization server verifies the binding. The resource authorization server enforces it ({{processing}}); where a resource authorization server does not, a key-bound grant is no less protected against theft than an unbound one ({{non-enforcing-consumers}}). If binding is required and the issuing authorization server does not bind the grant, it MUST NOT issue the grant; in response to a Token Exchange request, it returns the error code that {{required-binding}} gives. When binding is not required, {{binding-request}} defines the response to a DPoP binding request when `jkt` is not a permitted confirmation method for the grant.

When the issued token is a key-bound authorization grant, the issuing authorization server MUST set the `token_type` response parameter to `N_A` ({{Section 2.2.1 of RFC8693}}); the `DPoP` value that {{Section 5 of RFC9449}} requires applies only to a DPoP-bound access token.

{{ex-grant}} shows the claims of the authorization grant issued in response to the DPoP binding request in {{ex-binding-request}}.

## Token Exchange Response {#cnf-param}

This specification defines the use of the `cnf` token response parameter, registered by {{RFC9201}}, in a Token Exchange response:

`cnf`:
: REQUIRED if the authorization server bound the token in the `access_token` response parameter to a key or certificate and the `token_type` response parameter is `N_A`. For a bound token with another `token_type` value, OPTIONAL unless another specification requires it, as {{Section 3.2 of RFC9201}} does for a symmetric proof-of-possession key. It MUST NOT be present if the token is not bound. A JSON object containing exactly one confirmation member, which identifies that key or certificate. Its syntax and member semantics are those of the `cnf` claim ({{Section 3.1 of RFC7800}}).

The parameter describes only the token in the `access_token` response parameter, whatever its `issued_token_type`, including an opaque or encrypted token and an access token whose `token_type` also indicates the binding. It does not describe a `refresh_token` in the same response.

When the issued token is a JWT that contains a `cnf` claim, the parameter MUST identify the same key or certificate as that claim ({{cnf-comparison}}). The authorization server SHOULD use the same confirmation member in the parameter as in the claim, and SHOULD use a thumbprint member, such as `jkt` or `x5t#S256`, rather than a `jwk` member ({{privacy}}).

The following is an example of the Token Exchange response to the DPoP binding request in {{ex-binding-request}}:

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
{: #ex-te-response title="Token Exchange Response with the cnf Parameter"}

## Client Processing of the Response {#client-processing}

When the `token_type` response parameter is `N_A`, as it is for a JWT authorization grant, a client that sent a DPoP binding request, or that presented a TLS client certificate and requires a key-bound grant, processes the Token Exchange response as follows:

1. If the `cnf` parameter is present and is not a JSON object that contains exactly one confirmation member, the client MUST discard the issued token.
2. The client determines the binding from the `cnf` parameter and, if it can read the issued token, from the token's `cnf` claim. If neither is present, the authorization grant is not bound, and a client that requires a key-bound grant MUST discard it.
3. The client MUST verify the following, and MUST discard the issued token if any check fails:
   * if the `cnf` parameter is present and the client can read the authorization grant, the grant contains a `cnf` claim;
   * the `cnf` claim, if readable, contains exactly one confirmation member; and
   * the confirmation member of the `cnf` parameter, if present, and of the `cnf` claim, if readable, each match the client's own key or certificate ({{cnf-comparison}}): after a DPoP binding request, the public key of its DPoP proof; otherwise, its TLS client certificate. A confirmation member that the client does not recognize does not match.

For an access token, a `token_type` of `DPoP` shows a DPoP binding ({{Section 5 of RFC9449}}), but {{RFC8705}} defines no `token_type` value for a certificate binding; the `cnf` parameter, if present, identifies the key or certificate of either binding.

A client that did not send a DPoP binding request but receives a `cnf` parameter presents the token with the corresponding proof if it holds the identified key or certificate, such as the certificate it used for mutual-TLS client authentication. Otherwise, it SHOULD treat the response as an error.

For the response in {{ex-te-response}}, the client verifies that the `jkt` value in the `cnf` response parameter equals the JWK SHA-256 Thumbprint of its DPoP public key.

## Other Issuance Interfaces {#other-issuance}

When a JWT authorization grant is issued through an interface other than Token Exchange, such as a workload platform's token service, {{bind}} applies except for its requirements on the Token Exchange response, with that interface's request in place of the Token Exchange request. The interface defines how the client requests the binding and learns of it. {{redemption}} applies however the grant was issued.

# Redeeming a Key-Bound Authorization Grant {#redemption}

This section defines how a client presents a key-bound authorization grant, how the resource authorization server verifies its binding, and how that server binds the access and refresh tokens that it issues.

## Access Token Request {#token-request}

The client presents a key-bound authorization grant in an access token request ({{Section 4.1 of RFC7521}}) with the `jwt-pop`, the `jwt-bearer`, or, where {{legacy-jwt-dpop}} applies, the `jwt-dpop` grant type. It SHOULD use `jwt-pop` if the resource authorization server advertises it ({{as-metadata}}) and the client's registration permits it ({{client-metadata}}). Otherwise, a client that presents a grant with a `jkt` member SHOULD use `jwt-dpop` if the resource authorization server advertises it and the client's registration permits it. The `jwt-bearer` grant type is not a reliable fallback: the client's registration might not permit it, and a server that implements only {{JWT-DPOP-GRANT}} rejects such a grant presented with it.

The client proves possession of the key or certificate identified by the authorization grant's `cnf` claim:

* For a `jkt` or `jwk` member, the client MUST include a DPoP proof ({{Section 4 of RFC9449}}) signed with that key in the DPoP header field of the request.
* For an `x5t#S256` member, the client MUST send the request over a mutual-TLS connection that uses that certificate, to the token endpoint or to its mutual-TLS endpoint alias ({{Section 5 of RFC8705}}).

### Proof-of-Possession Grant Type {#jwt-pop}

To use the `jwt-pop` grant type, the client sends the following parameters and proves possession of the authorization grant's key or certificate as described in {{token-request}}:

`grant_type`:
: REQUIRED. The value `urn:ietf:params:oauth:grant-type:jwt-pop`.

`assertion`:
: REQUIRED. A single JWT authorization grant that contains a `cnf` claim.

`scope`:
: OPTIONAL. The requested scope, as defined in {{Section 4.1 of RFC7521}}.

The resource authorization server processes the request as defined in {{processing}}, and MUST also reject it with the `invalid_grant` error code if the authorization grant has no `cnf` claim.

The following is an example of an access token request that presents the authorization grant in {{ex-grant}} with the `jwt-pop` grant type, from a client that does not authenticate:

~~~ http-message
POST /token HTTP/1.1
Host: as.example
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6...

grant_type=urn:ietf:params:oauth:grant-type:jwt-pop
&assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0...
~~~
{: #ex-redeem title="Access Token Request with a DPoP-Bound Authorization Grant"}

### Legacy DPoP-Bound Grant Type {#legacy-jwt-dpop}

The `jwt-dpop` grant type, first defined in {{JWT-DPOP-GRANT}}, is a legacy grant type for existing clients. A resource authorization server MAY support it, and SHOULD continue to support it while clients are registered for it. A resource authorization server that supports it SHOULD include it in the `grant_types_supported` metadata parameter; an issuing authorization server that implements {{JWT-DPOP-GRANT}} can rely on that value to determine that the resource authorization server verifies DPoP-bound authorization grants. The resource authorization server processes a request with that grant type by the rules for the `jwt-pop` grant type ({{jwt-pop}}), and MUST also reject the request with the `invalid_grant` error code if the confirmation member of the grant's `cnf` claim is not `jkt`. A new client uses the `jwt-pop` grant type.

Unlike {{JWT-DPOP-GRANT}}, this specification:

* binds an authorization grant without first checking that the resource authorization server verifies the binding ({{non-enforcing-consumers}}), contrary to the recommendation in {{Section 7.1 of JWT-DPOP-GRANT}};
* verifies, instead of rejecting, a key-bound authorization grant presented with the `jwt-bearer` grant type ({{grant-type-substitution}}); and
* returns `invalid_grant`, not `invalid_dpop_proof`, when a resource authorization server receives a DPoP-bound authorization grant without a DPoP proof ({{processing}}).

## Resource Authorization Server Processing {#processing}

A resource authorization server validates a JWT authorization grant as defined in {{Section 3 of RFC7523}}, as updated by {{Section 4 of RFC10070}}, and by the profile of the grant. If the grant contains a `cnf` claim, a resource authorization server that supports this specification MUST also ensure the following, whichever grant type the request uses, and MUST otherwise reject the request with the indicated error code ({{errors}}):

1. For the `cnf` claim:
   * it is a JSON object that contains exactly one confirmation member; otherwise, `invalid_grant`;
   * that member is `jkt`, `jwk`, or `x5t#S256`; otherwise, `invalid_grant`; and
   * the resource authorization server supports its confirmation method; otherwise, `invalid_grant`.
2. For a `jkt` or `jwk` member:
   * the request contains a DPoP proof; otherwise, `invalid_grant`;
   * the DPoP proof is valid as described in {{Section 4.3 of RFC9449}}; otherwise, `invalid_dpop_proof`, or `use_dpop_nonce` as described in {{Section 8 of RFC9449}}; and
   * the JWK SHA-256 Thumbprint {{RFC7638}} of the public key in the `jwk` JOSE Header Parameter of the DPoP proof equals the `jkt` value, or the JWK SHA-256 Thumbprint of the key in the `jwk` member ({{cnf-comparison}}); otherwise, `invalid_grant`.
3. For an `x5t#S256` member:
   * the request was received on a TLS connection in whose handshake the client presented a certificate and proved possession of its private key; otherwise, `invalid_grant`; and
   * the SHA-256 thumbprint of the DER encoding of that certificate equals the `x5t#S256` value ({{Section 3.1 of RFC8705}}); otherwise, `invalid_grant`.

Item 1 overrides the rule in {{Section 3.1 of RFC7800}} that implementations ignore confirmation members they do not understand. Client authentication does not take the place of any of these checks.

A resource authorization server that supports DPoP rejects an invalid DPoP proof ({{Section 5 of RFC9449}}) whatever the binding of the grant. A request that presents a DPoP-bound grant without a DPoP proof is rejected with the `invalid_grant` error code (item 2), not `use_dpop_nonce`.

For an `x5t#S256` member, the resource authorization server need not authenticate the client with the certificate or trust the certificate's issuer ({{Section 4 of RFC8705}}).

If the profile of the authorization grant requires binding and the grant has no `cnf` claim, the resource authorization server MUST reject the request with the `invalid_grant` error code.

A resource authorization server that detects reuse of a JWT authorization grant, bound or not, for example by its `jti` claim ({{Section 4.1.7 of RFC7519}}), MUST NOT record the grant as used when it rejects the request ({{single-use}}).

The following `cnf` claim is rejected with the `invalid_grant` error code because it contains two confirmation members:

~~~ json
{
  "cnf": {
    "jkt": "0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I",
    "x5t#S256": "bwcK0esc3ACC3DB2Y5_lESsXE8o9ltc05O89jdN-dg0"
  }
}
~~~
{: title="Confirmation Claim with Two Confirmation Members"}

The following is an example of the error response when the request in {{ex-redeem}} carries no DPoP proof:

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

The following is an example of the response to the request in {{ex-redeem}} from a resource authorization server that requires a DPoP nonce:

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
{: title="Nonce Challenge for a DPoP-Bound Authorization Grant"}

The resource authorization server does not record the authorization grant as used. The client then repeats the request with the same grant and a DPoP proof that contains the nonce.

## Access Token Response {#token-response}

{{Section 5 of RFC9449}} and {{Section 3 of RFC8705}} determine the binding of the tokens that the resource authorization server issues, except as this section and {{refresh}} restrict it for a key-bound authorization grant.

When the authorization grant is bound, the resource authorization server SHOULD bind the access token it issues, and indicate the binding as defined in {{Section 5 of RFC9449}} or {{Section 3 of RFC8705}}. For a key-bound grant, it MUST NOT bind the access token, or a refresh token that it issues, to a key or certificate other than the following:

* for a DPoP-bound authorization grant, the grant's key, which is the key of the DPoP proof; or
* for a certificate-bound authorization grant, the grant's certificate or, if the resource authorization server supports DPoP and the request contains a valid DPoP proof, the public key of that proof.

A TLS client certificate presented with a DPoP-bound authorization grant therefore does not determine the binding, even when the client authenticates with it ({{proof-replay}}). {{binding-outcomes}} summarizes these rules. It lists the key or certificate to which an issued token may be bound, not whether the token is bound: the access token can be unbound, and {{refresh}} determines whether a refresh token is bound.

| Authorization grant's confirmation member | Proof in the request | Permitted key or certificate when the token is bound |
|---|---|---|
| `jkt` or `jwk` | DPoP proof, with or without a TLS client certificate | Authorization grant's key |
| `x5t#S256` | TLS client certificate only | Authorization grant's certificate |
| `x5t#S256` | TLS client certificate and DPoP proof | Authorization grant's certificate or key of the DPoP proof; for a refresh token issued to a public client by a resource authorization server that supports DPoP, key of the DPoP proof |
{: #binding-outcomes title="Permitted Binding of Tokens Issued for a Key-Bound Authorization Grant"}

For a public client, {{Section 5 of RFC9449}} narrows the last row of {{binding-outcomes}}: a resource authorization server that supports DPoP binds a refresh token issued to a public client that presents a valid DPoP proof to the key of that proof, even for a certificate-bound authorization grant. The statement in that section that a refresh token issued to a confidential client is not bound to the DPoP key does not apply ({{refresh}}).

The following is an example of the access token response to the request in {{ex-redeem}}:

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

## Refresh Tokens {#refresh}

If the resource authorization server issues a refresh token for a key-bound authorization grant, it MUST bind the refresh token as described in {{token-response}}, whether or not the client authenticates, with one exception: if the client authenticates and the resource authorization server verifies that the grant identifies the authenticated client, in a claim that the profile of the grant defines, it MAY instead bind the refresh token only to that client.

The resource authorization server decides on this exception when it redeems the grant, never at a later refresh. The replacement for a refresh token bound only to the client is bound only to that client; the replacement for a refresh token bound to a key or certificate is bound as {{refresh-processing}} describes. A refresh token's association with the authenticated client ({{Section 6 of RFC6749}}) never qualifies it for the exception.

A binding to a key or certificate is in addition to the binding to the client that {{Section 10.4 of RFC6749}} requires. It is stricter than {{Section 5 of RFC9449}}, under which a refresh token issued to a confidential client is not bound to the DPoP key, and than the recommendation in {{Section 4 of RFC8705}}; {{proof-replay}} gives the reason.

### Processing a Refresh Request {#refresh-processing}

When a client presents a refresh token that the resource authorization server bound to a key or certificate under {{refresh}}, the resource authorization server treats that key or certificate as the authorization grant's. It MUST verify the binding as items 2 and 3 of {{processing}} do, with the same error codes, and MUST bind the tokens that it issues in response as {{token-response}} and {{refresh}} require. The binding of the presented refresh token does not change, even when the issued tokens are bound to the key of a DPoP proof ({{binding-outcomes}}).

### Proofs in a Refresh Request {#refresh-proofs}

The client presents a refresh token issued for a key-bound authorization grant with each proof of possession that it presented when it obtained the refresh token: a DPoP proof for the same key, a mutual-TLS connection that uses the same certificate, or both. It does so because the response does not always show which binding was applied; for example, a resource authorization server can bind to the certificate although the request contained a DPoP proof, and return a `token_type` of `Bearer`.

A client that knows the binding of the refresh token, because the profile of the grant or an arrangement with the resource authorization server establishes it, need not present a proof of possession that the binding does not require, and needs none for a refresh token bound only to the client. It still presents a DPoP proof where its registration requires one ({{Section 5.2 of RFC9449}}) or where it needs DPoP-bound tokens, and it still needs a certificate that it uses for client authentication.

The following is an example of a refresh request. The refresh token is bound to the certificate in {{cert-example}}, and the client obtained it in a request that carried both that certificate and a DPoP proof. The client therefore sends the request over a mutual-TLS connection that uses the certificate, with which it also authenticates, and adds a DPoP proof signed with the key that it used in that earlier request:

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

The resource authorization server verifies the certificate against the refresh token's binding and, in this example, binds the issued tokens to the key of the DPoP proof:

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

The client presents the replacement refresh token with both proofs, because it obtained it with both; the resource authorization server verifies the DPoP proof against the replacement's binding, which is the key of the DPoP proof in the refresh request above.

## Key-Bound Authorization Grant in Place of Client Authentication {#client-auth}

Client authentication is optional for a JWT authorization grant ({{Section 4.1 of RFC7521}}, {{Section 3.1 of RFC7523}}). A profile can allow a resource authorization server to accept a key-bound grant from a client that does not authenticate, with proof of possession of the grant's key or certificate taking the place of client authentication.

Such a profile answers item 4 of {{profiles}}. The resource authorization server then relies on the issuing authorization server's authentication of the client ({{pop-identity}}).

Where a profile allows a key-bound authorization grant in place of client authentication, a resource authorization server MUST reject, with the `invalid_client` error code ({{Section 5.2 of RFC6749}}), a request that has no client authentication and whose grant has no `cnf` claim but passes every check of {{processing}} and of the request's grant type. A grant that fails one of those checks gets the error code that the check names ({{errors}}), as {{Section 3.1 of RFC7523}} requires `invalid_grant` for a JWT that is not valid.

# Error Responses {#errors}

An authorization server returns errors as defined in {{Section 5.2 of RFC6749}}, with the HTTP status code 400 (Bad Request) except where that section permits 401 (Unauthorized) for `invalid_client`. This specification defines no new error codes. It uses the following error codes, each at the check that names it:

`invalid_dpop_proof`:
: At either authorization server, the DPoP proof is invalid ({{Section 5 of RFC9449}}); or, at the issuing authorization server, binding is required, the request contains no DPoP proof, and {{required-binding}} gives this error code.

`use_dpop_nonce`:
: At either authorization server, the request contains a DPoP proof that {{Section 8 of RFC9449}} rejects with this error code. The authorization server MUST use this error code, and not `invalid_grant` or `invalid_request`, for such a proof, including when the DPoP proof is needed to verify a key-bound authorization grant or a bound `subject_token`.

`invalid_request`:
: At the issuing authorization server, a bound `subject_token` or `actor_token` is not proven and the requesting client is not exempt (item 2 of {{issuer-processing}}); or binding is required, the request contains neither a DPoP proof nor a TLS client certificate, and {{required-binding}} gives this error code.

`invalid_target`:
: At the issuing authorization server, binding is required and the issuing authorization server does not bind the authorization grant, in a case for which neither `invalid_dpop_proof` nor `invalid_request` applies ({{required-binding}}).

`invalid_grant`:
: At the resource authorization server, a check in {{processing}}, {{jwt-pop}}, {{legacy-jwt-dpop}}, or {{refresh}} that names `invalid_grant` fails, or the authorization grant lacks the `typ` value that its profile defines ({{cross-jwt}}).

`invalid_client`:
: At the resource authorization server, where a profile allows a key-bound authorization grant in place of client authentication, the request has no client authentication and its authorization grant has no `cnf` claim but passes every check of {{processing}} and of the request's grant type ({{client-auth}}).

`unauthorized_client`:
: At the resource authorization server, the client is not registered for the grant type of the request ({{client-metadata}}).

When more than one of these error codes, or the `invalid_client` error code for client authentication that the request includes and that fails ({{Section 5.2 of RFC6749}}), applies to a request, the authorization server MUST return the first in the following order:

1. `invalid_client`, if the request includes client authentication and that authentication fails;
2. `unauthorized_client`, if the client authenticated;
3. `invalid_dpop_proof` or `use_dpop_nonce`;
4. at the issuing authorization server, `invalid_request`, then `invalid_target`; and
5. at the resource authorization server, `invalid_grant`, then `unauthorized_client` for a client identified from the authorization grant ({{client-auth}}), then `invalid_client` for a request without client authentication.

# Metadata {#metadata}

This section defines how a resource authorization server advertises the confirmation methods that it verifies, and how client registration can limit a client to key-bound authorization grants.

## Authorization Server Metadata {#as-metadata}

This specification defines the following authorization server metadata parameter ({{Section 2 of RFC8414}}):

`jwt_grant_cnf_methods_supported`:
: RECOMMENDED for a resource authorization server that supports this specification; REQUIRED if its `grant_types_supported` parameter contains the `jwt-pop` grant type. A nonempty JSON array of Confirmation Method Values from the "JWT Confirmation Methods" registry ({{Section 6.2 of RFC7800}}) that lists each confirmation method, among `jkt`, `jwk`, and `x5t#S256`, whose binding the authorization server verifies in a JWT authorization grant ({{processing}}), whatever the grant type of the request or the profile of the authorization grant. Listing one confirmation method does not indicate support for another; in particular, `jkt` does not indicate `jwk`.

A resource authorization server that supports the `jwt-pop` grant type SHOULD include it in the `grant_types_supported` metadata parameter. The `jwt-pop` value alone indicates no confirmation method.

A client or an issuing authorization server can use `jwt_grant_cnf_methods_supported` to learn which confirmation methods a resource authorization server verifies, for example to choose a confirmation method, to confirm that a binding will be enforced ({{non-enforcing-consumers}}), or to decline a method that the server does not verify ({{issuer-processing}}). It retrieves and validates the metadata as described in {{Section 3 of RFC8414}}:

* A client that chooses a confirmation method retrieves the metadata of the authorization server to which it presents the authorization grant.
* A client that confirms that a binding will be enforced, and an issuing authorization server, retrieve the metadata for each `aud` value of the grant, using as the issuer identifier either:
   * that `aud` value; or
   * an issuer identifier that its configuration associates with that `aud` value, for example when the `aud` value is a token endpoint URL ({{Section 4 of RFC10070}}), provided that the `token_endpoint` value of the metadata equals the `aud` value. That equality alone does not establish the association, because only the `issuer` value of metadata is validated ({{Section 3.3 of RFC8414}}, {{Section 1 of RFC10070}}).

{{metadata-evidence}} shows what metadata is evidence of for one confirmation method. Validated metadata is metadata retrieved and validated as described above.

| Metadata | Evidence |
|---|---|
| Validated metadata whose `jwt_grant_cnf_methods_supported` parameter lists the method | That the resource authorization server verifies the method |
| Validated metadata whose `grant_types_supported` parameter contains `jwt-dpop`, when the method is `jkt` ({{legacy-jwt-dpop}}) | That the resource authorization server verifies the method, even if `jwt_grant_cnf_methods_supported` omits it |
| Validated metadata whose `jwt_grant_cnf_methods_supported` parameter omits the method, except as in the preceding row | That the resource authorization server does not verify the method |
| Validated metadata without `jwt_grant_cnf_methods_supported`, except as in the second row, or metadata that cannot be retrieved or validated | None |
{: #metadata-evidence title="Metadata as Evidence of a Confirmation Method"}

Where metadata is not evidence that the resource authorization server verifies a confirmation method (the last two rows), configuration or an established trust relationship can still provide that evidence.

The `dpop_signing_alg_values_supported` ({{Section 5.1 of RFC9449}}) and `tls_client_certificate_bound_access_tokens` ({{Section 3.3 of RFC8705}}) metadata parameters do not indicate that an authorization server verifies key-bound authorization grants.

The following is an example of the metadata of a resource authorization server at `https://as.example/` that verifies the `jkt` and `x5t#S256` confirmation methods:

~~~ json
{
  "issuer": "https://as.example/",
  "token_endpoint": "https://as.example/token",
  "grant_types_supported": [
    "authorization_code",
    "refresh_token",
    "urn:ietf:params:oauth:grant-type:jwt-bearer",
    "urn:ietf:params:oauth:grant-type:jwt-pop"
  ],
  "jwt_grant_cnf_methods_supported": ["jkt", "x5t#S256"],
  "dpop_signing_alg_values_supported": ["ES256"]
}
~~~
{: title="Authorization Server Metadata with the jwt-pop Grant Type"}

## Client Registration Metadata {#client-metadata}

The `jwt-pop` grant type requires a key-bound authorization grant in each request, whether or not the resource authorization server identifies the client ({{jwt-pop}}). A resource authorization server can also limit an identified client to key-bound grants by registering it with a `grant_types` client metadata value ({{Section 2 of RFC7591}}) that includes `jwt-pop` and not `jwt-bearer`. The server then rejects a `jwt-bearer` request from that client with the `unauthorized_client` error code ({{Section 5.2 of RFC6749}}), and {{jwt-pop}} and {{legacy-jwt-dpop}} reject an unbound grant.

This limit applies only when the resource authorization server identifies the client: by client authentication or, when a key-bound grant takes the place of client authentication ({{client-auth}}), from the grant as the profile defines. A request that does not identify a client is not limited by any client's registration.

A resource authorization server can instead reject unbound authorization grants by policy while accepting the `jwt-bearer` grant type, but `grant_types` then does not express that the client is limited to key-bound grants.

Support for both the `jwt-pop` and `jwt-dpop` grant types does not permit every client to use both: the resource authorization server rejects a request whose grant type the client's registration does not permit with the `unauthorized_client` error code. A new client is registered for the `jwt-pop` grant type; an existing client keeps the `jwt-dpop` grant type only where its registration permits it.

The `jwt-pop` grant type has no corresponding `response_types` value ({{Section 2.1 of RFC7591}}).

The `dpop_bound_access_tokens` client metadata parameter ({{Section 5.2 of RFC9449}}) requires a DPoP proof in each token request, but does not require the authorization grant to be bound.

The following is an example of client metadata for a client that the resource authorization server limits to key-bound authorization grants:

~~~ json
{
  "grant_types": [
    "urn:ietf:params:oauth:grant-type:jwt-pop",
    "refresh_token"
  ],
  "token_endpoint_auth_method": "client_secret_basic"
}
~~~
{: title="Client Metadata Limiting a Client to Key-Bound Authorization Grants"}

# Profiling This Specification {#profiles}

A specification that defines a JWT authorization grant, such as {{ID-JAG}}, uses this specification by stating:

1. The validation rules that distinguish the authorization grant from other kinds of JWTs that its issuer issues ({{Section 3.12 of RFC8725}}) or that a resource authorization server accepts, including the value of the grant's `typ` JOSE Header Parameter ({{Section 3.11 of RFC8725}}) and whether those rules rely on it, and, if the grant is issued through Token Exchange, its `requested_token_type` value.
2. Which confirmation methods an issuing authorization server may apply, which a resource authorization server must verify, which are mandatory to implement, and whether the `jwt-pop` grant type is mandatory to implement.
3. Whether binding is optional, recommended, or required, and whether that varies by resource authorization server. If the profile requires the binding to be enforced, and not only applied, who establishes that each resource authorization server in the authorization grant's audience verifies it, and from what evidence ({{non-enforcing-consumers}}).
4. The claim, if any, in which the issuing authorization server identifies the client ({{refresh}}), and whether a resource authorization server may accept a key-bound authorization grant in place of client authentication ({{client-auth}}). If it may: how the identity of the client is tied to the key or certificate; what the issuing authorization server verifies before it binds the grant; and how the resource authorization server maps that identity to its own client registration and policy.
5. Any error codes or processing beyond those in {{errors}} and {{processing}}, such as an error response for a request that carries no proof of possession when the requested resource requires a sender-constrained access token.

A profile that permits a key-bound authorization grant in place of client authentication MUST require the issuing authorization server to authenticate the client in the same request in which it verifies proof of possession of the key or certificate.

When a profile does not address an item above, the following defaults apply:

* Binding is optional, and either confirmation method may be used. An issuing authorization server can still require binding ({{required-binding}}); that does not make binding required at the resource authorization server.
* A resource authorization server verifies any `cnf` claim it receives.
* A key-bound authorization grant does not take the place of client authentication. Whether the client authenticates is the resource authorization server's policy ({{Section 3.1 of RFC7523}}), and {{client-auth}} does not apply.

# Security Considerations {#security}

The security considerations of {{RFC7521}}, {{RFC7523}}, {{RFC8693}}, {{RFC9449}}, and {{RFC8705}} apply to this specification.

## Non-Enforcing Resource Authorization Servers {#non-enforcing-consumers}

{{RFC7523}} does not process the `cnf` claim, and JWT implementations ignore claims and confirmation members that they do not understand ({{Section 4 of RFC7519}}, {{Section 3.1 of RFC7800}}). A resource authorization server that supports neither this specification nor another specification that verifies the binding, such as {{JWT-DPOP-GRANT}} or {{ID-JAG}}, therefore accepts a key-bound authorization grant as a bearer grant: a party that obtains the grant can redeem it without the key.

Only a resource authorization server that verifies the binding enforces it, and the issuing authorization server binds a grant without checking that the resource authorization server verifies the binding ({{bind}}). When binding is optional, the alternative is an unbound grant, and against a stolen grant, binding is never weaker: a key-bound grant is a bearer grant where its binding is not verified, and is protected wherever it is, including at a server whose support the issuing authorization server did not know of or whose configuration changed after the issuing authorization server last checked. Binding has two costs:

* A key-bound grant can be rejected where an unbound one is accepted, by a resource authorization server that does not support its confirmation method, or by one that implements only {{JWT-DPOP-GRANT}} and receives a DPoP-bound grant with the `jwt-bearer` grant type. For the latter, {{token-request}} directs the client to the `jwt-dpop` grant type where that server advertises it and the client's registration permits it; otherwise, the request fails. An issuing authorization server that has determined that a resource authorization server does not verify a confirmation method can decline that method for it ({{issuer-processing}}).
* The confirmation member reaches servers that do not verify it ({{privacy}}).

A requirement for binding ensures that the authorization grant is bound, not that the binding is enforced: the issuing authorization server can still issue a required key-bound grant for an audience that accepts it as a bearer grant. A profile or an issuing authorization server that needs a required binding to be enforced establishes that each authorization server in the grant's audience verifies it, for example through the profile (items 2 and 3 of {{profiles}}), through configuration that restricts the audience to such servers, or through the issuing authorization server's policy.

A key-bound authorization grant is only as protected as the least protective server in its audience: a party that obtains a grant whose `aud` claim identifies several authorization servers can present it to any of them. A client that needs the binding to be enforced therefore needs evidence that each of those servers verifies it, from that server's metadata ({{as-metadata}}), from configuration, or from the profile. A client that cannot read the grant's `aud` claim, for example because the grant is encrypted, relies on configuration or the profile to establish the grant's audience.

## Grant Type Substitution {#grant-type-substitution}

A separate grant type for key-bound authorization grants does not by itself prevent substitution: a party holding a stolen key-bound grant can present it with the `jwt-bearer` grant type. A resource authorization server that supports this specification therefore verifies the binding of every JWT authorization grant that contains a `cnf` claim, whichever grant type the request uses ({{processing}}).

The `jwt-pop` grant type adds no protection against a stolen authorization grant. What it adds is:

* a requirement for a key-bound grant in each request; a resource authorization server that does not support the grant type rejects the request with the `unsupported_grant_type` error code ({{Section 5.2 of RFC6749}}), whereas a server ignores client metadata that it does not understand ({{Section 2 of RFC7591}}); and
* a way to limit an identified client to key-bound grants through its registration ({{client-metadata}}).

The `jwt-pop` grant type indicates no confirmation method; the methods that a server verifies are advertised separately ({{as-metadata}}). Verifying a key-bound grant presented with the `jwt-bearer` grant type, instead of rejecting it as {{JWT-DPOP-GRANT}} does, lets clients registered only for that grant type keep working.

A resource authorization server that adopts this specification rejects an authorization grant that it previously accepted if the grant's `cnf` claim contains more than one confirmation member, a member other than `jkt`, `jwk`, or `x5t#S256`, or a member whose confirmation method the server does not support.

## Cross-JWT Confusion {#cross-jwt}

A key-bound authorization grant and a DPoP-bound JWT access token for the same client can carry the same `jkt` value. A party that receives one could present it where the other is expected. The `aud` claim of a grant identifies an authorization server ({{RFC10070}}), and an explicit `typ` value ({{Section 3.11 of RFC8725}}) lets a recipient reject a JWT of the wrong kind ({{Section 3.12 of RFC8725}}). When the profile of a grant defines a `typ` value, a resource authorization server SHOULD reject a grant that does not carry it with the `invalid_grant` error code, and MUST do so when the profile states that its validation rules rely on that value to distinguish its grants from other kinds of JWTs (item 1 of {{profiles}}).

## DPoP Proof Replay {#proof-replay}

The DPoP replay protections ({{Section 11.1 of RFC9449}}) and server-provided nonces ({{Section 8 of RFC9449}}) apply at the token endpoints of both the issuing and the resource authorization server. A DPoP proof sent to one cannot be replayed at the other when their token endpoint URIs do not match under the comparison rules of {{Section 4.3 of RFC9449}}, because the `htu` claim identifies the endpoint. Those rules ignore query and fragment parts, so token endpoint URIs that differ only in their query components ({{Section 3.2 of RFC6749}}) do not provide this separation.

A DPoP proof does not cover the authorization grant it accompanies or the TLS connection that carries it. An attacker that captures a grant together with a DPoP proof for its key can replay both, over its own TLS connection, within the window in which the resource authorization server accepts the proof. What the attacker obtains depends on the binding of the issued tokens:

* An access token or refresh token bound to the authorization grant's key is of no use to the attacker.
* An access token bound to a TLS client certificate that the attacker presents would be usable; {{token-response}} forbids that binding. The reverse case is safe: for a certificate-bound authorization grant, the mutual-TLS handshake shows that a DPoP proof comes from the holder of the grant's certificate, so {{token-response}} lets the issued tokens be bound to the key of that proof.
* A refresh token bound only to a client as which the attacker authenticates would be usable, at the first refresh or a later one. {{refresh}} therefore binds every refresh token issued for a key-bound authorization grant, and every replacement, to a key or certificate, unless the resource authorization server verifies that the grant identifies the authenticated client; an attacker that authenticates as its own client cannot meet that condition.
* An unbound access token is usable.

A resource authorization server that issues unbound access tokens for DPoP-bound authorization grants SHOULD reject a DPoP proof whose `jti` value it has already accepted at its token endpoint within that window ({{Section 11.1 of RFC9449}}). A server-provided nonce ({{Section 8 of RFC9449}}) shortens the window but does not by itself detect reuse, because a client can use the same nonce more than once. Neither measure denies an unbound access token to an attacker whose request arrives before the client's; only binding the access token does.

A captured request can also be replayed at the issuing authorization server. An attacker that captures a Token Exchange request that proves possession of a DPoP-bound `subject_token` or `actor_token` can replay the token and the DPoP proof over its own mutual-TLS connection. It has to do so within the window in which the issuing authorization server accepts the proof and, if that server rejects a reused `jti` value, before the client's request. It also has to satisfy any client authentication that the issuing authorization server requires: it can replay a client secret, or a client assertion that the server accepts more than once, or authenticate as its own client if the server lets that client exchange the token ({{pop-identity}}). What the attacker obtains depends on the binding of the issued authorization grant:

* A grant bound to the key of the DPoP proof is of no use to the attacker at a resource authorization server that verifies the binding, although it is usable at one that does not ({{non-enforcing-consumers}}).
* A grant bound to the attacker's TLS client certificate would be usable at every resource authorization server in its audience. This is one reason {{binding-request}} binds the grant only to the key of the DPoP proof.
* An unbound grant, which {{binding-request}} issues when `jkt` is not a permitted confirmation method and binding is not required, is usable ({{subject-token-theft}}).

When neither the `subject_token` nor the `actor_token` requires proof of possession of a DPoP key, binding the grant only to the key of the DPoP proof adds no protection against an attacker that can meet the other requirements of the exchange, because that attacker can request a grant bound to its own key.

## Exhaustion of Single-Use Authorization Grants {#single-use}

Binding limits who can redeem an authorization grant, not how often, so short grant lifetimes and single-use enforcement ({{Section 5.5 of ID-CHAINING}}) remain useful for key-bound grants. A resource authorization server that limits a grant to a single use could, however, mark a key-bound grant as used when a party without the key presents it, denying the legitimate client its redemption. It could also mark the grant as used when it rejects a request with the `use_dpop_nonce` error code, so that the client's retry with the nonce fails. {{processing}} prevents both by recording a grant as used only when the request succeeds. {{issuer-processing}} applies the same rule to a single-use `subject_token` at the issuing authorization server.

## Proof of Possession and Client Identity {#pop-identity}

Proof of possession shows that the presenter holds the key or certificate, not which client it is: even at the issuing authorization server, a DPoP proof is not cryptographically bound to the accompanying client authentication ({{Section 11.11 of RFC9449}}). A resource authorization server that accepts a key-bound authorization grant in place of client authentication ({{client-auth}}) therefore depends on the issuing authorization server's authentication of the client and on the client's protection of its private key.

## Theft of the Subject Token {#subject-token-theft}

Binding the issued authorization grant does not protect a bearer `subject_token` that is stolen before the exchange. An attacker that holds such a subject token, and can satisfy any client authentication that the issuing authorization server requires, can request a grant bound to the attacker's own key. Protection of bearer subject tokens is outside the scope of this specification; see {{RFC9700}}.

A bound `subject_token` is protected by item 2 of {{issuer-processing}}, except from an authenticated intended audience that {{bound-subject}} exempts. The exemption covers every authenticated client, other than the one to which the token was issued, that the issuing authorization server associates with the token's `aud` claim or its records of the token. When that claim identifies a resource that several resource servers serve, any of them that obtains the token, for example from another of them, can exchange it without the key. An issuing authorization server that associates several clients with one audience value therefore weakens the binding of tokens for that audience.

When `jkt` is not a permitted confirmation method for the authorization grant and binding is not required, a client that proves possession of a DPoP-bound `subject_token` receives an unbound grant ({{binding-request}}), which is a bearer token, as it would be without this specification.

## Informational Nature of the Response Parameter

The `cnf` response parameter is informational. The binding is enforced when the issued token is presented to its consumer, which verifies the token's `cnf` claim against the presenter's proof. A verified `cnf` parameter tells the client only that the authorization server applied the binding; it does not tell the client that the consumer enforces it ({{non-enforcing-consumers}}). The client's check of the parameter ({{client-processing}}) protects against an authorization server that fails to apply a binding through error or misconfiguration, not against a malicious one, which controls the contents of the token.

The token endpoint requires TLS ({{Section 3.2 of RFC6749}}), which prevents an on-path attacker from removing or modifying the `cnf` parameter.

# Privacy Considerations {#privacy}

The privacy considerations of {{RFC9449}} and {{RFC8705}} apply to this specification.

A confirmation member is a stable identifier for the bound key or certificate. It reaches the resource authorization server whether or not that server verifies the binding ({{bind}}). A client that uses the same key or certificate with several resource authorization servers lets them correlate its requests. A client that needs to avoid such correlation uses a separate key or certificate for each resource authorization server. This is not possible when the client exchanges a DPoP-bound `subject_token`, because a key-bound authorization grant is then bound to that token's key ({{te-request}}).

The `cnf` response parameter discloses the binding only to the client that made the request, which already holds the key or certificate. A thumbprint member discloses less than a `jwk` member and is sufficient for the client to verify the binding, which is why {{cnf-param}} recommends it.

# IANA Considerations {#iana}

## OAuth Parameters Registration

{{RFC9201}} registers the `cnf` parameter in the "OAuth Parameters" registry {{IANA.oauth-parameters}} with the usage location "token response". This specification requests that IANA add this document as a reference for that entry:

Parameter name:
: `cnf`

Parameter usage location:
: token response

Change controller:
: IETF

Specification document(s):
: {{Section 5 of RFC9201}} and {{cnf-param}} of this document

## OAuth URI Registration

This specification requests registration of the following values in the "OAuth URI" registry {{IANA.oauth-parameters}} established by {{RFC6755}}:

URN:
: `urn:ietf:params:oauth:grant-type:jwt-pop`

Common Name:
: Proof-of-Possession JWT Authorization Grant

Change Controller:
: IETF

Specification Document(s):
: {{jwt-pop}} of this document

URN:
: `urn:ietf:params:oauth:grant-type:jwt-dpop`

Common Name:
: DPoP-Bound JWT Authorization Grant (legacy)

Change Controller:
: IETF

Specification Document(s):
: {{legacy-jwt-dpop}} of this document

## OAuth Authorization Server Metadata Registration

This specification requests registration of the following value in the "OAuth Authorization Server Metadata" registry {{IANA.oauth-parameters}} established by {{RFC8414}}:

Metadata Name:
: `jwt_grant_cnf_methods_supported`

Metadata Description:
: JSON array of the Confirmation Method Values whose binding the authorization server verifies in JWT authorization grants

Change Controller:
: IETF

Specification Document(s):
: {{as-metadata}} of this document

--- back

# Profiles of This Specification {#profile-examples}

This appendix is informative. It shows how {{ID-JAG}} would answer the items in {{profiles}} and how the defaults of {{profiles}} apply to {{ID-CHAINING}}, and lists the conflicts between each document and this specification.

## Identity Assertion JWT Authorization Grant

{{ID-JAG}} would answer the items as follows:

1. The `typ` value is `oauth-id-jag+jwt`, and the validation rules rely on it ({{Section 4.4.1 of ID-JAG}}); the `requested_token_type` value is `urn:ietf:params:oauth:token-type:id-jag`.
2. Both confirmation methods, `jkt` and `x5t#S256`, may be used. Neither method, nor the `jwt-pop` grant type, is mandatory to implement, and a Resource Authorization Server verifies each method that it lists in `jwt_grant_cnf_methods_supported` ({{as-metadata}}). A DPoP-bound authorization grant is presented with the `jwt-dpop` grant type ({{Section 4.4 of ID-JAG}}), which this specification keeps for existing clients ({{legacy-jwt-dpop}}).
3. Binding is optional. When the DPoP proof is valid, the IdP Authorization Server binds the authorization grant: to the DPoP key, unless its local policy selects a certificate binding instead ({{Section 9.11.1.1 of ID-JAG}}). Its policy does not otherwise decline the `jkt` confirmation method ({{issuer-processing}}). The first conflict below concerns that certificate choice.
4. A Resource Authorization Server may accept a key-bound authorization grant in place of client authentication. The IdP Authorization Server authenticates the client in the Token Exchange request, binds the grant to the key or certificate for which that request carries proof of possession, and identifies the client in the grant's `client_id` claim.
5. A Resource Authorization Server also validates a certificate presented for an `x5t#S256`-bound authorization grant as described in {{Section 2 of RFC8705}} ({{Section 9.11.2.2 of ID-JAG}}). It rejects an unbound grant with the `invalid_grant` error code when the resource requires sender-constrained access tokens and the request carries neither a DPoP proof nor a TLS client certificate ({{Section 9.11.1.2.4 of ID-JAG}}, {{Section 9.11.2.2 of ID-JAG}}).

A client that detects a downgrade from the authorization grant's `cnf` claim ({{Section 9.11.1.1 of ID-JAG}}, {{Section 9.11.2.1 of ID-JAG}}) follows {{client-processing}}, which also accepts the `cnf` response parameter.

Version -06 of that document conflicts with this specification in two places. Each would be resolved by a change to that document:

* {{Section 9.11 of ID-JAG}} lets the IdP Authorization Server select either a DPoP or a certificate binding, according to local policy, when a request carries both a DPoP proof and a TLS client certificate. This specification binds an authorization grant issued in response to a DPoP binding request only to the key of the DPoP proof, if it binds it at all ({{binding-request}}), for two reasons. The client can authenticate to the IdP Authorization Server with a certificate that it does not use downstream and still obtain a grant that it can redeem with its DPoP key. And an attacker that replays a captured DPoP-bound `subject_token` or `actor_token` and its DPoP proof over its own mutual-TLS connection cannot obtain a grant bound to the attacker's certificate ({{proof-replay}}). The change is for the IdP Authorization Server never to choose a certificate binding when the request contains a DPoP proof. A narrower change is listed among the open issues ({{open-issues}}).
* {{Section 8 of ID-JAG}} requires a client that supports that profile to include `urn:ietf:params:oauth:grant-type:jwt-bearer` in its `grant_types`, which prevents a Resource Authorization Server from limiting the client to key-bound authorization grants ({{client-metadata}}), and {{Section 4.4 of ID-JAG}} presents a DPoP-bound grant with the `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type. The change is to present a key-bound grant of either method with the `urn:ietf:params:oauth:grant-type:jwt-pop` grant type, and to require `urn:ietf:params:oauth:grant-type:jwt-bearer` or `urn:ietf:params:oauth:grant-type:jwt-pop`.

## Identity and Authorization Chaining

{{ID-CHAINING}} defines no profile of this specification, so the defaults in {{profiles}} apply. Binding the authorization grant adds a mitigation to those listed in {{Section 5.5 of ID-CHAINING}}. The access token in {{Appendix B.3 of ID-CHAINING}} is bound to a key that the grant conveys in a `requested_cnf` claim. For a key-bound grant, {{token-response}} permits only the bindings in {{binding-outcomes}}, so a grant that carries both a `cnf` claim and a `requested_cnf` claim for a different key conflicts with this specification.

# Open Issues {#open-issues}

\[Editor's note: this appendix lists questions for the working group. It will be removed before publication.\]

1. Grant type or client metadata. A `jwt-pop` request has the same format as a `jwt-bearer` request, while {{Section 4.1 of RFC7521}} defines `grant_type` as "the format of the assertion". An alternative is to present every key-bound authorization grant with the `jwt-bearer` grant type and to express "key-bound grants only" with a boolean client metadata parameter, as `dpop_bound_access_tokens` does ({{Section 5.2 of RFC9449}}). That would remove the `jwt-pop` grant type and the conflict with {{Section 8 of ID-JAG}}. It would also lose the per-request requirement and the rejection of an unsupported grant type that {{grant-type-substitution}} describes; a client would instead confirm that the registration policy took effect from the registered metadata that the resource authorization server returns ({{Section 3.2.1 of RFC7591}}). Which should the working group prefer?
2. Error for a required binding. When binding is required and the issuing authorization server does not bind the authorization grant, this draft returns `invalid_dpop_proof` if a DPoP proof would let it bind the grant, `invalid_request` if only a TLS client certificate would, and `invalid_target` otherwise ({{required-binding}}). {{Section 5 of RFC9449}} defines `invalid_dpop_proof` for an invalid DPoP proof; this draft also uses it for a missing one at the issuing authorization server. {{JWT-DPOP-GRANT}} uses it for a missing proof at the authorization server that redeems the grant, where this draft returns `invalid_grant` ({{legacy-jwt-dpop}}). Is that extension acceptable, or should a new error code tell the client that proof of possession is required, for either confirmation method, distinct from the `invalid_request` error code that an unproven bound `subject_token` also receives?
3. Scope of the response parameter. {{RFC9201}} registers the `cnf` token response parameter for ACE. Should this document define its use for Token Exchange only, for every token response, or move it to a separate specification?
4. Client opt-in to certificate binding. Binding an authorization grant to the TLS client certificate that the client presents at issuance is an issuance policy that the client opts into, not only a capability: a client can support certificate-bound grants and still reserve the certificate it presents to the issuing authorization server for that relationship. This draft leaves the opt-in to the issuing authorization server's configuration for the client ({{cert-binding}}). Should a client metadata parameter express it? The `authorization_grant_profiles_supported` parameter of {{ID-JAG}} could carry such a signal, at the cost of mixing grant profiles with proof capabilities in one list. If a client can opt in, should the opt-in also let the issuing authorization server bind the grant to the certificate when the request also contains a DPoP proof, as {{Section 9.11 of ID-JAG}} permits, except when the request proves possession of a DPoP-bound `subject_token` or `actor_token` ({{proof-replay}})? That would narrow the certificate conflict with {{ID-JAG}} in {{profile-examples}}. It would need exceptions to {{binding-request}} and {{te-request}}, which bind the grant only to the key of the DPoP proof, to {{cert-binding}}, which applies only to a request without a DPoP proof, and to the `invalid_target` case of {{required-binding}}, and the client's check in {{client-processing}} would have to accept the certificate of a client that opted in.
5. Alignment with {{ID-JAG}}, {{ID-CHAINING}}, and {{JWT-DPOP-GRANT}}. The conflicts listed in {{profile-examples}}, two with {{ID-JAG}} and one with {{Appendix B.3 of ID-CHAINING}}, need to be resolved, by the proposed changes to those documents or by changes to this one. For the last, should a profile be able to permit binding the access token to a key that the authorization grant conveys in a `requested_cnf` claim? The replacement of the `jwt-dpop` grant type and the compatibility rules in {{legacy-jwt-dpop}} need the agreement of the authors of {{JWT-DPOP-GRANT}}, which also requests registration of the `jwt-dpop` grant type ({{Section 9.1 of JWT-DPOP-GRANT}}).
6. Unbound authorization grant after a DPoP binding request. When a valid DPoP binding request cannot be honored, {{binding-request}} requires an unbound grant unless binding is required by the profile, by the issuing authorization server's policy, or by the client's registration ({{required-binding}}). A client that requires binding discards an unbound grant ({{client-processing}}), but the successful exchange may already have consumed a single-use `subject_token`, for example under an issuer's refresh-token rotation policy. The protection in {{issuer-processing}} applies only when the issuer rejects the request. A client can already obtain a required binding through a profile that requires it, or through an arrangement that its registration at the issuing authorization server records ({{te-request}}). Is a per-request "bind or fail" option needed for a requirement that neither expresses, and how would it preserve compatibility with clients that send DPoP proofs for other reasons?
7. Refresh-token binding for confidential clients. {{refresh}} binds a refresh token issued for a key-bound authorization grant to a key or certificate even when the client authenticates, except when the grant identifies that authenticated client. This departs from {{Section 5 of RFC9449}}, which retains client binding for confidential clients so that authentication credentials can rotate without invalidating refresh tokens. The additional binding requires the client to retain the bound key or certificate for refresh, complicating rotation and recovery. Is the current exception sufficient, or should the binding requirement be a profile choice? How would an alternative prevent replay of a captured grant and DPoP proof by an attacker authenticating as a different client, including at later refreshes ({{proof-replay}})?
8. Strict confirmation-member rules. {{processing}} rejects a `cnf` claim unless it contains exactly one member, chosen from `jkt`, `jwk`, and `x5t#S256`. {{Section 3.1 of RFC7800}} permits other confirmation members and, in the absence of application-specific requirements, ignores members that an implementation does not understand. As {{grant-type-substitution}} notes, this draft therefore rejects some authorization grants that a server previously accepted. Should it retain the closed set and exactly-one-member rule, or permit additional members? If the rule is relaxed, how should unsupported or conflicting bindings be rejected while preserving RFC 7800's single-key constraint?
9. Acceptance of the `jwk` confirmation method. {{cnf-claim}} limits issuance under this specification to `jkt` and `x5t#S256`, but {{processing}} can also accept `jwk`, and {{as-metadata}} advertises that acceptance. Which deployments need interoperability with `jwk` authorization grants issued outside this specification? Is that sufficient reason to retain the additional redemption and comparison rules, or should issuance and redemption support the same set of confirmation methods?
10. Enforcement of a required binding. When binding is required, this draft requires the issuing authorization server to bind the authorization grant, but not to establish that each authorization server in the grant's audience verifies the binding, because such a determination can become stale ({{non-enforcing-consumers}}). The profile, configuration, or the issuing authorization server's policy can establish it instead. Should a requirement for binding also require that determination, and if so, from what evidence?

# Acknowledgments
{:numbered="false"}

This document combines and replaces {{JWT-DPOP-GRANT}} and {{TE-CNF}}, and generalizes the sender-constraining rules of {{ID-JAG}}. The author thanks the authors and reviewers of those documents.

# Document History
{:numbered="false"}

-00

* Initial version, combining {{JWT-DPOP-GRANT}} and {{TE-CNF}}.
* Replaces the `jwt-dpop` grant type with the method-neutral `jwt-pop` grant type and the `jwt_grant_cnf_methods_supported` metadata parameter, keeps `jwt-dpop` for existing clients, and verifies key-bound authorization grants whatever their grant type.
