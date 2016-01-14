---
layout: page
title: "Proposal: IRMA without APDU's"
permalink: /proposals/irma-without-apdus/
---
_**This document is still work in progress!**_


Until recently the IRMA token has always been a smart card as it offers very high levels of security. This focus allowed the IRMA system to thrive, but also imposed some restrictions on the system. Recently, we have been exploring other token carriers, like a smart phone. To enable working with the smart card, the interface has always been defined in terms of APDUs (the low level commands that you send to your smart card). However, when working with other tokens (for example a smart phone), this is not a natural interface.

We propose a new protocol here for requesting and sending disclosure proofs between the token and the verifier, based on JSON instead of on APDU's. For now we only consider disclosures; in the future we will expand to issuing.

# The setup

The workflow for the user should be as follows. Suppose she wants to watch a video on IrmaTube; we assume that she already has the proper credential on her token, which we assume to be the cardemu app for now (but see the section "Smart cards" below).

* The IrmaTube website displays a QR code as before,
* The user scans it,
* Her token either informs her that she does not have the required credentials or attributes, or asks her for permission to disclose the attributes that IrmaTube asked for. If she consents, her token sends a disclosure proofs to IrmaTube.

For code reusability and maintainability, it makes sense to split the logic of the website and the cryptography of verifying the credentials into separate projects. For that reason we have created the [`irma_api_server`](https://github.com/credentials/irma_api_server), which handles the cryptography and the communication with the token. This server sits between the service provider (the IrmaTube website in the example above) and the token, and generally works as follows.

* It accepts a disclosure proof request from the service provider (i.e., the IrmaTube website), and returns a session token.
* It is then the task of the service provider (the IrmaTube website) to inform the token of this session token and where to reach the `irma_api_server` - for example by using a QR code.
* The token connects to the `irma_api_server`, sending the session token. The `irma_api_server` informs the token of the required attributes. If the token does not possess these attributes, it aborts. Otherwise, it informs the user of the required attributes. If the user consents, the token creates the disclosure proof and sends it to `irma_api_server`.
* The `irma_api_server` verifies the proof. If anything went wrong, it reports failure. Otherwise, it sends the attributes contained in the proof back to the service provider (the IrmaTube website), in the form of a signed JSON web token.

Notice that the IrmaTube website now does almost nothing, apart from showing the QR-code; the token itself is responsible for informing its owner about what is happening. However, the service provider should be aware of the (RSA) public key with which `irma_api_server` signs the JSON web tokens, so that it can verify these.

# Smart cards

We can keep the smart cards in the game as follows:

* We build a translator that can translate our new protocol back and forth to APDU's.
* We endow the `irma_android_cardproxy` with this translator, and develop a similar desktop application for physical cardreaders that also uses this translator. These two apps can keep the user up-to-date through their GUI, but otherwise the card is authoritative.
* The service provider puts the URL to the verifier server in a new URL scheme like  `irma://something/something`
* The `irma_android_cardproxy` can scan QR codes like the cardemu can, but the desktop application will have to be notified of new sessions in some other way - perhaps the website explicitly shows the session key to the user, who can then copy and paste it into the desktop app.

# The protocol

We first focus on the protocol between the `irma_api_server` and the token. We first define a _disclosure proof request_ data type, that looks as follows.

```json
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
```

The `nonce` and `context` are optional large integers. Each entry of `content` is a disjunction of attributes, along with a label that can be shown to the user of the token. The disjunctions themselves should be ANDed together. Thus, this example asks for `MijnOverheid.ageLower.over18` or `Thalia.age.over18`.

Credential-only proofs are supported as follows. When one of the disjunctions contains an identifier of the form `issuer.credential` in the `attributes` list (for example, `MijnOverheid.ageLower`), the token should disclose none of the attributes contained in the credential apart from the metadata attribute. Since the metadata attribute is invisible to the user, this means that from the perspective of the user no attributes of the credential are disclosed, so that he only proves possession of the credential.

The `irma_api_server` is a web server listening at the following paths.

*   `POST /api/v2/verification`: accepts requests from
    service providers of the following form:

    ```json
    {
        "data": "...",
        "validity": "60",
        "request": "..."
    }
    ```
    Here `data` can be any string of the service provider's choosing, while `request` is a disclosure proof request (without a nonce or context). `validity` specifies how long the returned JSON web token should be valid (in seconds). Only `request` is required, the other two are optional (the default value of `validity` is 60 seconds). In response, the server returns a verification ID.

*   `GET /api/v2/verification/verificationID`: if `verificationID` is a
    valid session token (i.e., it has been assigned to a disclosure proof request at some point in the past), the server generates a nonce, puts this in the disclosure proof request associated to this session token, and returns this to the token.
*   `POST /api/v2/verification/verificationID/proofs`: if `verificationID`
    is a valid session token, this accepts a serialized `ProofList` object, that contains one or more disclosure proofs (`ProofD`s). The proofs are verified immediately, and the server informs the client (i. e., the IRMA token, not the service provider) of the validity of the proofs in the form of a string.

    The service provider is informed of the result in the form of a JSON web token signed with our RSA private key. If the proofs verified, then this token contains the attributes. If there was no `validity` specified in the request from the service proider, the web token's validity is set to 60 seconds. If the `data` field was present then this is included in the web token in the [`jti` field](https://tools.ietf.org/html/rfc7519#section-4.1.7). For example, (the payload of) a returned JSON web token might look as follows.

    ```json
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
    ```

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

# To do

* Authentication between `irma_api_server` and the service provider
* Issuing
