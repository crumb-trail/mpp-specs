---
title: MegaETH Charge Intent for HTTP Payment Authentication
abbrev: MegaETH Charge
docname: draft-megaeth-charge-00
version: 00
category: info
ipr: noModificationTrust200902
submissiontype: IETF
consensus: true

author:
  - name: Brett DiNovi
    ins: B. DiNovi
    email: bread@megaeth.com
    organization: MegaETH Labs

normative:
  RFC2119:
  RFC3339:
  RFC4648:
  RFC8174:
  RFC8259:
  RFC8785:
  I-D.httpauth-payment:
    title: "The 'Payment' HTTP Authentication Scheme"
    target: https://datatracker.ietf.org/doc/draft-ietf-httpauth-payment/
    author:
      - name: Jake Moxey
    date: 2026-01

informative:
  EIP-712:
    title: "Typed structured data hashing and signing"
    target: https://eips.ethereum.org/EIPS/eip-712
    author:
      - name: Remco Bloemen
    date: 2017-09
  EIP-3009:
    title: "Transfer With Authorization"
    target: https://eips.ethereum.org/EIPS/eip-3009
    author:
      - name: Peter Jihoon Kim
    date: 2020-09
  EIP-2612:
    title: "Permit — 712-signed Approvals"
    target: https://eips.ethereum.org/EIPS/eip-2612
    author:
      - name: Martin Lundfall
    date: 2020-04
  PERMIT2:
    title: "Permit2"
    target: https://github.com/Uniswap/permit2
    author:
      - org: Uniswap Labs
  MEGAETH:
    title: "MegaETH Documentation"
    target: https://docs.megaeth.com
    author:
      - org: MegaETH Labs
---

--- abstract

This document defines the "charge" intent for the "megaeth"
payment method in the Payment HTTP Authentication Scheme
{{I-D.httpauth-payment}}. It specifies how clients and
servers exchange one-time ERC-20 token transfers on the
MegaETH blockchain using Permit2 or EIP-3009 authorization
signatures.

--- middle

# Introduction

The `charge` intent represents a one-time payment of a
specified amount. The server submits the authorized
transfer on-chain any time before the challenge `expires`
auth-param timestamp.

MegaETH is an EVM-compatible blockchain with 10ms block
times and sub-second finality. These properties make it
well-suited for real-time machine-to-machine payments
where settlement latency is critical.

This specification defines two asset transfer methods:

- **Permit2** (RECOMMENDED): Works with any ERC-20 token
  via Uniswap's Permit2 contract {{PERMIT2}}. The client
  signs an off-chain `PermitWitnessTransferFrom` message;
  the server submits the transfer on-chain.

- **EIP-3009**: For tokens that natively implement
  `transferWithAuthorization` {{EIP-3009}}. The client
  signs an off-chain authorization; the server calls the
  token contract directly.

Both methods require only an off-chain signature from
the client. The server pays gas and submits the
transaction.

## Charge Flow

~~~
Client                Server             MegaETH
  |                      |                  |
  | (1) GET /resource    |                  |
  |--------------------->|                  |
  |                      |                  |
  | (2) 402 Payment Req  |                  |
  |     intent="charge"  |                  |
  |<---------------------|                  |
  |                      |                  |
  | (3) Sign EIP-712     |                  |
  |     authorization    |                  |
  |                      |                  |
  | (4) Authorization:   |                  |
  |     Payment <cred>   |                  |
  |--------------------->|                  |
  |                      | (5) Submit tx    |
  |                      |----------------->|
  |                      | (6) Receipt      |
  |                      |    (~10ms)       |
  |                      |<-----------------|
  | (7) 200 OK + Receipt |                  |
  |<---------------------|                  |
  |                      |                  |
~~~

# Requirements Language

{::boilerplate bcp14-tagged}

# Terminology

ERC-20
: The standard interface for fungible tokens on
  EVM-compatible blockchains.

Permit2
: Uniswap's universal token approval contract deployed
  at `0x000000000022D473030F116dDEE9F6B43aC78BA3`.
  Enables off-chain signed approvals for any ERC-20
  token.

EIP-712
: A standard for typed structured data hashing and
  signing {{EIP-712}}, used by both Permit2 and EIP-3009
  to create human-readable, replay-protected signatures.

EIP-3009
: A standard for gasless token transfers using off-chain
  signatures {{EIP-3009}}. Tokens implementing this
  standard expose `transferWithAuthorization`.

