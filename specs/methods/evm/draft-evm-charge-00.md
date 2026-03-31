---
title: EVM Charge Intent for HTTP Payment Authentication
abbrev: EVM Charge
docname: draft-evm-charge-00
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
  - name: Kartik Bhat
    ins: K. Bhat
    email: kartik@seinetwork.io
    organization: Sei Labs

normative:
  RFC2119:
  RFC3339:
  RFC4648:
  RFC8174:
  RFC8259:
  RFC8785:
  RFC9457:
  I-D.payment-intent-charge:
    title: "'charge' Intent for HTTP Payment Authentication"
    target: https://datatracker.ietf.org/doc/draft-payment-intent-charge/
    author:
      - name: Jake Moxey
      - name: Brendan Ryan
      - name: Tom Meagher
    date: 2026
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
  EIP-1559:
    title: "Fee market change for ETH 1.0 chain"
    target: https://eips.ethereum.org/EIPS/eip-1559
  EIP-55:
    title: "Mixed-case checksum address encoding"
    target: https://eips.ethereum.org/EIPS/eip-55
  ERC-20:
    title: "Token Standard"
    target: https://eips.ethereum.org/EIPS/eip-20
  PERMIT2:
    title: "Permit2"
    target: https://github.com/Uniswap/permit2
    author:
      - org: Uniswap Labs
---

--- abstract

This document defines the "charge" intent for the "evm" payment
method in the Payment HTTP Authentication Scheme
{{I-D.httpauth-payment}}. It specifies how clients and servers
exchange one-time ERC-20 token transfers on any EVM-compatible
blockchain.

Three credential types are supported: `type="transaction"`, where
the client sends a signed ERC-20 transfer transaction for the
server to broadcast; `type="permit2"`, where the client signs an
off-chain Permit2 authorization and the server submits the
transfer; and `type="hash"`, where the client broadcasts the
transaction itself and presents the on-chain transaction hash for
server verification.

--- middle

# Introduction

HTTP Payment Authentication {{I-D.httpauth-payment}} defines a
challenge-response mechanism that gates access to resources behind
payments. This document registers the "charge" intent for the
"evm" payment method.

The Ethereum Virtual Machine (EVM) is the execution environment
shared by Ethereum and a growing number of compatible blockchains.
These chains share a common smart contract interface (ERC-20
{{ERC-20}}), transaction format (EIP-1559 {{EIP-1559}}), address
encoding (EIP-55 {{EIP-55}}), and JSON-RPC API — making it
possible to define a single payment method that works across all
of them.

This specification inherits the shared request semantics of the
"charge" intent from {{I-D.payment-intent-charge}}. It defines
only the EVM-specific `methodDetails`, `payload`, and verification
procedures.

## Design Rationale

Prior drafts proposed separate payment methods for individual EVM
chains (`megaeth`, `sei`, etc.). However, the control flow, data
structures, and verification logic are identical across these
chains — the only differences are chain ID, block time, and
optional RPC extensions. A unified `evm` method avoids
fragmenting the registry while still allowing chain-specific
optimizations through `methodDetails`.

## Charge Flow

The following diagram illustrates the default charge flow using
a signed transaction credential:

~~~
Client                  Server               EVM Chain
  |                        |                      |
  | (1) GET /resource      |                      |
  |----------------------->|                      |
  |                        |                      |
  | (2) 402 Payment Req    |                      |
  |     intent="charge"    |                      |
  |<-----------------------|                      |
  |                        |                      |
  | (3) Sign transfer      |                      |
  |                        |                      |
  | (4) Authorization:     |                      |
  |     Payment <cred>     |                      |
  |----------------------->|                      |
  |                        | (5) Broadcast tx     |
  |                        |--------------------->|
  |                        | (6) Confirmation     |
  |                        |<---------------------|
  | (7) 200 OK + Receipt   |                      |
  |<-----------------------|                      |
  |                        |                      |
~~~

## Relationship to the Charge Intent

This document inherits the shared request semantics of the
"charge" intent from {{I-D.payment-intent-charge}}. It defines
only the EVM-specific `methodDetails`, `payload`, and
verification procedures for the "evm" payment method.

# Requirements Language

{::boilerplate bcp14-tagged}

# Terminology

ERC-20
: The standard token interface on EVM-compatible chains
  {{ERC-20}}. Tokens expose `transfer(address,uint256)` and
  emit `Transfer` events on successful transfers.

