```
title: Wallet Interaction Standard
status: Draft
author: Mike Godenzi <m.godenzi@papers.ch>, Andreas Gassmann <a.gassmann@papers.ch>, Alessandro De Carli <a.decarli@papers.ch>
discussions-to: https://forum.tezosagora.org/t/wallet-interaction-standard/1399
created: 2019-09-17
```

## Iterations

| Version | Date       | Changes                      |
| ------- | ---------- | ---------------------------- |
| v0.1    | 2019-09-17 | initial draft                |
| v0.2    | 2019-10-09 | adds workshop feedback notes |
| v0.3    | 2019-10-12 | specifies transport layer    |

## Simple Summary

Communication between wallets and decentralised applications.

## Abstract

This document describes the communication between an app (decentralised application on Tezos) and wallet (eg. browser extension).

## Motivation

A user wants to be able to interact with decentralised applications e.g. uniswap-like decentralised exchange or a game that uses NFTs, he wants to use his preferred wallet and avoid having to install a new wallet for each application he uses.

To enable adoption of decentralised applications in the Tezos ecosystem a standard for the communication between these apps and wallets is needed. Application developers shouldn't need to implement yet another wallet just for their use case and users shouldn't need a multitude of wallets just to use a certain service.

This standard should form consensus of how the types of messsages for this communication look like. They also should be applicable to a multitude of transport layers.

# Communication Protocol

## Message Types

The following chapters describe the different message types that exist between the app and wallet.

### 1. Permission

App requests permission to access account details from wallet.

#### Request

```typescript
type PermissionScopes = "addresses" | "sign" | "network";

interface PermissionRequest {
  appMetadata: {
    id: string;
    name: string;
    icon: string;
  };
  scope: PermissionScopes[];
}
```

`appMetadata`: An App should be identifiable by the user, for example with a name and icon.

`scope`: The App should ask for permissions to do certain things, for example for a list of addresses or if the App can request to sign a transaction (maybe even without confirmation for micro-transactions).

#### Response

```typescript
interface PermissionResponse {
  addresses: { address: string; permissions: PermissionScopes }[];
}
```

#### Errors

`NO_ADDRESS_ERROR`: Will be returned if there is no address present for the protocol / network requested.

`NOT_GRANTED_ERROR`: Will be returned if the permissions requested by the App were not granted.

### 2. Sign Transaction

App requests that a (forged) transaction is signed and broadcast (depending on a flag).

#### Request

```typescript
interface SignRequest {
  unsignedTransaction: string;
  sourceAddress: string;
  broadcast?: boolean;
}
```

`unsignedTransaction`: The unsigned, forged transaction as a hex-string.

`broadcast`: Whether or not the transaction should be broadcast by the wallet. Default: `false`.

#### Response

```typescript
interface SignResponse {
  signedTransaction: string;
  transactionHash?: string;
}
```

`signedTransaction`: The signed transaction as a hex-string.

`transactionHash`: The transaction hash if it has been broadcast to the node (see `SignRequest`).

#### Errors

`TRANSACTION_INVALID_ERROR`: Will be returned if the transaction is not parsable or is rejected by the node.

`BROADCAST_ERROR`: Will be returned if the user choses that the transaction is broadcast but there is an error (eg. node not available).

### 3. Payment Request

App sends parameters like `recipient` and `amount` to the wallet and the wallet will prepare the transaction and broadcast it (if the user sets the flag).

#### Request

```typescript
interface PaymentRequest {
  recipient: string;
  amount: string;
  source?: string;
  broadcast?: boolean;
}
```

`recipient`: The address of the recipient.

`amount`: The amount that will be sent, as a string in mutez (smallest unit).

`source`: The source address. This is optional. If no source is set, the wallet will let the user chose.

`broadcast`: Whether or not the transaction should be broadcast by the wallet. Default: `false`.

#### Response

```typescript
interface PaymentResponse {
  signedTransaction: string;
  transactionHash?: string;
}
```

`signedTransaction`: The signed transaction as a hex-string

`transactionHash`: The transaction hash if it has been broadcast to the node (see `PaymentRequest`)

#### Errors

`PARAMETERS_INVALID_ERROR`: Will be returned if any of the parameters are invalid

`BROADCAST_ERROR`: Will be returned if the user choses that the transaction is broadcast but there is an error (eg. node not available)

### 4. Broadcast Transactions

App requests a signed transaction to be broadcast.

#### Request

```typescript
interface BroadcastRequest {
  signedTransaction: string;
}
```

`signedTransaction`: The signed transaction as a hex-string

#### Response

```typescript
interface BroadcastResponse {
  transactionHash: string;
}
```

`transactionHash`: The transaction hash.

#### Errors

`TRANSACTION_INVALID_ERROR`: Will be returned if the transaction is not parsable or is rejected by the node.

`BROADCAST_ERROR`: Will be returned if the user choses that the transaction is broadcast but there is an error (eg. node not available).

# Transport Layer

This standard requires a transport layer that can cover the following use cases:

- Same Device Desktop to Desktop (e.g. Galleon)
- Same Device Desktop to Hardware Wallet (e.g. Ledger)
- Same Device Mobile to Mobile (e.g. Cortez)
- Two Device Desktop to online Mobile (e.g. Wetez)
- Two Device Desktop to offline Mobile (e.g. AirGap)

The transport layer used for app-wallet communication we propose is using a decentralised messaging service. We are proposing to use the [matrix protocol](https://matrix.org/). Other projects in the blockchain
space like [Raiden](https://medium.com/raiden-network/raiden-transport-explained-939d7741b6f4%5C) have been using matrix successfully to cover very similar requirements. The long term vision is that
the Tezos reference node will eventually include such a messaging service by default, replacing the matrix network in the long term.

The main reasons why we propose to use a decentralized messaging service as the transport layer between app and wallet are:

- decentralization
- enables modern user experience
- covers all of the scenarios described above
- easy to integrate with all kind of wallets
- minimal effort to "join" for a wallet
- support for oracles (e.g. push service)

## Serialisation

The serialisation used for the transport layer messages requires an algorithm which is space efficient and fast to encode/decode. These properties rule out the common encoding standards
XML and JSON. After researching various alternatives (BSON, RLP, Protocol buffers and amino) we propose to use the space efficient yet very easy to implement RLP encoding. The main reasons
why we are proposing RLP are:

- simple algorithm, no need to generate/compile code on a per-message basis (like with e.g. protocol buffers)
- very space efficient (only 1-2 bytes of overhead per new structure)
- same object always has same byte output, making it suitable to use with signatures <sup>1</sup>

<sup>1</sup> this is not a hard requirement for this initiative as of now, we can however foresee that it might become relevant in future.

### Handshake (opening a channel)

Communication between app and wallet requires a secure setup. This is the proposed handshake process:

0. we define the communication/transport layer as 'channel'
1. app generates and stores (e.g. local storage) a channel asymmetric key pair
1. app serialises public key and channel version in an handshake uri
1. handshake uri is either shown as QR or made openable by app
1. wallet receives handshake uri
1. wallet generates and stores a channel asymmetric key pair
1. wallet serialises public key and channel version in an handshake uri
1. wallet uses P2P network layer to reach out to app and send handshake uri <sup>2</sup>
1. wallet and app compute DH symmetric channel key
1. symmetric encrypted acknowledgment message is sent
1. channel is open

**Why not simply use directly the crypto address in the wallet?**  
There would be one big upside namely the wallet-app channel could be easily restored on any wallet that
imported the crypto private key material, this would also allow the user to accept/confirm messages coming from the app on different devices. The reason for the decision against such a solution is because it would imply that if a app sets up a channel once with a wallet it will always be able to send messages to it in future and since security is of outmost importance for this standard the decision has been made to have random channel keys which can be destroyed/ignored when an app turns rogue.

<sup>2</sup> public key is used to derive address on which both parties are listening.

### Unresponsive counter party

If for whatever reason one party will not respond to messages anymore the user should be informed accordingly. The user can then choose to either close the channel and reopen or just
retry at a later point in time.

### Closing a channel

Closing a channel will require one of the parties to send a channel close operation. After closing the channel the party will no longer listen to the channel (fire and forget).

### Cleanup

Channels without activity for 90 days should be considered as dead. All state related to that channel should be removed.

# Feedback - TQuorum Workshop

Feedback to this initial draft has been gathered in a workshop during the TQuorum Global Summit in New York with parties like Stove Labs, Cryptonomics, CamlCase, ECAD Labs, Keefer Taylor & Cryptium Labs.

### Feedback

- Tezos URI compatibility (check)
- contract monitoring communication between dapp <> wallet

* PermissionRequest

  - guaranteeing the security of the dapp not impersonating an other application
  - local storage of the browser
  - idea: use a smart contract to manage the permissions
  - idea: have different trust levels ex. on-chain access
  - idea: use SSL certificate for permission requests

* SignTransaction

  - broadcast and sign separate endpoints
  - dapp broadcasts signed transaction
  - Tezos URI compatibility
  - separate endpoint for arbitrary message signing
  - dapp forges transaction -> user decides if he signs transaction (extension should be able to unforge certain standards ex. NFT to display meaningful data to the user)

* PaymentRequest

  - use Tezos URI standard

* TransportLayer
  - use URI schemes to communicate with desktop application
  - use matrix messaging network in terms to act as a "relay" service (Tezos GitLab flag) for the nodes

### Questions

- avoid impersonation of dapp by other application
- how to avoid confusion between signing methods of existing libraries -> where is the implementation done in the the end?

### Inputs

- initial handshake with versioning etc.
- serialization, instead of json use protocol buffer etc.
- QR support with scheme url
- forging is dapps/sdk responsibility
- don't automatically broadcast have 2 api's

### Next steps

- Adapt draft with feedback
- Implementation -> first step: wallet extension (signing, broadcasting)

# Appendix

### Wallets implementing the standard

- [AirGap](https://airgap.it)

### Applications using the standard

- TBD

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