Fee Payer
: The server or a designated account that pays
  transaction gas fees on behalf of the client. On
  MegaETH, gas costs are negligible (<$0.001 per tx).

# Request Schema

The `request` parameter in the `WWW-Authenticate`
challenge contains a base64url-encoded JSON object. The
JSON MUST be serialized using JSON Canonicalization
Scheme (JCS) {{RFC8785}} before base64url encoding, per
{{I-D.httpauth-payment}}.

## Shared Fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `amount` | string | REQUIRED | Amount in base units |
| `currency` | string | REQUIRED | ERC-20 token address |
| `recipient` | string | REQUIRED | Recipient address |
| `description` | string | OPTIONAL | Human-readable desc |
| `externalId` | string | OPTIONAL | Merchant reference |

Challenge expiry is conveyed by the `expires` auth-param
in `WWW-Authenticate` per {{I-D.httpauth-payment}}.

## Method Details

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `chainId` | number | OPTIONAL | Chain ID (default: 4326) |
| `testnet` | boolean | OPTIONAL | If true, use testnet (chain 6343) |
| `assetTransferMethod` | string | OPTIONAL | `"permit2"` (default) or `"eip3009"` |
| `feePayer` | boolean | OPTIONAL | If true, server pays gas (default: false) |
| `permit2Address` | string | OPTIONAL | Permit2 contract (default: canonical) |
| `eip712Domain` | object | OPTIONAL | EIP-712 domain override for EIP-3009 tokens using a forwarder |

### Chain ID Resolution

If `testnet` is `true`, the chain ID is 6343 regardless
of the `chainId` field. Otherwise, the chain ID defaults
to 4326 (MegaETH mainnet).

| Network | Chain ID | RPC |
|---------|----------|-----|
| Mainnet | 4326 | `https://mainnet.megaeth.com/rpc` |
| Testnet | 6343 | `https://carrot.megaeth.com/rpc` |

### Asset Transfer Methods

#### Permit2 (Default)

When `assetTransferMethod` is `"permit2"` or omitted, the
client signs a Permit2 `PermitWitnessTransferFrom`
message. This works with any ERC-20 token.

**Prerequisite:** The client MUST have an active ERC-20
approval from the payment token to the Permit2 contract.
This is a one-time operation per token.

#### EIP-3009

When `assetTransferMethod` is `"eip3009"`, the client
signs a `TransferWithAuthorization` message per
{{EIP-3009}}. This requires the token to natively
implement EIP-3009 or to use a forwarder contract.

When a forwarder is used, the server MUST include the
`eip712Domain` field specifying the forwarder's EIP-712
domain parameters:

~~~json
{
  "eip712Domain": {
    "name": "USDm Forwarder",
    "version": "1",
    "verifyingContract": "0x2c2d8EF0...60aF7aB43a60B4"
  }
}
~~~

If `eip712Domain` is omitted, the client MUST use the
token contract's own EIP-712 domain (name, version, and
verifyingContract from the token itself).

**Example (Permit2):**

~~~json
{
  "amount": "1000000000000000000",
  "currency": "0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7",
  "recipient": "0x742d35Cc6634C0532925a3b844Bc9e7595f8fE00",
  "methodDetails": {
    "chainId": 4326,
    "assetTransferMethod": "permit2",
    "feePayer": false
  }
}
~~~

This requests a transfer of 1.0 USDm (10^18 base units,
18 decimals) using Permit2 with server-paid gas.

**Example (EIP-3009 with forwarder):**

~~~json
{
  "amount": "1000000000000000000",
  "currency": "0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7",
  "recipient": "0x742d35Cc6634C0532925a3b844Bc9e7595f8fE00",
  "methodDetails": {
    "chainId": 4326,
    "assetTransferMethod": "eip3009",
    "feePayer": false,
    "eip712Domain": {
      "name": "USDm Forwarder",
      "version": "1",
      "verifyingContract": "0x2c2d8EF0664432BA243deF0b8f60aF7aB43a60B4"
    }
  }
}
~~~

# Credential Schema

The credential in the `Authorization` header contains a
base64url-encoded JSON object per {{I-D.httpauth-payment}}.

## Credential Structure

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `challenge` | object | REQUIRED | Echoed challenge |
| `payload` | object | REQUIRED | Payment proof |
| `source` | string | OPTIONAL | Payer DID |

The `source` field, if present, SHOULD use the `did:pkh`
method with the MegaETH chain ID and the payer's address
(e.g., `did:pkh:eip155:4326:0x...`).

## Permit2 Payload

