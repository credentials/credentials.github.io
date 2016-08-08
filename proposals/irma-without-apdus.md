---
layout: page
title: "Proposal: IRMA without APDU's"
permalink: /proposals/irma-without-apdus/
---

This document has moved to [here](/protocols/irma-protocol).

Until recently the IRMA token has always been a smart card as it offers very high levels of security. This focus allowed the IRMA system to thrive, but also imposed some restrictions on the system. Recently, we have been exploring other token carriers, like a smart phone. To enable working with the smart card, the interface has always been defined in terms of APDUs (the low level commands that you send to your smart card). However, when working with other tokens (for example a smart phone), this is not a natural interface.

For this reason, we have created and implemented a more modern JSON-based protocol. The documentation of this protocol (i.e., the previous content of this document) is now no longer a proposal and can be found [here](/protocols/irma-protocol). The implementation is our [IRMA API server](https://github.com/credentials/irma_api_server).
