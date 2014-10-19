---
layout: page
title: Card Reader Protocol, version 2
permalink: /protocols/card-reader-protocol-v2/
---

*Version: 2* **Work in Progress**

This protocol offers a uniform interface to speak to card readers. These readers can be local, i.e. connected directly to the user's machine via a USB card reader, but can also be proxied via a card reader located elsewhere.

The uniform interface provides regular card reader functionality:

 * Sending APDUs to the card and receive the responses
 * Send notifications about card insertions and removals
 * Request the card reader to show a PIN dialog
 * Show status messages on the reader's display

History
-------

This document describes version 2 of this protocol. The predecessor of this protocol, version 1, was used in version 0.8 of the `irma_android_cardproxy`.


Message structure
=================

All messages are formatted as JSON messages and have the following general structure:

    {
      "type": "<messagetype>",
      "name": "<message_name>",
      "id": "<message_id>",
      "arguments": <JSON>, 
    }

The main types are: `command`, `response`, and `event`. The id is used to connect commands and responses. It is optional for events. The name field indicates what command, response or event is sent. The version is used to distinguish versions of this protocol. We will specify them shortly.

Messages can be accompanied by extra data. For example, the APDUs to send to the card, the responses to these APDUs and status messages. These are stored in the arguments field.

Commands
--------

There are types of commands that can be send to a card reader:

 * `setProtocolVersion` to determine which version of the protocol to use 
 * `authorizeWithPin` to ask the user to enter the PIN code
 * `transmitCommandSet` to send a batch of commands to the card

To sends such a command, set the type field to `command`, pick a unique id (for example by keeping a counter), lets say 42, and create the resulting JSON structure:

    {
      "type": "command",
      "name": "authorizeWithPin",
      "id": "42",
      "arguments": {},
    }

The `authorizeWithPin` command doesn't take an argument. Others do. All commands, which we describe in detail next, result in a corresponding `response`, see the next section.

### setProtocolVersion 

