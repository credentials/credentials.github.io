---
layout: page
title: Getting started with IRMA
---

IRMA is a distributed, attribute-based authentication technique which is very privacy-friendly.
This page provides a short overview on how to start using IRMA.
You can find more information on the IRMA project [here](https://www.irmacard.org/).

## IRMA user client
Users will need an IRMA client to manage their attributes.
Currently, there are two user client implementations of IRMA.
A smart card version and a much newer Android app.

### IRMA on a smart card
IRMA development started out on a smart card.
This code is no longer actively maintained and new users are encouraged to look at the IRMA app instead.
However, the smart card code is still functional and you can find it here: [idemix terminal](https://github.com/credentials/idemix_terminal)

### IRMA app
The IRMA app is currently the most up-to-date version of user-side IRMA stuff.
The IRMA app is available for download directly from the [Google Play store](https://play.google.com/store/apps/details?id=org.irmacard.cardemu).
If you have no access to the Google Play store, or do not wish to get the app from there, you can also find a binary [here](https://www.irmacard.org/irmaphone/#install).
Alternatively, you can build the app from the publicly available [source code](https://github.com/credentials/irma_android_cardemu).
Be aware that only an install via the app store will automatically update to the newest version.

After installing the app, users can obtain their first credentials through a self-enrollment process.
They can either convert the data from an electronic identity document to credentials, or they can self-issue credentials online through a demo enrollment website.

After obtaining attributes, users can use them to authenticate to service providers.
Several demos can be found on [demo.irmacard.org](https://demo.irmacard.org/).

More information on installing and using the IRMA app van be found [here](https://www.irmacard.org/irmaphone/).


## Verifying and Issuing credentials
If you want to verify and/or issue credentials you will need to run one of the following projects.

### [IRMA JavaScript client](https://github.com/credentials/irma_js)
[irma_js](https://github.com/credentials/irma_js) is a JavaScript client that will talk to an IRMA-API server, which does the actual verifying and issuing.
This JavaScript client essentially connects your webpage logic to the verification/issuing process, making it very easy to deploy IRMA technology on your websites.
The irma_js client can contact our own demo API server, so you can get started with only this JavaScript client.
When that works you can always look into running your own API server.

### [API server](https://github.com/credentials/irma_api_server)
The [API server](https://github.com/credentials/irma_api_server) handles all IRMA-specific cryptographic details of issuing and verifying attributes on behalf of the service or identity provider.
If you wish to run your own API server you can find the code and instructions [here](https://github.com/credentials/irma_api_server).

<!---
## Adding new credentials
New attributes can be added to the IRMA configuration project.
--->

## Support or Contact
Having trouble with the IRMA usage or development? Contact <a href="mailto:phone@demo.irmacard.org">phone@demo.irmacard.org</a> and we’ll help you sort it out.
