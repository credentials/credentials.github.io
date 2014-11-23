---
layout: page
title: CardProxy architecture
permalink: /docs/cardproxy-architecture/
---

*For now this is just a short write-up about why the card-proxy application communicates via the browser. It contains some content that should be moved to other articles and it is as of yet incomplete.*

# Using an NFC phone as a card reader

One of the bigger downsides of using a smart card as the primary carrier of attribute-based credentials is the scarcity of the card readers that enable their use. That is why the IRMA project promotes the use of NFC-enabled smart phones as an much wider spread alternative to regular card readers. We already have NFC-enabled applications like [the IRMA android card verifier application](https://github.com/credentials/irma_android_verifier) that can be used as a stand-alone off-line verifier, and [the IRMA android management application](https://github.com/credentials/irma_android_management) that enables users to manage their cards. In this document we will discuss a third application, [the IRMA android card proxy](https://github.com/credentials/irma_android_cardproxy).

The card proxy application covers a different use case: that of online verification of IRMA credentials. Traditionally, you would connect a card reader to you machine, and interface that reader to the browser (in our case we support this via a Java plugin). We will use the NFC-enabled smart phone as an alternative. However, the smart phone is only one of the alternatives. In the future we foresee several methods of interacting with a smart card from the browser:

 1. Using a regular card reader that is connected to the user's machine via USB. The card reader communicates with the browser using one of the following methods (the latter two are not yet supported):
    1. A Java applet in the browser directly connects to the card reader. Due to problems with Java, this method is no longer recommended, but it is supported by IRMA.
    2. The user installs a small daemon on its machine. The browser makes a local network connection to this daemon to talk to the card reader. This method is quite common in for example banking applications. IRMA does not yet support this method.
    3. A browser plugin enables direct communication to the card reader from the browser using, for example, javascript.
 2. The user uses his smart phone as a card reader. The card reader somehow connects to the existing infrastructure, and acts as a card reader. Most likely the smart phone is accessed via the network.

In the remainder of this document we'll see how can enable the second option, without making it more complicated to support the first options as well.