Permit2
: Uniswap's universal token approval contract {{PERMIT2}},
  deployed at the canonical address
  `0x000000000022D473030F116dDEE9F6B43aC78BA3` on all
  supported chains. Enables off-chain signed approvals for
  any ERC-20 token via `PermitWitnessTransferFrom`.

EIP-712
: A standard for typed structured data hashing and signing
  {{EIP-712}}, used by Permit2 to produce human-readable,
  replay-protected authorization signatures.

Base Units
: The smallest transferable unit of a token, determined by
  the token's decimal precision. For example, USDC (6
  decimals) uses 1,000,000 base units per 1 USDC; USDm
  (18 decimals) uses 10^18 base units per 1 USDm.

Fee Payer
: An account that pays transaction gas fees on behalf of
  the client. On low-fee chains, servers typically sponsor
  gas to simplify the client experience.

# Request Schema

The `request` parameter in the `WWW-Authenticate` challenge
contains a base64url-encoded JSON object. The JSON MUST be
serialized using JSON Canonicalization Scheme (JCS) {{RFC8785}}
before base64url encoding, per {{I-D.httpauth-payment}}.

## Shared Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | string | REQUIRED | Amount in base units (stringified integer) |
| `currency` | string | REQUIRED | ERC-20 token contract address |
| `recipient` | string | REQUIRED | Recipient address, EIP-55 encoded {{EIP-55}} |
| `description` | string | OPTIONAL | Human-readable payment description |
| `externalId` | string | OPTIONAL | Merchant's reference (order ID, invoice number, etc.) |

Challenge expiry is conveyed by the `expires` auth-param in
`WWW-Authenticate` per {{I-D.httpauth-payment}}.

Addresses in `currency` and `recipient` MUST be 0x-prefixed,
20-byte hex strings. Implementations SHOULD use EIP-55
mixed-case encoding but MUST compare addresses by decoded
20-byte value, not string form.

## Method Details

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `chainId` | number | REQUIRED | EIP-155 chain ID |
| `feePayer` | boolean | OPTIONAL | If `true`, server pays gas (default: `false`) |
| `permit2Address` | string | OPTIONAL | Permit2 contract address (default: canonical address) |
| `splits` | array | OPTIONAL | Additional payment splits |

### Chain Identification

The `chainId` field is REQUIRED and identifies the target
blockchain. Clients MUST reject challenges whose `chainId` does
not match a chain they support. The following table lists
commonly used EVM chain IDs, though this specification is not
limited to these chains:

| Chain ID | Network | Approx. Block Time |
|----------|---------|-------------------|
| 1 | Ethereum Mainnet | ~12s |
| 10 | Optimism | ~2s |
| 137 | Polygon | ~2s |
| 1329 | Sei Mainnet | ~400ms |
| 4326 | MegaETH Mainnet | ~10ms (mini blocks) |
| 8453 | Base | ~2s |
| 42161 | Arbitrum One | ~250ms |

Servers MUST include `chainId`. Clients MUST verify `chainId`
matches their configured chain before signing any transaction or
authorization.

### Credential Type Negotiation

Servers MAY indicate preferred credential types via the
`credentialTypes` field in `methodDetails`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `credentialTypes` | array | OPTIONAL | Ordered list of accepted credential types |

Valid values: `"transaction"`, `"permit2"`, `"hash"`.

If omitted, servers MUST accept `"transaction"` and SHOULD
accept `"hash"`. Support for `"permit2"` is OPTIONAL and
depends on Permit2 deployment on the target chain. Clients
SHOULD use the first type in the list that they support.

### Payment Splits {#split-payments}

The `splits` field enables a single charge to distribute
payment across multiple recipients. This is useful for
platform fees, revenue sharing, and marketplace payouts.

Each entry in the `splits` array is a JSON object:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `recipient` | string | REQUIRED | EIP-55 address of split recipient |
| `amount` | string | REQUIRED | Amount in base units for this recipient |
| `memo` | string | OPTIONAL | Human-readable label (max 256 chars) |

The top-level `amount` represents the total the client pays.
The primary `recipient` receives the remainder: `amount` minus
the sum of all split amounts.

Constraints:

- The sum of `splits[].amount` MUST be strictly less than
  `amount`. Clients MUST reject any request that violates
  this constraint.
- If present, `splits` MUST contain at least 1 entry.
  Servers SHOULD limit splits to 8 entries.
- All transfers MUST target the same `currency` token.
- Address fields are compared by decoded 20-byte value, not
  by string form.

The order of entries in `splits` is not significant for
verification. Clients SHOULD emit transfers in array order.
Servers MUST verify that the required payment effects are
present regardless of ordering.

