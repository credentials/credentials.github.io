---
layout: page
title: Technical introduction to IRMA
---

IRMA is a distributed, attribute-based authentication technique which is very privacy-friendly.
This page provides a short overview on how to start using IRMA.
You can find more general information on the IRMA project [here](https://privacybydesign.foundation/irma).
A much more detailed technical introduction to IRMA is available [here](/docs/irma.html)

## IRMA user token
Users will need an IRMA token to manage their attributes.
Currently, there are two user token implementations of IRMA.
A [smart card version](https://github.com/credentials/idemix_terminal)
which is no longer maintained, and a much newer and more versatile Android and iOS app.

### IRMA app
The IRMA app is currently the most up-to-date version of user-side IRMA stuff.
The IRMA app is available for download directly from the [Google Play store](https://play.google.com/store/apps/details?id=org.irmacard.cardemu) and from the [Apple App Store](https://itunes.apple.com/us/app/irma-authentication/id1294092994).
If you have no access to the Google Play store, or do not wish to get the app from there,
you can also find a binary [here](https://privacybydesign.foundation/irma.apk).
Alternatively, you can build the app from the publicly available [source code](https://github.com/privacybydesign/irma_mobile).
Be aware that only an install via the app store will automatically update to the newest version.

After installing the app, users can obtain their first credentials through a registration process.
After obtaining attributes, users can use them to authenticate to service providers.
Several demos can be found on [demo.irmacard.org](https://demo.irmacard.org/).

More information on installing and using the IRMA app can be found [here](https://privacybydesign.foundation/irma-begin/).

## Verifying and Issuing credentials
If you want to verify and/or issue credentials you can use the following projects.

### [IRMA API server](https://github.com/credentials/irma_api_server)
The [API server](https://github.com/credentials/irma_api_server) handles all IRMA-specific
cryptographic details of issuing and verifying attributes on behalf of the service or
identity provider. It sits between IRMA tokens on the one hand, and authorized service or
identity providers on the other hand. It exposes a RESTful JSON API driven by JWTs for authentication.
The protocol that the IRMA API server and the IRMA Android and iOS apps speak is documented
[here](/protocols/irma-protocol/).
If you wish to run your own API server you can find the code and instructions [here](https://github.com/credentials/irma_api_server).

### [IRMA Javascript client](https://github.com/credentials/irma_js)
[irma_js](https://github.com/credentials/irma_js) is a Javascript client of the
RESTful JSON API offered by the IRMA API server, which does the actual verifying and issuing.
This JavaScript client essentially connects your webpage logic to the verification/issuing
process, making it very easy to deploy IRMA technology on your websites.
The irma_js client can contact our own demo API server, so you can get started with only this JavaScript client.
When that works you can always look into running your own API server.

### IRMA session flow

The following image shows the dataflow between the IRMA software components in a typical IRMA session.

![IRMA flow](images/irmaflow.svg)

Explanation of the steps:

  1. The requestor (i.e., the service or identity provider wanting to verify or issue attributes) provides
     [a JWT](protocols/irma-protocol/#the-protocol) containing an IRMA session request,
     along with success and failure callbacks to `irma_js`
  2. `irma_js` `POST`s the JWT to the API server
  3. The API server replies with an IRMA session token
  4. `irma_js` renders the session token along with the URL to the API server in a QR that the IRMA app scans
  5. The IRMA app contact the API server, and they perform the actual IRMA session
  6. The API server informs `irma_js` of the result (in the case of a successful disclosure session, this includes a JWT containing the disclosed attributes)
  7. `irma_js` informs the requestor via the callbacks provided in step 1, including the disclosed attributes in verification sessions

<!---
## Adding new credentials
New attributes can be added to the IRMA configuration project.
--->

### Dependency graph

The following image shows the relationships between the most important IRMA projects. Legend: Ellipses are Java projects; rectangles are static files; normal arrows mean "depends on".

![IRMA stack](images/stack.svg)

### Project descriptions

 * [`irma_configuration`](https://github.com/credentials/irma_configuration): contains credential descriptions, issuer descriptions, and public and possibly private keys of issuers, grouped in scheme managers
 * [`credentials_api`](https://github.com/credentials/credentials_api): library that parses the credential and issuer descriptions from `irma_configuration`, and defines some of the semantics of attribute-based credential schemes
 * [`credentials_idemix`](https://github.com/credentials/credentials_idemix): library containing our Idemix implementation. Also parses the Idemix public and private keys from `irma_configuration`
 * [`irma_api_common`](https://github.com/credentials/irma_api_common): library containing classes that serve as the messages in the IRMA protocol, between the server (`irma_api_server`) and client (`irma_android_cardemu`)
 * [`irma_api_server`](https://github.com/credentials/irma_api_server): server for issuing and verifying attributes
 * [`irma_js`](https://github.com/credentials/irma_js): JavaScript frontend for easy handling of issuing and disclosure sessions with an `irma_api_server`
 * [`irma_android_cardemu`](https://github.com/credentials/irma_android_cardemu): the IRMA Android token

Additionally:

* [`irma_mobile`](https://github.com/privacybydesign/irma_mobile): A cross-platform iOS and Android mobile IRMA app,
* [`irmago`](https://github.com/privacybydesign/irmago): the underlying IRMA client implementation in Go,
* [`gabi`](https://github.com/mhe/gabi), Idemix implementation in Go used by `irmago`.

## Support or Contact
Having trouble with the IRMA usage or development? Contact `irma 'at' privacybydesign.foundation` and we’ll help you sort it out.
