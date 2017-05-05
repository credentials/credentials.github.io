---
layout: page
title: "IRMA issuance and disclosure protocol"
permalink: /protocols/irma-protocol/
---

This page documents the JSON-based protocol that IRMA (more specifically, the [IRMA Android app](https://github.com/credentials/irma_android_cardemu) and the [IRMA API server](https://github.com/credentials/irma_api_server)) uses for issuing and verifying attributes.

# Introduction

The workflow when a user discloses attributes is as follows (the workflow when issuing is almost the same). Suppose she wants to watch a video on IrmaTube; we assume that she already has the proper credential on her token, which we assume to be the IRMA Android app for now.

* The IrmaTube website displays a QR code as before,
* The user scans it,
* Her token either informs her that she does not have the required credentials or attributes, or asks her for permission to disclose the attributes that IrmaTube asked for. If she consents, her token sends a disclosure proofs to IrmaTube.

For code reusability and maintainability, it makes sense to split the logic of the website and the cryptography of verifying the credentials into separate projects. For that reason we have created the [`irma_api_server`](https://github.com/credentials/irma_api_server), which handles the cryptography and the communication with the token. This server sits between the service provider (the IrmaTube website in the example above) and the token, and generally works as follows.

* It accepts a disclosure proof request from the service provider (i.e., the IrmaTube website), and returns a session token.
* It is then the task of the service provider (the IrmaTube website) to inform the token of this session token and where to reach the `irma_api_server` - for example by using a QR code.
* The token connects to the `irma_api_server`, sending the session token. The `irma_api_server` informs the token of the required attributes. If the token does not possess these attributes, it aborts. Otherwise, it informs the user of the required attributes. If the user consents, the token creates the disclosure proof and sends it to `irma_api_server`.
* The `irma_api_server` verifies the proof. If anything went wrong, it reports failure. Otherwise, it sends the attributes contained in the proof back to the service provider (the IrmaTube website), in the form of a signed JSON web token.

Notice that the IrmaTube website now does almost nothing, apart from showing the QR-code; the token itself is responsible for informing its owner about what is happening. However, the service provider should be aware of the (RSA) public key with which `irma_api_server` signs the JSON web tokens, so that it can verify these.

# The protocol

## Verification

We first focus on the protocol between the `irma_api_server` and the token. We first define a _disclosure proof request_ data type, that looks as follows.

~~~ json
{
    "nonce": 123,
    "context": 456,
    "content": [
        {
            "label": "Over 18",
            "attributes": [
                "MijnOverheid.ageLower.over18",
                "Thalia.age.over18"
            ]
        },
        ...
    ]
}
~~~

The `nonce` and `context` are optional large integers. Each entry of `content` is a disjunction of attributes, along with a label that can be shown to the user of the token. The disjunctions themselves should be ANDed together. Thus, this example asks for `MijnOverheid.ageLower.over18` or `Thalia.age.over18`.

Credential-only proofs are supported as follows. When one of the disjunctions contains an identifier of the form `issuer.credential` in the `attributes` list (for example, `MijnOverheid.ageLower`), the token should disclose none of the attributes contained in the credential apart from the metadata attribute. Since the metadata attribute is invisible to the user, this means that from the perspective of the user no attributes of the credential are disclosed, so that he only proves possession of the credential.

The `irma_api_server` is a web server listening at the following paths.

*   `POST /api/v2/verification`: accepts requests from service providers in the form of a JSON web token, whose payload should be of the form

    ~~~ json
    {
        "iss": "Service provider name",
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

    *   `iss` identifies the service provider for the `irma_api_server` and the token. (`iss` is a standard JWT field, refering to the creator of the token, which in this case is, the service provider provider).
    *   `sub` is a fixed field whose value should be `verification_request`.
    *   `iat` is the time of the JWT creation. The server only accepts requests younger than a certain cutoff.
    *   `data` can be any string of the service provider's choosing.
    *   `validity` specifies how long the returned JSON web token should be valid (in seconds).
    *   `request` is a disclosure proof request as defined above (without a nonce or context).
    *   `timeout` specifies how long, in seconds, the server should wait for a token to contact it, before considering the disclosure proof request failed.
    *   `callbackUrl` (optional) If provided, this URL will be, concatenated with the session token, called by the `irma_api_server` when a proof has been posted by the token. The `irma_api_server` will post the result of the IRMA session in a signed JWT to this URL (see below how this JWT is defined).

    Possibly the server accepts unsigned JWT's. In response, the server returns a JSON object of the form

    ~~~ json
    {
        "u": "...",
        "v": "2.0"
    }
    ~~~
    where `u` is the session token (`v` is a fixed value numbering the API version). It is the responsibility of the service provider to prepend the url to the api server to the session token `u`, so that the token knows where to find the api server, and then to forward this to the token.

*   `GET /api/v2/verification/verificationID`: if `verificationID` is a
    valid session token (i.e., it has been assigned to a disclosure proof request at some point in the past), the server generates a nonce, puts this in the disclosure proof request associated to this session token, and returns this to the token.
*   `POST /api/v2/verification/verificationID/proofs`: if `verificationID`
    is a valid session token, this accepts a serialized `ProofList` object, that contains one or more disclosure proofs (`ProofD`s). The proofs are verified immediately, and the server informs the client (i. e., the IRMA token, not the service provider) of the validity of the proofs in the form of a string.

    The service provider is informed of the result in the form of a JSON web token signed with our RSA private key. If the proofs verified, then this token contains the attributes. If there was no `validity` specified in the request from the service proider, the web token's validity is set to 60 seconds. If the `data` field was present then this is included in the web token in the [`jti` field](https://tools.ietf.org/html/rfc7519#section-4.1.7). For example, (the payload of) a returned JSON web token might look as follows.

    ~~~ json
    {
        "exp": 1448636691,
        "sub": "disclosure_result",
        "jti": "foobar",
        "attributes": {
            "MijnOverheid.ageLower.over18": "yes",
            "IRMAWiki.member.email": "stuifje@kuifje.nl",
            "MijnOverheid.fullName": "present"
        },
        "iat": 1448636631,
        "status": "VALID"
    }
    ~~~
    The last entry in the `attributes` map is the response to a credential-only request: it indicates that the `fullName` credential issued by `MijnOverheid` is present on the token.

    The `status` is the same value that is sent to the token, and can be one of the following:

    - `VALID`: the proofs were valid
    - `INVALID`: the proofs were invalid
    - `EXPIRED`: one or more of the proofs came from an expired credential
    - `MISSING_ATTRIBUTES`: not all required attributes were disclosed in the proofs
    - `WAITING`: if the token has not yet posted a proof

    Note: it is [important to verify](https://auth0.com/blog/2015/03/31/critical-vulnerabilities-in-json-web-token-libraries/) that the JSON web token is actually signed with an RSA signature.

*   `DELETE /api/v2/verification/verificationID`: If the session
    exists it is deleted, and the service provider is informed of failure. Note that the token sends no reason, but this is what we want: the service provider does not need to learn if the user declined or if he does not have the required attributes.

The protocol is summarized by the following diagram (many thanks for Timen Olthof!).

![Disclosure diagram](/images/DisclosureProof.png)

## Issuing

The workflow here will be much the same as with disclosing. The `irma_api_server` sits between the token and the identity provider, handling all IRMA-specifics of issuing attributes to the token on behalf of the identity provider. In more detail, the following happens:

* The identity provider informs the `irma_api_server` that it wants to issue certain attributes with certain values to a token. These may be attributes from more than one credential-type. If the identity provider is authorized for this, and the `irma_api_server` has the appropriate Idemix secret keys, then the `irma_api_server` returns a session token.
* The identity provider informs the token of this session token, who then contacts `irma_api_server` mentioning the token. `irma_api_server` looks up the attributes that the identity provider wants issued and sends these (unsigned) to the token along with a nonce.
* If the token agrees to receive this credentials, then it engages in the Idemix issuing protocol with the `irma_api_server` normally, where the token uses this nonce in its first message.

Similarly to the disclosure proof request above, we define an _issuing request_ data type as follows:

~~~ json
{
    "nonce": 123,
    "context": 456,
    "credentials": [
        {
            "credential": "MijnOverheid.ageHigher",
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
                "MijnOverheid.ageLower.over18": "yes",
                "Thalia.age.over18": "Yes"
            }
        },
        ...
    ]
}
~~~

This indicates that we want to issue the `ageHigher` credential from `MijnOverheid`, that will expire on the corresponding Unix timestamp (which is in this case January 1 2017, 12:00 PM). In addition, the issuing will only happen if the token can satisfy the disclosure request in the `disclose` field (that is, in this case the token has to be able to show one of the two mentioned attributes with the corresponding value).

The server listens at the following paths.

*   `POST /api/v2/issue/`: accepts requests in the form of a JSON web token, whose payload should be of the form

    ~~~ json
    {
        "iss": "Identity provider name",
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

    *   `iss` identifies the identity provider for the `irma_api_server` and the token. (`iss` is a standard JWT field, refering to the creator of the token, which in this case is, perhaps slightly confusing, the identity provider).
    *   `sub` is a fixed field whose value should be `issue_request`.
    *   `iat` is the time of the JWT creation. The server only accepts requests younger than a certain cutoff.
    *   `data` can be any string of the issuer's choosing.
    *   `request` is an issuing request as defined above (without a nonce or context).
    *   `timeout` specifies how long, in seconds, the server should wait for a token to contact it, before considering the issuing request failed.

    The server keeps a list of public keys for the verification of incoming JSON web tokens; it uses the `iss` field to lookup the appropriate public key.

    Of the `iprequest` field, only `request` is required, the other three are optional (the default value of `timeout` is 10 seconds). In response, the server returns a JSON object of the form

    ~~~ json
    {
        "u": "...",
        "v": "2.0"
    }
    ~~~
    where `u` is the session token (`v` is a fixed value numbering the API version). It is the responsibility of the service provider to prepend the url to the api server to the session token `u`, so that the token knows where to find the api server, and then to forward this to the token.
*   `GET /api/v2/issue/issueID`: if `issueID` is a valid session token (i.e., it has been assigned to an issuing request
    request at some point in the past), the server generates a nonce, puts this in the issuing request associated to this session token, and returns this to the token.
*   `POST /api/v2/issue/issueID/commitments`: if `issueID` is a valid session token, then this accepts the token's
    commitments to its secret key, one for each credential type that will be issued, in the form of a serialized
    `IssueCommitmentMessage`. (This data type contains a `ProofList` instance; it is important that the proofs contained
    in that list are in the same order as the credentials in the `credentials` object of the issuing request. If the issuing request included the disclosure(s) of other credential(s), then this `ProofList` should contain the corresponding disclosure proof(s)).

    The `irma_api_server` verifies the correctness of the commitments using the included zero-knowledge proofs. If they
    are valid, and if the appropriate attributes were correctly disclosed, then it computes the corresponding Camenisch-Lysyanskaya signatures for each of the credentials, and returns
    these to the token, in the form of a list of `IssueSignatureMessage`s.
*   `DELETE /api/v2/issue/issueID`: If the session exists it is deleted, and the identity provider is informed of failure.

The protocol is summarized by the following diagram (many thanks for Timen Olthof!).

![Disclosure diagram](/images/Issuing.png)

## Attribute-based signatures
Besides just disclosure proofs, we can also use IRMA to produce attribute-based
digital signatures [1].

[1] : [http://www.cs.ru.nl/~brinda/publications/Space-paper.pdf](http://www.cs.ru.nl/~brinda/publications/Space-paper.pdf)

Most of the times, it is important to know with _which_ attributes a statement is
signed: this will describe the role of the signer. A person could for example
sign a document using his/her name, but also only using the fact that he/she is
over 18 and thus old enough to sign a contract.

To determine the required attributes, a signature specification is used: this
specification determines how a certain statement is signed in the same way as a
disclosure proof request determines which attributes are used in a disclosure
proof. If a service provider needs a signature from a user, it has to send this
signature specification to the irma_api_server. 

An example signature specification looks as follows:

~~~json
{
    "nonce": 123,
    "context": 456,
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

Like with the disclosure proof request data type, the `nonce` and `context` are
optional large integers. Each entry of `content` is a disjunction of attributes,
along with a label that can be shown to the user of the token. The disjunctions
themselves should be ANDed together. Besides the attributes, we can also require
optional conditions on the values of the attributes by specifying the attributes
in a key-value dictionary instead of a list. If a dictionary with values is
used, then the signer (the token) is required to have the attribute values
specified in this dictionary. If not, then the signature won't be accepted by
the irma_api_server.

This means in the above example that a signer is required to have a `firstname`
attribute with the value `Johan`. Furthermore, the `over18` and `over21`
attributes need to be disclosed, but a specific value for these attributes is not
required.

If `messageType` contains the keyword _STRING_, then the `message` must contain
a string that is the message that will be signed with the specified attributes.
In the future, the messageType could be extended with for example 'PDF'. In that
case, the message field could contain a URL to a PDF file, which can be retrieved 
and signed instead.

Like with the disclosure proofs, the irma_api_server will listen to the
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
    This is the same type of request as /api/v2/verification, but the request
    data type needs to be a signature specification instead of a disclosure proof
    request. In response, the server returns a JSON object of the form:

    ~~~ json
    {
        "u": "...",
        "v": "2.0"
    }
    ~~~
    where `u` is the session token, like with the disclosure proofs.

*   `GET /api/v2/signature/sessionToken`: if `sessionToken` is a
    valid session token (i.e., it has been assigned to a signature specification
    at some point in the past), the server generates a nonce, puts this in the
    request associated to this session token, and returns this to the token.

*   `POST /api/v2/signature/sessionToken/proofs`: if `sessionToken`
    is a valid session token, this accepts an IRMA signature that consists of a
    serialized `ProofList` object, that contains one or more disclosure proofs
    (`ProofD`s). The proofs are verified immediately, and the server informs the
    client (i. e., the IRMA token, not the service provider) of the validity of the
    proofs in the form of a string. Note that this disclosure proof must be a
    signature proof (so the message should be contained in the challenge of the
    proof).

    An example of a signature that corresponds to the earlier described
    signature proof request (this should be posted by the token to the api
    server):

    ~~~json
    {    
         "signature":
         { 
             "proofs": [ "List with ProofDs" ],
             "nonce": 6.579489688380443e+23,
             "context": 6.843305777297634e+75
         },
         "message": "Message to be signed",
         "messageType": "STRING",
         "status": "VALID",
         "attributes":
         {   
             "irma-demo.MijnOverheid.fullName": "present",
             "irma-demo.MijnOverheid.fullName.firstname": "Johan",
             "irma-demo.MijnOverheid.ageLower": "present",
             "irma-demo.MijnOverheid.ageLower.over18": "yes"
         },
         "data": "foobar"
    }
    ~~~

    The proofs are verified immediately, and the server informs the client (i.e.,
    the IRMA token, not the service provider) of the validity of the proofs in
    the form of a string.

    The service provider is informed of the result in the form of a JSON web
    token signed with our RSA private key. If the proofs verified, then this token
    contains the attributes. If there was no `validity` specified in the request
    from the service proider, the web token's validity is set to 60 seconds. If the
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
             "proofs": [ "List with ProofDs" ],
             "nonce": 6.579489688380443e+23,
             "context": 6.843305777297634e+75
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
    that the token sends no reason, but this is what we want: the service provider
    does not need to learn if the user declined or if he does not have the required
    attributes.

*   `POST /api/v2/signature/checksignature`:
    This is a simple stateless api request that allows a service provider to
    check a signature without having IRMA software. It accepts a signature, as
    returned in the json web token and returns a status (with the same structure as
    the status object that has been included in the json web token).
    Note that a session token is not needed to execute
    this request. Also note that the response on this request is not signed with
    the api_server's private key.

### Intermezzo: technical context about challenge

An attribute-based signature is still a disclosure proof with a modified
challenge. The challenge is now calculated as:

```
    challenge = SHA256(ASN1Sig(nonce, SHA256(message)))
```

The asn1 encoding makes sure that we have a clear separation between IRMA
disclosure proofs and IRMA signatures: in the case of signatures, we additionaly
encode a boolean, which is set to true, in front of the challenge. This was in
the original paper proposed as the `dbit`.

### To do

* timestamps in signatures
* messageTypes other than string
* signing 'on demand' (i.e. open a file with the irma-app to sign it without
  using an SP or api server)