When `assetTransferMethod` is `"permit2"`, the payload
contains the signed Permit2 witness transfer:

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `type` | string | REQUIRED | `"permit2"` |
| `permit` | object | REQUIRED | Permit2 permit data |
| `witness` | object | REQUIRED | Transfer witness |
| `signature` | string | REQUIRED | EIP-712 signature |

The `permit` object:

| Field | Type | Description |
|-------|------|-------------|
| `permitted` | object | `{ token, amount }` |
| `nonce` | string | Permit2 nonce |
| `deadline` | string | Unix timestamp |

The `witness` object:

| Field | Type | Description |
|-------|------|-------------|
| `transferDetails` | object | `{ to, requestedAmount }` |

**Example:**

~~~json
{
  "challenge": {
    "id": "aB3cDeF4gHiJkLmN",
    "realm": "api.example.com",
    "method": "megaeth",
    "intent": "charge",
    "request": "eyJ...",
    "expires": "2026-03-20T12:05:00Z"
  },
  "payload": {
    "type": "permit2",
    "permit": {
      "permitted": {
        "token": "0xFAfDdbb3...82079E7",
        "amount": "1000000000000000000"
      },
      "nonce": "1",
      "deadline": "1742472300"
    },
    "witness": {
      "transferDetails": {
        "to": "0x742d35Cc...5f8fE00",
        "requestedAmount": "1000000000000000000"
      }
    },
    "signature": "0x1b2c3d4e5f..."
  },
  "source": "did:pkh:eip155:4326:0x1234...5678"
}
~~~

## EIP-3009 Payload

When `assetTransferMethod` is `"eip3009"`, the payload
contains the signed transfer authorization:

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `type` | string | REQUIRED | `"eip3009"` |
| `authorization` | object | REQUIRED | Auth params |
| `signature` | string | REQUIRED | EIP-712 signature |

The `authorization` object:

| Field | Type | Description |
|-------|------|-------------|
| `from` | string | Payer address |
| `to` | string | Recipient address |
| `value` | string | Amount in base units |
| `validAfter` | string | Unix timestamp |
| `validBefore` | string | Unix timestamp |
| `nonce` | string | Random bytes32 |

**Example:**

~~~json
{
  "challenge": {
    "id": "aB3cDeF4gHiJkLmN",
    "realm": "api.example.com",
    "method": "megaeth",
    "intent": "charge",
    "request": "eyJ...",
    "expires": "2026-03-20T12:05:00Z"
  },
  "payload": {
    "type": "eip3009",
    "authorization": {
      "from": "0x1234...5678",
      "to": "0x742d...fE00",
      "value": "1000000000000000000",
      "validAfter": "0",
      "validBefore": "1742472300",
      "nonce": "0xabcd...1234"
    },
    "signature": "0x1b2c3d4e5f..."
  },
  "source": "did:pkh:eip155:4326:0x1234...5678"
}
~~~

# Fee Payment

MegaETH transaction fees are negligible (<$0.001 per
transaction). Servers SHOULD sponsor gas by default.

## Server-Paid Fees

When `feePayer` is `true`:

1. The client signs only the payment authorization
   (Permit2 or EIP-3009). No transaction is signed by
   the client.
2. The server constructs and signs the on-chain
   transaction using its own hot wallet.
3. The server pays gas from its own balance.

This is the expected flow. The client never interacts
with the chain directly.

## Client-Paid Fees (Default)

When `feePayer` is `false` or omitted, the client MAY submit the
transaction directly to the MegaETH network and provide
a hash credential (see {{hash-payload}}).

## Hash Payload {#hash-payload}

When the client has already broadcast the transaction,
the payload contains only the transaction hash:

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `type` | string | REQUIRED | `"hash"` |
| `hash` | string | REQUIRED | Tx hash (`0x`-prefixed) |

**Limitations:**

- Cannot be used with `feePayer: true`
- Server cannot modify the transaction

# Settlement Procedure

## Permit2 Settlement

1. Server receives credential with `type: "permit2"`
2. Server verifies the signature recovers to a valid
   signer with sufficient token balance and Permit2
   allowance
3. Server calls `Permit2.permitWitnessTransferFrom()`
   on-chain via its hot wallet
4. Server receives transaction receipt

## EIP-3009 Settlement

1. Server receives credential with `type: "eip3009"`
2. Server verifies the signature recovers to a valid
   signer with sufficient token balance
3. Server calls `token.transferWithAuthorization()` (or
   the forwarder equivalent) on-chain via its hot wallet