**Example:**

~~~json
{
  "amount": "1050000",
  "currency": "0xe15fc38f6d8c56af07bbcbe3baf5708a2bf42392",
  "recipient": "0x742d35Cc6634C0532925a3b844Bc9e7595f8fE00",
  "description": "Marketplace purchase",
  "methodDetails": {
    "chainId": 1329,
    "splits": [
      {
        "recipient": "0x8Ba1f109551bD432803012645Ac136ddd64DBA72",
        "amount": "50000",
        "memo": "platform fee"
      }
    ]
  }
}
~~~

This requests a total payment of 1.05 USDC. The platform
receives 0.05 USDC and the primary recipient receives 1.00
USDC.

### Split Atomicity

Split atomicity depends on the credential type:

- **`type="transaction"`**: Clients MAY batch multiple ERC-20
  transfers into a single transaction using multicall
  contracts or smart account batching. When batched, all
  transfers succeed or fail atomically. When executed as
  separate transactions, atomicity is not guaranteed.

- **`type="permit2"`**: Each split requires a separate
  `permitWitnessTransferFrom()` call. These are NOT atomic.
  Servers MUST execute the primary transfer before splits.
  If a split fails after the primary succeeds, servers MUST
  still return a receipt for the primary transfer and SHOULD
  log the partial failure.

Servers MAY mitigate partial execution risk by simulating all
transfers via `eth_call` before submitting any on-chain.

# Credential Schema

The credential in the `Authorization` header contains a
base64url-encoded JSON object per {{I-D.httpauth-payment}}.

## Credential Structure

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `challenge` | object | REQUIRED | Echo of the challenge from the server |
| `payload` | object | REQUIRED | EVM-specific payload |
| `source` | string | OPTIONAL | Payer identifier as a DID |

The `source` field, if present, SHOULD use the `did:pkh` method
with the chain ID from the challenge and the payer's address
(e.g., `did:pkh:eip155:4326:0x1234...`).

## Transaction Payload (type="transaction") {#transaction-payload}

The default credential type. The client signs a complete ERC-20
`transfer` transaction targeting the `currency` contract. The
server broadcasts the transaction to the chain.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | `"transaction"` |
| `signature` | string | REQUIRED | Hex-encoded RLP-serialized signed transaction |

The `signature` field contains an EIP-1559 (type 2) transaction,
RLP-encoded and hex-prefixed with `0x`. The transaction MUST
call `transfer(address,uint256)` on the ERC-20 token specified
in the challenge.

When `feePayer` is `false` or omitted, the client MUST sign
a fully valid transaction including gas parameters. When
`feePayer` is `true`, the client signs the transaction with
the server's designated address as `from` for fee purposes.
See {{fee-payment}} for details.

When `splits` are present, the client MUST include transfer
instructions for each split. Clients MAY use multicall
contracts, EIP-7702, or smart account batching to achieve
atomicity.

**Example:**

~~~json
{
  "challenge": {
    "id": "kM9xPqWvT2nJrHsY4aDfEb",
    "realm": "api.example.com",
    "method": "evm",
    "intent": "charge",
    "request": "eyJ...",
    "expires": "2026-04-01T12:05:00Z"
  },
  "payload": {
    "signature": "0x02f8...signed transaction bytes...",
    "type": "transaction"
  },
  "source": "did:pkh:eip155:1329:0x1234567890abcdef1234567890abcdef12345678"
}
~~~

## Permit2 Payload (type="permit2") {#permit2-payload}

The client signs an off-chain EIP-712 Permit2
`PermitWitnessTransferFrom` message. The server constructs and
submits the on-chain transaction. This type requires that the
Permit2 contract is deployed on the target chain and that the
client has an active ERC-20 approval to the Permit2 contract.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | `"permit2"` |
| `permit` | object | REQUIRED | Permit2 permit data |
| `witness` | object | REQUIRED | Transfer witness data |
| `signature` | string | REQUIRED | EIP-712 signature (`0x`-prefixed) |

The `permit` object:

| Field | Type | Description |
|-------|------|-------------|
| `permitted` | object | `{ token, amount }` — token address and maximum transfer amount |
| `nonce` | string | Permit2 nonce (stringified integer) |
| `deadline` | string | Unix timestamp (stringified integer) |

The `witness` object:

| Field | Type | Description |
|-------|------|-------------|
| `transferDetails` | object | `{ to, requestedAmount }` — recipient and exact amount |

