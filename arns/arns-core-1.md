# ARNS-CORE-1

## Status

**Draft**

## Version

| Version | Description                                           | Date       |
| ------- | ----------------------------------------------------- | ---------- |
| 1.0.0   | Initial version of the **ARNS-CORE-1** specification. | 2024-09-01 |
| 1.1.0   | Added priority field to Records object.               | 2025-07-29 |
| 2.0.0   | Rewrite for Solana: on-chain PDA records, multi-protocol targets, renamed fields, SDK/RPC interface. | 2026-04-20 |
| 2.1.0   | Pluggable ANT program: record PDAs are derived against the program named in the asset's Metaplex Core Attributes plugin (`ANT Program` key). Canonical fallback when absent. | 2026-05-03 |
| 2.1.1   | Audit corrections: keyword/description limits (8 / 256), AntRecordMetadata PDA split, sort/index is read-side, async `ANT.init`. | 2026-05-11 |
| 2.1.2   | Pinned `ANT Program` plugin key/value encoding; Get State naming note. | 2026-05-11 |
| 2.1.3   | Contract-drift corrections: keywords max 3, description max 128; non-root `priority` accepts any non-negative value (matches on-chain). | 2026-06-09 |

## Abstract

The **ARNS-CORE-1** specification defines the foundational requirements for resolving AR.IO Name System (ArNS) names to their corresponding content addresses and owners. It provides the essential framework upon which other functionalities and extensions are built, ensuring consistent name resolution across the AR.IO network.

On Solana, each AR.IO Name Token (ANT) is a Metaplex Core NFT. Its undername records are stored as individual on-chain PDA accounts derived from the NFT mint address and the undername. Reading records is done via Solana RPC account reads or through the AR.IO SDK.

## Motivation

The **ARNS-CORE-1** specification provides the essential building blocks for resolving AR.IO Names within the AR.IO Name System (ArNS). It defines the minimal structure and interface needed to map human-readable names to content addresses, ensuring consistent and reliable name resolution across AR.IO gateways.

This specification establishes a standardized record format and a read interface that are crucial for any ArNS implementation. By adhering to these foundational requirements, developers can build additional functionalities on top of a stable and predictable resolution framework, ensuring interoperability and ease of use within the AR.IO ecosystem.

### Implementation Note

This specification is implementation-agnostic. The canonical implementation stores records as Solana PDA accounts within the `ario-ant` program, and the AR.IO SDK provides a unified read interface. However, the rules and validation constraints defined here apply to all implementations and all ArNS name resolvers.

The program holding an ANT's per-mint state PDAs is declared by an `ANT Program` entry in the asset's Metaplex Core Attributes plugin. Resolvers MUST read this entry to find the program ID and derive `AntRecord` PDAs against it; absence (or any parse failure) MUST fall back to the canonical `ARIO_ANT_PROGRAM_ID`. This makes the ANT program pluggable per-asset (BYO-ANT) without changing the registry-side `ArnsRecord` schema. See ADR-016 / BD-100.

The plugin entry is a Metaplex Core Attribute key/value pair with:

| Field | Value |
| --- | --- |
| `key`   | `"ANT Program"` (exact string, case-sensitive, single space between words) |
| `value` | The ANT program's address as a base58-encoded string (e.g., `"ARioAntProg..."`)         |

```ts
// Resolver pattern (kit-style)
const asset = await fetchEncodedAccount(rpc, antMint).send();
const antProgram = readAttribute(asset.data, 'ANT Program') ?? ARIO_ANT_PROGRAM_ID;
const [recordPda] = await getProgramDerivedAddress({
  programAddress: antProgram,
  seeds: [
    Buffer.from('ant_record'),
    addressEncoder.encode(antMint),
    sha256(undername.toLowerCase()),
  ],
});
```

## Specification

### Overview

The **ARNS-CORE-1** specification includes the following requirements for valid, resolvable records:

**Records**:

- Each ANT MUST have at least one record: the root record identified by the undername `@`.
- Each record is identified by an **undername** (e.g., `@`, `blog`, `docs_v2`).
- Each record MUST include a `target`, which is the content address this undername resolves to.
- Each record MUST include a `targetProtocol`, which identifies the protocol used to retrieve the content at `target`.
- Each record MUST include a `ttlSeconds` value, which is the suggested time-to-live for caching by ArNS name resolvers.
- The `ttlSeconds` value MUST be between 60 and 86400 seconds (inclusive). Resolvers SHOULD cache for at least 900 seconds regardless of the declared TTL.
- Each record MAY include a `priority` value, which determines the sort order of undernames served by gateways.
- The `priority` value for the root record (`@`) MUST be 0.
- The `priority` value for non-root records MUST be a non-negative integer (0 or greater), or omitted (unset). When omitted, the record is sorted lexicographically after all priority-assigned records.
- The undername `@` denotes the root/base name record (i.e., the ArNS name itself with no subdomain prefix).
- The record undername plus its ArNS name MUST NOT exceed 63 characters in total length.
- Non-root undernames MUST match the regular expression `^[a-zA-Z0-9][a-zA-Z0-9_-]*$` (must start with an alphanumeric character; remaining characters may be alphanumeric, hyphens, or underscores).
- Records SHOULD be sorted by priority (ascending), then lexicographically by undername for records without a priority.

**Target and Target Protocol**:

- The `target` field is a content address string with a maximum length of 128 characters.
- The `targetProtocol` field is an integer that identifies how to interpret and retrieve the content at `target`.
- Defined protocol values:

  | Value | Protocol | Target Format | Example |
  | ----- | -------- | ------------- | ------- |
  | 0     | Arweave  | 43-character base64url Arweave transaction ID | `UyC5P5qKPZaltMmmZAWdakhlDXsBF6qmyrbWYFchRTk` |
  | 1     | IPFS     | CIDv0 (`Qm...`, exactly 46 chars) or CIDv1 (multibase-prefixed) | `QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG` (CIDv0) / `bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi` (CIDv1) |

- Protocol values 2 and above are reserved for future use.
- When `targetProtocol` is 0 (Arweave), the `target` MUST be a valid 43-character base64url string (character set: `[a-zA-Z0-9_-]`).
- When `targetProtocol` is 1 (IPFS), the `target` MUST be a valid CID — either CIDv0 (exactly 46 chars, `Qm` prefix, base58btc body) or CIDv1 (multibase-prefixed, alphanumeric body).
- `UyC5P5qKPZaltMmmZAWdakhlDXsBF6qmyrbWYFchRTk` MAY be used as a placeholder target for the Arweave protocol (protocol 0).

**Interface**:

- Implementations MUST provide a way to read a single record by undername.
- Implementations MUST provide a way to read all records for an ANT.
- Implementations MUST provide a way to read the state of the ANT, which includes all records, the current owner, and controller list.

**Resolvability**:

AR.IO Name owners can configure their ANT records as they wish, but all of the above rules are also enforced at each ArNS name resolver that is part of AR.IO gateways.

Any AR.IO Name that is to be resolved by the AR.IO network MUST have an active registration in the ArNS Registry. Additionally, records within the ANT MUST be valid, or else ArNS resolvers and AR.IO gateways MAY refuse to resolve them.

AR.IO Names may not be resolvable under the following scenarios:

- Invalid records or malformed state (e.g., missing `target`, invalid `targetProtocol`, TTL out of range).
- Undernames exceeding the amount paid for (registered) within the ArNS Registry.
- The ANT account does not exist on-chain or is not owned by the expected program.
- The record PDA does not exist or cannot be deserialized.

### Objects

The **ARNS-CORE-1** specification defines the following objects needed to support ArNS name resolution.

#### Record

A record represents a single undername resolution entry within an ANT. On Solana, each record is stored as two PDA accounts under the ANT program:

- **`AntRecord`** — required fields (resolution-critical). Seeds: `["ant_record", <mint>, <undername_hash>]`.
- **`AntRecordMetadata`** — optional descriptive fields (display name, logo, description, keywords). Seeds: `["ant_record_meta", <mint>, <undername_hash>]`. Created lazily on first write to an optional field.

`<mint>` is the Metaplex Core NFT public key and `<undername_hash>` is the SHA-256 hash of the lowercased undername. Resolvers MAY skip reading the metadata PDA if they only need core resolution data; the logical "Record" presented below is the union of both accounts.

**Required fields:**

| Field            | Type          | Description |
| ---------------- | ------------- | ----------- |
| `undername`      | string        | The undername identifier (`@` for root, or a valid subdomain label). Stored lowercase. Max 61 characters. |
| `target`         | string        | Content address (Arweave TX ID, IPFS CID, etc.). Max 128 characters. |
| `targetProtocol` | integer (u8)  | Protocol identifier for content retrieval. `0` = Arweave, `1` = IPFS. |
| `ttlSeconds`     | integer (u32) | Time-to-live in seconds for resolver caching. Range: 60-86400. |

> **SDK property naming note:** The AR.IO SDK exposes `target` as `transactionId` in its TypeScript interface for backward compatibility with the AO-era API. The on-chain Borsh field is `target`; the SDK property is `transactionId`. Both refer to the same value. The `targetProtocol` name is consistent across the on-chain schema and the SDK.

**Optional fields:**

| Field         | Type             | Description |
| ------------- | ---------------- | ----------- |
| `priority`    | integer (u32) or null | Sort order. `0` required for `@`, `>0` for others, `null`/omitted for lexicographic ordering. |
| `owner`       | string or null   | Delegated record owner address. When set, this address can modify the record independently of the ANT owner. |
| `displayName` | string or null   | Human-readable display name. Max 61 characters. |
| `logo`        | string or null   | Logo image address (Arweave TX ID, 43 characters). |
| `description` | string or null   | Free-text description. Max 128 characters. |
| `keywords`    | array of strings or null | Keyword tags. Max 3 keywords, each max 32 characters. |

#### Record Examples

**Root record with Arweave target:**

```
{
  "undername": "@",
  "target": "UyC5P5qKPZaltMmmZAWdakhlDXsBF6qmyrbWYFchRTk",
  "targetProtocol": 0,
  "ttlSeconds": 3600,
  "priority": 0
}
```

**Undername record with IPFS target:**

```
{
  "undername": "docs",
  "target": "bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi",
  "targetProtocol": 1,
  "ttlSeconds": 900,
  "priority": 1
}
```

**Undername record with Arweave target and optional metadata:**

```
{
  "undername": "blog",
  "target": "bNaUFJlfMQ0uqBamLkhNKF96DGCo3bE-5_0Lx3EXAMPLE",
  "targetProtocol": 0,
  "ttlSeconds": 1800,
  "priority": 2,
  "displayName": "My Blog",
  "description": "Personal blog hosted on Arweave",
  "keywords": ["blog", "arweave", "permanent"]
}
```

### Interface

The following read operations are defined by the **ARNS-CORE-1** specification. Implementations MUST support all three.

#### Get Record

Retrieves a single record by undername. Executable by any reader (no authentication required).

##### Parameters

| Name      | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| undername  | string | Yes      | The undername to look up (e.g., `@`, `blog`, `docs_v2`). |

##### Behavior

- If the record exists, return it with all fields (required and optional).
- If the record does not exist, return `undefined` / `null` / an empty result (implementation-dependent).

##### SDK Example

```typescript
import { ANT } from '@ar.io/sdk';
import { createSolanaRpc, mainnet } from '@solana/kit';

const rpc = createSolanaRpc(mainnet('https://api.mainnet-beta.solana.com'));

const ant = await ANT.init({
  backend: 'solana',
  processId: '<ant-mint-address>',
  rpc,
});

// Read the root record
const rootRecord = await ant.getRecord({ undername: '@' });
// => { transactionId: "UyC5P5...", targetProtocol: 0, ttlSeconds: 3600, priority: 0 }

// Read a specific undername
const docsRecord = await ant.getRecord({ undername: 'docs' });
// => { transactionId: "bafybei...", targetProtocol: 1, ttlSeconds: 900, priority: 1 }
```

