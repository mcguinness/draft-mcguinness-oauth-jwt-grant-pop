# Background for Appendices A and B

The -00 draft shortens Appendix A (Profiles of This Specification) and
Appendix B (Open Issues). This file keeps the full text of both appendices
as they stood before they were shortened, at commit 65feacc, including the
reasoning behind each proposed change to ID-JAG and the background for each
open issue. Section numbers and references are those of that version of the
draft.

```text
Appendix A.  Profiles of This Specification

   This appendix is informative.  It shows how [ID-JAG] would answer the
   items in Section 8 and how the defaults of Section 8 apply to
   [ID-CHAINING], and lists the conflicts between each document and this
   specification.

A.1.  Identity Assertion JWT Authorization Grant

   [ID-JAG] would answer the items as follows:

   1.  The typ value is oauth-id-jag+jwt, and the validation rules rely
       on it (Section 4.4.1 of [ID-JAG]); the requested_token_type value
       is urn:ietf:params:oauth:token-type:id-jag.

   2.  Both confirmation methods, jkt and x5t#S256, may be used.
       Neither method, nor the jwt-pop grant type, is mandatory to
       implement, and a Resource Authorization Server verifies each
       method that it lists in jwt_grant_cnf_methods_supported
       (Section 7.1).  A DPoP-bound authorization grant is presented
       with the jwt-dpop grant type (Section 4.4 of [ID-JAG]), which
       this specification keeps for existing clients (Section 5.1.2).

   3.  Binding is optional.  When the DPoP proof is valid, the IdP
       Authorization Server binds the authorization grant: to the DPoP
       key, unless its local policy selects a certificate binding
       instead (Section 9.11.1.1 of [ID-JAG]).  Its policy does not
       otherwise decline the jkt confirmation method (Section 4.2).  The
       first conflict below concerns that certificate choice.

   4.  A Resource Authorization Server may accept a key-bound
       authorization grant in place of client authentication.  The IdP
       Authorization Server authenticates the client in the Token
       Exchange request, binds the grant to the key or certificate for
       which that request carries proof of possession, and identifies
       the client in the grant's client_id claim.

   5.  A Resource Authorization Server also validates a certificate
       presented for an x5t#S256-bound authorization grant as described
       in Section 2 of [RFC8705] (Section 9.11.2.2 of [ID-JAG]).  It
       rejects an unbound grant with the invalid_grant error code when
       the resource requires sender-constrained access tokens and the
       request carries neither a DPoP proof nor a TLS client certificate
       (Section 9.11.1.2.4 of [ID-JAG], Section 9.11.2.2 of [ID-JAG]).

   A client that detects a downgrade from the authorization grant's cnf
   claim (Section 9.11.1.1 of [ID-JAG], Section 9.11.2.1 of [ID-JAG])
   follows Section 4.4, which also accepts the cnf response parameter.

   Version -06 of that document conflicts with this specification in two
   places.  Each could be resolved by a change to either document
   (Appendix B); the changes proposed for that document are:

   *  Section 9.11 of [ID-JAG] lets the IdP Authorization Server select
      either a DPoP or a certificate binding, according to local policy,
      when a request carries both a DPoP proof and a TLS client
      certificate.  This specification binds an authorization grant
      issued in response to a DPoP binding request only to the key of
      the DPoP proof, if it binds it at all (Section 4.2.2), for two
      reasons.  The client can authenticate to the IdP Authorization
      Server with a certificate that it does not use downstream and
      still obtain a grant that it can redeem with its DPoP key.  And an
      attacker that replays a captured DPoP-bound subject_token or
      actor_token and its DPoP proof over its own mutual-TLS connection
      cannot obtain a grant bound to the attacker's certificate
      (Section 9.4).  The change is for the IdP Authorization Server
      never to choose a certificate binding when the request contains a
      DPoP proof.  A narrower change is listed among the open issues
      (Appendix B).

   *  Section 8 of [ID-JAG] requires a client that supports that profile
      to include urn:ietf:params:oauth:grant-type:jwt-bearer in its
      grant_types, which prevents a Resource Authorization Server from
      limiting the client to key-bound authorization grants
      (Section 7.2).  The change is to require
      urn:ietf:params:oauth:grant-type:jwt-bearer or
      urn:ietf:params:oauth:grant-type:jwt-pop, and to present a key-
      bound grant of either method with the urn:ietf:params:oauth:grant-
      type:jwt-pop grant type, where Section 4.4 of [ID-JAG] uses
      urn:ietf:params:oauth:grant-type:jwt-dpop for a DPoP-bound grant
      and urn:ietf:params:oauth:grant-type:jwt-bearer otherwise.

A.2.  Identity and Authorization Chaining

   [ID-CHAINING] defines no profile of this specification, so the
   defaults in Section 8 apply.  Binding the authorization grant adds a
   mitigation to those listed in Section 5.5 of [ID-CHAINING].  The
   access token in Appendix B.3 of [ID-CHAINING] is bound to a key that
   the grant conveys in a requested_cnf claim.  For a key-bound grant,
   Section 5.3 permits only the bindings in Table 3, so a grant that
   carries both a cnf claim and a requested_cnf claim for a different
   key conflicts with this specification.

Appendix B.  Open Issues

   [Editor's note: this appendix lists questions for the working group.
   It will be removed before publication.]

   1.   Grant type or client metadata.  A jwt-pop request has the same
        format as a jwt-bearer request, while Section 4.1 of [RFC7521]
        defines grant_type as "the format of the assertion".  An
        alternative is to present every key-bound authorization grant
        with the jwt-bearer grant type and to express "key-bound grants
        only" with a boolean client metadata parameter, as
        dpop_bound_access_tokens does (Section 5.2 of [RFC9449]), or for
        every client with a boolean authorization server metadata
        parameter, as dpop_bound_access_tokens_required does for a
        protected resource (Section 2 of [RFC9728]).  That would remove
        the jwt-pop grant type and the conflict with Section 8 of
        [ID-JAG].  It would also lose the per-request requirement and
        the rejection of an unsupported grant type that Section 9.2
        describes.  With client metadata, a client would instead confirm
        that the registration policy took effect from the registered
        metadata that the resource authorization server returns
        (Section 3.2.1 of [RFC7591]); with server-wide metadata, it
        would learn the policy from the resource authorization server's
        validated metadata (Section 7.1).  Which should the working
        group prefer?

   2.   Error for a required binding.  When binding is required and the
        issuing authorization server does not bind the authorization
        grant, this draft returns invalid_dpop_proof if a DPoP proof
        would let it bind the grant, invalid_request if only a TLS
        client certificate would, and invalid_target otherwise
        (Section 4.2.4).  Section 5 of [RFC9449] defines
        invalid_dpop_proof for an invalid DPoP proof; this draft also
        uses it for a missing one at the issuing authorization server.
        [JWT-DPOP-GRANT] uses it for a missing proof at the
        authorization server that redeems the grant, where this draft
        returns invalid_grant (Section 5.1.2).  Is that extension
        acceptable, or should a new error code tell the client that
        proof of possession is required, for either confirmation method,
        distinct from the invalid_request error code that an unproven
        bound subject_token also receives?

   3.   Scope of the response parameter.  [RFC9201] registers the cnf
        token response parameter for ACE.  Should this document define
        its use for Token Exchange only, for every token response, or
        move it to a separate specification?

   4.   Client opt-in to certificate binding.  Binding an authorization
        grant to the TLS client certificate that the client presents at
        issuance is an issuance policy that the client opts into, not
        only a capability: a client can support certificate-bound grants
        and still reserve the certificate it presents to the issuing
        authorization server for that relationship.  This draft leaves
        the opt-in to the issuing authorization server's configuration
        for the client (Section 4.2.3).  Should a client metadata
        parameter express it?  The
        authorization_grant_profiles_supported parameter of [ID-JAG]
        could carry such a signal, at the cost of mixing grant profiles
        with proof capabilities in one list.  If a client can opt in,
        should the opt-in also let the issuing authorization server bind
        the grant to the certificate when the request also contains a
        DPoP proof, as Section 9.11 of [ID-JAG] permits, except when the
        request proves possession of a DPoP-bound subject_token or
        actor_token (Section 9.4)?  That would narrow the certificate
        conflict with [ID-JAG] in Appendix A.  It would need exceptions
        to Section 4.2.2 and Section 4.1, which bind the grant only to
        the key of the DPoP proof, to Section 4.2.3, which applies only
        to a request without a DPoP proof, and to the invalid_target
        case of Section 4.2.4, and the client's check in Section 4.4
        would have to accept the certificate of a client that opted in.

   5.   Alignment with [ID-JAG], [ID-CHAINING], and [JWT-DPOP-GRANT].
        The conflicts listed in Appendix A, two with [ID-JAG] and one
        with Appendix B.3 of [ID-CHAINING], need to be resolved, by the
        proposed changes to those documents or by changes to this one.
        For the last, should a profile be able to permit binding the
        access token to a key that the authorization grant conveys in a
        requested_cnf claim?  The replacement of the jwt-dpop grant type
        and the compatibility rules in Section 5.1.2 need the agreement
        of the author of [JWT-DPOP-GRANT], which also requests
        registration of the jwt-dpop grant type (Section 9.1 of
        [JWT-DPOP-GRANT]).

   6.   Unbound authorization grant after a DPoP binding request.  When
        a valid DPoP binding request cannot be honored, Section 4.2.2
        forbids the issuing authorization server to reject the request
        for that reason, so any grant that it issues is unbound, unless
        binding is required by the profile, by the issuing authorization
        server's policy, or by the client's registration
        (Section 4.2.4).  A client that requires binding discards an
        unbound grant (Section 4.4), but the successful exchange may
        already have consumed a single-use subject_token, for example
        under an issuer's refresh-token rotation policy.  The protection
        in Section 4.2 applies only when the issuer rejects the request.
        A client can already obtain a required binding through a profile
        that requires it, or through an arrangement that its
        registration at the issuing authorization server records
        (Section 4.1).  Is a per-request "bind or fail" option needed
        for a requirement that neither expresses, and how would it
        preserve compatibility with clients that send DPoP proofs for
        other reasons?

   7.   Refresh-token binding for confidential clients.  Section 5.4
        binds a refresh token issued for a key-bound authorization grant
        to a key or certificate even when the client authenticates,
        except when the grant identifies that authenticated client.
        This departs from Section 5 of [RFC9449], which retains client
        binding for confidential clients so that authentication
        credentials can rotate without invalidating refresh tokens.  The
        additional binding requires the client to retain the bound key
        or certificate for refresh, complicating rotation and recovery.
        Is the current exception sufficient, or should the binding
        requirement be a profile choice?  How would an alternative
        prevent replay of a captured grant and DPoP proof by an attacker
        authenticating as a different client, including at later
        refreshes (Section 9.4)?

   8.   Strict confirmation-member rules.  Section 5.2 rejects a cnf
        claim unless it contains exactly one member, chosen from jkt,
        jwk, and x5t#S256.  Section 4.2.1 treats a subject_token or
        actor_token with any cnf claim or value as bound and accepts
        proof of possession only for one with exactly one member.
        Section 3.1 of [RFC7800] permits other confirmation members and,
        in the absence of application-specific requirements, ignores
        members that an implementation does not understand.  Both rules
        also reject a cnf claim that is valid under RFC 7800 and has two
        members, such as jku with the kid member that Section 3.5 of
        [RFC7800] requires when the referenced JWK Set contains more
        than one key.  As Section 9.2 notes, this draft therefore
        rejects some authorization grants and Token Exchange requests
        that a server previously accepted.  Should it retain the closed
        set and exactly-one-member rule, or permit additional members?
        If the rule is relaxed, how should unsupported or conflicting
        bindings be rejected while preserving RFC 7800's single-key
        constraint?

   9.   Acceptance of the jwk confirmation method.  Section 3.1 limits
        issuance under this specification to jkt and x5t#S256, but
        Section 5.2 can also accept jwk, and Section 7.1 advertises that
        acceptance.  Which deployments need interoperability with jwk
        authorization grants issued outside this specification?  Is that
        sufficient reason to retain the additional redemption and
        comparison rules, or should issuance and redemption support the
        same set of confirmation methods?

   10.  Enforcement of a required binding.  When binding is required,
        this draft requires the issuing authorization server to bind the
        authorization grant, but not to establish that each
        authorization server in the grant's audience verifies the
        binding, because such a determination can become stale
        (Section 9.1).  The profile, configuration, or the issuing
        authorization server's policy can establish it instead.  Should
        a requirement for binding also require that determination, and
        if so, from what evidence?

   11.  Mandatory-to-implement confirmation method.  By default, neither
        confirmation method is mandatory to implement (Section 8).  An
        issuing authorization server binds the authorization grant to
        the key of a DPoP binding request whenever jkt is a permitted
        confirmation method (Section 4.2.2), so a resource authorization
        server that verifies only x5t#S256 rejects a grant that it would
        have accepted unbound, unless the issuing authorization server's
        policy declines jkt for that audience (Section 4.2).  Should jkt
        be mandatory to implement for both authorization servers by
        default?
```
