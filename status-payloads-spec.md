# Status Message Payloads Specification

> Version: 0.1 (Draft)
>
> Authors: Adam Babik <adam@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

## Abstract

This specifications decribes how the payload of each message in Status looks
like. It is primarly centered around chat and chat-related use cases.

The payloads aims be flexible enough to support messaging but also cases
described in the [Status Whitepaper](https://status.im/whitepaper.pdf) as well
as various clients created using different technologies.

## Table of Contents

- [Status Message Payloads Specification](#status-message-payloads-specification)
  - [Abstract](#abstract)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Payload wrapper](#payload-wrapper)
  - [Encoding](#encoding)
  - [Message](#message)
    - [Payload](#payload)
    - [Content types](#content-types)
    - [Message types](#message-types)
    - [Clock vs Timestamp and message ordering](#clock-vs-timestamp-and-message-ordering)
  - [Chats](#chats)
  - [Upgradability](#upgradability)
  - [Security Considerations](#security-considerations)
  - [Design rationale](#design-rationale)
    - [Why are you using Transit and Protobuf?](#why-are-you-using-transit-and-protobuf)

## Introduction

In this document we describe the payload format and some special considerations.

## Payload wrapper

All payloads are wrapped in a [protobuf record](https://developers.google.com/protocol-buffers/)
record:

```protobuf
message StatusProtocolMessage {
  bytes signature = 4001;
  bytes payload = 4002;
}
```

`signature` is the bytes of the signed `SHA3-256` of the payload, signed with the key of the author of the message.
The signature is needed to validate authorship of the message, so that the message can be relayed to third parties.
If a signature is not present but an author is provided by a layer below, the message is not to be relayed to third parties and its considered plausibly deniable.

## Encoding

The payload is encoded using [Transit format](https://github.com/cognitect/transit-format). Transit was chosen over JSON in order to reduce the bandwidth.

Example of a valid encoded payload:

```
["~#c4",["abc123","text/plain","~:public-group-user-message",154593077368201,1545930773682,["^ ","~:chat-id","testing-adamb","~:text","abc123"]]]
```

The message is an array and each index value has its meaning:
* 0: `c4` is a decoder handler identification for the current payload format. Identifications allow to register handlers for many different types of payload
* 1: array which items correspond to the described payload fields above

For more details regarding serialization and deserialization please consult [transit format](https://github.com/cognitect/transit-format) specification.

<!-- TODO: This requires a lot more detail since c4 is only one of several types, and also possibly links to implementation
ANDREA: Not sure this section is really needed (other then a brief mention of the fact that we use transit), explaining how transit is encoded is outside of the scope of this document, as well because that's not the only way transit can be encoded. -->

## Message

The type `Message` represents a text message exchanged between clients.

<!-- TODO: It is not clear how this relates to StatusProtocolMessage above -->

### Payload

Payload is a struct (a compound data type) with the following fields (order is important):

<!-- TODO: Be more precise in struct description, a la RFC, e.g. TLS style https://tools.ietf.org/html/rfc8446 -->

| Field | Name | Type |
| ----- | ---- | ---- |
| 1 | text | `string` |
| 2 | content type | `enum` (more in [Content types](#content-types)) |
| 3 | message type | `enum` (more in [Message types](#message-types)) |
| 4 | clock | `int64` |
| 5 | timestamp | `int64` |
| 6 | content | `struct { chat-id string, text string }` |

### Content types

Content types are required for a proper interpretation of incoming messages. Not each message is a plain text but may carry a different information.

The following content types MUST be supported:
* `text/plain` identifies a message which content is a plain text.

There are also other content types that MAY be implemented by the client:
* `sticker`
* `status`
* `command`
* `command-request`
* `emoji`

These are currently underspecified. We refer to real-world implementations for clients who wish to interoperate.

<!-- TODO: Ideally specify this, but barring that, link to implementation. -->

### Message types

Message types are required to decide how a particular message is encrypted and what metadata needs to be attached when passing a message to the transport layer. For more on this, see [Status Whisper Usage Specification](./status-whisper-usage-spec.md).

<!-- TODO: This reference is a bit odd, considering the layer payloads should interact with is Secure Transport, and not Whisper. This requires more detail -->


The following messages types MUST be supported:
* `public-group-user-message` is a message to the public group
* `user-message` is a private message
* `group-user-message` is a message to the private group.

### Clock vs Timestamp and message ordering

`timestamp` MUST be Unix time calculated when the message is created. Because the peers in the Whisper network should have synchronized time, `timestamp` values should be fairly accurate among all Whisper network participants.

`clock` SHOULD be calculated using the algorithm of [Lamport timestamps](https://en.wikipedia.org/wiki/Lamport_timestamps). When there are messages available in a chat, `clock`'s value is calculated based on the last received message in a particular chat: `last-message-clock-value + 1`. If there are no messages, `clock` is initialized with `timestamp * 100`'s value.

`clock` value is used for the message ordering. Due to the used algorithm and distributed nature of the system, we achieve casual ordering which might produce counterintuitive results in some edge cases. For example, when one joins a public chat and sends a message before receiving the exist messages, their message `clock` value might be lower and the message will end up in the past when the historical messages are fetched.

<!-- TODO: Document section on replies 
     TODO: Document timestamp, is it in seconds/ms ? -->

## Chats
<!-- This section should probably fall under Message, as it's only valid for Message-type messages -->

Chat is a structure that helps organize messages. It's usually desired to display messages only from a single recipient or a group of recipients at a time and chats help to achieve that.

All incoming messages can be matched against a chat. Below you can find a table that describes how to calculate a chat ID for each message type.

|Message Type|Chat ID Calculation|Direction|Comment|
|------------|-------------------|---------|-------|
|public-group-user-message|chat ID is equal to a public channel name; it should equal `chat-id` from message's `content` field|Incoming/Outgoing||
|user-message|let `P` be a public key of the recipient; `hex-encode(P)` is a chat ID; use it as `chat-id` value in message's `content` field|Outgoing||
|user-message|let `P` be a public key of message's signature; `hex-encode(P)` is a chat ID; discard `chat-id` from message's `content` field|Incoming|if there is no matched chat, it might be the first message from public key `P`; you can discard it or create a new chat; Status official clients create a new chat|
|group-user-message|use `chat-id` from message's `content` field|Incoming/Outgoing|find an existing chat by `chat-id`; if none is found discard the message (TODO: incomplete)|

<!-- TODO: "group-user-message" is not complete. Does it require to explicitly join the group chat? Is there a way to invite someone? Also, if I start a new group chat (or join an existing one), I need to somehow calculate this chatID by myself. How to do it? -->

## Upgradability

The current protocol format is hardly upgradable without breaking backward compatibility. Because Transit is used in this particular way described above, the only reliable option is to append a new field to the Transit record definition. It will be simply ignored by the old clients.
<!-- Not sure I agree with this statement, seems very arbitrary, appending to an array is just as upgradable as adding an entry in a map, just less convenient, I would remove the qualitative statement, and just describe how to upgrade -->

## Security Considerations

TBD.

## Design rationale

### Why are you using Transit and Protobuf?

Transit was initially chose for encoding, and Protobuf was added afterwards. This is partly due to the history of the protocol living inside of `status-react`, which is written in Clojurescript. In future versions of payload and data sync client specifications it is likely we'll move towards Protobuf only. See e.g. [Dasy](https://github.com/vacp2p/dasy) for a research proof of concept.
<!-- I would remove the link to dasy, I find it a bit confusing, and the repo just implements something totally different and a fraction of the functionalities -->