**Prerequisite:** The client MUST have an active ERC-20
approval from the `currency` token to the Permit2 contract.
This is a one-time operation per token per chain.

When `splits` are present, the client MUST provide separate
Permit2 signatures for each split. The `payload` is extended
with a `splitSignatures` array:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `splitSignatures` | array | Conditionally REQUIRED | Array of `{ permit, witness, signature }` objects, one per split entry |

Each entry in `splitSignatures` follows the same schema as the
top-level `permit`, `witness`, and `signature` fields.

**Example:**

~~~json
{
  "challenge": {
    "id": "aB3cDeF4gHiJkLmN",
    "realm": "api.example.com",
    "method": "evm",
    "intent": "charge",
    "request": "eyJ...",
    "expires": "2026-04-01T12:05:00Z"
  },
  "payload": {
    "type": "permit2",
    "permit": {
      "permitted": {
        "token": "0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7",
        "amount": "1000000000000000000"
      },
      "nonce": "1",
      "deadline": "1743523500"
    },
    "witness": {
      "transferDetails": {
        "to": "0x742d35Cc6634C0532925a3b844Bc9e7595f8fE00",
        "requestedAmount": "1000000000000000000"
      }
    },
    "signature": "0x1b2c3d4e5f..."
  },
  "source": "did:pkh:eip155:4326:0x1234...5678"
}
~~~

## Hash Payload (type="hash") {#hash-payload}

When the client has already broadcast the transaction to the
chain, the payload contains only the transaction hash:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | `"hash"` |
| `hash` | string | REQUIRED | Transaction hash (`0x`-prefixed, 32 bytes hex) |

**Limitations:**

- MUST NOT be used when `feePayer` is `true`. Servers MUST
  reject `type="hash"` credentials when the challenge
  specifies `feePayer: true`.
- Server cannot modify or retry the transaction.
- Weaker challenge binding than other payload types (see
  {{hash-binding}}).

**Example:**

~~~json
{
  "challenge": {
    "id": "kM9xPqWvT2nJrHsY4aDfEb",
    "realm": "api.example.com",
    "method": "evm",
    "intent": "charge",
    "request": "eyJ...",
    "expires": "2026-04-01T12:05:00Z"
  },
  "payload": {
    "hash": "0x1a2b3c...7890",
    "type": "hash"
  },
  "source": "did:pkh:eip155:4326:0x1234567890abcdef1234567890abcdef12345678"
}
~~~

# Fee Payment {#fee-payment}

Gas costs vary significantly across EVM chains. On high-fee
chains (Ethereum L1), fee sponsorship is expensive. On low-fee
chains (MegaETH, Sei, L2 rollups), gas costs are negligible
and servers SHOULD sponsor fees by default.

## Server-Paid Fees

When `feePayer` is `true`:

- **For `type="permit2"`:** The client signs only the
  off-chain Permit2 authorization. The server constructs the
  on-chain transaction using its own hot wallet and pays gas
  from its own balance. The client never interacts with the
  chain directly.

- **For `type="transaction"`:** The client signs a transaction
  but the server's fee payer account is the `from` address
  on the transaction itself. Implementations SHOULD prefer
  `type="permit2"` with `feePayer: true` as it provides a
  cleaner separation of concerns.

## Client-Paid Fees

When `feePayer` is `false` or omitted, the client constructs
and signs a complete transaction including gas. The client MAY
use `type="transaction"` (server broadcasts) or `type="hash"`
(client broadcasts).

## Server Requirements

When acting as fee payer, servers:

- MUST maintain sufficient native token balance to cover gas
- MUST verify credential contents before spending gas
- SHOULD implement rate limiting to mitigate gas exhaustion
  attacks
- SHOULD simulate transactions via `eth_call` before broadcast

## Client Requirements

- When `feePayer` is `true`: clients MUST use `type="permit2"`
  or `type="transaction"` and MUST NOT use `type="hash"`
- When `feePayer` is `false` or omitted: clients MAY use any
  supported credential type

# Verification Procedure {#verification}

Upon receiving a request with a credential, the server MUST:

1. Decode the base64url credential and parse the JSON.
2. Verify that `payload.type` is present and is one of
   `"transaction"`, `"permit2"`, or `"hash"`.
3. Look up the stored challenge using `credential.challenge.id`.
   If no matching challenge is found, reject the request.
4. Verify that all fields in `credential.challenge` exactly
   match the stored challenge auth-params.
5. If `payload.type` is `"hash"` and the challenge specifies
   `feePayer: true`, reject the request.
