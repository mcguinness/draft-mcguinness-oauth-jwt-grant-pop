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
  RFC9728:
  ID-CHAINING: I-D.ietf-oauth-identity-chaining
  ID-JAG: I-D.ietf-oauth-identity-assertion-authz-grant
  JWT-DPOP-GRANT: I-D.parecki-oauth-jwt-dpop-grant
  TE-CNF: I-D.mcguinness-oauth-token-exchange-cnf
  WAG: I-D.carleton-workload-authz-grant

--- abstract

This specification defines how an authorization server binds a JSON Web Token (JWT) authorization grant (RFC 7523) to a key or certificate held by the client, using OAuth 2.0 Demonstrating Proof of Possession (DPoP) or OAuth 2.0 Mutual-TLS. It specifies how the authorization server that receives the grant verifies the binding and binds the access and refresh tokens that it issues. It profiles OAuth 2.0 Token Exchange so that a client can request a DPoP-bound grant and learn which binding was applied. It also defines a grant type that requires a key-bound grant and an authorization server metadata parameter that lists the confirmation methods that a server verifies in JWT authorization grants.

--- middle

# Introduction

A client can present a JWT authorization grant to an authorization server to obtain an access token ({{Section 2.1 of RFC7523}}). A bearer grant can also be presented by a party that obtains it and meets the server's other requirements ({{Section 5.5 of ID-CHAINING}}, {{Section 10 of WAG}}). In {{ID-CHAINING}} and {{ID-JAG}}, the client obtains a JWT authorization grant in one trust domain and redeems it in another; in {{WAG}}, a workload platform issues one to a workload.

This specification lets the issuing authorization server bind a grant to the client's DPoP key {{RFC9449}} or TLS client certificate {{RFC8705}} by identifying the key or certificate in the grant's `cnf` claim {{RFC7800}}. The client presents the grant to the resource authorization server and proves possession; that server checks the proof against the `cnf` claim ({{processing}}) and limits the binding of the tokens that it issues ({{token-response}}, {{refresh}}). These rules apply however the client obtained the grant.

When the client obtains the grant through Token Exchange {{RFC8693}}, it can request a DPoP binding and learn which binding was applied, even when the grant is encrypted {{RFC7516}}.

## Protocol Overview {#overview}

In the following figure, a client obtains an authorization grant bound to its DPoP key K and redeems it at the resource authorization server:

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

The client sends a separate DPoP proof, signed with K, to each authorization server ({{te-request}}), and checks the binding that the issuing authorization server returns ({{client-processing}}). With mutual TLS, the client's TLS client certificate takes the place of K: the client proves possession in the TLS handshake instead of with a DPoP proof.

This specification involves three separate bindings: that of a `subject_token` or `actor_token` presented to the issuing authorization server ({{bound-subject}}); that of the authorization grant ({{processing}}); and that of the access and refresh tokens that the resource authorization server issues ({{token-response}}, {{refresh}}). Proof of possession shows that the presenter holds a key or certificate, not which client it is ({{pop-identity}}).

## Relationship to Other Documents {#relationship}

* {{JWT-DPOP-GRANT}} defines the `jwt-dpop` grant type. This specification proposes to replace that document, subject to its author's agreement ({{open-issues}}). The `jwt-pop` grant type ({{jwt-pop}}) supports both DPoP and mutual TLS. Existing clients can continue to use `jwt-dpop` as described in {{legacy-jwt-dpop}}, which lists the other differences.
* {{TE-CNF}} specifies the use of the `cnf` response parameter in Token Exchange. This specification incorporates that use ({{cnf-param}}) and replaces {{TE-CNF}}. {{RFC9201}} registers the parameter for the Authentication and Authorization for Constrained Environments (ACE) framework.
* {{ID-JAG}} defines sender-constraining rules for the Identity Assertion JWT Authorization Grant. This specification generalizes those rules. {{profile-examples}} shows how that grant fits the profile requirements and identifies conflicts between the two documents.

## Conventions and Terminology

{::boilerplate bcp14-tagged}

This specification uses the terms "access token", "authorization grant", "authorization server", "client", "confidential client", "public client", "refresh token", "resource server", and "token endpoint" defined by {{RFC6749}}; the terms "JSON Web Token (JWT)" and "claim" defined by {{RFC7519}}; the term "confirmation method" from the "JWT Confirmation Methods" registry ({{Section 6.2 of RFC7800}}); and the term "DPoP proof" defined by {{RFC9449}}. It also defines the following terms:

JWT authorization grant:
: A JWT used as an authorization grant as defined in {{Section 2.1 of RFC7523}}.

Issuing authorization server:
: The party that issues a JWT authorization grant, such as the IdP Authorization Server of {{ID-JAG}} or the Workload Identity Provider of {{WAG}}. It acts as an authorization server when it issues the grant through Token Exchange ({{issuance}}), and can also issue it through another interface ({{other-issuance}}).

Resource authorization server:
: The authorization server to which a client presents a JWT authorization grant in an access token request ({{Section 4.1 of RFC7521}}), typically the authorization server for the resources that the client wants to access, such as the Resource Authorization Server of {{ID-JAG}}.

Confirmation member:
: A member of a `cnf` claim or of the `cnf` response parameter, such as `jkt` or `x5t#S256`.

Key-bound authorization grant:
: A JWT authorization grant that contains a `cnf` claim ({{format}}). A DPoP-bound authorization grant has a `jkt` or `jwk` confirmation member; a certificate-bound authorization grant has an `x5t#S256` confirmation member. A JWT authorization grant without a `cnf` claim is unbound.

DPoP binding request:
: A Token Exchange request that contains a DPoP proof ({{te-request}}).

Profile:
: A specification that defines a particular kind of JWT authorization grant, such as {{ID-JAG}} or {{WAG}}. {{profiles}} lists the choices that a profile can make, with defaults for some of them.

Permitted confirmation method:
: A confirmation method that the issuing authorization server supports and that both the profile of an authorization grant and the issuing authorization server's policy permit for that grant ({{issuer-processing}}).

This specification refers to grant types by the short names in {{grant-type-names}}.

