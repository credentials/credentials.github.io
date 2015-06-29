---
layout: page
title: Proposal: two interfaces for IRMA tokens
permalink: /proposals/two-interfaces-to-IRMA/
---

*This document is still work in progress!*

Until recently the IRMA token has always been a smart card as it offers very high levels of security. This focus allowed the IRMA system to thrive, but also imposed some restrictions on the system. Recently, we have been exploring other token carriers, like a smart phone. To enable working with the smart card, the interface has always been defined in terms of APDUs (the low level commands that you send to your smart card). However, when working with other tokens (for example a smart phone), this is not a natural interface. Hence we propose two interfaces for interacting with an IRMA token:

 1. A low level APDU interface for communicating with smart cards
 2. A higher level JSON interface for communicating with more powerful tokens.

# Creating a simple disclosure proof

To demonstrate that the user has a credential the token creates a disclosure proof in response to a nonce. A verifier checks this proof for correctness. So conceptually, to show that you have a credential, the proof is what you sent.

However, such a proof consists of multiple values (one for each attribute, one for the public key, and some more for the CL-signature). When using the APDU interface multiple commands are send:

 * A start proof command, to inform the card's state machine that it will now create a disclosure proof
 * A command to set the nonce, the card will internally create the proof
 * Commands to retrieve the blinded CL-Signature
 * Commands to retrieve the disclosed attributes
 * Commands to retrieve the responses for the hidden attributes.

The responses to these APDUs can be combined to recover the original proof. Compare this to the high level protocol that it actually implements:

 1. Send the nonce to the token
 2. The token replies with a disclosure proof (containing the desired values)

A similar comparison can be made for the issuing of credentials (although that is a three step protocol). Up to this point, this is just a translation/aggregation of values. However, the secure channel (while necessary) makes this approach impossible.

# A secure channel

We do not use this basic approach directly. Instead we setup uan authenticated secure channel  between the token and the verifier (both the verifier and the card are authenticated, the card remains anonymous). This solves the following problems:

 1. If the verifier is not authenticated there is no way to restrict what credentials and attributes the verifier can see.
 2. The card can only produce one proof at the time. However, sometimes you need to show two credentials, and demonstrate that they belong to the same person. At a high level, you could bind them together mathematically, but the smart card cannot produce this proof. Using the secure channel binds the two showings together.
 3. The values are normally transmitted over NFC, but the attributes and their authenticity is sensitive information. A secure channel protects them from this.

For a token that can deal with a higher level protocol, like a phone, we can solve (1) and (3) differently, for example by signing the verifiers request and using a TLS channel directly. However, option (2) really requires the mathematical approach of binding credentials together. So we cannot directly translate between a low level APDU interface and a higher level JSON interface as far as showing multiple credentials is concerned.

# Design choices due to APDU interface

Some of the design choices in the IRMA project are a direct result of using a smart card as a base of operations. A few of them are:

 1. The number of attributes per credential is limited
 2. Credential types are identified using a 2-byte id.
 3. Verification types are identified using a 2-byte id (not implemented).

# Summary

To summarize, we want two interfaces, one low-level (based on APDUs) and one high level (based on for example JSON, although the carrier format does not matter). The proofs themselves are easily translated between these two forms. However, the other required properties imply that these two version of the protocol necessarily have to diverge.