6. Proceed with type-specific verification:
   - For `type="transaction"`: see {{transaction-verification}}.
   - For `type="permit2"`: see {{permit2-verification}}.
   - For `type="hash"`: see {{hash-verification}}.

## Transaction Verification {#transaction-verification}

Before broadcasting, servers MUST verify:

1. Deserialize the RLP-encoded transaction from
   `payload.signature`
2. Verify the transaction `chainId` matches
   `methodDetails.chainId`
3. Verify the transaction `to` address matches the `currency`
   token contract
4. Verify the transaction calldata begins with the
   `transfer(address,uint256)` function selector
   (`0xa9059cbb`)
5. Decode the calldata and verify `recipient` and `amount`
   match the challenge request
6. If `splits` are present, verify that additional transfer
   instructions are included for each split entry
7. Broadcast the transaction via `eth_sendRawTransaction`
8. Wait for confirmation and fetch the transaction receipt
9. Verify the receipt `status` is `0x1` (success)
10. Verify the receipt contains `Transfer` event logs matching
    the challenge parameters

## Permit2 Verification {#permit2-verification}

Before submitting, servers MUST verify:

1. The EIP-712 signature is valid and recovers to the
   `source` address
2. The `permitted.token` matches `currency`
3. The `permitted.amount` is sufficient for the transfer
4. The `witness.transferDetails.to` matches `recipient`
5. The `witness.transferDetails.requestedAmount` matches
   `amount` (minus sum of splits, if primary transfer)
6. The `deadline` has not passed
7. The signer has sufficient token balance
8. The signer has sufficient Permit2 allowance
9. If `splits` are present, verify each entry in
   `splitSignatures` with the same checks
10. Call `Permit2.permitWitnessTransferFrom()` for the
    primary transfer, then for each split in order
11. Verify transaction receipt(s) indicate success

## Hash Verification {#hash-verification}

For hash credentials, servers MUST:

1. Verify `payload.hash` has not been previously consumed
   (see {{replay-protection}})
2. Fetch the transaction receipt via
   `eth_getTransactionReceipt`
3. Verify `status` is `0x1` (success)
4. Verify the receipt contains `Transfer` event log(s):
   - Log `address` matches `currency`
   - `to` parameter matches `recipient`
   - `value` parameter matches `amount` (for the primary
     transfer: `amount` minus sum of splits if present)
5. If `splits` are present, verify additional `Transfer`
   logs for each split entry
6. Mark the hash as consumed

# Settlement Procedure

## Transaction Settlement

~~~
Client                  Server               EVM Chain
  |                        |                      |
  | (1) Authorization:     |                      |
  |     Payment <cred>     |                      |
  |----------------------->|                      |
  |                        | (2) Broadcast tx     |
  |                        |--------------------->|
  |                        | (3) Confirmation     |
  |                        |<---------------------|
  | (4) 200 OK + Receipt   |                      |
  |<-----------------------|                      |
~~~

## Permit2 Settlement

~~~
Client                  Server               EVM Chain
  |                        |                      |
  | (1) Authorization:     |                      |
  |     Payment <cred>     |                      |
  |  (Permit2 signature)   |                      |
  |----------------------->|                      |
  |                        | (2) Verify sig       |
  |                        | (3) Submit           |
  |                        |  permitWitness-      |
  |                        |  TransferFrom()      |
  |                        |--------------------->|
  |                        | (4) Receipt          |
  |                        |<---------------------|
  | (5) 200 OK + Receipt   |                      |
  |<-----------------------|                      |
~~~

## Hash Settlement

~~~
Client                  Server               EVM Chain
  |                        |                      |
  | (1) Broadcast tx       |                      |
  |---------------------------------------------->|
  | (2) Confirmed          |                      |
  |<----------------------------------------------|
  |                        |                      |
  | (3) Authorization:     |                      |
  |     Payment <cred>     |                      |
  |  (tx hash)             |                      |
  |----------------------->|                      |
  |                        | (4) getTransaction-  |
  |                        |     Receipt          |
  |                        |--------------------->|
  |                        | (5) Verify           |
  |                        |<---------------------|
  | (6) 200 OK + Receipt   |                      |
  |<-----------------------|                      |
~~~

## Chain-Specific Optimizations

EVM chains MAY offer RPC extensions that improve settlement
latency. Servers SHOULD use these when available:

| Chain | Optimization | Benefit |
|-------|-------------|---------|
| MegaETH | `realtime_sendRawTransaction` | Returns receipt inline (no polling) |
| Any | WebSocket `eth_subscribe` | Push-based confirmation |
| Any | `eth_call` simulation | Pre-flight validation |

