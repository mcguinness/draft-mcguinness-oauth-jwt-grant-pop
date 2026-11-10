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

informative:
  IANA.oauth-parameters:
  RFC6755:
  RFC7516:
  RFC7591:
  RFC8725:
  RFC9201:
  RFC9700:
  ID-CHAINING: I-D.ietf-oauth-identity-chaining
  ID-JAG: I-D.ietf-oauth-identity-assertion-authz-grant
  JWT-DPOP-GRANT: I-D.parecki-oauth-jwt-dpop-grant
  TE-CNF: I-D.mcguinness-oauth-token-exchange-cnf

--- abstract

This specification defines how an authorization server binds a JSON Web Token (JWT) authorization grant (RFC 7523) to a key or certificate held by the client, using OAuth 2.0 Demonstrating Proof of Possession (DPoP) or OAuth 2.0 Mutual-TLS, and how the authorization server that receives the grant verifies that binding. It profiles OAuth 2.0 Token Exchange so that a client can request a DPoP-bound grant and learn which binding was applied, and it defines a grant type that requires a bound grant and an authorization server metadata parameter that lists the confirmation methods that a server verifies. It also lists the choices that a specification defining a particular JWT authorization grant makes to use this mechanism.

--- middle

# Introduction

A JWT authorization grant ({{Section 2.1 of RFC7523}}) lets a client obtain an access token from an authorization server by presenting a JWT issued by a party that the authorization server trusts. Identity and authorization chaining {{ID-CHAINING}} uses OAuth 2.0 Token Exchange {{RFC8693}} to obtain such a grant in one trust domain and present it in another, and the Identity Assertion JWT Authorization Grant {{ID-JAG}} profiles that pattern for deployments that share a single sign-on identity provider. {{Section 5.5 of ID-CHAINING}} notes that "The authorization grant obtained from the Token Exchange process is a bearer token": any party that obtains the grant can redeem it until it expires. The mitigations listed there do not include binding the grant to a key.

OAuth 2.0 Demonstrating Proof of Possession (DPoP) {{RFC9449}} and OAuth 2.0 Mutual-TLS {{RFC8705}} bind the access token, and in some cases the refresh token, issued in a token response to a key or certificate held by the client. They do not define:

* how an authorization server binds a JWT authorization grant that Token Exchange issues, for which the `token_type` response parameter is `N_A` ({{Section 2.2.1 of RFC8693}});
* how an authorization server that receives a JWT authorization grant containing a `cnf` claim {{RFC7800}} verifies proof of possession; or
* how a client learns which binding was applied when it cannot read the issued token, for example because the token is opaque or is encrypted {{RFC7516}} for its consumer.

This specification defines both halves of the binding. {{format}} defines what a bound grant contains. {{issuance}} defines how a grant is bound when it is issued and how the client learns the binding. {{redemption}} defines how the resource authorization server verifies the binding, however the grant was issued. {{metadata}} defines discovery and per-client enforcement, and {{profiles}} lists the choices that a specification defining a particular JWT authorization grant makes.

This specification defines no new confirmation method. It uses the `jkt` ({{Section 6.1 of RFC9449}}) and `x5t#S256` ({{Section 3.1 of RFC8705}}) confirmation methods, and also accepts the `jwk` confirmation method ({{Section 3.2 of RFC7800}}) in a grant that it redeems.

## Relationship to Other Documents {#relationship}

This specification relates to other documents as follows:

* It replaces {{JWT-DPOP-GRANT}}. It extends that document from DPoP to mutual TLS, and replaces its `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type with the method-neutral `urn:ietf:params:oauth:grant-type:jwt-pop` grant type ({{jwt-pop}}), keeping the former for existing clients ({{legacy-jwt-dpop}}). Unlike that document, it:
  * verifies the binding of a grant presented with the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type instead of rejecting it ({{grant-type-substitution}}); and
  * returns the `invalid_grant` error code, not `invalid_dpop_proof`, for a request that carries no DPoP proof ({{processing}}).
* It replaces {{TE-CNF}}, whose Token Exchange response parameter it incorporates ({{cnf-param}}). That parameter is the `cnf` token response parameter that {{RFC9201}} registers for the Authentication and Authorization for Constrained Environments (ACE) framework.
* It generalizes the sender-constraining rules that the Identity Assertion JWT Authorization Grant {{ID-JAG}} defines for its grant. {{profile-examples}} describes that grant as a profile of this specification and lists the conflicts between the two documents.

## Conventions and Terminology

{::boilerplate bcp14-tagged}

This specification uses the terms "access token", "authorization grant", "authorization server", "client", "refresh token", and "token endpoint" defined by {{RFC6749}}, the terms "JSON Web Token (JWT)" and "claim" defined by {{RFC7519}}, and the term "DPoP proof" defined by {{RFC9449}}. It also defines the following terms:

JWT authorization grant:
: A JWT used as an authorization grant as defined in {{Section 2.1 of RFC7523}}.

Issuing authorization server:
: The authorization server that issues a JWT authorization grant, such as the IdP Authorization Server of {{ID-JAG}}.

Resource authorization server:
: The authorization server to which a client presents a JWT authorization grant in an access token request ({{Section 4.1 of RFC7521}}), typically the authorization server for the resources that the client wants to access, such as the Resource Authorization Server of {{ID-JAG}}.

Bound grant:
: A JWT authorization grant that contains a `cnf` claim ({{format}}). A DPoP-bound grant has a `jkt` or `jwk` confirmation member; a certificate-bound grant has an `x5t#S256` confirmation member.

Confirmation member:
: A member of a `cnf` claim or of the `cnf` response parameter, such as `jkt` or `x5t#S256`.

Binding request:
: A Token Exchange request that contains a DPoP proof ({{te-request}}).

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
{: title="Issuing and Redeeming a DPoP-Bound Grant"}

1. The client sends a Token Exchange request with a DPoP proof for its key K ({{te-request}}).
2. The issuing authorization server issues a grant whose `cnf` claim identifies K ({{bind}}) and returns the binding in the `cnf` response parameter ({{cnf-param}}), which the client checks ({{client-processing}}).
3. The client presents the grant to the resource authorization server with a DPoP proof for K ({{token-request}}).
4. The resource authorization server verifies the proof against the grant's `cnf` claim ({{processing}}) and issues an access token, normally bound to K ({{token-response}}).

The client sends a separate DPoP proof to each authorization server, signed with the same key, because the `htu` claim of a DPoP proof identifies the endpoint that receives it ({{Section 4.2 of RFC9449}}). With mutual TLS, the client's TLS client certificate takes the place of K: the client proves possession in the TLS handshake instead of with a DPoP proof.

