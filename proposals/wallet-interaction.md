```
title: Wallet Interaction Standard
status: Draft
author: Mike Godenzi <m.godenzi@papers.ch>, Andreas Gassmann <a.gassmann@papers.ch>, Alessandro De Carli <a.decarli@papers.ch>
discussions-to:
created: 2019-09-17
```

## Simple Summary

Communication between wallets and decentralised applications.

## Abstract

This document describes the communication between an app (decentralised application on Tezos) and wallet (eg. browser extension).

## Motivation

A user wants to be able to interact with decentralised applications e.g. uniswap-like decentralised exchange or a game that uses NFTs, he wants to use his preferred wallet and avoid having to install a new wallet for each application he uses.

To enable adoption of decentralised applications in the Tezos ecosystem a standard for the communication between these apps and wallets is needed. Application developers shouldn't need to implement yet another wallet just for their use case and users shouldn't need a multitude of wallets just to use a certain service.

This standard should form consensus of how the types of messsages for this communication look like. They also should be applicable to a multitude of transport layers.

# Message Types

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

# Transport Layers

In this chapter, the interaction flow between the different entities with concrete example of the messages that are being passed between them.

The protocol can be used with many more transports, we only describe 2 here that are distinctly different, the others fall into either of one of those 2 categories.

### QR Code

In this communication, the App and the signing device have a direct, "one-way" communication channel open through QR codes. It is not possible to have back and forth communication unless multiple QRs are being scanned. This means that the messages should contain all necessary information so only one round trip is necessary.

![TransportLayer_QRCode](uploads/a418051544ed7a3562de5923557edb9f/TransportLayer_QRCode.png)

### Push Notification / Relay Server

Push notifications always require a central server. This server is also used as a "relay server" to forward messages between the app and the wallet/signing device.

![TransportLayer_RelayServer](uploads/a5cf03a6d7fd654cda4563215fc3f264/TransportLayer_RelayServer.png)

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
- instead of json use protocol buffer etc.

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