These optimizations are transparent to the client and do not
affect the credential format or verification procedure.

## Confirmation Requirements

Servers MUST wait for at least one block confirmation before
returning a receipt. The required confirmation depth is a
server policy decision and MAY vary based on transaction value
and chain finality characteristics.

As a guideline:

| Chain Property | Recommended Confirmations |
|---------------|--------------------------|
| Sub-second finality (MegaETH, Sei) | 1 block |
| L2 rollups (Optimism, Base, Arbitrum) | 1 block |
| Ethereum L1 | 1-12 blocks (value-dependent) |

## Receipt Generation

Upon successful settlement, servers MUST return a
`Payment-Receipt` header per {{I-D.httpauth-payment}}.
Servers MUST NOT include a `Payment-Receipt` header on error
responses; failures are communicated via HTTP status codes
and Problem Details {{RFC9457}}.

The receipt payload:

| Field | Type | Description |
|-------|------|-------------|
| `method` | string | `"evm"` |
| `challengeId` | string | The `id` from the original challenge |
| `reference` | string | Transaction hash (`0x`-prefixed) |
| `status` | string | `"success"` |
| `timestamp` | string | {{RFC3339}} settlement time |
| `chainId` | number | Chain ID where settlement occurred |
| `externalId` | string | OPTIONAL. Echoed from the challenge request |

# Replay Protection {#replay-protection}

Servers MUST maintain a set of consumed credential identifiers.
The replay prevention token depends on the credential type:

- **`type="transaction"`**: The transaction hash (derived
  after broadcast) serves as the replay token.
- **`type="permit2"`**: The combination of signer address
  and Permit2 nonce serves as the replay token. The nonce
  is consumed on-chain by the Permit2 contract.
- **`type="hash"`**: The transaction hash provided by the
  client serves as the replay token.

Before accepting a credential, the server MUST check whether
its replay token has already been consumed. After successful
verification, the server MUST atomically mark it as consumed.

# Error Responses

When rejecting a credential, the server MUST return HTTP 402
(Payment Required) with a fresh `WWW-Authenticate: Payment`
challenge per {{I-D.httpauth-payment}}. The server SHOULD
include a response body conforming to Problem Details
{{RFC9457}} with `Content-Type: application/problem+json`.

Servers MUST use the standard problem types defined in
{{I-D.httpauth-payment}}: `malformed-credential`,
`invalid-challenge`, and `verification-failed`. The `detail`
field SHOULD contain a human-readable description of the
specific failure.

Example:

~~~json
{
  "type": "https://paymentauth.org/problems/verification-failed",
  "title": "Transfer Mismatch",
  "status": 402,
  "detail": "Transfer amount does not match challenge request"
}
~~~

# Security Considerations

## Transport Security

All communication MUST use TLS 1.2 or higher per
{{I-D.httpauth-payment}}. Credentials MUST only be transmitted
over HTTPS connections.

## Transaction Replay

EIP-1559 transactions include chain ID and nonce, preventing
cross-chain and same-chain replay. Permit2 signatures include
chain ID in the EIP-712 domain separator and consume nonces
on-chain. The `expires` auth-param limits the temporal window
for credential use.

## Amount Verification

Clients MUST parse and verify the `request` payload before
signing:

1. Verify `amount` is reasonable for the service
2. Verify `currency` is the expected token address
3. Verify `recipient` is controlled by the expected party
4. Verify `chainId` matches the expected network
5. If `splits` are present, verify the sum of split amounts
   is strictly less than `amount` and all split recipients
   are expected

## Hash Credential Binding {#hash-binding}

Hash credentials (`type="hash"`) provide weaker challenge
binding than transaction or Permit2 credentials. The server
verifies that a payment matching the challenge terms exists
on-chain, but cannot prove the payment was created
specifically for this challenge instance. If multiple valid
challenges have identical terms, the same transaction could
satisfy any one of them.

Servers MAY mitigate this by:

- Requiring unique `externalId` values per challenge and
  verifying them on-chain (e.g., via event data)
- Preferring `type="transaction"` or `type="permit2"`
  over `type="hash"` in `credentialTypes`
- Restricting `type="hash"` to low-value transactions

## Fee Payer Risks

Servers acting as fee payers accept financial risk:

**Denial of Service**: Malicious clients could submit
credentials that fail on-chain, causing the server to pay
gas without receiving payment. Mitigations:

- Simulate transactions via `eth_call` before broadcast
- Rate limit per client address and IP
- Verify client token balance before signing
- Require client authentication before accepting
  fee-sponsored credentials

**Balance Exhaustion**: Servers MUST monitor fee payer
balance and reject new fee-sponsored requests when
insufficient.

## Permit2-Specific Risks

**Allowance Prerequisite**: Permit2 requires a one-time
ERC-20 `approve()` to the Permit2 contract. Clients should
understand they are granting approval to a third-party
contract. The Permit2 contract is widely deployed and
audited, but clients SHOULD verify the contract address
matches the canonical deployment.

**Nonce Management**: Permit2 nonces are consumed on-chain.
If a server fails to submit a Permit2 credential, the nonce
remains unconsumed and the client can reuse it. Servers MUST
handle nonce conflicts gracefully.

## Split Payment Risks

**Recipient Transparency**: Clients SHOULD present each
split recipient and amount so the user can verify the payment
distribution. Clients SHOULD highlight when the primary
recipient receives a small remainder relative to the total
`amount`.

**Partial Execution**: When splits are non-atomic (Permit2
credentials), a split may fail after the primary transfer
succeeds. Servers MUST handle partial execution gracefully
and SHOULD simulate all transfers before submitting.

## RPC Trust

Servers rely on their RPC endpoint for transaction data. A
compromised RPC could return fabricated data. Servers SHOULD
use trusted RPC providers or run their own nodes. This is
especially important for hash credential verification where
the server relies entirely on RPC-provided receipts.

# IANA Considerations

## Payment Method Registration

This document registers the following payment method in the
"HTTP Payment Methods" registry established by
{{I-D.httpauth-payment}}:

| Method Identifier | Description | Reference |
|-------------------|-------------|-----------|
| `evm` | EVM-compatible blockchain ERC-20 token transfer | This document |

Contact: Brett DiNovi (<bread@megaeth.com>),
Kartik Bhat (<kartik@seinetwork.io>)

## Payment Intent Registration

This document registers the following payment intent in the
"HTTP Payment Intents" registry established by
{{I-D.httpauth-payment}}:

| Intent | Applicable Methods | Description | Reference |
|--------|-------------------|-------------|-----------|
| `charge` | `evm` | One-time ERC-20 token transfer on any EVM chain | This document |

--- back

# Chain-Specific Notes

This appendix documents notable chain-specific behaviors that
implementers should be aware of. These notes are informational
and do not change the core specification.

## MegaETH (Chain ID: 4326)

- **Block time**: ~10ms (mini blocks), ~1s (EVM blocks)
- **Settlement latency**: Sub-50ms end-to-end when using
  `realtime_sendRawTransaction`
- **Intrinsic gas**: 60,000 (vs 21,000 on Ethereum) due
  to multidimensional gas model. Servers setting gas limits
  for Permit2 transactions MUST account for this.
- **Permit2**: Deployed at canonical address
- **Gas costs**: <$0.001 per tx. Servers SHOULD set
  `feePayer: true` by default.
- **`SELFDESTRUCT`**: Disabled. Contracts relying on this
  opcode will fail.

## Sei (Chain ID: 1329)

- **Block time**: ~400ms
- **Gas costs**: Low. Servers SHOULD consider `feePayer: true`.
- **EVM compatibility**: Full ERC-20 and EIP-1559 support
- **Permit2**: Deployed at canonical address

## Ethereum L1 (Chain ID: 1)

- **Block time**: ~12s
- **Gas costs**: Variable, potentially high. Fee sponsorship
  may not be economical; servers MAY set `feePayer: false`.
- **Confirmation**: Servers SHOULD require more confirmations
  for high-value transactions due to reorg risk.

## L2 Rollups (Optimism, Base, Arbitrum)

- **Block time**: 250ms-2s depending on chain
- **Gas costs**: Low to moderate
- **Finality**: Soft finality is fast; full L1 finality
  takes longer. For most payment use cases, soft finality
  is sufficient.

# Full Example: ERC-20 Charge with Permit2

**1. Challenge (402 response):**

~~~http
HTTP/1.1 402 Payment Required
WWW-Authenticate: Payment id="aB3cDeF4gHiJkLmN",
  realm="api.example.com",
  method="evm",
  intent="charge",
  request="eyJhbW91bnQiOiIxMDAwMDAwMDAwMDAwMDAwMDAwIiwiY3
    VycmVuY3kiOiIweEZBZkRkYmIzRkM3Njg4NDk0OTcxYTc5Y2M2NU
    RDYTNFRjgyMDc5RTciLCJtZXRob2REZXRhaWxzIjp7ImNoYWluSWQ
    iOjQzMjYsImZlZVBheWVyIjp0cnVlfSwicmVjaXBpZW50IjoiMHg3
    NDJkMzVDYzY2MzRDMDUzMjkyNWEzYjg0NEJjOWU3NTk1ZjhmRTAwI
    n0",
  expires="2026-04-01T12:05:00Z"