# Bound JWT Authorization Grants {#format}

This section defines the `cnf` claim of a bound grant and how confirmation values are compared.

## Confirmation Claim {#cnf-claim}

An issuing authorization server binds a grant by including in it a `cnf` claim ({{Section 3.1 of RFC7800}}) that contains exactly one confirmation member ({{bind}}). The member identifies the key or certificate whose possession the client proves when it redeems the grant:

* `jkt` ({{Section 6.1 of RFC9449}}), in a DPoP-bound grant, contains the JWK SHA-256 Thumbprint {{RFC7638}} of the client's public key.
* `x5t#S256` ({{Section 3.1 of RFC8705}}), in a certificate-bound grant, contains the SHA-256 thumbprint of the DER encoding of the client's certificate.

An issuing authorization server uses only these members. A resource authorization server also accepts a `jwk` member ({{Section 3.2 of RFC7800}}) in a DPoP-bound grant, and rejects a `cnf` claim that does not contain exactly one supported confirmation member ({{processing}}). Requiring exactly one member is narrower than {{Section 3.1 of RFC7800}}, which requires only that a `cnf` claim "represent only a single proof-of-possession key".

The following are the claims of a DPoP-bound grant:

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
{: #ex-grant title="Claims of a DPoP-Bound Grant"}

A certificate-bound grant instead contains a `cnf` claim such as the following:

~~~ json
{
  "cnf": {
    "x5t#S256": "bwcK0esc3ACC3DB2Y5_lESsXE8o9ltc05O89jdN-dg0"
  }
}
~~~
{: title="Confirmation Claim of a Certificate-Bound Grant"}

## Comparing Confirmation Values {#cnf-comparison}

Two confirmation values, or a confirmation value and the client's own key or certificate, are compared by confirmation method and value, not by byte representation:

* JSON member order and insignificant whitespace are ignored.
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

# Issuing a Bound Grant {#issuance}

This section applies to a JWT authorization grant that an issuing authorization server issues in response to a Token Exchange request ({{Section 2.1 of RFC8693}}); {{cnf-param}} applies to any token that Token Exchange issues, and {{other-issuance}} covers other issuance interfaces. An issued token is a JWT authorization grant if it is a JWT whose `aud` claim identifies an authorization server. The issuing authorization server decides whether to issue one from the `requested_token_type`, `audience`, and `resource` parameters and its policy.

## Token Exchange Request {#te-request}

A client requests a DPoP-bound grant by including a DPoP proof ({{Section 4 of RFC9449}}) in the DPoP header field of a Token Exchange request, as it would to request a DPoP-bound access token ({{Section 5 of RFC9449}}). Such a request is a binding request. The client signs the DPoP proof with the key that it will use to redeem the grant ({{token-request}}).

A TLS client certificate is not a binding request: a client presents one for mutual-TLS client authentication ({{Section 2 of RFC8705}}) without necessarily intending to bind the grant. An issuing authorization server binds a grant to that certificate only as {{cert-binding}} describes.

If the `subject_token` or `actor_token` is bound to a key or certificate, the client proves possession of that key or certificate in the request, with a DPoP proof or over a mutual-TLS connection that uses the certificate, unless {{bound-subject}} exempts it.

No request parameter lets a client require a bound grant; a client that requires one checks the response ({{client-processing}}).

The following is an example of a binding request from a client that authenticates with a client secret, for a grant for the resource authorization server `https://as.example/`:

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

An issuing authorization server that supports this specification implements {{cnf-param}} and at least one of the `jkt` and `x5t#S256` confirmation methods, and binds a grant only as described in this section. When it rejects a request for a JWT authorization grant, the issuing authorization server MUST NOT treat the `subject_token` or `actor_token` as used, for example by rotating a refresh token ({{single-use}}).

### DPoP Binding {#binding-request}

The issuing authorization server MUST validate the DPoP proof of a binding request as described in {{Section 4.3 of RFC9449}}, and responds to an invalid proof or a missing nonce as described in {{errors}}.

In response to a binding request with a valid DPoP proof, the issuing authorization server MUST bind the issued grant to the public key of the DPoP proof ({{bind}}), unless it does not support the `jkt` confirmation method or cannot bind the grant for the resource authorization server ({{bind}}). In that case, it MUST issue an unbound grant, without the `cnf` response parameter, unless the profile of the grant requires binding; if the profile requires binding, it MUST reject the request with the `invalid_target` error code. It MUST NOT bind the grant to any other key or certificate, including when the client also presents a TLS client certificate.

A client that requires a bound grant discards an unbound one ({{client-processing}}). Issuing an unbound grant keeps working a client that sends a DPoP proof in every token request, and a client that proves possession of a DPoP-bound `subject_token` ({{bound-subject}}), when the resource authorization server does not verify bindings.

### Certificate Binding {#cert-binding}

If a request contains no DPoP proof and arrives over a mutual-TLS connection in which the client presented a TLS client certificate, an issuing authorization server that supports the `x5t#S256` confirmation method MAY bind the grant to that certificate ({{bind}}). It SHOULD NOT do so unless the profile of the grant requires certificate binding, or the client's registration or configuration at the issuing authorization server indicates that the client presents that certificate when it redeems the grant. This is because a client that does not support this specification would receive a grant that it cannot redeem. The `tls_client_certificate_bound_access_tokens` client metadata parameter ({{Section 3.4 of RFC8705}}) concerns access tokens and is not such an indication.

The following is an example of a Token Exchange response for a certificate-bound grant, issued to a client whose configuration at the issuing authorization server indicates that it redeems grants over mutual TLS with the certificate it used to authenticate:

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
{: #cert-example title="Token Exchange Response for a Certificate-Bound Grant"}

### Bound Subject and Actor Tokens {#bound-subject}

If the `subject_token` or `actor_token` is bound to a key or certificate, such as a refresh token issued to a public client ({{Section 5 of RFC9449}}), the issuing authorization server MUST reject the request with the `invalid_request` error code ({{Section 2.2.2 of RFC8693}}) unless one of the following holds:

* The request proves possession of that key or certificate, with a DPoP proof signed with the key or a mutual-TLS handshake using the certificate.
* The requesting client authenticates and is an intended audience of the token, other than the client to which the token was issued, such as a resource server that exchanges an access token it received ({{Section 2.3 of RFC8693}}).

The issuing authorization server determines whether the requesting client is an intended audience of the token from the token's `aud` claim or its own records of the token, for example by associating a resource server's client registration with the resource identifiers that the `aud` claim contains. A grant issued in response to a binding request that proves possession of a DPoP-bound `subject_token` is therefore bound to the same key, or unbound ({{binding-request}}).

### Binding the Grant {#bind}

To bind a grant, the issuing authorization server includes in it a `cnf` claim as described in {{format}}, whose confirmation member identifies the client's key or certificate. For a binding request, that key is the public key in the `jwk` JOSE Header Parameter of the DPoP proof ({{binding-request}}); for a Token Exchange request bound to a certificate, that certificate is the TLS client certificate ({{cert-binding}}). The issuing authorization server MUST NOT bind a grant to a key or certificate whose possession the client has not proven in the request for the grant.

The issuing authorization server MUST NOT bind a grant using a confirmation method unless it has determined that each authorization server that the grant's `aud` claim identifies verifies that confirmation method as described in {{processing}} (see {{non-enforcing-consumers}}). It makes this determination from configuration, from an established trust relationship, or from the resource authorization server's metadata ({{as-metadata}}). {{binding-request}} defines the response to a binding request when the issuing authorization server cannot bind the grant.

When the issued token is a bound grant, the issuing authorization server MUST set the `token_type` response parameter to `N_A` ({{Section 2.2.1 of RFC8693}}); the `DPoP` value that {{Section 5 of RFC9449}} requires applies only to a DPoP-bound access token.

{{ex-grant}} shows the claims of the grant issued in response to the binding request in {{ex-binding-request}}. The following is an example of the `invalid_target` error response to a binding request for a grant whose profile requires binding:

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

## Token Exchange Response {#cnf-param}

This specification defines the use of the `cnf` token response parameter, registered by {{RFC9201}}, in a Token Exchange response ({{Section 2.2.1 of RFC8693}}):

`cnf`:
: REQUIRED if the authorization server bound the token in the `access_token` response parameter to a key or certificate; otherwise, it MUST NOT be present. A JSON object containing exactly one confirmation member, which identifies that key or certificate. Its syntax and member semantics are those of the `cnf` claim ({{Section 3.1 of RFC7800}}).

The parameter describes only the token in the `access_token` response parameter, whatever its `issued_token_type`, including an opaque or encrypted token and an access token whose `token_type` also indicates the binding. It does not describe a `refresh_token` in the same response.

When the issued token is a JWT that contains a `cnf` claim, the parameter MUST identify the same key or certificate as that claim ({{cnf-comparison}}). The authorization server SHOULD use the same confirmation member in the parameter as in the claim, and SHOULD use a thumbprint member, such as `jkt` or `x5t#S256`, rather than a `jwk` member ({{privacy}}).

The following is an example of the Token Exchange response to the binding request in {{ex-binding-request}}:

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

When the `token_type` response parameter is `N_A`, as it is for a JWT authorization grant ({{bind}}), a client that sent a binding request, or that presented a TLS client certificate and requires a bound grant, processes the Token Exchange response as follows:

1. If the `cnf` parameter is present and is not a JSON object that contains exactly one confirmation member, the client MUST discard the issued token.
2. The client determines the binding from the `cnf` parameter and, if it can read the issued token, from the token's `cnf` claim. If neither is present, the grant is not bound, and a client that requires a bound grant MUST discard it.
3. The client MUST verify the following, and MUST discard the issued token if any check fails:
   * if the `cnf` parameter is present and the client can read the grant, the grant contains a `cnf` claim;
   * the `cnf` claim, if readable, contains exactly one confirmation member; and
   * the confirmation member of the `cnf` parameter, if present, and of the `cnf` claim, if readable, each match the client's own key or certificate ({{cnf-comparison}}): after a binding request, the public key of its DPoP proof; otherwise, its TLS client certificate. A confirmation member that the client does not recognize does not match.

For an access token, a `token_type` of `DPoP` indicates a DPoP binding ({{Section 5 of RFC9449}}). {{RFC8705}} defines no `token_type` value, so the `token_type` does not show a certificate binding; the `cnf` parameter, if present, identifies the key or certificate of either binding ({{cnf-comparison}}).

A client that did not send a binding request but receives a `cnf` parameter presents the token with the corresponding proof if it holds the identified key or certificate, such as the certificate it used for mutual-TLS client authentication. Otherwise, it SHOULD treat the response as an error.

For the response in {{ex-te-response}}, the client verifies that the `jkt` value in the `cnf` response parameter equals the JWK SHA-256 Thumbprint of its DPoP public key.

## Other Issuance Interfaces {#other-issuance}

When a JWT authorization grant is issued through an interface other than Token Exchange, such as a workload platform's token service, {{bind}} applies except for its requirements on the Token Exchange response, with that interface's request in place of the Token Exchange request. The interface defines how the client requests the binding and learns of it. {{redemption}} applies however the grant was issued.

# Redeeming a Bound Grant {#redemption}

## Access Token Request {#token-request}

The client presents a bound grant in an access token request ({{Section 4.1 of RFC7521}}) with the `jwt-pop` grant type ({{jwt-pop}}), the `jwt-bearer` grant type, or, where {{legacy-jwt-dpop}} applies, the `jwt-dpop` grant type. It SHOULD use the `jwt-pop` grant type if the resource authorization server advertises it ({{as-metadata}}) and the client's registration permits it ({{client-metadata}}). If the resource authorization server advertises the `jwt-dpop` grant type and not the `jwt-pop` grant type, and the client's registration permits it, a client that presents a grant with a `jkt` member SHOULD use the `jwt-dpop` grant type, because a server that implements only {{JWT-DPOP-GRANT}} rejects such a grant presented with the `jwt-bearer` grant type. The client proves possession of the key or certificate identified by the grant's `cnf` claim:

* For a `jkt` or `jwk` member, the client MUST include a DPoP proof ({{Section 4 of RFC9449}}) signed with that key in the DPoP header field of the request.
* For an `x5t#S256` member, the client MUST send the request over a mutual-TLS connection that uses that certificate ({{Section 3 of RFC8705}}), to the token endpoint or to its mutual-TLS endpoint alias ({{Section 5 of RFC8705}}).

### Proof-of-Possession Grant Type {#jwt-pop}

To use the `jwt-pop` grant type, the client sends the following parameters and proves possession of the grant's key or certificate as described in {{token-request}}:

`grant_type`:
: REQUIRED. The value `urn:ietf:params:oauth:grant-type:jwt-pop`.

`assertion`:
: REQUIRED. A single JWT authorization grant that contains a `cnf` claim.

`scope`:
: OPTIONAL. The requested scope, as defined in {{Section 4.1 of RFC7521}}.

The resource authorization server processes the request as defined in {{processing}}, and MUST also reject it with the `invalid_grant` error code if the grant has no `cnf` claim.

The following is an example of an access token request that presents the grant in {{ex-grant}} with the `jwt-pop` grant type, from a client that does not authenticate:

~~~ http-message
POST /token HTTP/1.1
Host: as.example
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6...

grant_type=urn:ietf:params:oauth:grant-type:jwt-pop
&assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0...
~~~
{: #ex-redeem title="Access Token Request with a DPoP-Bound Grant"}

### Legacy DPoP-Bound Grant Type {#legacy-jwt-dpop}

The `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type, first defined in {{JWT-DPOP-GRANT}}, is a legacy grant type for existing clients. A resource authorization server MAY support it, and SHOULD continue to support it while clients are registered for it. A resource authorization server that supports it SHOULD include it in the `grant_types_supported` metadata parameter, on which an issuing authorization server that implements {{JWT-DPOP-GRANT}} can rely to determine that the resource authorization server verifies DPoP-bound grants. It processes a request with that grant type by the rules of this specification for the `jwt-pop` grant type ({{jwt-pop}}), and MUST also reject the request with the `invalid_grant` error code if the confirmation member of the grant's `cnf` claim is not `jkt`. These rules differ from {{JWT-DPOP-GRANT}} as {{relationship}} describes. A new client uses the `jwt-pop` grant type.

## Resource Authorization Server Processing {#processing}

A resource authorization server validates a JWT authorization grant as defined in {{Section 3 of RFC7523}}, as updated by {{Section 4 of RFC10070}}, and by the profile of the grant. If the grant contains a `cnf` claim, a resource authorization server that supports this specification MUST also ensure the following, whichever grant type the request uses, and MUST otherwise reject the request with the indicated error code ({{errors}}):

1. For the `cnf` claim:
   * it is a JSON object that contains exactly one confirmation member; otherwise, `invalid_grant`;
   * that member is `jkt`, `jwk` ({{Section 3.2 of RFC7800}}), or `x5t#S256`; otherwise, `invalid_grant`; and
   * the resource authorization server supports its confirmation method; otherwise, `invalid_grant`.
2. For a `jkt` or `jwk` member:
   * the request contains a DPoP proof; otherwise, `invalid_grant`;
   * the DPoP proof is valid as described in {{Section 4.3 of RFC9449}}; otherwise, `invalid_dpop_proof`, or `use_dpop_nonce` as described in {{Section 8 of RFC9449}}; and
   * the JWK SHA-256 Thumbprint {{RFC7638}} of the public key in the `jwk` JOSE Header Parameter of the DPoP proof equals the `jkt` value, or the JWK SHA-256 Thumbprint of the key in the `jwk` member ({{cnf-comparison}}); otherwise, `invalid_grant`.
3. For an `x5t#S256` member:
   * the request was received on a TLS connection in whose handshake the client presented a certificate and proved possession of its private key; otherwise, `invalid_grant`; and
   * the SHA-256 thumbprint of the DER encoding of that certificate equals the `x5t#S256` value ({{Section 3.1 of RFC8705}}); otherwise, `invalid_grant`.

Item 1 overrides the rule in {{Section 3.1 of RFC7800}} that implementations ignore confirmation members they do not understand. Client authentication does not take the place of any of these checks.

A resource authorization server that supports DPoP validates a DPoP proof in the request, including its nonce, before it validates the grant, and rejects an invalid proof as {{Section 5 of RFC9449}} requires, whatever the binding of the grant. A request for a DPoP-bound grant that carries no DPoP proof is rejected with the `invalid_grant` error code (item 2), not `use_dpop_nonce`, unless {{client-auth}} requires the `invalid_client` error code.

For an `x5t#S256` member, the resource authorization server need not authenticate the client with the certificate or trust the certificate's issuer ({{Section 4 of RFC8705}}). How the certificate reaches a resource authorization server behind a TLS-terminating proxy is outside the scope of this specification, as it is in {{Section 6.5 of RFC8705}}.

If the profile of the grant requires binding ({{profiles}}) and the grant has no `cnf` claim, the resource authorization server MUST reject the request with the `invalid_grant` error code.

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

A resource authorization server that requires a DPoP nonce ({{Section 8 of RFC9449}}) responds to the request in {{ex-redeem}} as follows, without recording the grant as used. The client then repeats the request with the same grant and a DPoP proof that contains the nonce.

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

## Access Token Response {#token-response}

{{Section 5 of RFC9449}} and {{Section 3 of RFC8705}} determine the binding of the tokens that the resource authorization server issues, except as this section and {{refresh}} restrict it for a bound grant.

When the grant is bound, the resource authorization server SHOULD bind the access token it issues, and indicate the binding as defined in {{Section 5 of RFC9449}} or {{Section 3 of RFC8705}}. For a bound grant, it MUST NOT bind the access token, or a refresh token that it issues, to a key or certificate other than the following:

* for a DPoP-bound grant, the grant's key, which is the key of the DPoP proof ({{processing}}); or
* for a certificate-bound grant, the grant's certificate or, if the resource authorization server supports DPoP and the request contains a valid DPoP proof ({{Section 4.3 of RFC9449}}), the public key of that proof.

A TLS client certificate presented with a DPoP-bound grant therefore does not determine the binding, even when the client authenticates with it ({{proof-replay}}). {{binding-outcomes}} summarizes these rules; the access token can instead be unbound.

| Grant's confirmation member | Proof in the request | Binding of issued tokens |
|---|---|---|
| `jkt` or `jwk` | DPoP proof, with or without a TLS client certificate | Grant's key |
| `x5t#S256` | TLS client certificate only | Grant's certificate |
| `x5t#S256` | TLS client certificate and DPoP proof | Grant's certificate or key of the DPoP proof |
{: #binding-outcomes title="Binding of Tokens Issued for a Bound Grant"}

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

If the resource authorization server issues a refresh token for a bound grant, it MUST bind the refresh token as described in {{token-response}}, whether or not the client authenticates, with one exception: if the client authenticates and the resource authorization server verifies that the grant identifies the authenticated client, in a claim that the profile of the grant defines ({{profiles}}), it MAY instead bind the refresh token only to that client. The resource authorization server decides this exception when it redeems the grant, never at a later refresh: the replacement for a refresh token bound only to the client is bound only to that client, and the replacement for a refresh token bound to a key or certificate is bound as described below. A refresh token's association with the authenticated client ({{Section 6 of RFC6749}}) never qualifies it for the exception. A binding to a key or certificate is in addition to the binding to the client that {{Section 10.4 of RFC6749}} requires. It is stricter than {{Section 5 of RFC9449}}, under which a refresh token issued to a confidential client is not bound to the DPoP key, and than the recommendation in {{Section 4 of RFC8705}}; {{proof-replay}} gives the reason.

When a client presents a refresh token that the resource authorization server bound to a key or certificate under this section, the resource authorization server treats that key or certificate as the grant's. It MUST verify the binding as items 2 and 3 of {{processing}} do, with the same error codes, and MUST bind the tokens that it issues in response as {{token-response}} and this section require. The binding of the presented refresh token does not change, even when the issued tokens are bound to the key of a DPoP proof ({{binding-outcomes}}).

The client presents a refresh token issued for a bound grant with each proof of possession that it presented in the request in which it obtained that refresh token: a DPoP proof for the same key, a mutual-TLS connection that uses the same certificate, or both. The response does not always show which binding was applied; for example, a resource authorization server can bind to the certificate although the request contained a DPoP proof, and return a `token_type` of `Bearer` ({{binding-outcomes}}).

The following is an example of a refresh request for a refresh token bound to the certificate in {{cert-example}}. The client sends it over a mutual-TLS connection that uses that certificate, with which it also authenticates ({{Section 2 of RFC8705}}), and adds a DPoP proof:

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

The client presents the replacement refresh token with both proofs, because it obtained it with both; the resource authorization server verifies the DPoP proof.

## Bound Grant in Place of Client Authentication {#client-auth}

Client authentication is optional for a JWT authorization grant ({{Section 4.1 of RFC7521}}, {{Section 3.1 of RFC7523}}). A profile ({{profiles}}) can allow a resource authorization server to accept a bound grant from a client that does not authenticate, with proof of possession of the grant's key or certificate taking the place of client authentication.

Such a profile specifies the items listed in item 4 of {{profiles}}. The resource authorization server then relies on the issuing authorization server's authentication of the client ({{pop-identity}}).

Where a profile allows a bound grant in place of client authentication, a resource authorization server MUST reject a request that has neither client authentication nor a bound grant with the `invalid_client` error code ({{Section 5.2 of RFC6749}}). For this purpose, a grant that fails validation ({{processing}}) is not a bound grant, and this error code takes precedence over the `invalid_grant` error code.

# Error Responses {#errors}

An authorization server returns errors as defined in {{Section 5.2 of RFC6749}}, with the HTTP status code 400 (Bad Request) except where that section permits 401 (Unauthorized) for `invalid_client`. This specification defines no new error codes. It uses the following error codes, each at the check that names it:

`invalid_dpop_proof`:
: At either authorization server, the DPoP proof is invalid ({{Section 5 of RFC9449}}).

`use_dpop_nonce`:
: At either authorization server, the authorization server requires a nonce in the DPoP proof, or the nonce is missing or not current ({{Section 8 of RFC9449}}). The authorization server MUST use this error code, and not `invalid_grant` or `invalid_request`, for these conditions, including when the DPoP proof is needed to verify a bound grant or a bound `subject_token`.

`invalid_request`:
: At the issuing authorization server, the `subject_token` or `actor_token` is bound, and the request meets neither condition listed in {{bound-subject}}.

`invalid_target`:
: At the issuing authorization server, the profile of the grant requires binding and a binding request cannot be honored for the resource authorization server ({{binding-request}}).

`invalid_grant`:
: At the resource authorization server, a check in {{processing}}, {{jwt-pop}}, {{legacy-jwt-dpop}}, or {{refresh}} that names `invalid_grant` fails, or the grant lacks the `typ` value that its profile defines ({{cross-jwt}}).

`invalid_client`:
: At the resource authorization server, the request has neither client authentication nor a bound grant, where a profile allows a bound grant in place of client authentication ({{client-auth}}).

`unauthorized_client`:
: At the resource authorization server, the client is not registered for the grant type of the request ({{client-metadata}}).

# Metadata {#metadata}

## Authorization Server Metadata {#as-metadata}

This specification defines the following authorization server metadata parameter ({{Section 2 of RFC8414}}):

`jwt_grant_cnf_methods_supported`:
: RECOMMENDED for a resource authorization server that supports this specification; REQUIRED if its `grant_types_supported` parameter contains the `jwt-pop` grant type. A nonempty JSON array of Confirmation Method Values from the "JWT Confirmation Methods" registry ({{Section 6.2 of RFC7800}}) that lists each confirmation method, among `jkt`, `jwk`, and `x5t#S256`, whose binding the authorization server verifies in a JWT authorization grant ({{processing}}), whatever the grant type of the request or the profile of the grant. Listing one member does not indicate support for another; in particular, `jkt` does not indicate `jwk`.

A resource authorization server that supports the `jwt-pop` grant type SHOULD include it in the `grant_types_supported` metadata parameter. The `jwt-pop` value alone indicates no confirmation method.

An issuing authorization server MAY rely on a confirmation member listed in `jwt_grant_cnf_methods_supported` to determine that the resource authorization server verifies that confirmation method ({{bind}}), in metadata that it retrieved and validated as described in {{Section 3 of RFC8414}} for each `aud` value of the grant, using one of the following issuer identifiers:

* that `aud` value; or
* an issuer identifier that the issuing authorization server's configuration associates with that `aud` value, for example when it is a token endpoint URL ({{Section 4 of RFC10070}}), provided that the `token_endpoint` value of the metadata equals the `aud` value. That equality alone does not establish the association, because only the `issuer` value of metadata is validated ({{Section 3.3 of RFC8414}}, {{Section 1 of RFC10070}}).

Metadata that does not list a confirmation member is no evidence that the resource authorization server verifies it; configuration or an established trust relationship can still provide that evidence ({{bind}}). In metadata obtained in the same way, an issuing authorization server MAY also treat the `jwt-dpop` value of `grant_types_supported` as evidence that the resource authorization server verifies the `jkt` confirmation method ({{legacy-jwt-dpop}}).

The `dpop_signing_alg_values_supported` ({{Section 5.1 of RFC9449}}) and `tls_client_certificate_bound_access_tokens` ({{Section 3.3 of RFC8705}}) metadata parameters do not indicate that an authorization server verifies bound grants.

The following is an example of metadata on which an issuing authorization server can rely to bind a grant whose `aud` value is `https://as.example/` with the `jkt` or `x5t#S256` confirmation method:

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

The `jwt-pop` grant type requires a bound grant in each request, whether or not the resource authorization server identifies the client ({{jwt-pop}}). Client registration can additionally restrict an identified client to that grant type. A resource authorization server can require a client to present bound grants by registering the client with a `grant_types` client metadata value ({{Section 2 of RFC7591}}) that includes the `jwt-pop` grant type and not the `jwt-bearer` grant type. The resource authorization server then rejects a request from that client that uses the `jwt-bearer` grant type with the `unauthorized_client` error code ({{Section 5.2 of RFC6749}}), and {{jwt-pop}} and {{legacy-jwt-dpop}} reject an unbound grant. This applies only to a request in which the resource authorization server identifies the client: by client authentication or, when a bound grant takes the place of client authentication ({{client-auth}}), from the grant as the profile defines. A request that does not identify a client is not limited by any client's registration.

A resource authorization server can instead reject unbound grants by policy while accepting the `jwt-bearer` grant type, but `grant_types` then does not express that the client is limited to bound grants.

Support for both the `jwt-pop` and `jwt-dpop` grant types does not permit every client to use both: the resource authorization server rejects a request whose grant type the client's registration does not permit with the `unauthorized_client` error code. A new client is registered for the `jwt-pop` grant type; an existing client keeps the `jwt-dpop` grant type only where its registration permits it.

The `jwt-pop` grant type has no corresponding `response_types` value ({{Section 2.1 of RFC7591}}).

The `dpop_bound_access_tokens` client metadata parameter ({{Section 5.2 of RFC9449}}) requires a DPoP proof in each token request, but does not require the grant to be bound.

The following is an example of client metadata for a client that the resource authorization server limits to bound grants:

~~~ json
{
  "grant_types": [
    "urn:ietf:params:oauth:grant-type:jwt-pop",
    "refresh_token"
  ],
  "token_endpoint_auth_method": "client_secret_basic"
}
~~~
{: title="Client Metadata Limiting a Client to Bound Grants"}

# Profiling This Specification {#profiles}

A specification that defines a JWT authorization grant, such as {{ID-JAG}}, uses this specification by stating:

1. The value of the grant's `typ` JOSE Header Parameter ({{Section 3.11 of RFC8725}}) and, if the grant is issued through Token Exchange, its `requested_token_type` value.
2. Which confirmation methods an issuing authorization server may apply, which a resource authorization server must verify, which are mandatory to implement, and whether the `jwt-pop` grant type is mandatory to implement.
3. Whether binding is optional, recommended, or required, and whether that varies by resource authorization server.
4. The claim, if any, in which the issuing authorization server identifies the client ({{refresh}}), and whether a resource authorization server may accept a bound grant in place of client authentication ({{client-auth}}). If it may: how the identity of the client is tied to the key or certificate; what the issuing authorization server verifies before it binds the grant; and how the resource authorization server maps that identity to its own client registration and policy.
5. Any error codes or processing beyond those in {{errors}} and {{processing}}, such as an error response for a request that carries no proof of possession when the requested resource requires a sender-constrained access token.

A profile that permits a bound grant in place of client authentication MUST require the issuing authorization server to authenticate the client in the same request in which it verifies proof of possession of the key or certificate.

When a profile does not address an item above, the following defaults apply:

* Binding is optional, and either confirmation method may be used.
* A resource authorization server verifies any `cnf` claim it receives ({{processing}}).
* A bound grant does not take the place of client authentication. Whether the client authenticates is the resource authorization server's policy ({{Section 3.1 of RFC7523}}), and {{client-auth}} does not apply.

# Security Considerations {#security}

The security considerations of {{RFC7521}}, {{RFC7523}}, {{RFC8693}}, {{RFC9449}}, and {{RFC8705}} apply to this specification.

## Non-Enforcing Resource Authorization Servers {#non-enforcing-consumers}

{{Section 3.1 of RFC7800}} requires implementations to ignore confirmation members they do not understand, and {{RFC7523}} does not process the `cnf` claim. A resource authorization server that does not support this specification therefore accepts a bound grant as a bearer grant, and a party that obtains the grant can redeem it without the key. {{bind}} therefore forbids an issuing authorization server to bind a grant for a resource authorization server that it has not determined verifies the binding.

## Grant Type Substitution {#grant-type-substitution}

A separate grant type for bound grants does not by itself prevent substitution: a party holding a stolen bound grant can present it with the `jwt-bearer` grant type. This specification therefore requires a resource authorization server that supports this specification to verify the binding of every JWT authorization grant that contains a `cnf` claim, whichever grant type the request uses ({{processing}}).

The `jwt-pop` grant type adds no protection against a stolen grant. It requires a bound grant in each request, and a resource authorization server that does not support it rejects the request with the `unsupported_grant_type` error code ({{Section 5.2 of RFC6749}}), whereas a server ignores client metadata that it does not understand ({{Section 2 of RFC7591}}). It also lets a resource authorization server limit an identified client to bound grants through its registration ({{client-metadata}}); the confirmation methods that the server verifies are advertised separately ({{as-metadata}}). Verifying a bound grant presented with the `jwt-bearer` grant type, instead of rejecting it as {{JWT-DPOP-GRANT}} does, keeps clients registered only for that grant type working.

A resource authorization server that adopts this specification rejects a grant that it previously accepted if the grant's `cnf` claim contains more than one confirmation member, or a member other than `jkt`, `jwk`, and `x5t#S256`.

## Cross-JWT Confusion {#cross-jwt}

A bound grant and a DPoP-bound JWT access token for the same client can carry the same `jkt` value. A party that receives one could present it where the other is expected. The `aud` claim of a grant identifies an authorization server ({{RFC10070}}), and an explicit `typ` value ({{Section 3.11 of RFC8725}}) lets a recipient reject a JWT of the wrong kind ({{Section 3.12 of RFC8725}}). When the profile of a grant defines a `typ` value, a resource authorization server SHOULD reject a grant that does not carry it with the `invalid_grant` error code.

## DPoP Proof Replay {#proof-replay}

The DPoP proof replay protections of {{Section 11.1 of RFC9449}} and the authorization server-provided nonce of {{Section 8 of RFC9449}} apply at the token endpoints of both the issuing and the resource authorization server. A DPoP proof sent to one cannot be replayed at the other, because the `htu` claim identifies the token endpoint ({{Section 4.3 of RFC9449}}).

A DPoP proof does not cover the grant it accompanies or the TLS connection that carries it. An attacker that captures a grant together with a DPoP proof for its key can replay both, over its own TLS connection, within the window in which the resource authorization server accepts the proof ({{Section 11.1 of RFC9449}}). What the attacker obtains depends on the binding of the issued tokens:

* An access token or refresh token bound to the grant's key is of no use to the attacker.
* An access token bound to a TLS client certificate that the attacker presents would be usable; {{token-response}} forbids that binding. The reverse case is safe: for an `x5t#S256`-bound grant, the mutual-TLS handshake shows that a DPoP proof comes from the holder of the grant's certificate, so {{token-response}} lets the issued tokens be bound to the key of that proof.
* A refresh token bound only to a client as which the attacker authenticates would be usable, at the first refresh or a later one; {{refresh}} therefore binds every refresh token issued for a bound grant, and every replacement, to its key or certificate, unless the resource authorization server verifies that the grant identifies the authenticated client, which an attacker that authenticates as its own client cannot satisfy.
* An unbound access token is usable.

A resource authorization server that issues unbound access tokens for DPoP-bound grants SHOULD reject a DPoP proof whose `jti` value it has already accepted at its token endpoint within that window ({{Section 11.1 of RFC9449}}). A server-provided nonce ({{Section 8 of RFC9449}}) shortens the window but does not by itself detect reuse, because a client can use the same nonce more than once. Neither measure denies an unbound access token to an attacker whose request arrives before the client's; only binding the access token does.

## Replay by the Key Holder

Binding limits who can redeem a grant, not how often. A client, or an attacker that has obtained the client's key, can redeem a bound grant until it expires unless the resource authorization server limits it to a single use. Short grant lifetimes and single-use enforcement ({{Section 5.5 of ID-CHAINING}}) remain applicable to bound grants.

## Exhaustion of Single-Use Grants {#single-use}

A resource authorization server that limits a grant to a single use, as {{Section 5.5 of ID-CHAINING}} suggests, could mark a bound grant as used when a party without the key presents it, denying the legitimate client its redemption. It could also mark the grant as used when it rejects a request with the `use_dpop_nonce` error code, so that the client's retry with the nonce fails. {{processing}} prevents both by recording a grant as used only when the request succeeds. {{issuance}} applies the same rule to a single-use `subject_token` at the issuing authorization server.

## Proof of Possession and Client Identity {#pop-identity}

Proof of possession shows that the presenter holds the key or certificate, not which client the presenter is. Even at the issuing authorization server, a DPoP proof is not cryptographically bound to the client authentication that accompanies it ({{Section 11.11 of RFC9449}}). A resource authorization server that accepts a bound grant in place of client authentication ({{client-auth}}) therefore depends on the issuing authorization server's authentication of the client and on the client's protection of its private key: any party that obtains the key can use a grant bound to it.

## Theft of the Subject Token

Binding the issued grant does not protect a bearer `subject_token` that is stolen before the exchange. An attacker that holds such a subject token, and can satisfy any client authentication that the issuing authorization server requires, can request a grant bound to the attacker's own key. A bound `subject_token` is protected by the check in {{bound-subject}}, except from an authenticated intended audience that the check exempts. When the issuing authorization server cannot bind the grant for the resource authorization server, a client that proves possession of a DPoP-bound `subject_token` receives an unbound grant ({{binding-request}}), which is a bearer token, as it would be without this specification. Protection of bearer subject tokens is outside the scope of this specification; see {{RFC9700}}.

## Informational Nature of the Response Parameter

The `cnf` response parameter is informational. The binding is enforced when the issued token is presented to its consumer, which verifies the token's `cnf` claim against the presenter's proof. A verified `cnf` parameter tells the client only that the authorization server applied the binding; it does not tell the client that the consumer enforces it ({{non-enforcing-consumers}}). The check protects against an authorization server that fails to apply a binding through error or misconfiguration, not against a malicious one, which controls the contents of the token.

The token endpoint requires TLS ({{Section 3.2 of RFC6749}}), which prevents an on-path attacker from removing or modifying the `cnf` parameter.

# Privacy Considerations {#privacy}

The privacy considerations of {{RFC9449}} and {{RFC8705}} apply to this specification.

A confirmation member is a stable identifier for the bound key or certificate. A client that uses the same key or certificate with several resource authorization servers lets them correlate its requests. A client that needs to avoid such correlation uses a separate key for each resource authorization server. This is not possible when the client exchanges a DPoP-bound `subject_token`, because a bound grant is then bound to that token's key ({{bound-subject}}).

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

This appendix is informative. It shows how two specifications answer the items in {{profiles}}.

## Identity Assertion JWT Authorization Grant

{{ID-JAG}} would answer the items as follows:

1. The `typ` value is `oauth-id-jag+jwt`, and the `requested_token_type` value is `urn:ietf:params:oauth:token-type:id-jag`.
2. Both confirmation methods, `jkt` and `x5t#S256`, may be used. Neither method, nor the `jwt-pop` grant type, is mandatory to implement, and a Resource Authorization Server verifies each method that it lists in `jwt_grant_cnf_methods_supported` ({{as-metadata}}). A DPoP-bound grant is presented with the `jwt-dpop` grant type ({{Section 4.4 of ID-JAG}}), which this specification keeps for existing clients ({{legacy-jwt-dpop}}).
3. Binding is optional.
4. A Resource Authorization Server may accept a bound grant in place of client authentication. The IdP Authorization Server authenticates the client in the Token Exchange request, binds the grant to the key or certificate for which that request carries proof of possession, and identifies the client in the grant's `client_id` claim.
5. A Resource Authorization Server also validates a certificate presented for an `x5t#S256`-bound grant as described in {{Section 2 of RFC8705}} ({{Section 9.11.2.2 of ID-JAG}}). It rejects an unbound grant with the `invalid_grant` error code when the resource requires sender-constrained access tokens and the request carries neither a DPoP proof nor a TLS client certificate ({{Section 9.11.1.2.4 of ID-JAG}}, {{Section 9.11.2.2 of ID-JAG}}).

A client that detects a downgrade from the grant's `cnf` claim ({{Section 9.11.1.1 of ID-JAG}}, {{Section 9.11.2.1 of ID-JAG}}) follows {{client-processing}}, which also accepts the `cnf` response parameter.

Version -06 of that document conflicts with this specification in three places. Each would be resolved by a change to that document:

* {{Section 9.11.1.1 of ID-JAG}} requires the IdP Authorization Server to include a `cnf` claim whenever the DPoP proof is valid. This specification issues an unbound grant when the issuing authorization server has not determined that the resource authorization server verifies the binding ({{binding-request}}, {{non-enforcing-consumers}}). The change is to bind the grant only for a Resource Authorization Server known to verify the binding.
* {{Section 9.11 of ID-JAG}} lets the IdP Authorization Server choose a certificate binding, according to local policy, when a request carries both a DPoP proof and a TLS client certificate. This specification binds a grant issued in response to a binding request only to the key of the DPoP proof, if it binds it at all ({{binding-request}}). The change is for the IdP Authorization Server never to choose a certificate binding when the request contains a DPoP proof.
* {{Section 8 of ID-JAG}} requires a client that supports that profile to include `urn:ietf:params:oauth:grant-type:jwt-bearer` in its `grant_types`, which prevents a Resource Authorization Server from limiting the client to bound grants ({{client-metadata}}), and {{Section 4.4 of ID-JAG}} presents a DPoP-bound grant with the `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type. The change is to present a bound grant of either method with the `urn:ietf:params:oauth:grant-type:jwt-pop` grant type, and to require `urn:ietf:params:oauth:grant-type:jwt-bearer` or `urn:ietf:params:oauth:grant-type:jwt-pop`.

## Identity and Authorization Chaining

{{ID-CHAINING}} defines no profile of this specification, so the defaults in {{profiles}} apply. Binding the grant adds a mitigation to those listed in {{Section 5.5 of ID-CHAINING}}. {{Appendix B.3 of ID-CHAINING}} binds the access token to a key that the grant conveys in a `requested_cnf` claim. For a bound grant, {{token-response}} permits only the bindings in {{binding-outcomes}}, so a grant that carries both a `cnf` claim and a `requested_cnf` claim for a different key conflicts with this specification.

# Open Issues {#open-issues}

\[Editor's note: this appendix lists questions for the working group. It will be removed before publication.\]

1. Grant type or client metadata. A `jwt-pop` request has the same format as a `jwt-bearer` request, while {{Section 4.1 of RFC7521}} defines `grant_type` as "the format of the assertion". An alternative is to present every bound grant with the `jwt-bearer` grant type and to express "bound grants only" with a boolean client metadata parameter, as `dpop_bound_access_tokens` does ({{Section 5.2 of RFC9449}}). That would remove the `jwt-pop` grant type and the conflict with {{Section 8 of ID-JAG}}. It would also lose the per-request requirement ({{jwt-pop}}), and a client would need to confirm that the registration policy took effect: a resource authorization server ignores client metadata that it does not understand ({{Section 2 of RFC7591}}), but returns all registered metadata, which the client can check ({{Section 3.2.1 of RFC7591}}), whereas it rejects an unsupported grant type outright ({{Section 5.2 of RFC6749}}). Which should the working group prefer?
2. Error for a required binding. When the profile of a grant requires binding and the issuing authorization server cannot bind the grant for the resource authorization server, this draft returns the `invalid_target` error code ({{binding-request}}). {{Section 2.2.2 of RFC8693}} also uses that code for an unacceptable audience. Should this be `invalid_request` or a new error code instead?
3. Scope of the response parameter. {{RFC9201}} registers the `cnf` token response parameter for ACE. Should this document define its use for Token Exchange only, for every token response, or move it to a separate specification?
4. Client opt-in to certificate binding. Binding a grant to the TLS client certificate that the client presents at issuance is an issuance policy that the client opts into, not only a capability: a client can support certificate-bound grants and still reserve the certificate it presents to the issuing authorization server for that relationship. This draft leaves the opt-in to the issuing authorization server's configuration for the client ({{cert-binding}}). Should a client metadata parameter express it? The `authorization_grant_profiles_supported` parameter of {{ID-JAG}} could carry such a signal, at the cost of mixing grant profiles with proof capabilities in one list.
5. Alignment with {{ID-JAG}}, {{ID-CHAINING}}, and {{JWT-DPOP-GRANT}}. The conflicts listed in {{profile-examples}}, three with {{ID-JAG}} and one with {{Appendix B.3 of ID-CHAINING}}, need to be resolved, by the proposed changes to those documents or by changes to this one. For the last, should a profile be able to permit binding the access token to a key that the grant conveys in a `requested_cnf` claim? The replacement of the `jwt-dpop` grant type and the compatibility rules in {{legacy-jwt-dpop}} need the agreement of the authors of {{JWT-DPOP-GRANT}}.
6. Unbound grant after a binding request. When a valid binding request cannot be honored, {{binding-request}} requires an unbound grant unless the profile requires binding. A client that requires binding discards it ({{client-processing}}), but the successful exchange may already have consumed a single-use `subject_token`, for example under an issuer's refresh-token rotation policy. The protection in {{issuer-processing}} applies only when the issuer rejects the request. Should a client be able to request "bind or fail", or should the issuer be allowed to reject instead of issuing an unbound grant? How would either option preserve compatibility with clients that send DPoP proofs for other reasons?
7. Refresh-token binding for confidential clients. {{refresh}} binds a refresh token issued for a bound grant to a key or certificate even when the client authenticates, except when the grant identifies that authenticated client. This departs from {{Section 5 of RFC9449}}, which retains client binding for confidential clients so that authentication credentials can rotate without invalidating refresh tokens. The additional binding requires the client to retain the bound key or certificate for refresh, complicating rotation and recovery. Is the current exception sufficient, or should the binding requirement be a profile choice? How would an alternative prevent replay of a captured grant and DPoP proof by an attacker authenticating as a different client, including at later refreshes ({{proof-replay}})?
8. Strict confirmation-member rules. {{processing}} rejects a `cnf` claim unless it contains exactly one member, chosen from `jkt`, `jwk`, and `x5t#S256`. {{Section 3.1 of RFC7800}} permits other confirmation members and, in the absence of application-specific requirements, ignores members that an implementation does not understand. As {{grant-type-substitution}} notes, this draft therefore rejects some grants that a server previously accepted. Should it retain the closed set and exactly-one-member rule, or permit additional members? If the rule is relaxed, how should unsupported or conflicting bindings be rejected while preserving RFC 7800's single-key constraint?
9. Acceptance of the `jwk` confirmation method. {{cnf-claim}} limits issuance under this specification to `jkt` and `x5t#S256`, but {{processing}} also accepts `jwk`, and {{as-metadata}} advertises that acceptance. Which deployments need interoperability with `jwk` grants issued outside this specification? Is that sufficient reason to retain the additional redemption and comparison rules, or should issuance and redemption support the same set of confirmation methods?

# Acknowledgments
{:numbered="false"}

This document combines and replaces {{JWT-DPOP-GRANT}} and {{TE-CNF}}, and generalizes the sender-constraining rules of {{ID-JAG}}. The author thanks the authors and reviewers of those documents.

# Document History
{:numbered="false"}

-00

* Initial version, combining {{JWT-DPOP-GRANT}} and {{TE-CNF}}.
* Replaces the `jwt-dpop` grant type with the method-neutral `jwt-pop` grant type and the `jwt_grant_cnf_methods_supported` metadata parameter, keeps `jwt-dpop` for existing clients, and verifies bound grants whatever their grant type.