##### Direct RPC Example (Solana)

Record PDA derivation: `["ant_record", <mint_pubkey>, SHA256(lowercase(undername))]` under the `ario-ant` program.

```typescript
import {
  address,
  fetchEncodedAccount,
  getAddressEncoder,
  getProgramDerivedAddress,
} from '@solana/kit';

const [recordPDA] = await getProgramDerivedAddress({
  programAddress: ARIO_ANT_PROGRAM_ID,
  seeds: [
    'ant_record',
    getAddressEncoder().encode(mintAddress),
    sha256(undername.toLowerCase()),
  ],
});

const account = await fetchEncodedAccount(rpc, recordPDA);
// Deserialize account.data per the AntRecord account layout
```

#### Get Records

Retrieves all records for an ANT. Returns a map of undername to record, sorted by priority (ascending) then lexicographically. Each entry includes an `index` field indicating its position in the sorted order. Executable by any reader.

> **Note.** Sort order and the `index` field are computed at read time by the SDK or resolver. On-chain, each record is a standalone PDA with no enforced ordering — the program does not store or emit an index.

##### Parameters

None.

##### Behavior

- Return all records for the ANT as a map keyed by undername.
- Records MUST be sorted: priority-assigned records first (ascending by priority value), followed by records without a priority (sorted lexicographically by undername).
- Each record in the result includes an `index` field (integer, 0-based) reflecting its position in the sorted order.

##### SDK Example

```typescript
const records = await ant.getRecords();
// => {
//   "@":    { transactionId: "UyC5P5...", targetProtocol: 0, ttlSeconds: 3600, priority: 0, index: 0 },
//   "docs": { transactionId: "bafybei...", targetProtocol: 1, ttlSeconds: 900, priority: 1, index: 1 },
//   "blog": { transactionId: "bNaUFJ...", targetProtocol: 0, ttlSeconds: 1800, priority: 2, index: 2 },
//   "api":  { transactionId: "xR3kLP...", targetProtocol: 0, ttlSeconds: 60, index: 3 }
// }
```

#### Get State

Returns the complete state of the ANT, including all records, the current owner, and the controller list. Executable by any reader.

> **SDK property naming note:** The SDK preserves the legacy AO-era field naming (`Owner`, `Controllers`, `Records`, `Name`, `Ticker`, `Logo`, `Description`, `Keywords`) in this composed view for backward compatibility. The underlying on-chain fields are snake_case (e.g., `last_known_owner`); the SDK aggregates and renames them at read time.

##### Parameters

None.

##### Behavior

- Return a state object containing:
  - `Owner` -- the current owner address of the ANT (the Metaplex Core NFT holder).
  - `Controllers` -- the list of authorized controller addresses.
  - `Records` -- all records (same format as Get Records, but without the `index` field).
  - Additional ANT metadata: `Name`, `Ticker`, `Logo`, `Description`, `Keywords`.

##### SDK Example

```typescript
const state = await ant.getState();
// => {
//   Owner: "5fSD3...",
//   Controllers: ["5fSD3...", "7xKL2..."],
//   Records: {
//     "@":    { transactionId: "UyC5P5...", targetProtocol: 0, ttlSeconds: 3600, priority: 0 },
//     "docs": { transactionId: "bafybei...", targetProtocol: 1, ttlSeconds: 900, priority: 1 }
//   },
//   Name: "my-name",
//   Ticker: "ANT-MY-NAME",
//   Logo: "Sie_26dvGYXLv7c9S4wznlmswoIenDpCY...",
//   Description: "My AR.IO Name Token",
//   Keywords: ["example", "ario"]
// }
```