Cache-Control: no-store
~~~

Decoded `request`:

~~~json
{
  "amount": "1000000000000000000",
  "currency": "0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7",
  "recipient": "0x742d35Cc6634C0532925a3b844Bc9e7595f8fE00",
  "methodDetails": {
    "chainId": 4326,
    "feePayer": true
  }
}
~~~

This requests 1.0 USDm (10^18 base units) on MegaETH with
server-paid gas.

**2. Credential (Permit2 authorization):**

~~~http
GET /api/resource HTTP/1.1
Host: api.example.com
Authorization: Payment eyJjaGFsbGVuZ2UiOnsiaWQiOiJhQjNjRGVG
  NGdIaUprTG1OIn0sInBheWxvYWQiOnsidHlwZSI6InBlcm1pdDIiLCJw
  ZXJtaXQiOnsicGVybWl0dGVkIjp7InRva2VuIjoiMHhGQWZEZGJiM0ZD
  NzY4ODQ5NDk3MWE3OWNjNjVEQ2EzRUY4MjA3OUU3IiwiYW1vdW50Ijoi
  MTAwMDAwMDAwMDAwMDAwMDAwMCJ9LCJub25jZSI6IjEiLCJkZWFkbGlu
  ZSI6IjE3NDM1MjM1MDAifSwid2l0bmVzcyI6eyJ0cmFuc2ZlckRldGFp
  bHMiOnsidG8iOiIweDc0MmQzNUNjNjYzNEMwNTMyOTI1YTNiODQ0QmM5
  ZTc1OTVmOGZFMDAiLCJyZXF1ZXN0ZWRBbW91bnQiOiIxMDAwMDAwMDAw
  MDAwMDAwMDAwIn19LCJzaWduYXR1cmUiOiIweDFiMmMzZDRlNWYuLi4i
  fX0
~~~

**3. Response (with receipt):**

~~~http
HTTP/1.1 200 OK
Payment-Receipt: eyJtZXRob2QiOiJldm0iLCJjaGFsbGVuZ2VJZCI6Im
  FCM2NEZUY0Z0hpSmtMbU4iLCJyZWZlcmVuY2UiOiIweGFiYzEyMy4u
  LiIsInN0YXR1cyI6InN1Y2Nlc3MiLCJ0aW1lc3RhbXAiOiIyMDI2LTA
  0LTAxVDEyOjA0OjU4WiIsImNoYWluSWQiOjQzMjZ9
Content-Type: application/json

{"response": "resource data"}
~~~

Decoded receipt:

~~~json
{
  "method": "evm",
  "challengeId": "aB3cDeF4gHiJkLmN",
  "reference": "0xabc123...",
  "status": "success",
  "timestamp": "2026-04-01T12:04:58Z",
  "chainId": 4326
}
~~~

# Full Example: ERC-20 Charge with Signed Transaction

**Challenge** requests 1.0 USDC on Sei:

~~~json
{
  "amount": "1000000",
  "currency": "0xe15fc38f6d8c56af07bbcbe3baf5708a2bf42392",
  "recipient": "0x742d35Cc6634C0532925a3b844Bc9e7595f8fE00",
  "description": "Premium API call",
  "methodDetails": {
    "chainId": 1329
  }
}
~~~

**Credential** (signed EIP-1559 transaction):

~~~json
{
  "challenge": {
    "id": "kM9xPqWvT2nJrHsY4aDfEb",
    "realm": "api.example.com",
    "method": "evm",
    "intent": "charge",
    "request": "eyJ...",
    "expires": "2026-04-01T12:05:00Z"
  },
  "payload": {
    "type": "transaction",
    "signature": "0x02f8...signed transaction bytes..."
  },
  "source": "did:pkh:eip155:1329:0x1234567890abcdef1234567890abcdef12345678"
}
~~~

# Acknowledgements

The authors thank Georgios Konstantopoulos for guidance on
consolidating chain-specific specs into a unified EVM method,
Brendan Ryan and Jake Moxey at Tempo Labs for the MPP framework,
and the Sei and MegaETH communities for their contributions to
earlier drafts.
