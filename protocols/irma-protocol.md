---
layout: page
title: IRMA issuance and disclosure protocol
permalink: /protocols/irma-protocol/
---

This page documents the JSON-based RESTful protocol that IRMA uses for verifying and issuing attributes, as well as for creating attribute-based signatures. This protocol is used between the [IRMA API server](https://github.com/privacybydesign/irma_api_server), which communicates with an [IRMA client](https://github.com/privacybydesign/irmago) (e.g., the [IRMA mobile app](https://github.com/privacybydesign/irma_mobile)) on the one hand, and a party wishing to verify or issue attributes, or which wants an attribute-based signature on the other hand. The `irma_api_server` exposes a JWT-driven API for requestors, which is also documented here. The main consumer of this part of the API is the [`irma_js`](https://github.com/privacybydesign/irma_js) Javascript library, for integration into websites.

A comprehensive general introduction to the core concepts and functionality of IRMA can be found [here](/docs/irma.html). An overview of all IRMA technical documentation is [here](https://privacybydesign.foundation/documentation). The [keyshare protocol](/docs/irma.html#irma-pin-codes-using-the-keyshare-server) spoken between the IRMA client and an [IRMA keyshare server](https://github.com/privacybydesign/irma_keyshare_server) is documented [here](/protocols/keyshare-protocol).

# Introduction

First we introduce some terminology:

* *Client*: software that can receive, store, and disclose attributes. Functions as the client in the client-server protocol in which the `irma_api_server` acts as the server. [`irmago`](https://github.com/privacybydesign/irmago) is an IRMA client, which is the basis of the [IRMA mobile app](https://github.com/privacybydesign/irma_mobile).
* *Service provider*: a party (or software acting on its behalf) wishing to receive and verify attributes of a user.
* *Identity provider*: a party (or software acting on its behalf) wishing to issue attributes to a user.
* *Signature requestor*: a party (or software acting on its behalf) that wants the user to sign a message with certain attributes.
* *Requestor*: a service provider, identity provider, or a signature provider, depending on the session type.
* *IRMA session*: sequence of messages exchanged between the `irma_api_server` and client in which attributes are disclosed, issued, or used in an attribute-based signature.
* *IRMA protocol*: the JSON-based protocol that the `irma_api_server` speaks with the client during IRMA sessions.
* *Disclosure proof*: a set of disclosed attributes, along with a proof of knowledge showing that these disclosed attributes originated from a credential that was validly signed by the issuer.

The purpose of the `irma_api_server` is to enable IRMA session that generally happen as follows. We assume that the requestor runs a website and manages IRMA sessions with the `irma_api_server` through [`irma_js`](https://github.com/privacybydesign/irma_js).

* The requestor displays a QR code on its website,
* The user scans it with her IRMA app, which forwards it to the contained IRMA client,
* The client either informs her that she does not have the required attributes, in case attributes need to be disclosed, or asks her for permission to proceed with the IRMA session, listing the involved attributes. If she consents, the client computes the appropriate response, sends this to the `irma_api_server`, which handles it according to the session type and informs the requestor of the result.

The [`irma_api_server`](https://github.com/privacybydesign/irma_api_server) thus handles all IRMA-specific cryptography and the communication with the client, sitting between the requestor and the client. In case of attribute disclosures, in more detail the workflow above results in the following:

* The `irma_api_server` accepts a disclosure proof request from the service provider (for example, the [IRMATube website](https://privacybydesign.foundation/demo/irmaTube/)), and returns a session token.
* It is then the task of the service provider (the IRMATube website) to inform the client of this session token and where to reach the `irma_api_server`. This is done by embedding this information in a QR code which is scanned by the IRMA app.
* The client connects to the `irma_api_server`, sending the session token. The `irma_api_server` informs the client of the required attributes. If the client does not possess these attributes, it aborts. Otherwise, it informs the user of the required attributes. If the user consents, the client creates the disclosure proof and sends it to `irma_api_server`.
* The `irma_api_server` verifies the proof. If anything went wrong, it reports failure. Otherwise, it sends the attributes contained in the proof back to the service provider (the IRMATube website), in the form of a signed JSON web token.

Notice that the IRMATube website has to do very little: it shows the QR code and waits for the outcome of the IRMA session. In particular, it is the client itself and not the requestor that is responsible for informing its owner about what is happening. However, the requestor should be aware of the (RSA) public key with which the `irma_api_server` signs the outcome of the IRMA protocol, in order to be able to verify this.

# The protocol

Below we specify the protocol and API endpoints of the `irma_api_server` for all three kinds of IRMA session: verification of attributes, issuance of attributes, and creation of attribute-based signatures. Instead of fully describe the more technical datatypes we sometimes link to their Java equvalent in the IRMA Java implementation, as in for example [`ProofD`](https://github.com/privacybydesign/irma_api_common/blob/master/src/main/java/org/irmacard/credentials/idemix/proofs/ProofD.java) for a disclosure proof. (Each of these messages have a corresponding struct in the IRMA Go implementation, i.e., in [`gabi`](https://github.com/mhe/gabi) or [`irmago`](https://github.com/privacybydesign/irmago)).

## Verification

We start with the verification protocol between the `irma_api_server` and the client. We first define a _disclosure proof request_ data type, that looks as follows.

~~~ json
{
    "content": [
        {
            "label": "Over 18",
            "attributes": [
                "irma-demo.MijnOverheid.ageLower.over18",
                "irma-demo.Thalia.age.over18"
            ]
        },
        ...
    ]
}
~~~

Each entry of `content` is a disjunction of attributes, along with a label that can be shown to the user of the client. For each such disjunctions the user chooses one of the contained attributes, which is then disclosed. The disjunctions themselves should be ANDed together. Thus, this example asks for either `irma-demo.MijnOverheid.ageLower.over18` or `irma-demo.Thalia.age.over18`.

The service provider can also demand that certain attributes have pre-determined values, by replacing the `attributes` list with a map, for example:

~~~ json
{
    "attributes": {
        "irma-demo.MijnOverheid.ageLower.over18": "yes",
        "irma-demo.Thalia.age.over18": "Yes"
    }
}
~~~
The client's disclosure proof is then only considered valid by the `irma_api_server` if the disclosure proof created by the client contains, in this case, either an instance of `irma-demo.MijnOverheid.ageLower.over18` with the value `yes`, or `irma-demo.Thalia.age.over18` with the value `Yes`.

IRMA also supports credential-only proofs, in which the client proves ownership of a credential without disclosing any of the contained attributes, as follows. When one of the disjunctions contains an identifier of the form `schememanager.issuer.credential` in the `attributes` list (for example, `irma-demo.MijnOverheid.ageLower`), the client should disclose none of the attributes contained in the credential apart from the metadata attribute. Since the metadata attribute is invisible to the user, this means that from the perspective of the user no attributes of the credential are disclosed, so that she only proves possession of the credential.

The `irma_api_server` is a web server listening at the following paths.

*   `POST /api/v2/verification`: accepts requests from service providers in the form of a JSON web token, whose payload should be of the form

    ~~~ json
    {
        "iss": "Service provider name",
        "kid": "service_provider_name",
        "sub": "verification_request",
        "iat": 1453377600,
        "sprequest": {
            "data": "...",
            "validity": 60,
            "timeout": 60,
            "callbackUrl": "...",
            "request": "..."
        }
    }
    ~~~
    The fields mean the following:

    *   `iss` identifies the service provider for the `irma_api_server` and the client. (`iss` is a standard JWT field, refering to the creator of the token, which in this case is, the service provider provider).
    *   `kid` is used by the `irma_api_server` to lookup the service provider's public key with which to verify its JWT. If this field is absent, `iss` is used instead.
    *   `sub` is a fixed field whose value should be `verification_request`.
    *   `iat` is the time of the JWT creation. The server only accepts requests younger than a certain cutoff.
    *   `data` can be any string of the service provider's choosing.
    *   `validity` specifies how long the returned JSON web token should be valid (in seconds).
    *   `request` is a disclosure proof request as defined above (without a nonce or context).
    *   `timeout` specifies how long, in seconds, the server should wait for a client to contact it, before considering the disclosure proof request failed.
    *   `callbackUrl` (optional) If provided, this URL will be, concatenated with the session token, called by the `irma_api_server` when a proof has been posted by the client. The `irma_api_server` will post the result of the IRMA session in a signed JWT to this URL (see below how this JWT is defined).

    Possibly the `irma_api_server` accepts unsigned JWT's. If not, then it keeps track of a list specifying which service provider may verify which attributes, and what the value of the `iss` field must be for each of them.

    In response, the server returns a JSON object of the form

    ~~~ json
    {
        "irmaqr": "disclosing",
        "u": "...",
        "v": "2.0",
        "vmax": "2.3"
    }
    ~~~
    where `u` is the session token, `v` is the lowest API version supported by the `irma_api_server`, and `vmax` is the highest API version supported by the `irma_api_server`. The value of `irmaqr` is always `disclosing` for disclosure sessions. It is the responsibility of the service provider to prepend the url to the api server to the session token `u`, so that the client knows where to find the api server, and then to forward this to the client.

*   `GET /api/v2/verification/verificationID`: if `verificationID` is a
    valid session token (i.e., it has been assigned to a disclosure proof request at some point in the past), the server generates a nonce, puts this in the disclosure proof request associated to this session token, and returns this to the client.
*   `GET /api/v2/verification/verificationID/jwt`: This returns the signed JWT with which
    the service provider started the request (and which contains the disclosure proof request listing the required attributes), along with the context and nonce which are generated by the `irma_api_server`. This allows the client to verify itself the service provider's JWT. For example:

    ```json
    {
        "jwt": "...",
        "nonce": 123,
        "context": 456
    }
    ```
*   `POST /api/v2/verification/verificationID/proofs`: if `verificationID`
    is a valid session token, this accepts a serialized [`ProofList`](https://github.com/privacybydesign/irma_api_common/blob/master/src/main/java/org/irmacard/credentials/idemix/proofs/ProofList.java) object, that contains one or more disclosure proofs ([`ProofD`](https://github.com/privacybydesign/irma_api_common/blob/master/src/main/java/org/irmacard/credentials/idemix/proofs/ProofD.java)'s. The proofs are verified immediately, and the server informs the client of the validity of the proofs in the form of a string.

    The service provider is informed of the result in the form of a JSON web token signed with the `irma_api_server`'s private key. If the proofs verified, then this token contains the disclosed attributes. If there was no `validity` specified in the request from the service provider, the web token's validity is set to 60 seconds. If the `data` field was present then this is included in the web token in the [`jti` field](https://tools.ietf.org/html/rfc7519#section-4.1.7). For example, (the payload of) a returned JSON web token might look as follows.

    ~~~ json
    {
        "exp": 1448636691,
        "sub": "disclosure_result",
        "jti": "foobar",
        "attributes": {
            "irma-demo.MijnOverheid.ageLower.over18": "yes",
            "irma-demo.IRMATube.member": "present",
        },
        "iat": 1448636631,
        "status": "VALID"
    }
    ~~~
    The last entry in the `attributes` map is the response to a credential-only request: it indicates that the `member` credential issued by `IRMATube` is present on the client.

    The `status` is the same value that is sent to the client, and can be one of the following:

    - `VALID`: the proofs were valid
    - `INVALID`: the proofs were invalid
    - `EXPIRED`: one or more of the proofs came from an expired credential
    - `MISSING_ATTRIBUTES`: not all required attributes were disclosed in the proofs
    - `WAITING`: if the client has not yet posted a proof

    Note: it is [important to verify](https://auth0.com/blog/2015/03/31/critical-vulnerabilities-in-json-web-token-libraries/) that the JSON web token is actually signed with an RSA signature.

*   `DELETE /api/v2/verification/verificationID`: If the session
    exists it is deleted, and the service provider is informed of failure. Note that the client sends no reason, but this is what we want: the service provider does not need to learn if the user declined or if he does not have the required attributes.

The protocol is summarized by the following diagram (many thanks to Timen Olthof!).

![Disclosure diagram](/images/DisclosureProof.png)

## Issuing

The workflow here will be much the same as with disclosing. The `irma_api_server` sits between the client and the identity provider, handling all IRMA-specifics of issuing attributes to the client on behalf of the identity provider. In more detail, the following happens:

* The identity provider informs the `irma_api_server` that it wants to issue certain attributes with certain values to a client. These may be attributes from more than one credential type. If the identity provider is authorized for this, and the `irma_api_server` has the appropriate Idemix secret keys, then the `irma_api_server` returns a session token.
* The identity provider informs the client of this session token, who then contacts `irma_api_server` mentioning the session token. `irma_api_server` looks up the attributes that the identity provider wants issued and sends these (unsigned) to the client along with a nonce.
* If the client agrees to receive this credentials, then it engages in the Idemix issuing protocol with the `irma_api_server`, where the client uses this nonce in its first message.

Similarly to the disclosure proof request above, we define an _issuing request_ data type as follows:

~~~ json
{
    "credentials": [
        {
            "credential": "irma-demo.MijnOverheid.ageHigher",
            "validity": 1484481600,
            "attributes": {
                "over50": "yes",
                "over60": "no",
                "over65": "no",
                "over75": "no"
            }
        },
        ...
    ],
    "disclose": [
        {
            "label": "Over 18",
            "attributes": {
                "irma-demo.MijnOverheid.ageLower.over18": "yes",
                "irma-demo.Thalia.age.over18": "Yes"
            }
        },
        ...
    ]
}
~~~

This indicates that the `MijnOverheid` issuer want to issue the `ageHigher` credential, that will expire on the corresponding Unix timestamp (which is in this case January 1 2017, 12:00 PM). In addition, the issuing will only happen if the client can satisfy the disclosure request in the `disclose` field (that is, in this case the client has to be able to show one of the two mentioned attributes with the corresponding value).

The server listens at the following paths.

*   `POST /api/v2/issue/`: accepts requests in the form of a JSON web token, whose payload should be of the form

    ~~~ json
    {
        "iss": "Identity provider name",
        "kid": "identity_provider_name",
        "sub": "issue_request",
        "iat": 1453377600,
        "iprequest": {
            "data": "...",
            "timeout": 10,
            "request": "..."
        }
    }
    ~~~
    The fields mean the following:

    *   `iss` identifies the identity provider for the `irma_api_server` and the client. (`iss` is a standard JWT field, refering to the creator of the client, which in this case is, perhaps slightly confusing, the identity provider and not the IRMA issuer).
    *   `kid` is used by the `irma_api_server` to lookup the identity provider's public key with which to verify its JWT. If this field is absent, `iss` is used instead.
    *   `sub` is a fixed field whose value should be `issue_request`.
    *   `iat` is the time of the JWT creation. The server only accepts requests younger than a certain cutoff.
    *   `data` can be any string of the issuer's choosing.
    *   `request` is an issuing request as defined above (without a nonce or context).
    *   `timeout` specifies how long, in seconds, the server should wait for a client to contact it, before considering the issuing request failed.

    Of the `iprequest` field, only `request` is required, the other three are optional (the default value of `timeout` is 10 seconds). In response, the server returns a JSON object of the form

    ~~~ json
    {
        "irmaqr": "issuing",
        "u": "...",
        "v": "2.0",
        "vmax": "2.3"
    }
    ~~~
    where `u` is the session token, `v` is the lowest API version supported by the `irma_api_server`, and `vmax` is the highest API version supported by the `irma_api_server`. The value of `irmaqr` is always `issuing` for issuance sessions. It is the responsibility of the service provider to prepend the url to the api server to the session token `u`, so that the client knows where to find the api server, and then to forward this to the client.
*   `GET /api/v2/issue/issueID`: if `issueID` is a valid session token (i.e., it has been assigned to an issuing request
    request at some point in the past), the server generates a nonce, puts this in the issuing request associated to this session token, and returns the issuing request.
*   `GET /api/v2/issue/issueID/jwt`: This returns the signed JWT with which
    the identity provider started the request (and which contains the attributes to be issued and attributes to be disclosed, if any), along with the context and nonce which are generated by the `irma_api_server`. This allows the client to verify itself the identity provider's JWT. In addition, the issuer public keys against which the new attributes can be verified are included. For example:

    ```json
    {
        "jwt": "...",
        "nonce": 123,
        "context": 456,
        "keys": {
            "irma-demo.MijnOverheid": 1
        }
    }
    ```
    The one entry of the `keys` field indicate that the second public key (they are numbered zero-based) of the `MijnOverheid` issuer.
*   `POST /api/v2/issue/issueID/commitments`: if `issueID` is a valid session token, then this accepts the client's
    commitments to its secret key, one for each credential type that will be issued, in the form of a serialized
    [`IssueCommitmentMessage`](https://github.com/privacybydesign/irma_api_common/blob/master/src/main/java/org/irmacard/credentials/idemix/messages/IssueCommitmentMessage.java). (This data type contains a [`ProofList`](https://github.com/privacybydesign/irma_api_common/blob/master/src/main/java/org/irmacard/credentials/idemix/proofs/ProofList.java) instance; it is important that the proofs contained
    in that list are in the same order as the credentials in the `credentials` object of the issuing request. If the issuing request included the disclosure(s) of other credential(s), then this [`ProofList`](https://github.com/privacybydesign/irma_api_common/blob/master/src/main/java/org/irmacard/credentials/idemix/proofs/ProofList.java) should contain the corresponding disclosure proof(s)).

    The `irma_api_server` verifies the correctness of the commitments using the included zero-knowledge proofs. If they
    are valid, and if the appropriate attributes were correctly disclosed, then it computes the corresponding Camenisch-Lysyanskaya signatures for each of the credentials, and returns
    these to the client, in the form of a list of [`IssueSignatureMessage`](https://github.com/privacybydesign/irma_api_common/blob/master/src/main/java/org/irmacard/credentials/idemix/messages/IssueSignatureMessage.java)s.
*   `DELETE /api/v2/issue/issueID`: If the session exists it is deleted, and the identity provider is informed of failure.

The protocol is summarized by the following diagram (many thanks to Timen Olthof!).

![Disclosure diagram](/images/Issuing.png)

## Attribute-based signatures
Besides disclosing them, IRMA attributes can also be used to produce *attribute-based signatures*. These are digital signatures over any set of bytes (although currently only strings are supported by IRMA), to which the attributes are attached. This is achieved by a single change in the IRMA disclosure protocol: instead of signing the `nonce` provided by the API server, the client sings an actual message (or rather its `SHA256`, perhaps combined with a `nonce` depending on the setup). In this fashion, any IRMA client can produce an attribute-based signature over any message, with any subset of its attributes attached to it, which can be verified against the IRMA issuer's public keys that issued the attributes. For example, the user could sign a document using her name, but also only using the fact that she is over 18 and thus old enough to sign a contract.

Like the message, the attributes cannot be changed, removed, added or otherwise manipulated without invalidating the included signature. For more details on attribute-based signatures, see [here](/docs/irma.html#attribute-based-signatures).

The `irma_api_server` supports attribute-based signature with an API that is *mutatis mutandis* a copy of the API for attribute disclosure. The party that initiates an attribute-based signature session with the `irma_api_server`, called the *signature requestor*, thus posts a signed JWT to the `irma_api_server` containing the message she wants signed, along with a conjunction of disjuntions of attributes as in disclosure session. The `irma_api_server` returns a session token, which is forwarded to the client by the signature requestor. If the user approves the client signs the message with the specified attributes, and sends the signature and attributes to the `irma_api_server`, which forwards it to the client if the signature and attributes were valid.

We define a *signature request* data type (the analog of the disclosure proof request from disclosure sessions), which specifies the message and the attributes that should be used, as follows:

~~~json
{
    "message" : "Message to be signed",
    "messageType" : "STRING",
    "content": [
        {
            "label": "Name",
            "attributes": {"irma-demo.MijnOverheid.fullName.firstname": "Johan"}
        },
        {
            "label": "Over 21",
            "attributes": ["irma-demo.MijnOverheid.ageLower.over18", "irma-demo.MijnOverheid.ageLower.over21"]
        }
    ]
}
~~~

Each entry of `content` is a disjunction of attributes,
along with a label that can be shown to the user of the client. The disjunctions
themselves should be ANDed together. Besides the attributes, the signature requestor can also require
optional conditions on the values of the attributes by specifying the attributes
in a key-value dictionary instead of a list, as in disclosure sessions. If a dictionary with values is
used, then the signer (the client) is required to have the attribute values
specified in this dictionary. If not, then the signature will be rejected by
the irma_api_server.

This means in the above example that a signer is required to have a `firstname`
attribute with the value `Johan`. Furthermore, the `over18` and `over21`
attributes need to be disclosed, but a specific value for these attributes is not
required.

If `messageType` contains the keyword `STRING`, then the `message` must contain
a string that is the message that will be signed with the specified attributes.
In the future, the `messageType` could be extended with for example `PDF`. In that
case, the message field could contain a URL to a PDF file, which can be retrieved 
and signed instead.

Like with the disclosure proofs, the `irma_api_server` will listen to the
following paths:

*   `POST /api/v2/signature`: accepts requests from
    service providers of the following form:

    ~~~ json
    {
        "data": "...",
        "validity": "60",
        "request": "..."
    }
    ~~~
    This is the same type of request as `/api/v2/verification`, but the request
    data type needs to be a signature specification instead of a disclosure proof
    request. In response, the server returns a JSON object of the form:

    ~~~ json
    {
        "irmaqr": "signing",
        "u": "...",
        "v": "2.0",
        "vmax": "2.3"
    }
    ~~~
    where `u` is the session token, `v` is the lowest API version supported by the `irma_api_server`, and `vmax` is the highest API version supported by the `irma_api_server`. The value of `irmaqr` is always `signing` for attribute-based signature sessions.

*   `GET /api/v2/signature/sessionToken`: if `sessionToken` is a
    valid session token (i.e., it has been assigned to a signature specification
    at some point in the past), the server generates a nonce, puts this in the
    request associated to this session token, and returns this to the client.

*   `GET /api/v2/signature/sessionToken/jwt`: This returns the signed JWT with which
    the signature requestor started the request (and which contains the disclosure proof request listing the required attributes and the message to be signed), along with the context and nonce which are generated by the `irma_api_server`. This allows the client to verify itself the service provider's JWT. For example:

    ```json
    {
        "jwt": "...",
        "nonce": 123,
        "context": 456
    }
    ```

*   `POST /api/v2/signature/sessionToken/proofs`: if `sessionToken`
    is a valid session token, this accepts a a
    serialized [`ProofList`](https://github.com/privacybydesign/irma_api_common/blob/master/src/main/java/org/irmacard/credentials/idemix/proofs/ProofList.java) object that contains one or more disclosure proofs
    ([`ProofD`](https://github.com/privacybydesign/irma_api_common/blob/master/src/main/java/org/irmacard/credentials/idemix/proofs/ProofD.java)'s. The proofs are verified immediately, ensuring that they sign the right message as requested by the signature requestor, and the server informs the
    client of the validity of the
    proofs in the form of a string.

    An example of a signature that corresponds to the earlier described
    signature proof request:

    ~~~json
    {    
         "signature":
         { 
             "proofs": [ "A ProofD", "another ProofD", ... ],
             "nonce": 123,
             "context": 456,
         },
         "message": "Message to be signed",
         "messageType": "STRING",
         "status": "VALID",
         "attributes":
         {   
             "irma-demo.MijnOverheid.fullName.firstname": "Johan",
             "irma-demo.MijnOverheid.ageLower.over18": "yes"
         },
         "data": "foobar"
    }
    ~~~

    The service provider is informed of the result in the form of a JSON web
    token signed with our RSA private key. If the proofs verified, then this token
    contains the attributes, the signed message, and the nonce; this is sufficient information for later verification of the the attributes and signature over the message. If there was no `validity` specified in the request
    from the service provider, the web token's validity is set to 60 seconds. If the
    `data` field was present then this is included in the web token in the [`jti`
    field](https://tools.ietf.org/html/rfc7519#section-4.1.7). Such token looks
    almost the same as with discosure proofs:

    ~~~ json
    {
        "exp": 1448636691,
        "sub": "signature_result",
        "jti": "foobar",
         "signature":
         { 
             "proofs": [ "A ProofD", "another ProofD", ... ],
             "nonce": 123,
             "context": 4565
         },
        "message": "Message to be signed",
        "messageType": "STRING",
        "attributes":
        {
            "irma-demo.MijnOverheid.fullName": "present",
            "irma-demo.MijnOverheid.fullName.firstname": "Johan",
            "irma-demo.MijnOverheid.ageLower": "present",
            "irma-demo.MijnOverheid.ageLower.over18": "yes"
        },
        "iat": 1448636631,
        "status": "VALID"
    }
    ~~~
    
*   `DELETE /api/v2/signature/sessiontoken`: If the session
    exists it is deleted, and the service provider is informed of failure. Note
    that the client sends no reason, but this is what we want: the service provider
    does not need to learn if the user declined or if he does not have the required
    attributes.

*   `POST /api/v2/signature/checksignature`:
    This is a stateless api request that allows a service provider to
    check a signature without having IRMA software. It accepts a signature, as
    returned in the json web token and returns a status (with the same structure as
    the status object that has been included in the json web token).
    Note that a session token is not needed to execute
    this request. Also note that the response on this request is not signed with
    the api_server's private key.