4. Server receives transaction receipt

## Hash Settlement

1. Server receives credential with `type: "hash"`
2. Server fetches the transaction receipt from the chain
3. Server verifies the emitted `Transfer` event logs
   match the challenge parameters (token, from, to,
   amount)

## Transaction Submission

MegaETH offers `realtime_sendRawTransaction`, a custom
RPC method that returns the full transaction receipt
inline with the response. Unlike standard
`eth_sendRawTransaction` which returns only a transaction
hash (requiring subsequent polling for the receipt), this
eliminates one round-trip and provides confirmation in a
single call.

Servers SHOULD use `realtime_sendRawTransaction` when
available. Servers MAY fall back to standard
`eth_sendRawTransaction` with receipt polling.

## Settlement Latency

MegaETH's 10ms block times and sub-second finality
enable settlement in under 50ms end-to-end (signature
verification + transaction submission + receipt). This
is measured against Base mainnet where equivalent
settlement takes 500-3000ms depending on optimization
level.

## Transaction Verification

Before broadcasting, servers MUST verify:

1. The signature is valid and recovers to the `from`
   address
2. The `from` address has sufficient token balance
3. The amount matches the challenge `amount`
4. The recipient matches the challenge `recipient`
5. The token address matches the challenge `currency`
6. For Permit2: the `from` address has sufficient
   Permit2 allowance
7. For EIP-3009: the nonce has not been used
8. Validity timestamps are current

## Receipt Generation

Upon successful settlement, servers MUST return a
`Payment-Receipt` header per {{I-D.httpauth-payment}}.

| Field | Type | Description |
|-------|------|-------------|
| `method` | string | `"megaeth"` |
| `reference` | string | Transaction hash |
| `status` | string | `"success"` |
| `timestamp` | string | {{RFC3339}} settlement time |
| `externalId` | string | OPTIONAL. Echoed from request |

# Security Considerations

## Signature Replay Protection

### Permit2

Permit2 nonces are consumed on-chain upon use. Each
nonce can only be used once per (owner, token, spender)
tuple. The `deadline` field provides temporal bounds.

### EIP-3009

The `nonce` field in `TransferWithAuthorization` is a
random bytes32 value. The token contract tracks used
nonces per sender. The `validAfter` and `validBefore`
fields provide temporal bounds.

### Cross-Chain Replay

Both Permit2 and EIP-3009 signatures include the chain
ID in the EIP-712 domain separator. Signatures for
MegaETH mainnet (4326) cannot be replayed on testnet
(6343) or other EVM chains.

## Amount Verification

Clients MUST verify before signing:

1. The `amount` is reasonable for the resource
2. The `recipient` is the expected party
3. The `currency` is the expected token
4. The `chainId` matches the expected network

Clients MUST NOT rely on the `description` field for
payment verification.

## Fee Payer Risks

Servers acting as fee payers accept the risk of paying
gas for transactions that may fail on-chain. Servers
SHOULD:

- Simulate transactions via `eth_call` before submission
- Implement rate limiting per client address
- Monitor hot wallet balance

On MegaETH, gas costs are negligible, limiting the
financial impact of this risk.

## Permit2 Approval Scope

Clients granting Permit2 approval SHOULD use bounded
amounts rather than unlimited approval where possible.
The approval is to the Permit2 contract, not to the
server, limiting exposure.

## Server Hot Wallet Security

The server's hot wallet holds ETH for gas and submits
transactions on behalf of clients. Servers MUST:

- Use a dedicated hot wallet for payment settlement
- Monitor for anomalous transaction patterns
- Implement key rotation procedures
- Keep minimal balance sufficient for operations

# IANA Considerations

## Payment Method Registration

This document registers the following payment method in
the "HTTP Payment Methods" registry established by
{{I-D.httpauth-payment}}:

| Method Identifier | Description | Reference |
|-------------------|-------------|-----------|
| `megaeth` | MegaETH ERC-20 token transfer | This document |

Contact: MegaETH Labs (<bread@megaeth.com>)

## Payment Intent Registration

This document registers the following payment intent in
the "HTTP Payment Intents" registry established by
{{I-D.httpauth-payment}}:

| Intent | Methods | Description | Reference |
|--------|---------|-------------|-----------|
| `charge` | `megaeth` | One-time ERC-20 transfer | This document |

--- back

# Known Tokens

The following tokens are commonly used with the MegaETH
payment method:

| Token | Address | Decimals |
|-------|---------|----------|
| USDm | `0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7` | 18 |

