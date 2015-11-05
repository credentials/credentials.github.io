---
layout: page
title: "Proposal: IRMA without APDU's"
permalink: /proposals/irma-without-apdus/
---
_**This document is still work in progress!**_


Until recently the IRMA token has always been a smart card as it offers very high levels of security. This focus allowed the IRMA system to thrive, but also imposed some restrictions on the system. Recently, we have been exploring other token carriers, like a smart phone. To enable working with the smart card, the interface has always been defined in terms of APDUs (the low level commands that you send to your smart card). However, when working with other tokens (for example a smart phone), this is not a natural interface.

We propose a new protocol here for requesting and sending disclosure proofs between the token and the verifier, based on JSON instead of on APDU's. For now we deal with the disclosure of just one credential, instead of multiples that are bound by either cryptography or a safe channel. In the future we may expand to disclosures of multiple credentials as well as issuing.

# The setup

The workflow for the user should be as follows. Suppose she wants to watch a video on IrmaTube; we assume that she already has the proper credential on her token, which we assume to be the cardemu app for now (but see the section "Smart cards" below).

* The IrmaTube website displays a QR code as before,
* The user scans it,
* Her token either informs her that she does not have the required credentials or attributes, or asks her for permission to disclose the attributes that IrmaTube asked for. If she consents, her token sends a disclosure proofs to IrmaTube.

For code reusability and maintainability, it makes sense to split the logic of the website and the cryptography of verifying the credentials into separate projects. Thus we propose to create a new `irma_verification_server` which handles the cryptography and the communication with the token. This server sits between the verifier (the IrmaTube website in the example above) and the token, and generally works as follows.

* It accepts a disclosure proof request from the verifier (i.e., the IrmaTube website), and returns a session token.
* It is then the task of the verifier (the IrmaTube website) to inform the token of this session token and where to reach the `irma_verification_server` - for example by using a QR code.
* The token connects to the `irma_verification_server`, sending the session token. The `irma_verification_server` informs the token of the required attributes. If the token does not possess these attributes, it aborts. Otherwise, it informs the user of the required attributes. If the user consents, the token creates the disclosure proof and sends it to `irma_verification_server`.
* The `irma_verification_server` verifies the proof. If successful, it sends the attributes contained in the proof back to the verifier (the IrmaTube website). Otherwise, it reports failure.

Notice that the IrmaTube website now does almost nothing, apart from showing the QR-code; the token itself is responsible for informing its owner about what is happening. The `irma_verification_server` can be a Java web application much like the enrollment server. We do not focus on the trust relationship between the verifier and the `irma_verification_server` here; we assume here that these two just trust each other.

# Smart cards

We can keep the smart cards in the game as follows:

* We build a translator that can translate our new protocol back and forth to APDU's.
* We endow the `irma_android_cardproxy` with this translator, and develop a similar desktop application for physical cardreaders that also uses this translator. These two apps can keep the user up-to-date through their GUI, but otherwise the card is authoritative.
* The `irma_android_cardproxy` can scan QR codes like the cardemu can, but the desktop application will have to be notified of new sessions in some other way - perhaps the website explicitly shows the session key to the user, who can then copy and paste it into the desktop app.

# The protocol

We first focus on the protocol between the `irma_verification_server` and the token. The `irma_verification_server` is a web server listening at the following paths.

* `/start`: accepts a POSTed json object like the following: `{session: "$session"}`. If `$token` is a valid session token (i.e., it has been assigned to a disclosure proof request at some point in the past (TODO: how?)), the server replies with a JSON object like the following:

```
{
    "session": "...",
    "request": {
        "nonce": "...",
        "content": [
            {
                "label" = "Over 18",
                "attributes" = [
                    "MijnOverheid.ageLower.over18",
                    "Thalia.age.over18"
                ]
            },
            ...
        ]
    }
}
```
The `session` and the `nonce` are base64-encoded. Each entry of `content` is a disjunction of attributes, along with a label that can be shown to the user of the token. The disjunctions themselves should be ANDed together. Thus, this example asks for `MijnOverheid.ageLower.over18` or `Thalia.age.over18`.

* `/abort` accepts a POSTed json object `{session: "$session"}`. If the session exists it is deleted, and the verifier is somehow informed of failure (TODO).
* `/verify`: accepts a disclosure proof like the following.

```
{
    "session": "...",
    "proof": { ... }
}
```
  where the `proof` is a serialized `ProofD` object. If the proof verifies, the attributes are sent to the verifier.

# To do

* The protocol between the verifier `irma_verification_server` and the verifier
* Authentication between those two
* Issuing?

# To decide

* When the `irma_verification_server` informs the verifier of the session token, it could perhaps return a URL to a pregenerated QR-code instead of letting the verifier generate this QR code.
* If the `irma_verification_server` is a Java web application, i.e., a web server, how will it notify the verifier that the disclosure proof is ready? Perhaps the verifier is also a web server to which stuff can be POSTed, or the verifier will just have to poll the `irma_verification_server`?
* Instead of posting the session token in the JSON objects, we could include it in the URLs at which the `irma_verification_server` listens. This way we don't need to have these tokens as a member in the Java classes corresponding to the JSON objects.