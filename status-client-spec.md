# Status Client Specification

> Version: 0.1 (Draft)
>
> Authors: Adam Babik <adam@status.im>, Dean Eigenmann <dean@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

## Abstract

In this specification, we describe how to write a Status client for
communicating with other Status clients.

We present a reference implementation of the protocol <sup>1</sup> that is used
in a command line client <sup>2</sup> and a mobile app <sup>3</sup>.

This document consists of two parts. The first outlines the specifications that
have to be implemented in order to be a full Status client. The second gives a design rationale and answers some common questions.

## Table of Contents
- [Status Client Specification](#status-client-specification)
  - [Abstract](#abstract)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
    - [Protocol layers](#protocol-layers)
  - [Components](#components)
    - [P2P Overlay](#p2p-overlay)
      - [Node discovery and roles](#node-discovery-and-roles)
      - [Bootstrapping](#bootstrapping)
      - [Discovery](#discovery)
      - [Mobile nodes](#mobile-nodes)
    - [Transport privacy and Whisper usage](#transport-privacy-and-whisper-usage)
    - [Secure Transport](#secure-transport)
    - [Data Sync](#data-sync)
    - [Payloads and clients](#payloads-and-clients)
  - [Security Considerations](#security-considerations)
    - [Censorship-resistance](#censorship-resistance)
  - [Design Rationale](#design-rationale)
    - [P2P Overlay](#p2p-overlay-1)
      - [Why devp2p? Why not use libp2p?](#why-devp2p-why-not-use-libp2p)
      - [What about other RLPx subprotocols like LES, and Swarm?](#what-about-other-rlpx-subprotocols-like-les-and-swarm)
      - [Why do you use Whisper?](#why-do-you-use-whisper)
      - [I heard you were moving away from Whisper?](#i-heard-you-were-moving-away-from-whisper)
      - [Why is PoW for Whisper set so low?](#why-is-pow-for-whisper-set-so-low)
      - [Why do you not use Discovery v5 for node discovery?](#why-do-you-not-use-discovery-v5-for-node-discovery)
      - [I heard something about mailservers being trusted somehow?](#i-heard-something-about-mailservers-being-trusted-somehow)
    - [Data sync](#data-sync)
      - [Why is MVDS not used for public chats?](#why-is-mvds-not-used-for-public-chats)
  - [Footnotes](#footnotes)
  - [Acknowledgements](#acknowledgements)

## Introduction

### Protocol layers

Implementing a Status clients means implementing the following layers. Additionally, there are separate specifications for things like key management and account lifecycle.

| Layer             | Purpose                         | Technology                   |
|-------------------|---------------------------------|------------------------------|
| Data and payloads | End user functionality          | 1:1, group chat, public chat |
| Data sync         | Data consistency                | MVDS Ratchet                 |
| Secure transport  | Confidentiality, PFS, etc       | Double Ratchet               |
| Transport privacy | Routing, Metadata protection    | Whisper                      |
| P2P Overlay       | Overlay routing, NAT traversal  | devp2p                       |

## Components

### P2P Overlay

Status clients run on the public Ethereum network, as specified by the devP2P
network protocols. devP2P provides a protocol for node discovery which is in
draft mode
[here](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md). See
more on node discovery and management in the next section.

To communicate between Ethereum nodes, the [RLPx Transport
Protocol, v5](https://github.com/ethereum/devp2p/blob/master/rlpx.md) is used, which
allows for TCP-based communication between nodes.

On top of this we run the RLPx-based subprotocol [Whisper
v6](https://eips.ethereum.org/EIPS/eip-627) for privacy-preserving messaging.

There MUST be an Ethereum node that is capable of discovering peers and
implements Whisper V6 specification.

#### Node discovery and roles

There are four types of node roles:
1. Bootstrap nodes
2. Whisper relayers
3. Mailservers (servers and clients)
4. Mobile nodes (Status Clients)

To implement a standard Status client you MUST implement the last node type. The
other node types are optional, but we RECOMMEND you implement a mailserver
client mode, otherwise the user experience is likely to be poor.

#### Bootstrapping

To connect to other Status nodes you need to connect to a bootstrap node. These
nodes allow you to discover other nodes of the network.

Currently the main bootstrap nodes are provided by Status Gmbh, but anyone can
run these provided they are connected to the rest of the Whisper network.

Status maintains a list of boootstrap nodes in the following locations:
- Asia:
  - `enode://e8a7c03b58911e98bbd66accb2a55d57683f35b23bf9dfca89e5e244eb5cc3f25018b4112db507faca34fb69ffb44b362f79eda97a669a8df29c72e654416784@47.91.224.35:443`
  - `enode://43947863cfa5aad1178f482ac35a8ebb9116cded1c23f7f9af1a47badfc1ee7f0dd9ec0543417cc347225a6e47e46c6873f647559e43434596c54e17a4d3a1e4@47.52.74.140:443`
- Europe:
  - `enode://436cc6f674928fdc9a9f7990f2944002b685d1c37f025c1be425185b5b1f0900feaf1ccc2a6130268f9901be4a7d252f37302c8335a2c1a62736e9232691cc3a@174.138.105.243:443`
  - `enode://5395aab7833f1ecb671b59bf0521cf20224fe8162fc3d2675de4ee4d5636a75ec32d13268fc184df8d1ddfa803943906882da62a4df42d4fccf6d17808156a87@206.189.243.57:443`
- North America:
  - `enode://7427dfe38bd4cf7c58bb96417806fab25782ec3e6046a8053370022cbaa281536e8d64ecd1b02e1f8f72768e295d06258ba43d88304db068e6f2417ae8bcb9a6@104.154.88.123:443`
  - `enode://ebefab39b69bbbe64d8cd86be765b3be356d8c4b24660f65d493143a0c44f38c85a257300178f7845592a1b0332811542e9a58281c835babdd7535babb64efc1@35.202.99.224:443`

These bootstrap nodes do not change, however, we can't guarantee that it will stay this way forever
and at some point we might be forced to change them.

#### Discovery

To implement a Status client you need to discover peers to connect to. We use a
light discovery mechanism based on a combination of [Discovery
v5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md) and
[Rendezvous Protocol](https://github.com/libp2p/specs/tree/master/rendezvous),
(with some
[modifications](https://github.com/status-im/rendezvous#differences-with-original-rendezvous)).
Additionally, some static nodes MAY also be used.

A Status client MUST use at least one discovery method or use static nodes
to communicate with other clients.

Discovery V5 uses bootstrap nodes to discover other peers. Bootstrap nodes MUST support
Discovery V5 protocol as well in order to provide peers. It is kademlia-based discovery mechanism
and it might consume significant (at least on mobile) amount of network traffic to operate.

In order to take advantage from simpler and more mobile-friendly peers discovery mechanism,
i.e. Rendezvous protocol, one MUST provide a list of Rendezvous nodes which speak
Rendezvous protocol. Rendezvous protocol is request-response discovery mechanism.
It uses Ethereum Node Records (ENR) to report discovered peers.

Both peers discovery mechanisms use topics to provide peers with certain capabilities.
There is no point in returning peers that do not support a particular protocol.
Status nodes that want to be discovered MUST register to Discovery V5 and/or Rendezvous
with the `whisper` topic. Status nodes that are mail servers and want to
be discoverable MUST additionally register with the `whispermail` topic.

The recommended strategy is to use both mechanisms but at the same time implement a structure
called `PeerPool`. `PeerPool` is responsible for maintaining an optimal number of peers.
For mobile nodes, there is no significant advantage to have more than 2-3 peers and one mail server.
`PeerPool` can notify peers discovery protocol implementations that they should suspend
their execution because the optimal number of peers is found. They should resume
if the number of connected peers drops or a mail server disconnects.

It is worth noticing that an efficient caching strategy can be of great use, especially,
on mobile devices. Discovered peers can be cached as they rarely change and used
when the client starts again. In such a case, there might be no need to even start
peers discovery protocols because cached peers will satisfy the optimal number of peers.

Alternatively, a client MAY rely exclusively on a list of static peers. This is the most efficient
way because there is no peers discovery algorithm overhead introduced. The disadvantage
is that these peers might be gone and without peers discovery mechanism, it won't be possible to find
new ones.

The current list of static peers is published on https://fleets.status.im/. `eth.beta` is the current
group of peers the official Status client uses. The others are test networks.

#### Mobile nodes

This is a Whisper node which connects to part of the Whisper network. It MAY
relay messages. See next section for more details on how to use Whisper to
communicate with other Status nodes.

### Transport privacy and Whisper usage

Once a Whisper node is up and running there are some specific settings required
to commmunicate with other Status nodes.

See [Status Whisper Usage Spec](status-whisper-usage-spec.md) for more details.

For providing offline inboxing, see the complementary [Whisper Mailserver
Spec](status-whisper-mailserver-spec.md).

### Secure Transport

In order to provide confidentiality, integrity, authentication and forward
secrecy of messages we implement a secure transport on top of Whisper. This is
used in 1:1 chats and group chats, but not for public chats. See [Status Secure
Transport Spec](status-secure-transport-spec.md) for more.

### Data Sync

[MVDS](https://specs.vac.dev/mvds.html) is used for 1:1 and group chats, however it is currently not in use for public chats.

[Status payloads](#payloads-and-clients) are serialized and then wrapped inside a MVDS message which is added to an [MVDS payload](https://specs.vac.dev/mvds.html#payloads), this payload is then encrypted (if necessary for 1-to-1 / group-chats) and sent using whisper which also encrypts it.

### Payloads and clients

On top of secure transport, we have various types of data sync clients and
payload formats for things like 1:1 chat, group chat and public chat. These have
various degrees of standardization. Please refer to [Initial Message Payload
Specification](status-payloads-spec.md) for more details.

## Security Considerations

TBD.

<!-- TODO: Fill this out. -->

### Censorship-resistance

With default settings Whisper over DevP2P runs on odd ports in 30k range, which
are easy to block. One workaround for this is to run ports on 443. This doesn't
take care of all cases though, and this quickly leads into efforts such as
obfuscated transports a la Tor.

See https://github.com/status-im/status-react/issues/6351 for some discussion.

<!-- TODO: More detail on interop of ports and what we do precisely -->

## Design Rationale

### P2P Overlay

#### Why devp2p? Why not use libp2p?

At the time the main Status clients were being developed, devp2p was the most
mature. However, it is likely we'll move over to libp2p in the future, as it'll
provide us with multiple transports, better protocol negotiation, NAT traversal,
etc.

For very experimental bridge support, see the bridge between libp2p and devp2p
in [Murmur](https://github.com/status-im/murmur).

#### What about other RLPx subprotocols like LES, and Swarm?

Status is primarily optimized for resource restricted devices, and at present
time light client support for these protocols are suboptimal. This is a work in
progress.

For better Ethereum light client support, see [Re-enable LES as
option](https://github.com/status-im/status-go/issues/1025). For better Swarm
support, see [Swarm adaptive
nodes](https://github.com/ethersphere/SWIPs/pull/12).

For transaction support, Status clients currently have to rely on Infura.

Status clients currently do not offer native support for file storage.

#### Why do you use Whisper?

Whisper is one of the [three parts](http://gavwood.com/dappsweb3.html) of the
vision of Ethereum as the world computer, Ethereum and Swarm being the other
two. Status was started as an encapsulation of and a clear window to this world
computer.

#### I heard you were moving away from Whisper?

Whisper is not currently under active development, and it has several drawbacks.
Among others:

- It is very wasteful bandwidth-wise and it doesn't appear to be scalable
- Proof of work is a poor spam protection mechanism for heterogenerous devices
- The privacy guarantees provided are not rigorous
- There's no incentives to run a node

Finding a more suitable transport privacy is an ongoing research effort,
together with [Vac](https://vac.dev/vac-overview) and other teams in the space.

#### Why is PoW for Whisper set so low?

A higher PoW would be desirable, but this kills the battery on mobilephones,
which is a prime target for Status clients.

This means the network is currently vulnerable to DDoS attacks. Alternative
methods of spam protection are currently being researched.

#### Why do you not use Discovery v5 for node discovery?

At the time we implemented dynamic node discovery, Discovery v5 wasn't completed
yet. Additionally, running a DHT on a mobile leads to slow node discovery, bad
battery and poor bandwidth usage. Instead, each client can choose to turn on
Discovery v5 for a short period of time until their peer list is populated.

For some further investigation, see
[here](https://github.com/status-im/swarms/blob/master/ideas/092-disc-v5-research.md).

#### I heard something about mailservers being trusted somehow?

In order to use a mail server, a given node needs to connect to it directly, i.e. add the mailserver as its peer and mark it as trusted. This means that the mail server is able to send direct p2p messages to the node instead of broadcasting them. Effectively, it knows the bloom filter of the topics the node is interested in, when it is online as well as many metadata like IP address.

### Data sync

#### Why is MVDS not used for public chats?

Currently public chats are broadcast-based, and there's no direct way of finding
out who is receiving messages. Hence there's no clear group sync state context
whereby participants can sync. Additionally, MVDS is currently not optimized for
large group contexts, which means bandwidth usage will be a lot higher than
reasonable. See [P2P Data Sync for
Mobile](https://vac.dev/p2p-data-sync-for-mobile) for more. This is an active
area of research. The bandwidth issue is further exacerbated by Whisper being
very bandwidth heavy.

## Footnotes

1. <https://github.com/status-im/status-protocol-go/>
2. <https://github.com/status-im/status-console-client/>
3. <https://github.com/status-im/status-react/>

## Acknowledgements