| Short name | Grant type | Defined in |
|---|---|---|
| `jwt-bearer` | `urn:ietf:params:oauth:grant-type:jwt-bearer` | {{Section 2.1 of RFC7523}} |
| `jwt-pop` | `urn:ietf:params:oauth:grant-type:jwt-pop` | {{jwt-pop}} |
| `jwt-dpop` | `urn:ietf:params:oauth:grant-type:jwt-dpop` | {{JWT-DPOP-GRANT}}, {{legacy-jwt-dpop}} |
{: #grant-type-names title="Grant Type Short Names"}

The examples in this specification are non-normative. They use the public key from the examples in {{RFC9449}}, an issuing authorization server at `https://issuer.example/`, and a resource authorization server at `https://as.example/`. Request bodies are shown with extra line breaks and without URL encoding, and tokens are truncated, for display purposes only.

# Key-Bound JWT Authorization Grants {#format}

## Confirmation Claim {#cnf-claim}

An issuing authorization server binds an authorization grant by including in it a `cnf` claim ({{Section 3.1 of RFC7800}}). The confirmation member of the claim identifies the key or certificate whose possession the client proves when it redeems the grant:

* `jkt` ({{Section 6.1 of RFC9449}}), in a DPoP-bound authorization grant, contains the JWK SHA-256 Thumbprint {{RFC7638}} of the client's public key.
* `x5t#S256` ({{Section 3.1 of RFC8705}}), in a certificate-bound authorization grant, contains the SHA-256 thumbprint of the DER encoding of the client's certificate.

The `cnf` claim of an authorization grant that an issuing authorization server binds MUST contain exactly one of these members. Requiring exactly one member is narrower than {{Section 3.1 of RFC7800}}, which requires only that a `cnf` claim "represent only a single proof-of-possession key".

A resource authorization server MAY also accept a `jwk` member ({{Section 3.2 of RFC7800}}) in a DPoP-bound authorization grant.

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

The following is an example of the `cnf` claim of a certificate-bound authorization grant:

~~~ json
{
  "cnf": {
    "x5t#S256": "bwcK0esc3ACC3DB2Y5_lESsXE8o9ltc05O89jdN-dg0"
  }
}
~~~
{: #cert-example title="Confirmation Claim of a Certificate-Bound Authorization Grant"}

## Comparing Confirmation Members {#cnf-comparison}

Two confirmation members, or a confirmation member and the client's own key or certificate, are compared by confirmation method and value, not by byte representation:

* A thumbprint member, such as `jkt` or `x5t#S256`, is compared as the base64url-encoded string it contains, character by character.
* A public key in a `jwk` member is compared with a `jkt` member, or with the client's own public key, by computing JWK SHA-256 Thumbprints {{RFC7638}}.
* Members of different confirmation methods, other than a `jwk` and `jkt` pair, do not match.

The following is an example of a `jwk` member that matches a `jkt` member with the value `0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I`, the JWK SHA-256 Thumbprint of its key:

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
{: title="A jwk Confirmation Member"}

# Issuing a Key-Bound Authorization Grant {#issuance}

This section covers JWT authorization grants issued through Token Exchange ({{Section 2.1 of RFC8693}}). The issuing authorization server decides whether to issue a JWT for use as an authorization grant from its policy, its configuration for `audience` and `resource`, and the requested token type (`requested_token_type`), which a profile can define. {{cnf-param}} applies to any token issued through Token Exchange. For other issuance interfaces, see {{other-issuance}}.

## Token Exchange Request {#te-request}

A client requests a DPoP-bound authorization grant by including a DPoP proof ({{Section 4 of RFC9449}}) in the DPoP HTTP header field of a Token Exchange request, as it would for a DPoP-bound access token. Such a request is a DPoP binding request. The client signs the DPoP proof with the key that it will use to redeem the grant.

The client sends a separate DPoP proof to each authorization server, signed with the same key, because the `htu` claim of a DPoP proof identifies the endpoint that receives it ({{Section 4.2 of RFC9449}}). The key's algorithm therefore needs to be one that both authorization servers accept and, if the access token is bound to the key, one that the resource server also accepts. Each server can list the algorithms that it accepts in its `dpop_signing_alg_values_supported` metadata parameter ({{Section 5.1 of RFC9449}}, {{Section 2 of RFC9728}}).

A TLS client certificate in a request does not ask for a certificate binding: the client might be presenting the certificate only for mutual-TLS client authentication ({{Section 2 of RFC8705}}). An issuing authorization server binds a grant to that certificate only as {{cert-binding}} describes.

If the `subject_token` or `actor_token` is bound to a key or certificate, the client MUST prove possession of that same key or certificate in the request, with a DPoP proof signed with the key or over a mutual-TLS connection that uses the certificate, unless the client is exempt ({{bound-subject}}). Two consequences follow:

* A request carries at most one DPoP proof ({{Section 4.3 of RFC9449}}). A request whose `subject_token` and `actor_token` are bound to different DPoP keys therefore proves possession of at most one of them, and is rejected unless the client is exempt for each token whose key the DPoP proof does not match.
* An authorization grant issued in response to a DPoP binding request is bound to the key of that request's DPoP proof, or unbound ({{binding-request}}). A key-bound grant therefore has the same key as each DPoP-bound `subject_token` or `actor_token` whose possession the request proves.

No request parameter lets a client require a key-bound authorization grant for an individual request. A client that always needs one can request a grant whose profile requires binding, or arrange for its registration at the issuing authorization server to require binding ({{required-binding}}); this specification defines no client metadata for that arrangement. A client that requires a key-bound grant also checks the response ({{client-processing}}).

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

The DPoP proof in that request carries the example public key ({{cnf-comparison}}) in its `jwk` JOSE Header Parameter. Its decoded claims are:

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

An issuing authorization server that supports this specification MUST support the `cnf` response parameter ({{cnf-param}}) and at least one of the `jkt` and `x5t#S256` confirmation methods, and MUST NOT bind an authorization grant other than as described in this section.

For a Token Exchange request for a JWT authorization grant, the issuing authorization server MUST also ensure the following, and MUST otherwise reject the request with the indicated error code ({{errors}}):

1. Any DPoP proof that the request contains is valid as described in {{Section 4.3 of RFC9449}}; otherwise, `invalid_dpop_proof`, or `use_dpop_nonce` as described in {{Section 8 of RFC9449}}.
2. For each `subject_token` or `actor_token` that is bound to a key or certificate, the request proves possession of that key or certificate, or the requesting client is exempt ({{bound-subject}}); otherwise, `invalid_request` ({{Section 2.2.2 of RFC8693}}).
3. Binding is not required, or the issuing authorization server binds the authorization grant ({{binding-request}}, {{cert-binding}}); otherwise, the error code that {{required-binding}} gives.

If several checks fail, {{errors}} determines which error to return. When it rejects a request for a JWT authorization grant, the issuing authorization server MUST NOT treat the `subject_token` or `actor_token` as used, for example by rotating a refresh token ({{single-use}}).

The issuing authorization server determines which confirmation methods are permitted independently of the proofs in the request. Its policy can decline a method for an audience that does not verify it, based on configuration, an established trust relationship, or metadata ({{as-metadata}}). {{binding-request}} and {{cert-binding}} determine whether the request permits binding with a permitted confirmation method, and {{issuer-outcomes}} summarizes the outcome.

### Bound Subject and Actor Tokens {#bound-subject}

Item 2 of {{issuer-processing}} applies to a `subject_token` or `actor_token` that is bound to a key or certificate, such as a refresh token issued to a public client ({{Section 5 of RFC9449}}). A request proves possession of that key or certificate with a DPoP proof signed with the key, or over a mutual-TLS connection that uses the certificate. A requesting client, such as a resource server that exchanges an access token it received ({{Section 2.3 of RFC8693}}), is exempt if all of the following hold:

* it authenticates;
* it is an intended audience of the token; and
* the token was issued to a client other than the requesting client.

The issuing authorization server establishes the second and third conditions from the token's claims, such as `aud` and `client_id` ({{Section 4.3 of RFC8693}}), or from its own records of the token. If it cannot establish them, the requesting client is not exempt.

The issuing authorization server MUST treat a `subject_token` or `actor_token` as bound if the token contains a `cnf` claim, or if the issuing authorization server's validation of the token, including its own records of the token, yields a `cnf` value, whatever that claim or value contains. Confirmation members that the issuing authorization server does not recognize are not ignored, as they would be under the default in {{Section 3.1 of RFC7800}}, which applies only in the absence of application-specific requirements. Possession of such a token is proven only if its `cnf` claim or value is a JSON object that contains exactly one confirmation member, and the request proves possession of the key or certificate that the member identifies. Otherwise, the token fails item 2 unless the requesting client is exempt.

### DPoP Binding {#binding-request}

When it issues an authorization grant in response to a DPoP binding request, the issuing authorization server MUST bind the grant to the public key of the DPoP proof if `jkt` is a permitted confirmation method for the grant. If `jkt` is not a permitted confirmation method for the grant and binding is not required, it MUST NOT reject a DPoP binding request because it cannot bind the grant; any grant that it issues is unbound, and the response omits the `cnf` response parameter. It MUST NOT bind the grant to any other key or certificate, including when the client also presents a TLS client certificate.

A client can therefore use a TLS client certificate solely for client authentication or to prove possession of a certificate-bound `subject_token` or `actor_token`, even if it cannot use that certificate at redemption. {{proof-replay}} explains the replay risk of selecting the certificate instead.

Issuing an unbound grant when `jkt` is not a permitted confirmation method lets the exchange succeed for a client that sends a DPoP proof in every token request, and for a client that proves possession of a DPoP-bound `subject_token` or `actor_token`. A client that requires binding discards an unbound grant ({{client-processing}}).

### Certificate Binding {#cert-binding}

The issuing authorization server MAY bind the authorization grant to the client's TLS client certificate if:

* the request contains no DPoP proof;
* the request arrives over a mutual-TLS connection in which the client presented that certificate; and
* `x5t#S256` is a permitted confirmation method for the grant.

The issuing authorization server SHOULD NOT bind the grant to a TLS client certificate unless the profile of the grant requires certificate binding, or the client's registration or configuration at the issuing authorization server indicates that the client presents that certificate when it redeems the grant. Neither the `tls_client_certificate_bound_access_tokens` client metadata parameter ({{Section 3.4 of RFC8705}}), which concerns access tokens, nor a requirement for binding is such an indication. Binding in any other case could give a client that does not support this specification a grant that it cannot redeem.

### Required Binding {#required-binding}

Binding is required if the grant's profile requires it. The issuing authorization server MAY also require binding by policy or by the client's registration. A requirement for binding ensures that an issued grant is bound, not that each authorization server in the grant's audience verifies the binding ({{non-enforcing-consumers}}).

{{issuer-outcomes}} shows the outcome of a Token Exchange request for a JWT authorization grant that passes items 1 and 2 of {{issuer-processing}}; a request with an invalid DPoP proof fails item 1. In the table, "permitted" means a permitted confirmation method for the grant. If the issuing authorization server does not apply a required binding, it rejects the request with the error code that the table gives (item 3 of {{issuer-processing}}). The other cells summarize {{binding-request}} and {{cert-binding}}.

| Request contains | Binding not required | Binding required |
|---|---|---|
| A valid DPoP proof; `jkt` permitted | Bound to the key of the DPoP proof | Bound to the key of the DPoP proof |
| A valid DPoP proof; `jkt` not permitted | Unbound | `invalid_target` ({{Section 2.2.2 of RFC8693}}) |
| A TLS client certificate and no DPoP proof | Bound to the certificate if the server applies {{cert-binding}}; otherwise unbound | Bound to the certificate if the server applies {{cert-binding}}; otherwise `invalid_dpop_proof` if `jkt` is permitted, and `invalid_target` if not |
| Neither a DPoP proof nor a TLS client certificate | Unbound | `invalid_dpop_proof` if `jkt` is permitted; `invalid_request` if `x5t#S256` is permitted, `jkt` is not, and the server would bind the grant to the client's TLS client certificate, if the request had one, as {{cert-binding}} describes; otherwise `invalid_target` |
{: #issuer-outcomes title="Outcome of a Request for a JWT Authorization Grant"}

A client that receives the `invalid_dpop_proof` error code for a required binding can repeat the request with a DPoP proof, and one that receives `invalid_request` for a required binding can repeat it over a mutual-TLS connection with its TLS client certificate. In an error response with the `invalid_dpop_proof` error code, the issuing authorization server MAY include a DPoP-Nonce HTTP header field ({{Section 8.2 of RFC9449}}) so that the repeated request can include the nonce and need not first be rejected with `use_dpop_nonce`.

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

### Binding the Authorization Grant {#bind}

To bind an authorization grant, the issuing authorization server includes a `cnf` claim ({{format}}) whose confirmation member identifies the client's key or certificate: for a DPoP binding request, the public key in the `jwk` JOSE Header Parameter of the DPoP proof; for a certificate binding, the TLS client certificate that the client presented. The issuing authorization server MUST NOT bind a grant to a key or certificate whose possession the client has not proven in the request for the grant.

The issuing authorization server need not establish before issuance that the resource authorization server verifies the binding ({{non-enforcing-consumers}}). If binding is required and the issuing authorization server does not bind the grant, it MUST NOT issue the grant. For Token Exchange, {{required-binding}} gives the error code.

When the issued token is a key-bound authorization grant, the issuing authorization server MUST set the `token_type` response parameter to `N_A` ({{Section 2.2.1 of RFC8693}}); the `DPoP` value that {{Section 5 of RFC9449}} requires applies only to a DPoP-bound access token.

{{ex-grant}} shows the claims of the authorization grant issued in response to the DPoP binding request in {{ex-binding-request}}, and the following is an example of the Token Exchange response that carries it:

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

Whatever the value of the `token_type` response parameter, a client that will present the issued token as a JWT authorization grant processes the Token Exchange response as follows if (a) it sent a DPoP binding request, or (b) it presented a TLS client certificate and requires a key-bound grant.

The client determines the binding from the `cnf` response parameter and, if it can read the issued token, from the token's `cnf` claim. A client that cannot read the token, for example because it is encrypted, relies on the `cnf` response parameter. The client compares each confirmation member with its own key or certificate ({{cnf-comparison}}): after a DPoP binding request, the public key of its DPoP proof; otherwise, its TLS client certificate. A confirmation member that the client does not recognize does not match.

A client in case (a) or case (b) MUST discard the issued token if:

* the `cnf` response parameter is present and is not a JSON object that contains exactly one confirmation member;
* the `cnf` response parameter is present, the client can read the token, and the token contains no `cnf` claim;
* the token's `cnf` claim is readable and does not contain exactly one confirmation member;
* a confirmation member of the `cnf` response parameter or of a readable `cnf` claim does not match the client's own key or certificate; or
* neither the `cnf` response parameter nor a readable `cnf` claim is present, and the client requires a key-bound grant.

A client in neither case (a) nor case (b) that receives a `cnf` response parameter presents the token with the corresponding proof if it holds the identified key or certificate, such as the certificate it used for mutual-TLS client authentication. Otherwise, it MUST discard the issued token.

For the response in {{ex-te-response}}, the client verifies that the `jkt` value in the `cnf` response parameter equals the JWK SHA-256 Thumbprint of its DPoP public key.

## Other Issuance Interfaces {#other-issuance}

When a JWT authorization grant is issued through an interface other than Token Exchange, such as a workload platform's token service, {{bind}} applies except for its requirements on the Token Exchange response, with that interface's request in place of the Token Exchange request. The interface defines how the client requests the binding and learns of it. {{redemption}} applies however the grant was issued.

# Token Exchange Response Parameter {#cnf-param}

This specification defines the use of the `cnf` token response parameter, which {{RFC9201}} registers, in a Token Exchange response. It applies to any token issued through Token Exchange ({{Section 2.1 of RFC8693}}), not only to a JWT authorization grant; {{client-processing}} describes how a client that requested a key-bound authorization grant processes it. The `access_token` parameter of the response carries the issued token, whatever its type ({{Section 2.2.1 of RFC8693}}). The `cnf` parameter identifies the key or certificate to which the authorization server bound that token, including an opaque or encrypted one. It does not describe a `refresh_token` in the same response.

`cnf`:
: A JSON object that contains exactly one confirmation member, with the syntax and semantics of the `cnf` claim ({{Section 3.1 of RFC7800}}). Its presence depends on the issued token:

  * Bound, and `token_type` is `N_A`, as for a key-bound authorization grant ({{bind}}): REQUIRED.
  * Bound, with another `token_type`, such as `DPoP`: OPTIONAL, unless another specification requires it, as {{Section 3.2 of RFC9201}} does for a `pop` token with a symmetric proof-of-possession key.
  * Not bound: MUST NOT be present.

When the issued token is a JWT that contains a `cnf` claim, the parameter MUST identify the same key or certificate as that claim ({{cnf-comparison}}). The authorization server SHOULD use the same confirmation member in the parameter as in the claim, and SHOULD use a thumbprint member, such as `jkt` or `x5t#S256`, rather than a `jwk` member ({{privacy}}).

For an access token, a `token_type` of `DPoP` shows a DPoP binding ({{Section 5 of RFC9449}}), but {{RFC8705}} defines no `token_type` value for a certificate binding; the `cnf` response parameter, if present, identifies the key or certificate of either binding.

# Redeeming a Key-Bound Authorization Grant {#redemption}

## Access Token Request {#token-request}

The client presents a key-bound authorization grant in an access token request ({{Section 4.1 of RFC7521}}) and chooses the grant type in this order:

1. `jwt-pop` ({{jwt-pop}}): the client SHOULD use it if the resource authorization server advertises it ({{as-metadata}}) and the client's registration permits it ({{client-metadata}}).
2. `jwt-dpop` ({{legacy-jwt-dpop}}): otherwise, for a grant with a `jkt` member, the client SHOULD use it under the same two conditions.
3. `jwt-bearer`: the client can use this grant type if its registration permits it. A server that implements only {{JWT-DPOP-GRANT}} rejects a grant with a `jkt` member presented this way.

If the request is rejected with the `unauthorized_client` or `unsupported_grant_type` error code, the client can retry with the same grant and another grant type, with a new DPoP proof where one is needed ({{Section 4.2 of RFC9449}}). A resource authorization server that supports this specification does not record a grant as used when it rejects a request ({{processing}}).

The client proves possession of the key or certificate identified by the authorization grant's `cnf` claim:

* For a `jkt` or `jwk` member, the client MUST include a DPoP proof ({{Section 4 of RFC9449}}) signed with that key in the DPoP HTTP header field of the request.
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

The `jwt-dpop` grant type, first defined in {{JWT-DPOP-GRANT}}, is a legacy grant type for existing clients; a new client uses `jwt-pop`. A resource authorization server:

* MAY support `jwt-dpop`, and SHOULD continue to support it while clients are registered for it;
* SHOULD, if it supports `jwt-dpop`, include that value in its `grant_types_supported` metadata parameter; and
* processes a `jwt-dpop` request by the rules for `jwt-pop` ({{jwt-pop}}), and MUST also reject it with the `invalid_grant` error code if the confirmation member of the grant's `cnf` claim is not `jkt`.

An issuing authorization server that implements {{JWT-DPOP-GRANT}} can rely on that `grant_types_supported` value to determine that the resource authorization server verifies DPoP-bound authorization grants.

This specification differs from {{JWT-DPOP-GRANT}} as follows:

* An issuing authorization server binds an authorization grant without first checking that the resource authorization server verifies the binding ({{non-enforcing-consumers}}), contrary to the recommendation in {{Section 7.1 of JWT-DPOP-GRANT}}.
* A resource authorization server verifies, instead of rejecting, a key-bound authorization grant presented with the `jwt-bearer` grant type ({{grant-type-substitution}}).
* A resource authorization server returns the `invalid_grant` error code, not `invalid_dpop_proof`, for a DPoP-bound authorization grant presented without a DPoP proof ({{processing}}).

## Resource Authorization Server Processing {#processing}

A resource authorization server validates a JWT authorization grant as defined in {{Section 3 of RFC7523}}, as updated by {{Section 4 of RFC10070}}, and as the profile of the grant defines.

When the profile of a grant defines a `typ` value, a resource authorization server SHOULD reject, with the `invalid_grant` error code, a grant that does not carry it, and MUST do so when the profile states that its validation rules rely on that value to distinguish its grants from other kinds of JWTs (item 1 of {{profiles}}, {{cross-jwt}}).

If the profile of the authorization grant requires binding and the grant has no `cnf` claim, the resource authorization server MUST reject the request with the `invalid_grant` error code.

If the grant contains a `cnf` claim, a resource authorization server that supports this specification MUST also ensure the following, whichever grant type the request uses, and MUST otherwise reject the request with the indicated error code ({{errors}}):

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

If several checks fail, {{errors}} determines which error to return.

Item 1 is an application-specific requirement that replaces the default in {{Section 3.1 of RFC7800}}, under which implementations ignore confirmation members they do not understand. Client authentication does not take the place of any of these checks.

A resource authorization server that supports DPoP rejects an invalid DPoP proof ({{Section 5 of RFC9449}}) whatever the binding of the grant. A request that presents a DPoP-bound grant without a DPoP proof is rejected with the `invalid_grant` error code (item 2), not `use_dpop_nonce`.

For an `x5t#S256` member, the resource authorization server need not authenticate the client with the certificate or trust the certificate's issuer ({{Section 4 of RFC8705}}).

A resource authorization server that detects reuse of a JWT authorization grant, bound or not, for example by its `jti` claim ({{Section 4.1.7 of RFC7519}}), MUST NOT record the grant as used when it rejects the request ({{single-use}}).

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

The client then repeats the request with the same grant and a DPoP proof that contains the nonce.

## Access Token Response {#token-response}

For a key-bound authorization grant, {{Section 5 of RFC9449}} and {{Section 3 of RFC8705}} determine the binding of the tokens that the resource authorization server issues, except as this section and {{refresh}} restrict it. This specification does not restrict the binding of tokens issued for an unbound authorization grant, such as a binding to a key that the grant conveys in a `requested_cnf` claim ({{Appendix B.3 of ID-CHAINING}}).

When the authorization grant is key-bound, the resource authorization server SHOULD bind the access token that it issues, and indicate the binding as defined in {{Section 5 of RFC9449}} or {{Section 3 of RFC8705}}. For a key-bound grant, it MUST NOT bind the access token, or a refresh token that it issues, to a key or certificate other than the following:

* for a DPoP-bound authorization grant, the grant's key, which is the key of the DPoP proof; or
* for a certificate-bound authorization grant, the grant's certificate or, if the resource authorization server supports DPoP and the request contains a valid DPoP proof, the public key of that proof.

A TLS client certificate presented with a DPoP-bound authorization grant therefore does not determine the binding, even when the client authenticates with it ({{proof-replay}}). {{binding-outcomes}} summarizes these rules. It lists the key or certificate to which an issued token can be bound, not whether the token is bound: the access token can be unbound, and {{refresh}} determines whether a refresh token is bound.

| Authorization grant's confirmation member | Proof in the request | Permitted key or certificate when the token is bound |
|---|---|---|
| `jkt` or `jwk` | DPoP proof, with or without a TLS client certificate | Authorization grant's key |
| `x5t#S256` | TLS client certificate only | Authorization grant's certificate |
| `x5t#S256` | TLS client certificate and DPoP proof | Authorization grant's certificate or key of the DPoP proof; for a refresh token issued to a public client by a resource authorization server that supports DPoP, key of the DPoP proof |
{: #binding-outcomes title="Permitted Binding of Tokens Issued for a Key-Bound Authorization Grant"}

The last row of {{binding-outcomes}} follows {{Section 5 of RFC9449}} for a refresh token issued to a public client; {{refresh}} departs from that section for a refresh token issued to a confidential client.

## Refresh Tokens {#refresh}

If the resource authorization server issues a refresh token for a key-bound authorization grant, it MUST bind the refresh token as described in {{token-response}}, whether or not the client authenticates, with one exception. It MAY instead bind the refresh token only to the client if:

* the client authenticates; and
* the resource authorization server verifies that the grant identifies the authenticated client, in a claim that the profile of the grant defines.

The resource authorization server decides on this exception when the client redeems the grant, never at a later refresh. The replacement for a refresh token bound only to the client is bound only to that client; the replacement for a refresh token bound to a key or certificate is bound as {{refresh-processing}} describes. A refresh token's association with the authenticated client ({{Section 6 of RFC6749}}) never qualifies it for the exception.

A binding to a key or certificate is in addition to the binding to the client that {{Section 10.4 of RFC6749}} requires. The requirement to bind the refresh token to a key or certificate is stricter than {{Section 5 of RFC9449}}, under which a refresh token issued to a confidential client is not bound to the DPoP key, and than the recommendation in {{Section 4 of RFC8705}}; {{proof-replay}} gives the reason.

### Processing a Refresh Request {#refresh-processing}

When a client presents a refresh token that the resource authorization server bound to a key or certificate under {{refresh}}, the resource authorization server treats that key or certificate as the authorization grant's. It MUST verify the binding as items 2 and 3 of {{processing}} do, with the same error codes, and MUST bind the tokens that it issues in response as {{token-response}} and {{refresh}} require. The binding of the presented refresh token does not change, even when the issued tokens are bound to the key of a DPoP proof ({{binding-outcomes}}).

### Proofs in a Refresh Request {#refresh-proofs}

The client presents a refresh token issued for a key-bound authorization grant with each proof of possession that it presented when it obtained the refresh token: a DPoP proof for the same key, a mutual-TLS connection that uses the same certificate, or both. It does so because the response does not always show the binding: a resource authorization server can bind to the certificate although the request contained a DPoP proof, and return a `token_type` of `Bearer`.

A client that knows the binding of the refresh token, because the profile of the grant or an arrangement with the resource authorization server establishes it, need not present a proof of possession that the binding does not require, and needs none for a refresh token bound only to the client. It still presents a DPoP proof where its registration requires one ({{Section 5.2 of RFC9449}}) or where it needs DPoP-bound tokens, and it still needs a certificate that it uses for client authentication.

In the following example, a confidential client obtained a refresh token bound to the certificate in {{cert-example}} with a request that carried both that certificate and a DPoP proof. It therefore refreshes over a mutual-TLS connection that uses the certificate, with which it also authenticates, and with a DPoP proof signed with the key of that earlier DPoP proof:

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

Where a profile allows a key-bound authorization grant in place of client authentication, a resource authorization server MUST reject a request with the `invalid_client` error code ({{Section 5.2 of RFC6749}}) if:

* the request has no client authentication;
* its grant has no `cnf` claim; and
* the request passes every check of {{processing}} and of its grant type.

A request that fails a check of {{processing}} or of its grant type is rejected with the error code that the check names ({{errors}}); for a JWT that is not valid, {{Section 3.1 of RFC7523}} requires `invalid_grant`.

# Error Responses {#errors}

An authorization server returns the error specified by the failed check, as defined in {{Section 5.2 of RFC6749}}. The HTTP status is 400 (Bad Request), except where that section permits or requires 401 (Unauthorized) for `invalid_client`.

At either authorization server, if a DPoP proof in the request requires a nonce challenge under {{Section 8 of RFC9449}}, the server MUST use the `use_dpop_nonce` error code for that failure. This also applies when the proof is needed to verify a key-bound grant or a bound `subject_token` or `actor_token`.

When two or more of the errors listed below apply, the authorization server MUST return the first applicable error in the order shown. Both authorization servers start with:

1. `invalid_client`, if the request includes client authentication and that authentication fails;
2. `unauthorized_client`, if the client authenticated and is not permitted to use the grant type of the request ({{Section 5.2 of RFC6749}}); and
3. `invalid_dpop_proof` or `use_dpop_nonce`.

The issuing authorization server continues with `invalid_request`, then `invalid_target`. The resource authorization server continues with `invalid_grant`, then `unauthorized_client` for a client identified from the authorization grant ({{client-auth}}), then `invalid_client` for a request without client authentication.

# Metadata {#metadata}

## Authorization Server Metadata {#as-metadata}

This specification defines the following authorization server metadata parameter ({{Section 2 of RFC8414}}):

`jwt_grant_cnf_methods_supported`:
: RECOMMENDED for a resource authorization server that supports this specification; REQUIRED if its `grant_types_supported` metadata parameter contains the `jwt-pop` grant type. A nonempty JSON array of Confirmation Method Values from the "JWT Confirmation Methods" registry ({{Section 6.2 of RFC7800}}) that lists each confirmation method, among `jkt`, `jwk`, and `x5t#S256`, whose binding the authorization server verifies in a JWT authorization grant ({{processing}}), whatever the grant type of the request or the profile of the authorization grant. Listing one confirmation method does not indicate support for another; in particular, `jkt` does not indicate `jwk`.

A resource authorization server that supports the `jwt-pop` grant type SHOULD include it in the `grant_types_supported` metadata parameter. The `jwt-pop` value alone indicates no confirmation method.

A client or an issuing authorization server can use the `jwt_grant_cnf_methods_supported` metadata parameter to learn which confirmation methods a resource authorization server verifies, for example to choose a confirmation method, to confirm that a binding will be enforced ({{non-enforcing-consumers}}), or to decline a method that the server does not verify ({{issuer-processing}}). A client that chooses a confirmation method retrieves the metadata of the authorization server to which it presents the authorization grant.

{{metadata-evidence}} shows what metadata is evidence of for one confirmation method. Validated metadata is metadata retrieved and validated as described in {{Section 3 of RFC8414}}; for the authorization servers in a grant's audience, it is retrieved and validated as {{non-enforcing-consumers}} describes.

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

A resource authorization server can limit an identified client to key-bound authorization grants by registering it with a `grant_types` client metadata value ({{Section 2 of RFC7591}}) that includes `jwt-pop` and not `jwt-bearer`. The server then rejects a `jwt-bearer` request from that client with the `unauthorized_client` error code ({{Section 5.2 of RFC6749}}), and rejects an unbound grant presented with `jwt-pop` or `jwt-dpop` ({{jwt-pop}}, {{legacy-jwt-dpop}}). In addition:

* The limit applies only when the server identifies the client: by client authentication or, where a key-bound grant takes the place of client authentication ({{client-auth}}), from the grant as the profile defines. A request that does not identify a client is not limited by any client's registration, but the `jwt-pop` grant type still requires a key-bound grant in each request ({{jwt-pop}}).
* A server can instead reject unbound grants by policy while accepting the `jwt-bearer` grant type, but the `grant_types` client metadata parameter then does not express that the client is limited to key-bound grants.
* Support for both the `jwt-pop` and `jwt-dpop` grant types does not let every client use both: the server rejects, with the `unauthorized_client` error code, a request whose grant type the client's registration does not permit. A new client is registered for `jwt-pop`; an existing client keeps `jwt-dpop` only where its registration permits it.
* The `jwt-pop` grant type has no corresponding `response_types` value ({{Section 2.1 of RFC7591}}).
* The `dpop_bound_access_tokens` client metadata parameter ({{Section 5.2 of RFC9449}}) requires a DPoP proof in each token request, but not a key-bound grant.

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

A specification that defines a JWT authorization grant, such as {{ID-JAG}}, uses this specification by stating the following items. Where a profile does not address an item, the default that the item states applies.

1. Validation rules: the rules that distinguish the authorization grant from other kinds of JWTs that its issuing authorization server issues ({{Section 3.12 of RFC8725}}) or that a resource authorization server accepts, including the value of the grant's `typ` JOSE Header Parameter ({{Section 3.11 of RFC8725}}) and whether those rules rely on it, and, if the grant is issued through Token Exchange, its `requested_token_type` value.
2. Confirmation methods: which confirmation methods an issuing authorization server can apply, which a resource authorization server is required to verify, which are mandatory to implement, and whether the `jwt-pop` grant type is mandatory to implement. By default, either confirmation method can be used, and a resource authorization server verifies any `cnf` claim it receives.
3. Binding policy: whether binding is optional, recommended, or required, and whether that varies by resource authorization server. If the profile requires the binding to be enforced, and not only applied, who establishes that each resource authorization server in the authorization grant's audience verifies it, and from what evidence ({{non-enforcing-consumers}}). By default, binding is optional. An issuing authorization server can still require binding ({{required-binding}}); that does not make binding required at the resource authorization server.
4. Client identity: the claim, if any, in which the issuing authorization server identifies the client ({{refresh}}), and whether a resource authorization server can accept a key-bound authorization grant in place of client authentication ({{client-auth}}). If it can: how the identity of the client is tied to the key or certificate; what the issuing authorization server verifies before it binds the grant; and how the resource authorization server maps that identity to its own client registration and policy. By default, a key-bound grant does not take the place of client authentication: whether the client authenticates is the resource authorization server's policy ({{Section 3.1 of RFC7523}}), and {{client-auth}} does not apply.
5. Additional processing: any error codes or processing beyond those in {{errors}} and {{processing}}, such as an error response for a request that carries no proof of possession when the requested resource requires a sender-constrained access token.

A profile that permits a key-bound authorization grant in place of client authentication MUST require the issuing authorization server to authenticate the client in the same request in which it verifies proof of possession of the key or certificate.

# Security Considerations {#security}

The security considerations in {{Section 8 of RFC7521}}, {{Section 6 of RFC7523}}, {{Section 5 of RFC8693}}, {{Section 11 of RFC9449}}, and {{Section 7 of RFC8705}} apply to this specification.

## Non-Enforcing Resource Authorization Servers {#non-enforcing-consumers}

{{RFC7523}} defines no processing of the `cnf` claim, and JWT implementations ignore claims and confirmation members that they do not understand ({{Section 4 of RFC7519}}, {{Section 3.1 of RFC7800}}). A resource authorization server that supports neither this specification nor another specification that verifies the binding, such as {{JWT-DPOP-GRANT}} or {{ID-JAG}}, therefore accepts a key-bound authorization grant as a bearer grant: a party that obtains the grant can redeem it without the key.

The issuing authorization server can bind a grant without first checking the audience's support ({{bind}}). When binding is optional, the alternative is an unbound grant. Against a stolen grant, binding is never weaker: a key-bound grant is a bearer grant where its binding is not verified, and is protected wherever it is verified. This holds even if the issuing authorization server's information about the audience is incomplete or stale. Binding has two costs:

* A key-bound grant can be rejected where an unbound one is accepted, by a resource authorization server that does not support its confirmation method, or by one that implements only {{JWT-DPOP-GRANT}} and receives a DPoP-bound grant with the `jwt-bearer` grant type. For the latter, {{token-request}} directs the client to the `jwt-dpop` grant type where that server advertises it and the client's registration permits it; otherwise, the request fails. An issuing authorization server that has determined that a resource authorization server does not verify a confirmation method can decline that method for it ({{issuer-processing}}).
* The confirmation member reaches servers that do not verify it ({{privacy}}).

A party holding a grant can present it to any authorization server in its audience. Where enforcement is required, the profile, configuration, or the issuing authorization server's policy establishes that every such server verifies the binding ({{profiles}}). A client that needs this assurance obtains evidence for each audience from validated metadata ({{as-metadata}}), configuration, or the profile. If the client cannot read the grant's `aud` claim, it relies on configuration or the profile to establish the audience.

A client that confirms that a binding will be enforced, and an issuing authorization server, retrieve the metadata for each `aud` value of the grant, using as the issuer identifier either:

* that `aud` value; or
* an issuer identifier that its configuration associates with that `aud` value, for example when the `aud` value is a token endpoint URL ({{Section 4 of RFC10070}}). When the `aud` value is a token endpoint URL, the `token_endpoint` value of the metadata also has to equal the `aud` value. That equality alone does not establish the association, because only the `issuer` value of metadata is validated ({{Section 3.3 of RFC8414}}, {{Section 1 of RFC10070}}).

## Grant Type Substitution {#grant-type-substitution}

A separate grant type for key-bound authorization grants does not by itself prevent substitution: a party holding a stolen key-bound grant can present it with the `jwt-bearer` grant type. A resource authorization server that supports this specification therefore verifies the binding of every JWT authorization grant that contains a `cnf` claim, whichever grant type the request uses ({{processing}}).

The `jwt-pop` grant type adds no protection against a stolen authorization grant. It adds the following:

* a requirement for a key-bound grant in each request; a resource authorization server that does not support the grant type rejects the request with the `unsupported_grant_type` error code ({{Section 5.2 of RFC6749}}), whereas a server ignores client metadata that it does not understand ({{Section 2 of RFC7591}}); and
* a way to limit an identified client to key-bound grants through its registration ({{client-metadata}}).

Verifying a key-bound grant presented with the `jwt-bearer` grant type, instead of rejecting it as {{JWT-DPOP-GRANT}} does, lets clients registered only for that grant type continue to work.

Adopting this specification can make a resource authorization server reject authorization grants that it previously accepted (item 1 of {{processing}}), and an issuing authorization server reject Token Exchange requests that it previously accepted ({{bound-subject}}).

## Cross-JWT Confusion {#cross-jwt}

A key-bound authorization grant and a DPoP-bound JWT access token for the same client can carry the same `jkt` value. A party that receives one could present it where the other is expected. The `aud` claim of a grant identifies an authorization server ({{RFC10070}}), and an explicit `typ` value ({{Section 3.11 of RFC8725}}) lets a recipient reject a JWT of the wrong kind ({{Section 3.12 of RFC8725}}). {{processing}} specifies when a resource authorization server rejects a grant that lacks the `typ` value that its profile defines.

## DPoP Proof Replay {#proof-replay}

The DPoP replay protections ({{Section 11.1 of RFC9449}}) and server-provided nonces ({{Section 8 of RFC9449}}) apply at the token endpoints of both the issuing and the resource authorization server. A DPoP proof sent to one cannot be replayed at the other when their token endpoint URIs do not match under the comparison rules of {{Section 4.3 of RFC9449}}, because the `htu` claim identifies the endpoint. Those rules ignore query and fragment parts, so token endpoint URIs that differ only in their query components ({{Section 3.2 of RFC6749}}) do not provide this separation.

A DPoP proof does not cover the authorization grant it accompanies or the TLS connection that carries it. An attacker that captures a grant together with a DPoP proof for its key can replay both, over its own TLS connection, within the window in which the resource authorization server accepts the proof. What the attacker obtains depends on the binding of the issued tokens:

* An access token or refresh token bound to the authorization grant's key is of no use to the attacker.
* An access token bound to a TLS client certificate that the attacker presents would be usable; {{token-response}} forbids that binding. The reverse case is safe: for a certificate-bound authorization grant, the mutual-TLS handshake shows that a DPoP proof comes from the holder of the grant's certificate, so {{token-response}} lets the issued tokens be bound to the key of that proof.
* A refresh token bound only to a client as which the attacker authenticates would be usable, at the first refresh or a later one. {{refresh}} therefore requires the resource authorization server to bind every refresh token issued for a key-bound authorization grant, and every replacement, to a key or certificate, unless it verifies that the grant identifies the authenticated client; an attacker that authenticates as its own client cannot meet that condition.
* An unbound access token is usable.

A resource authorization server that issues unbound access tokens for DPoP-bound authorization grants SHOULD reject a DPoP proof whose `jti` value it has already accepted at its token endpoint within that window ({{Section 11.1 of RFC9449}}). A server-provided nonce ({{Section 8 of RFC9449}}) shortens the window but does not by itself detect reuse, because a client can use the same nonce more than once. Neither measure denies an unbound access token to an attacker whose request arrives before the client's; only binding the access token does.

A captured request can also be replayed at the issuing authorization server. An attacker that captures a Token Exchange request that proves possession of a DPoP-bound `subject_token` or `actor_token` can replay the token and the DPoP proof over its own mutual-TLS connection. It has to do so within the window in which the issuing authorization server accepts the proof and, if that server rejects a reused `jti` value, before the client's request. It also has to satisfy any client authentication that the issuing authorization server requires: it can replay a client secret, or a client assertion that the server accepts more than once, or authenticate as its own client if the server lets that client exchange the token ({{pop-identity}}). What the attacker obtains depends on the binding of the issued authorization grant:

* A grant bound to the key of the DPoP proof is of no use to the attacker at a resource authorization server that verifies the binding, although it is usable at one that does not ({{non-enforcing-consumers}}).
* A grant bound to the attacker's TLS client certificate would be usable at every resource authorization server in its audience. This is one reason the issuing authorization server binds the grant only to the key of the DPoP proof ({{binding-request}}).
* An unbound grant, which the issuing authorization server issues when `jkt` is not a permitted confirmation method and binding is not required ({{binding-request}}), is usable ({{subject-token-theft}}).

An issuing authorization server that issues unbound authorization grants in response to requests that prove possession of a DPoP-bound `subject_token` or `actor_token` SHOULD likewise reject a DPoP proof whose `jti` value it has already accepted at its token endpoint within the window in which it accepts the proof ({{Section 11.1 of RFC9449}}).

When neither the `subject_token` nor the `actor_token` requires proof of possession of a DPoP key, binding the grant only to the key of the DPoP proof adds no protection against an attacker that can meet the other requirements of the exchange, because that attacker can request a grant bound to its own key.

## Exhaustion of Single-Use Authorization Grants {#single-use}

Binding limits who can redeem an authorization grant, not how often, so short grant lifetimes and single-use enforcement ({{Section 5.5 of ID-CHAINING}}) remain useful for key-bound grants. A resource authorization server that limits a grant to a single use could, however, mark a key-bound grant as used when a party without the key presents it, denying the legitimate client its redemption. It could also mark the grant as used when it rejects a request with the `use_dpop_nonce` error code, so that the client's retry with the nonce fails. The resource authorization server prevents both by not recording a grant as used when it rejects the request ({{processing}}). The issuing authorization server applies the same rule to a single-use `subject_token` or `actor_token` ({{issuer-processing}}).

## Proof of Possession and Client Identity {#pop-identity}

Proof of possession shows that the presenter holds the key or certificate, not which client it is: even at the issuing authorization server, a DPoP proof is not cryptographically bound to the accompanying client authentication ({{Section 11.11 of RFC9449}}). A resource authorization server that accepts a key-bound authorization grant in place of client authentication ({{client-auth}}) therefore depends on the issuing authorization server's authentication of the client and on the client's protection of its private key.

## Theft of the Subject Token {#subject-token-theft}

Binding the issued authorization grant does not protect a bearer `subject_token` that is stolen before the exchange. An attacker that holds such a `subject_token`, and can satisfy any client authentication that the issuing authorization server requires, can request a grant bound to the attacker's own key. Protection of a bearer `subject_token` is outside the scope of this specification; see {{RFC9700}}.

The exemption in {{bound-subject}} can weaken binding when several resource servers share one audience value. Each server that qualifies for the exemption can exchange a bound `subject_token` that it obtains, including one obtained from another server, without holding the key.

When `jkt` is not a permitted confirmation method for the authorization grant and binding is not required, a client that proves possession of a DPoP-bound `subject_token` or `actor_token` receives an unbound grant ({{binding-request}}), which is a bearer token, as it would be without this specification.

## Informational Nature of the Response Parameter

The `cnf` response parameter is informational. The binding is enforced by the party to which the issued token is presented, which verifies the token's `cnf` claim against the presenter's proof. A verified `cnf` response parameter tells the client only that the authorization server applied the binding; it does not tell the client that this party enforces it ({{non-enforcing-consumers}}). The client's check of the parameter ({{client-processing}}) protects against an authorization server that fails to apply a binding through error or misconfiguration, not against a malicious one, which controls the contents of the token.

The token endpoint requires TLS ({{Section 3.2 of RFC6749}}), which prevents an on-path attacker from removing or modifying the `cnf` response parameter.

# Privacy Considerations {#privacy}

The privacy considerations in {{Section 8 of RFC8705}} apply to this specification.

A confirmation member is a stable identifier for the bound key or certificate. It reaches the resource authorization server whether or not that server verifies the binding ({{bind}}). A client that uses the same key or certificate with several resource authorization servers lets them correlate its requests. A client that needs to avoid such correlation uses a separate key or certificate for each resource authorization server. This is not possible when the client proves possession of a DPoP-bound `subject_token` or `actor_token`, because a key-bound authorization grant is then bound to that token's key ({{te-request}}).

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
: DPoP-Bound JWT Authorization Grant

Change Controller:
: IETF

Specification Document(s):
: {{JWT-DPOP-GRANT}} and {{legacy-jwt-dpop}} of this document

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

This appendix is informative. It shows how {{ID-JAG}} would answer the items in {{profiles}} and lists the conflicts between this specification and {{ID-JAG}} and {{ID-CHAINING}}.

## Identity Assertion JWT Authorization Grant

{{ID-JAG}} would answer the items as follows:

1. Validation rules: the `typ` value is `oauth-id-jag+jwt`, and the validation rules rely on it ({{Section 4.4.1 of ID-JAG}}); the `requested_token_type` value is `urn:ietf:params:oauth:token-type:id-jag`.
2. Confirmation methods: both `jkt` and `x5t#S256` can be used, and neither method, nor the `jwt-pop` grant type, is mandatory to implement. A Resource Authorization Server verifies each method that it lists in its `jwt_grant_cnf_methods_supported` metadata parameter ({{as-metadata}}). A DPoP-bound grant is presented with the `jwt-dpop` grant type ({{Section 4.4 of ID-JAG}}, {{legacy-jwt-dpop}}).
3. Binding policy: optional. With a valid DPoP proof, the IdP Authorization Server binds the grant to the DPoP key, unless its local policy selects a certificate binding ({{Section 9.11.1.1 of ID-JAG}}). Its policy does not otherwise decline the `jkt` confirmation method ({{issuer-processing}}).
4. Client identity: a Resource Authorization Server can accept a key-bound grant in place of client authentication. The IdP Authorization Server authenticates the client in the Token Exchange request, binds the grant to the key or certificate for which that request carries proof of possession, and identifies the client in the grant's `client_id` claim.
5. Additional processing: a Resource Authorization Server validates a certificate presented for a certificate-bound authorization grant as described in {{Section 2 of RFC8705}} ({{Section 9.11.2.2 of ID-JAG}}), and rejects an unbound grant with the `invalid_grant` error code when the resource requires sender-constrained access tokens and the request carries neither a DPoP proof nor a TLS client certificate ({{Section 9.11.1.2.4 of ID-JAG}}, {{Section 9.11.2.2 of ID-JAG}}).

A client that detects a downgrade from the grant's `cnf` claim ({{Section 9.11.1.1 of ID-JAG}}, {{Section 9.11.2.1 of ID-JAG}}) follows {{client-processing}}, which also accepts the `cnf` response parameter.

## Conflicts

Each conflict could be resolved by a change to either document (item 2 of {{open-issues}}):

* Certificate binding with a DPoP proof. {{Section 9.11 of ID-JAG}} lets the IdP Authorization Server's local policy choose a certificate binding when a request carries both a DPoP proof and a TLS client certificate, whereas under this specification the issuing authorization server binds only to the key of the DPoP proof ({{binding-request}}, {{proof-replay}}).
* Required `jwt-bearer` registration. {{Section 8 of ID-JAG}} requires a client that supports that profile to include `jwt-bearer` in its `grant_types` client metadata value, which prevents a Resource Authorization Server from limiting the client to key-bound grants ({{client-metadata}}). {{Section 4.4 of ID-JAG}} presents a DPoP-bound grant with `jwt-dpop` and other grants with `jwt-bearer`.
* Key in a `requested_cnf` claim. {{ID-CHAINING}} defines no profile of this specification, so the defaults in {{profiles}} apply. The access token in {{Appendix B.3 of ID-CHAINING}} is bound to a key that the grant conveys in a `requested_cnf` claim, a binding that {{binding-outcomes}} does not permit when the grant's `cnf` claim identifies a different key. Item 2 of {{open-issues}} asks whether a profile should be able to permit that binding.

# Open Issues {#open-issues}

1. Grant type or metadata. Should key-bound authorization grants keep the `jwt-pop` grant type, or use `jwt-bearer` with metadata that requires key-bound grants? [](https://github.com/mcguinness/draft-mcguinness-oauth-jwt-grant-pop/issues/27)
2. Alignment with other documents. How should the conflicts with other documents be resolved ({{profile-examples}}), should a profile be able to permit a key in a `requested_cnf` claim, and should a client be able to opt into a certificate binding? Both the proposal to replace {{JWT-DPOP-GRANT}} and the rules for the `jwt-dpop` grant type ({{legacy-jwt-dpop}}) need the agreement of the author of {{JWT-DPOP-GRANT}}. [](https://github.com/mcguinness/draft-mcguinness-oauth-jwt-grant-pop/issues/28)
3. Required binding. Is a per-request "bind or fail" option needed, and should a required binding also require establishing that each authorization server in the grant's audience verifies it? [](https://github.com/mcguinness/draft-mcguinness-oauth-jwt-grant-pop/issues/29)
4. Mandatory-to-implement confirmation method. Should `jkt` be mandatory to implement for both authorization servers by default? [](https://github.com/mcguinness/draft-mcguinness-oauth-jwt-grant-pop/issues/30)
5. Holder of the key. Should a profile state whether the proven key belongs to the client, the subject, or an actor, and when proof of possession can then take the place of client authentication? [](https://github.com/mcguinness/draft-mcguinness-oauth-jwt-grant-pop/issues/31)

# Acknowledgments
{:numbered="false"}

This document replaces {{TE-CNF}}, proposes to replace {{JWT-DPOP-GRANT}}, and generalizes the sender-constraining rules of {{ID-JAG}}. The author thanks the authors and reviewers of those documents.

# Document History
{:numbered="false"}

-00

* Initial version, building on {{JWT-DPOP-GRANT}} and {{TE-CNF}}.
* Generalizes the `jwt-dpop` grant type into the method-neutral `jwt-pop` grant type, adds the `jwt_grant_cnf_methods_supported` metadata parameter, keeps `jwt-dpop` for existing clients, and verifies key-bound authorization grants whatever their grant type.