# Key Contracts

| Contract | Address | Chain |
|----------|---------|-------|
| Permit2 | `0x000000000022D473030F116dDEE9F6B43aC78BA3` | 4326, 6343 |
| USDm Forwarder (Meridian) | `0x2c2d8EF0664432BA243deF0b8f60aF7aB43a60B4` | 4326 |

# Example

## Full Charge Flow (Permit2)

**Challenge:**

~~~http
HTTP/1.1 402 Payment Required
Cache-Control: no-store
WWW-Authenticate: Payment id="mE9xPqWvT2nJrHsY4aDfEb",
  realm="api.example.com",
  method="megaeth",
  intent="charge",
  request="eyJhbW91bnQiOiIxMDAwMDAwMDAwMDAwMDAwMDAwIiwiY3VycmVuY3kiOiIweEZBZkRkYmIzRkM3Njg4NDk0OTcxYTc5Y2M2NURDYTNFRjgyMDc5RTciLCJyZWNpcGllbnQiOiIweDc0MmQzNUNjNjYzNEMwNTMyOTI1YTNiODQ0QmM5ZTc1OTVmOGZFMDAiLCJtZXRob2REZXRhaWxzIjp7ImNoYWluSWQiOjQzMjYsImFzc2V0VHJhbnNmZXJNZXRob2QiOiJwZXJtaXQyIiwiZmVlUGF5ZXIiOnRydWV9fQ",
  expires="2026-03-20T12:05:00Z"
~~~

Decoded `request`:

~~~json
{
  "amount": "1000000000000000000",
  "currency": "0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7",
  "recipient": "0x742d35Cc6634C0532925a3b844Bc9e7595f8fE00",
  "methodDetails": {
    "chainId": 4326,
    "assetTransferMethod": "permit2",
    "feePayer": false
  }
}
~~~

**Credential:**

~~~http
GET /api/resource HTTP/1.1
Host: api.example.com
Authorization: Payment eyJjaGFsbGVuZ2UiOnsiaWQiOiJtRTl4UHFXdlQybkpySHNZNGFEZkViIiwicmVhbG0iOiJhcGkuZXhhbXBsZS5jb20iLCJtZXRob2QiOiJtZWdhZXRoIiwiaW50ZW50IjoiY2hhcmdlIiwicmVxdWVzdCI6ImV5Si4uLiIsImV4cGlyZXMiOiIyMDI2LTAzLTIwVDEyOjA1OjAwWiJ9LCJwYXlsb2FkIjp7InR5cGUiOiJwZXJtaXQyIiwicGVybWl0Ijp7InBlcm1pdHRlZCI6eyJ0b2tlbiI6IjB4RkFmRGRiYjNGQzc2ODg0OTQ5NzFhNzljYzY1RENhM0VGODIwNzlFNyIsImFtb3VudCI6IjEwMDAwMDAwMDAwMDAwMDAwMDAifSwibm9uY2UiOiIxIiwiZGVhZGxpbmUiOiIxNzQyNDcyMzAwIn0sIndpdG5lc3MiOnsidHJhbnNmZXJEZXRhaWxzIjp7InRvIjoiMHg3NDJkMzVDYzY2MzRDMDUzMjkyNWEzYjg0NEJjOWU3NTk1ZjhmRTAwIiwicmVxdWVzdGVkQW1vdW50IjoiMTAwMDAwMDAwMDAwMDAwMDAwMCJ9fSwic2lnbmF0dXJlIjoiMHgxYjJjM2Q0ZTVmLi4uIn19
~~~

**Success:**

~~~http
HTTP/1.1 200 OK
Cache-Control: private
Payment-Receipt: eyJzdGF0dXMiOiJzdWNjZXNzIiwibWV0aG9kIjoibWVnYWV0aCIsInRpbWVzdGFtcCI6IjIwMjYtMDMtMjBUMTI6MDA6MDFaIiwicmVmZXJlbmNlIjoiMHhhYmNkZWYxMjM0NTY3ODkwIn0
Content-Type: application/json

{"data": "..."}
~~~

Decoded receipt:

~~~json
{
  "status": "success",
  "method": "megaeth",
  "timestamp": "2026-03-20T12:00:01Z",
  "reference": "0xabcdef1234567890..."
}
~~~

# Acknowledgements

The authors thank the Tempo Labs team for the MPP
specification framework that this method builds upon,
and the MegaETH engineering team for chain-level
optimizations that enable sub-50ms settlement.
