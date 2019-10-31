

_The following notes and spec changes and scenarios are intended only for discussion and not intended to be merged. We present this as a PR to facilitate fine-grained discussion via GitHubs code-review features._


## Suggestions

### PermissionRequests / Granted Permissions

The spec does not inform the wallet author of how to deal with PermissionScopes.

Can the spec make suggestions to help contextualize this? For example:

The wallet (must?|should?|may?) persist an inventory of permission scopes that it grants to a dApp, and duly enforce those permissions for future `SigningRequests`

The questions that arise from this are;

As a user, will the dApp prompt me for permissions every time I reload or leave and return to the dApp? (poor UX, simpler implementation).

#### thinking

The ability for a wallet to auth itself is not the ability to sign transactions, it is the ability to produce an operation.

Since KT1 contracts cannot sign stuff directly, but they can own assets and produce operations, if you want to authenticate your KT1 account to a dApp the KT1 needs to be able to produce an operation.

# Questions;

## Permission Request;

When does the `PermissionRequest` get sent by the dApp?

* Immediately before every signing request?
* When the user first opens the dApp?
* Does this spec permit that the permission response get stored for later use? 
  * If so, how long does the permission grant last?
  * How does a user audit, list, or revoke these permissions?

The wallet returns a  `PermissionResponse` that contains `address`. Is this `address` value intended to be used as the value of `sourceAddress` in the subsequent `SignPayloadRequest`?

Example:

Wallet that can send a dApp transaction as a KT1 (AFAIK you cannot produce signature directly using a KT1)

## How do we get a list of already granted permission scopes?

Maybe make:

PermissionGrantRequest - Grant me permissions, please
PermissionRequest      - Tell me what permissions you already granted me (assumes wallet persists the grants it gives)

## PaymentRequest

Is `PaymentRequest` the best name? In many cases, we are invoking a smart contract for reasons that do not include commerce concepts like Payment. The NL Nodes RPC methods use the `Transaction` word, which usefully generalizes the concept.

What's the differentiation between `PaymentRequest` and `SignPayloadRequest`.

We assume that `SignPayloadRequest` is for signing a raw operation forged by the dApp and `PaymentRequest` is an API artifact sent to the wallet with the intention that the wallet will forge an operation based on the provided `PaymentRequest`.

`SignPayloadRequest` is very useful for operations that cannot involve a KT1 address, such as ballot operations, protocol injections or baking/endorsing operations (maybe YAGNI but, whoa! remote signers could use this spec for M2M operations... )

# Scenarios;

We layout scenarios around an NFT contract called "CryptoTacos" that deals with anthropomorphic collectable Taco's.

The NFT contract is referred to in the text as: `KT1TACOS...`

The CryptoTacos is a typical NFT contract that maintains a ledger tracking ownership of CryptoTacos collectibles.

The `KT1TACOS...` contract as an entry point:

```
// Buy taco allow the `SENDER` to buy a taco in exchange of Tez
// After this entry point gets called the CryptoTacos is transferred to the `SENDER` of the transaction
buyTaco: (tacosID: string) 
```


## Multi-sig Scenario

Bob owns a few collectable "CryptoTacos". Bob's Taco's are "owned" by Bob's multi-sig KT1 contract named `KT1BMS...`.

* Bob has multi-sig wallet `KT1BMS...`
* multi-sig wallet has three collectable tacos from the `KT1TACOS...` NFT smart contract
* Bob uses his `KT1BMS...` multi-sig wallet to pay for his collectable tacos and the collectible tacos are owned by the multi-sig

### Sequence

* Bob opens the CryptoTaco dApp.
  * dApp sends `PermissionRequest` with `read_address` and `payment_request` permissions via matrix
  * Bob's wallet receives request and specifies the address of his KT1 multi-sig contract.
  * Bob's wallet stores the `PermissionScope` it has granted to the CryptoTaco dApp.
  * dApp receives `PermissionResponse` stores the granted Scopes in state

```json
// PermissionResponse received from wallet
{
  address: 'KT1BMS...'
  permissions: ['read_address', 'payment_request']
}
```

* Display Bob's tacos
  * dApp queries the `KT1TACOS...` token contract storage using the supplied `PermissionResponse.address`, which in this case is: `KT1BMS...`
  * dApp displays Bob's Tacos (Bob marvels at his collection, and his brain release endorphins)

* Bob buys a new Taco...
  * Bob chooses to purchase a new CryptoTaco with ID `tacoXYZ` to purchase with 1 tez
  * The dApp creates a `PaymentRequest` object that it will send to Bob's wallet via the Matrix transport.

```typescript
interface PaymentRequest {
  network: string;
  recipient: string;
  amount: string;
  parameters?: {
    entrypoint: "string",
    value: MichelsonValue
  },
  source?: string;
}
```

// Actual payload of the PaymentRequest shown as JSON

```json
// PaymentRequest
{
    "recipient": "KT1TACOS...",
    "amount": "1",
    "parameter": {
        "entrypoint": "buyTaco",
        "parameters: "tacoXYZ"
    },
    "source": "KT1BMS..."
}
```

The payload does the following:

* call buyTaco function on `KT1TACOS...` using `KT1BMS...` as the `SENDER`
* implementation detail of `KT1TACOS...` check the `SENDER` of the transaction and transfer the NFT to it

The wallet receives the `PaymentRequest` via Matrix

_The following steps are implementation details of the wallet and are outside of the wallet-interaction spec. These steps are provided for illustrative purposes only_

The wallet forges a manager operation containing a `transaction` operation according to the PaymentRequest payload.

In the case of `KT1BMS...` it would forge an operation to initiate the transaction in the multi-sig contract. 
This involve:
 - collecting signature from each party that are required in the multi-sig contract
 - craft the lambda to initiate this transfer
 - broadcast the transaction to the network.

Then it would return the operationHash to the dApp

```json
{
    "transactionHash": "op..." 
}
`