This command should be the first command that the application executes after receiving the [`cardReaderFound` event](#eventCardReaderFound). It will tell the card reader which version of the protocol the application wants to use. The desired version should be one of the versions that is supported by the reader.

The partial for this command is:

    {
        "type": "command"
        "name": "setProtocolVersion",
        "id": "<message_id>",
        "arguments": { desired_version: <desired_version> }
    }

The desired version is an integer. 

### authorizeWithPin

The `authorizeWithPin` command requests the user to enter his/her pin on the card reader. It is the responsibility of the implementing side to offer a software alternative if a hardware pin pad is not available or not supported.  This command takes no arguments.

The partial for this command is:

    {
        "type": "command"
        "name": "authorizeWithPin",
        "id": "<message_id>",
        "arguments": {}
    }

### transmitCommandSet

The `transmitCommandSet` sends a batch of APDUs to the card. As arguments it takes a list of APDUs that has to be sent to the card (in the order in which they appear in the list). An APDU (or `ProtocolCommand` in the Java stack) is represented by the hex encoded string of the APDU that is to be sent. For example, the select applet APDU would be encoded as:

    "00A4040009F849524D416361726400"
    
The complete partial for this command is:

    {
        "type": "command"
        "name": "transmitCommandSet",
        "id": "<message_id>",
        "arguments": {
            "commands": [
                <hex_encoded_apdu>, ...
                ],
        },
    }

Responses
---------

For each of the three commands we described above there is a corresponding set of responses.

### setProtocolVersion 

When the reader is asked to set the protocol version there are two possible outcomes:

 * `success`: the reader supports the desired version, and accepts the proposal
 * `failure`: the reader does not support the desired version.

The partial for this response is:

    {
        "type": "response"
        "name": "setProtocolVersion",
        "id": "<message_id>",
        "arguments": { result: <success/failure> }
    }


### authorizeWithPin

When asked to enter the pin the user has the option to abort/cancel, furthermore, the card can be blocked when a wrong PIN is entered too frequently. This gives three possible outcomes:

 * `success`: the card accepted the PIN entered by the user
 * `cancel`: the user cancelled the operation
 * `blocked`: the card is blocked, a wrong PIN was entered too often.

 *This may be IRMA specific. In particular, I'm not very familiar with the smart card practices, it may be that the way we send a PIN to the card and the way the card returns a counter of available tries is in fact the normal way.*

The partial for this response is:

    {
        "type": "response"
        "name": "authorizeWithPin",
        "id": "<message_id>",
        "arguments": {"result": "<success/blocked/cancel>"},
    }

### transmitCommandSet

The response to a batched set of commands is a list of responses, sent in the same order as the commands. The card reader is required to abort when it receives a non-successful response code (so the list of responses may be shorter than the corresponding list of commands).  The responses themselves are again hex encoded strings. For example, the response to the select applet APDU described above could be **TODO**:

    "SOMETHING_FIXME"

The complete partial for this response is:

    {
        "type": "response"
        "name": "transmitCommandSet",
        "id": "<message_id>",
        "arguments": {
            "responses": [
                "<hex_encoded_response_apdu>", ...
                ],
        },
    }


Events
------

The card reader can send and receive events. The card reader can send the following events to the application:

 * `cardReaderFound` to notify that there is a card reader on the other side
 * `cardInserted` to notify that a card has been inserted
 * `cardRemoved` to notify that a card has been removed

In addition, the application can send the following events to the card reader:

 * `statusUpdate` to change the status display on the reader
 * `doneCard` to notify the card reader that application is finished with the current card.
 * `done` to notify that the application is done with the reader and will disconnect.

We specify these next.

### cardReaderFound <a name="eventCardReaderFound"></a>

The `cardReaderFound` event is sent to the application whenever a card reader connects. It is also the first message that is sent over the channel. In this sense it acts as a "Hello, here I am"-message.

It is useful to separate this from the channel itself, as the channel might be forwarded, in which case a connecting channel does not mean that there is a card reader on the other side.

The arguments describe the type of reader (currently the only option is `phone`), and the versions that are supported by the reader. A protocol version is indicated by an integer.

The partial for this event is:

    {
        "type": "event"
        "name": "cardReaderFound",
        "arguments": {
            "type": <type_of_reader>,
            "supported_versions": [<version1>, <version2>, ...],
        },
    }

### cardInserted

The reader sends this message is when a card is inserted. (We use the term inserted loosely, for a contactless card, it can also mean that it is presented to the reader).

The partial for this event is:

    {
        "type": "event"
        "name": "cardInserted",
        "arguments": {},
    }

### cardRemoved

The reader sends this message when it detects that the card is removed. This can happen due to a physical trigger, but also when during transmission of commands contact with the card is lost.

If the reader cannot detect card removal (this is the case on android) it should periodically test if the card is still present. When the card is no longer present, it should also send this event.

The partial for this event is:

    {
        "type": "event"
        "name": "cardRemoved",
        "arguments": {},
    }

### statusUpdate

This event is sent from the application to the card reader to display a message, and to change the status icon. There are four possible status types (in increasing order of severity):

 * `none`: don't display a special icon, for regular status updates
 * `success`: interaction with the card was successful
 * `warning`: something went wrong, warn the user
 * `failure`: the interaction failed, signal an error

Additionally, the application sends a feedback message that can be displayed by the reader. Readers that cannot display statusUpdates do not have to.

The partial for this event is:

    {
        "type": "event"
        "name": "cardRemoved",
        "arguments": {},
    }

### doneCard

When the application is done interacting with the card reader it sends a `doneCard` event to indicate this. The reader will remain connected to the application, in the state that it was after negotiating a protocol. This is useful if the application needs to communicate with many cards presented to the same reader.

The partial for this event is:

    {
        "type": "event"
        "name": "doneCard",
        "arguments": {},
    }

### done

When the application is done interacting with the card reader it sends a `done` event to indicate this. The reader shuts down (or returns to an idle state to connect with a new application).

The partial for this event is:

    {
        "type": "event"
        "name": "done",
        "arguments": {},
    }
