---
layout: page
title: IRMA documentation
---

The IRMA documentation can be found here: https://irma.app/docs

<!---
IRMA is a distributed, attribute-based authentication platform which is very privacy-friendly.
It is at its core an implementation of the [Idemix attribute-based credential scheme](https://eprint.iacr.org/2001/019).
This page provides a short technical overview of IRMA.
You can find more general information on the IRMA project [here](https://privacybydesign.foundation/irma-en).
A much more detailed technical introduction to IRMA is available [here](/docs/irma.html).
All other technical IRMA documentation can be found [here](https://privacybydesign.foundation/documentation).

## IRMA client
Users will need an IRMA client to manage their attributes.
Currently, there are two client implementations of IRMA.
A [smart card version](https://github.com/privacybydesign/idemix_terminal)
which is no longer maintained, and a much newer and more versatile Android and iOS app.

### IRMA app
The IRMA app is currently the only maintained IRMA client.
The IRMA app is available for download directly from the [Google Play store](https://play.google.com/store/apps/details?id=org.irmacard.cardemu) and from the [Apple App Store](https://itunes.apple.com/us/app/irma-authentication/id1294092994).
If you have no access to the Google Play store, or do not wish to get the app from there,
you can also find a binary [here](https://privacybydesign.foundation/irma.apk).
Alternatively, you can build the app from the publicly available [source code](https://github.com/privacybydesign/irma_mobile).
Be aware that only an install via the app store will automatically update to the newest version.

After installing the app, users obtain their first credentials through a registration process.
After obtaining attributes, users can use them to authenticate to service providers.
Several demos can be found on [here](https://privacybydesign.foundation/demo-en/).

More information on installing and using the IRMA app can be found [here](https://privacybydesign.foundation/irma-begin/).

## Verifying and Issuing credentials
If you want to verify and/or issue credentials you can use the following projects.

### [IRMA API server](https://github.com/privacybydesign/irma_api_server)
The [API server](https://github.com/privacybydesign/irma_api_server) handles all IRMA-specific
cryptographic details of issuing and verifying attributes on behalf of the service or
identity provider. It sits between IRMA tokens on the one hand, and authorized service or
identity providers on the other hand. It exposes a RESTful JSON API driven by JWTs for authentication.
The protocol that the IRMA API server and the IRMA Android and iOS apps speak is documented
[here](/protocols/irma-protocol/).
If you wish to run your own API server you can find the code and instructions [here](https://github.com/privacybydesign/irma_api_server).

### [IRMA Javascript client](https://github.com/privacybydesign/irma_js)
[irma_js](https://github.com/privacybydesign/irma_js) is a Javascript client of the
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

### Project descriptions

 * [`irma_mobile`](https://github.com/privacybydesign/irma_mobile): A cross-platform iOS and Android mobile IRMA app,
 * [`irmago`](https://github.com/privacybydesign/irmago): the underlying IRMA client implementation in Go; also parses the `irma_configuration` folder, and contains structs that serve as the messages in the IRMA protocol
 * [`irma_api_common`](https://github.com/privacybydesign/irma_api_common): Java library that parses the `irma_configuration` folder; contains our Idemix implementation, and contains classes that serve as the messages in the IRMA protocol
 * [`irma_api_server`](https://github.com/privacybydesign/irma_api_server): server for issuing and verifying attributes
 * [`irma_js`](https://github.com/privacybydesign/irma_js): JavaScript frontend for easy handling of issuing and disclosure sessions with an `irma_api_server`
 * The [`pbdf`](https://github.com/privacybydesign/pbdf-schememanager) and [`irma-demo`](https://github.com/privacybydesign/irma-demo-schememanager) scheme managers contain credential descriptions, issuer descriptions, and public and possibly private keys of issuers, grouped in scheme managers. These should be put in an `irma_configuration` folder for use by the `irma_mobile` app or `irma_api_server`.
* [`gabi`](https://github.com/mhe/gabi), Idemix implementation in Go used by `irmago`.

## Support or Contact
Having trouble with the IRMA usage or development? Contact `irma 'at' privacybydesign.foundation` and we’ll help you sort it out.

--->
