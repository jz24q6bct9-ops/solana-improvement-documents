---
simd: '0465'
title: Token Account Status Check API
authors:
  - Solana Foundation
category: Standard
type: Interface
status: Idea
created: 2026-02-24
---

## Summary

This proposal introduces a new JSON-RPC method `getTokenAccountStatus`
that allows clients to query the status of one or more SPL Token accounts,
including whether the account is frozen (stuck), its current balance, and
whether any transfer restrictions are in place.

## Motivation

Users and application developers frequently encounter situations where token
balances appear unavailable or transfers fail unexpectedly. The most common
cause is a frozen SPL Token account, but diagnosing this today requires
parsing raw `AccountInfo` data and understanding the SPL Token program
layout. There is no first-class RPC method that surfaces a human-readable
token account status in a single call.

This proposal adds a dedicated endpoint so that:

- Wallet software can proactively warn users that a token account is frozen
  before they attempt a transfer.
- Explorers and support tools can display an actionable status rather than
  a raw binary flag.
- Developers can quickly determine whether a token account is usable without
  fetching and decoding the full account data themselves.

## New Terminology

**Frozen account**: An SPL Token account whose `state` field is set to
`Frozen`. Token transfers from or to a frozen account are rejected by the
token program.

**Stuck tokens**: Informally, tokens held in a frozen account that cannot
be transferred until the account is thawed by the mint authority.

**Token account status**: A structured summary of an SPL Token account's
operability, including its freeze state, delegation, and close-authority
information.

## Detailed Design

### New RPC method: `getTokenAccountStatus`

#### Request

```
POST /
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "getTokenAccountStatus",
  "params": [
    "<base-58 encoded token account public key>",
    {
      "commitment": "confirmed"   // optional, default "finalized"
    }
  ]
}
```

The method MUST accept a single base-58 encoded public key that identifies
an SPL Token account (either the legacy SPL Token program or the Token-2022
extension program).

An optional configuration object MAY include a `commitment` field whose
value MUST be one of `processed`, `confirmed`, or `finalized`.

#### Response

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "context": { "slot": 12345678 },
    "value": {
      "address": "<base-58 token account pubkey>",
      "mint": "<base-58 mint pubkey>",
      "owner": "<base-58 owner pubkey>",
      "amount": "1000000",
      "decimals": 6,
      "uiAmount": 1.0,
      "state": "frozen",
      "isNative": false,
      "delegate": null,
      "delegatedAmount": "0",
      "closeAuthority": null
    }
  }
}
```

The `state` field MUST be one of:

| Value        | Meaning                                              |
|--------------|------------------------------------------------------|
| `initialized`| Account is active and transfers are permitted.       |
| `frozen`     | Account is frozen; transfers are blocked.            |
| `uninitialized`| Account exists on-chain but has not been initialized |
|              | by the token program.                                |

If the supplied public key does not exist on-chain, `value` MUST be `null`.

If the supplied public key exists but is not owned by a recognized SPL
Token program, the RPC node MUST return an error with code `-32602`
(Invalid params) and a message indicating the account is not a token
account.

#### Batch queries

Clients SHOULD use JSON-RPC batching (an array of request objects) when
querying the status of multiple accounts in a single HTTP round-trip.
RPC nodes MUST support batched `getTokenAccountStatus` requests.

### Implementation guidance

An RPC node implementing this method SHOULD:

1. Resolve the account at the requested commitment level.
2. Deserialise the account data using the SPL Token account layout (165
   bytes for the base program, variable length for Token-2022 extensions).
3. Extract the `state` discriminator byte and map it to the string
   enumeration above.
4. Return the structured response; no additional on-chain calls are needed.

The deserialization logic MUST handle both the original SPL Token program
(`TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`) and the Token-2022
program (`TokenzQdBNbEqunb4HrP5QnN73MtM5UWm5nzE2yK`).

## Alternatives Considered

### Reuse `getAccountInfo` with client-side decoding

Clients can already call `getAccountInfo` and decode the binary account
data themselves. This is the status quo and it works, but it places a
significant burden on every developer: they must understand the SPL Token
binary layout, keep up with Token-2022 extension changes, and write
defensive error-handling for malformed data.

A dedicated method reduces this burden without adding complexity to the
validator.

### Extend `getTokenAccountBalance`

The existing `getTokenAccountBalance` method returns amount and decimals
but deliberately omits account state. Extending it to include a `state`
field would be a backwards-compatible change; however, the method name no
longer accurately describes the richer response. A new method avoids
confusion.

### Add a `frozen` field to `getTokenAccountBalance`

Adding a single `frozen` boolean field to the existing response is the
minimal change, but boolean fields do not accommodate the three-way
`state` enumeration (`initialized`, `frozen`, `uninitialized`), nor
future states that Token-2022 extensions may introduce.

## Impact

**Application developers**: A single RPC call is sufficient to diagnose
a frozen account and surface a clear message to end-users, reducing
support burden.

**Wallet software**: Wallets can gate the transfer UI based on the
`state` field and display an appropriate warning when tokens are frozen.

**Block explorers**: Explorers can surface the freeze status directly in
the token account detail view without custom deserialization logic.

**Validators and RPC node operators**: Minimal additional computation is
required per call; the deserialization of 165 bytes is negligible. No
consensus-layer changes are involved.

## Security Considerations

This proposal introduces a read-only RPC method that deserialises existing
on-chain state. No new on-chain instructions or state transitions are
introduced, so the consensus layer is unaffected.

RPC nodes MUST validate the length and owner of the account data before
attempting to deserialise it to prevent panics or incorrect responses from
malformed or adversarially crafted accounts. An account whose data length
does not match the expected token account layout MUST result in an error
response rather than a partially-decoded value.

Rate-limiting and denial-of-service considerations for the new endpoint
are identical to those for existing read methods such as
`getTokenAccountBalance` and `getAccountInfo`.
