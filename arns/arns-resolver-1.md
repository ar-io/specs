# ARNS-RESOLVER-1

## Status:

**Draft**

## Version:

| Version | Description                                                          | Date       |
| ------- | -------------------------------------------------------------------- | ---------- |
| 1.0.0   | Initial version of the ARNS-RESOLVER-1 specification.                | 2024-09-01 |
| 2.0.0   | Solana migration: multi-protocol resolution, updated data model.     | 2026-04-20 |
| 2.1.0   | Pluggable ANT program: AntRecord PDAs are derived against the program named in the asset's `ANT Program` Attributes-plugin entry. New `x-arns-ant-program` response header. | 2026-05-03 |
| 2.1.1   | Audit corrections: `NameRegistry` slot count is 200,000 (not 50,000). | 2026-05-11 |
| 2.1.2   | Added explicit `ArnsRecord` / `ReturnedName` / `ReservedName` PDA derivation; AntRecordMetadata fields split out of AntRecord listing; event-subscription guidance. | 2026-05-11 |

## Abstract

The **ARNS-RESOLVER-1** specification defines the protocols and patterns that an ArNS Resolver must follow to accurately retrieve and serve the latest state of AR.IO Names from the ArNS Registry. It ensures that names are correctly resolved to their associated content targets and AR.IO Name Token (ANT) identifiers within the AR.IO network, supporting multiple storage protocols (Arweave, IPFS, and future networks) through a single resolution layer.

## Motivation

Resolvers are critical to ensuring that users can reliably access content associated with ArNS names. This specification standardizes the resolution process, enabling consistent behavior across different AR.IO gateways and infrastructure.

With the migration from AO to Solana, the data model has changed: ArNS Registry records are now on-chain Solana accounts managed by the `ario-arns` program, and ANT records are PDA accounts managed by the `ario-ant` program. ANTs are Metaplex Core NFTs on Solana, and each ANT record carries a `target_protocol` field that declares which storage protocol the content address belongs to (Arweave, IPFS, or future protocols).

By establishing clear protocols for reading registry state, resolving ANT records, routing by storage protocol, and caching results, **ARNS-RESOLVER-1** ensures that resolvers operate efficiently and securely. This foundation is crucial for maintaining the integrity and performance of the AR.IO Name System, particularly as the ecosystem scales to support multiple storage networks.

## Specification

### Overview

The resolver must adhere to the following protocols to correctly identify, resolve, and serve ArNS names:

- **Expired Names**: Must not serve names that have expired from the ArNS Registry. The `ArnsRecord` account stores `purchase_type` (Lease or Permabuy) and `end_timestamp`. Leases with `end_timestamp < now` are expired and must not be served.
- **Undernames**:
  - Must not serve undernames (i.e., subdomains) beyond the `undername_limit` purchased for the name.
  - Must not serve undernames that exceed the maximum DNS label limit. The maximum length of each label is 63 characters, and a full domain name can have a maximum of 253 characters.
  - The maximum undername length is 61 characters. Undernames must start with an alphanumeric character and may contain alphanumeric characters, hyphens, and underscores.
- **Root Record**: The record with undername `"@"` is the root of the ArNS Name. All undernames for a given name must be sorted by priority, with the root `@` at priority 0.
- **Regex Compliance**: Must not serve undernames that do not match the ArNS standard pattern: `^@$` or `^[a-zA-Z0-9]+[a-zA-Z0-9_-]*$`.
- **Caching**:
  - Must cache each record for the time-to-live (TTL) stored in the record's `ttl_seconds` field.
  - The on-chain minimum TTL is 60 seconds. The resolver SHOULD enforce a minimum cache TTL of 900 seconds (15 minutes) to reduce RPC load and improve performance.
- **Multi-Protocol Resolution**: Must read the `target_protocol` field from each ANT record and route content fetching accordingly. See the [Multi-Protocol Resolution](#multi-protocol-resolution) section.
- **Response Headers**: Must respond to requests with the following headers:
  - `x-arns-ant-id` — the ANT's Metaplex Core NFT mint address on Solana (base58).
  - `x-arns-ant-program` — the program ID resolved from the asset's `ANT Program` Attributes-plugin entry (or canonical `ARIO_ANT_PROGRAM_ID` when the trait is absent). Surfaces the BYO-ANT routing decision so observers can audit which program a given name resolved through.
  - `x-arns-resolved-id` — the content target that the name resolved to (Arweave TX ID, IPFS CID, etc.).
  - `x-arns-resolved-protocol` — the storage protocol used: `arweave` or `ipfs`.
  - Example:
    - `x-arns-ant-id: 7xKXjH7YCdMw5vZwmTpKNoBUtFbqRXEBiPRdHE3dBF3`
    - `x-arns-ant-program: ARioAntProgXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX`
    - `x-arns-resolved-id: bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi`
    - `x-arns-resolved-protocol: ipfs`

### Standard Operations

#### 1. Reading Records from the ArNS Registry

The resolver must read `ArnsRecord` accounts from the `ario-arns` program via Solana RPC (or the AR.IO SDK). Each `ArnsRecord` provides:

- The associated ANT (Metaplex Core NFT mint address).
- Lease or permabuy status (`purchase_type`).
- The number of undernames allowed (`undername_limit`).
- The expiration timestamp (`end_timestamp`, present for leases; absent for permabuys).
- The name owner (`owner`).

The resolver may read records individually by deriving the PDA from the name hash, or enumerate all active records via the `NameRegistry` zero-copy account. The AR.IO SDK provides helper methods for both patterns.

PDA derivation under the `ario-arns` program:

```
ArnsRecord    seeds = ["arns_record",   sha256(name.toLowerCase())]
ReturnedName  seeds = ["returned_name", sha256(name.toLowerCase())]
ReservedName  seeds = ["reserved_name", sha256(name.toLowerCase())]
NameRegistry  seeds = ["name_registry"]  (singleton, zero-copy)
```

A name is "live" (resolvable) iff an `ArnsRecord` PDA exists for it. Presence of a `ReturnedName` or `ReservedName` PDA without an `ArnsRecord` means the name is unregistered (auction or reservation only) and MUST NOT be served. See [Name Lifecycle](arns-overview.md#name-lifecycle) in ARNS-OVERVIEW for the full state model.

The resolver SHOULD subscribe to Solana WebSocket notifications (account subscriptions) for real-time updates to registry records, rather than relying solely on polling. This enables faster propagation of new registrations, transfers, and expirations.

#### 2. Reading ANT Record State

For each ArNS name, the resolver MUST first determine which program owns the ANT's record PDAs. Each ANT carries an `ANT Program` entry in its Metaplex Core Attributes plugin (set at `CreateV1` time by the SDK and migration mint paths). The resolver:

1. Fetches the ANT asset (`getAccountInfo(antMint)`).
2. Walks the asset's plugin chain to read the `ANT Program` Attribute (key/value encoding per [ARNS-CORE-1 §Implementation Note](arns-core-1.md#implementation-note) — base58-encoded program address).
3. Treats absence (or any parse failure) as a directive to use the canonical `ARIO_ANT_PROGRAM_ID`.
4. Derives `AntRecord` PDAs against that program: `seeds = ["ant_record", antMint, sha256(undername.toLowerCase())]`.

Reading `AntRecord` accounts uses Solana RPC (or the AR.IO SDK's `SolanaANTReadable.fromAsset(rpc, mint)` factory, which encapsulates the lookup). Each `AntRecord` PDA provides:

- The content target (`target`) — an Arweave TX ID, IPFS CID, or other protocol-specific content address, up to 128 characters.
- The storage protocol (`target_protocol`) — `0` for Arweave, `1` for IPFS, with higher values reserved for future protocols.
- Time-to-live settings (`ttl_seconds`) — between 60 and 86,400 seconds (inclusive).
- The undername (`undername`) — `"@"` for root, or a subdomain label.
- Sort priority (`priority`) and optional delegated record owner (`owner`).
- Optional descriptive metadata (`display_name`, `record_logo`, `record_description`, `record_keywords`) lives on a companion `AntRecordMetadata` PDA (seeds: `["ant_record_meta", mint, sha256(undername.lowercase())]`). Resolvers MAY skip reading this account when only resolution data is needed.

If the ANT's record accounts cannot be read (e.g., the ANT mint does not exist, or the `@` record PDA does not exist under the resolved program), the resolver will be unable to fully resolve the name and MUST return an error or empty resolution. Resolvers MUST NOT silently retry against the canonical program when the asset names a third-party program — the assets's declared program is authoritative for that asset.

#### 3. Local Caching

The resolver should store all retrieved information locally to facilitate fast lookups by users or downstream infrastructure, such as an AR.IO Gateway.

- **ArNS Registry records** should be cached and refreshed at a regular interval. A 15-minute refresh interval is the recommended default.
- **ANT records** should be cached per-undername using the record's `ttl_seconds` value, with a resolver-enforced minimum of 900 seconds.
- **Cache invalidation** may be triggered by Solana WebSocket account change notifications, enabling near-real-time updates without frequent polling.

Cache keys should incorporate the name and undername. Each undername resolves to exactly one (protocol, target) pair, so the cache entry naturally includes the protocol.

### Multi-Protocol Resolution

#### Protocol Routing

Each ANT record declares a `target_protocol` field that determines how the resolver fetches the referenced content:

| `target_protocol` | Protocol | Target Format | Fetch Strategy |
| --- | --- | --- | --- |
| `0` | Arweave | 43-character base64url TX ID | Fetch from Arweave network via the gateway's existing Arweave data path. |
| `1` | IPFS | CIDv0 (`Qm...`, exactly 46 chars) or CIDv1 (multibase-encoded, typically 46-64 chars) | Fetch from IPFS network via the gateway's IPFS sidecar or IPFS gateway. |
| `2+` | Reserved | — | Unsupported. Resolver MUST NOT serve records with unrecognized protocols. Return 502 (Bad Gateway). |

The resolver MUST read `target_protocol` for every record and route accordingly. It MUST NOT assume all targets are Arweave TX IDs.

#### URI Semantics

| URI Pattern | Behavior |
| --- | --- |
| `ar://name` | Resolve the ANT `@` record. Fetch content from whatever protocol the record specifies. `ar://` means "resolve via AR.IO gateway" — it is protocol-agnostic. |
| `ar://name/undername` | Resolve the specified undername record. Route by its `target_protocol`. |
| `ipfs://name` | Resolve the ANT `@` record. The resolver MUST verify that `target_protocol == 1` (IPFS). If the record's protocol is not IPFS, return 404. `ipfs://` is protocol-specific. |
| `ipfs://name/undername` | Resolve the specified undername record. The resolver MUST verify that `target_protocol == 1` (IPFS). If the record's protocol is not IPFS, return 404. |

#### Target Validation

Before serving content, the resolver MUST validate the target value against the declared protocol:

- **Arweave (protocol 0)**: Target must be exactly 43 characters, consisting of base64url characters (`[A-Za-z0-9_-]`).
- **IPFS (protocol 1)**: Target must be a valid CID. Two forms are accepted:
  - **CIDv0**: exactly 46 characters, starts with `Qm`, base58btc body.
  - **CIDv1**: multibase-prefixed (`b`/`B`/`z`/`f`/`u`/`k`), at least 10 characters total, alphanumeric after the prefix byte.

Invalid targets MUST NOT be served. The resolver should log a warning and return an error response.

#### Graceful Degradation

If a gateway does not yet support a record's declared protocol (e.g., a record points to IPFS but the gateway has no IPFS sidecar configured), the resolver SHOULD:

1. Successfully resolve the name (return the ANT ID, target, and protocol in response headers).
2. Return HTTP 502 (Bad Gateway) for the content fetch, indicating the protocol is not supported by this gateway.
3. NOT return 404, which would imply the name does not exist.

This allows clients to discover the content address and protocol even when the serving gateway cannot fetch the content.

#### Subdomain Routing and Name/CID Disambiguation

AR.IO gateways conventionally serve ArNS names at the apex subdomain: `<arns-name>.<gateway-host>` (e.g., `ardrive.arweave.net`). When a gateway also serves IPFS content, the resolver MUST avoid ambiguity between ArNS-name lookups and CID lookups.

##### Why registry-level CID prohibition isn't needed

ArNS name validation (per ARNS-CORE-1) already prevents every collision class that occurs in practice. The table below enumerates each IPFS CID form against ArNS validation rules:

| CID form                          | Length    | Charset / shape                     | Collides with ArNS? |
| --------------------------------- | --------- | ----------------------------------- | ------------------- |
| CIDv0 (`Qm...`)                   | exactly 46 | starts with uppercase `Q`           | No — ArNS rejects uppercase |
| CIDv1 sha-256 base32lower (`b...`) | ~59 chars | lowercase alphanumeric              | No — exceeds 51-char ArNS cap |
| CIDv1 sha-256 base32upper (`B...`) | ~59 chars | uppercase alphanumeric              | No — both reasons |
| CIDv1 sha-256 base58btc (`z...`)   | ~48 chars | mixed-case alphanumeric             | No — base58btc body almost certainly contains uppercase chars (probability of all-lowercase: ~10⁻¹³) |
| CIDv1 sha-256 base16lower (`f...`) | ~70 chars | lowercase hex                       | No — exceeds ArNS cap |
| CIDv1 sha-256 base64url (`u...`)   | ~47 chars | mixed-case + `_-`                   | No — practically always contains uppercase or underscore |
| CIDv1 sha-256 base36lower (`k...`) | ~53 chars | lowercase alphanumeric              | No — exceeds ArNS cap |
| CIDv1 with sha-1 multihash         | ~38 chars | varies                              | **Yes, theoretically** — the only residual collision class. Sha-1 is deprecated for new IPFS content. |

The on-chain ArNS registry deliberately does NOT add CID-shape rejection rules beyond those above. A general "looks like a CID" check would have to reject any name starting with `b`, `f`, or `k` in a wide length range, which would block enormous numbers of legitimate names (`blog`, `bird`, `forum`, `kid`, `bookworm`, etc.) for vanishingly rare collision benefit. The 43-char Arweave-address prohibition works because every Arweave address is exactly 43 chars; CIDs have no equivalent narrow fingerprint.

##### Recommended gateway pattern: path-style IPFS resolution

To avoid the residual sha-1 collision and to avoid provisioning a second wildcard TLS certificate for `*.ipfs.<gateway-host>`, gateways SHOULD serve IPFS content via the path-style endpoint defined by the [IPFS Path Gateway specification](https://specs.ipfs.tech/http-gateways/path-gateway/):

```
<gateway-host>/ipfs/<cid>            ← IPFS content (path-style)
<arns-name>.<gateway-host>           ← ArNS resolution (apex subdomain)
```

Path-style resolution removes ambiguity entirely: ArNS lookups and CID lookups occupy disjoint URL spaces. The existing wildcard certificate that covers `*.<gateway-host>` is sufficient — no additional certificates required.

##### Apex-subdomain CID resolution (alternative)

Gateways MAY also serve CIDs at the apex subdomain (`<cid>.<gateway-host>`) but MUST then implement and document an explicit disambiguation policy. The recommended policy is "ArNS-first":

1. If `<id>` is exactly 46 chars and starts with `Qm` → resolve as CIDv0.
2. If `<id>` length > 51 → resolve as CID (cannot be ArNS).
3. If `<id>` length is exactly 43 → reject (Arweave-address rule).
4. Otherwise: attempt ArNS lookup first. If the name is registered, serve it. If not registered AND `<id>` parses as a valid CID, fall back to IPFS fetch. Otherwise return 404.

Under this policy, the only consequence is that a registered ArNS name which happens to also parse as a valid (rare) sha-1 CIDv1 will shadow that specific CID at the gateway's apex subdomain. The CID remains accessible via the path-style endpoint.

##### Reserving operationally-significant names

The `ario-arns` program exposes a `reserve_name` instruction (admin-only) that prevents a specific string from being registered. This is the appropriate tool for protecting individual sensitive strings — for example `ipfs`, `dweb`, `ar`, `gateway` — that have routing or branding significance for the gateway operator. Broader CID-shape rejection should not be encoded in the registry; targeted reservations should.

### Performance and Compliance

#### Data Source Performance

Solana RPC queries are significantly faster than the prior AO Compute Unit queries. The resolver benefits from:

- **Direct account reads**: `ArnsRecord` and `AntRecord` PDAs are deterministically derivable and readable in a single RPC call.
- **Batched reads**: Multiple records can be fetched in a single `getMultipleAccounts` RPC call.
- **WebSocket subscriptions**: The resolver can subscribe to account changes for real-time cache invalidation, eliminating the need for frequent full-registry polling.
- **Zero-copy registries**: The `NameRegistry` account (200,000 slots) enables full enumeration of all active names in a single read, useful for bootstrap and periodic reconciliation.
- **Event log subscriptions**: All state-changing instructions on the `ario-arns`, `ario-ant`, and `ario-core` programs emit Anchor `#[event]` records (see [ARNS-OVERVIEW §Events](arns-overview.md#events)). Resolvers and indexers SHOULD subscribe via `logsSubscribe` rather than diffing account state; the canonical event ABI is published alongside each program's IDL.

#### Refresh Cadence

By default, the AR.IO ArNS Resolver should refresh the ArNS Registry at least every 15 minutes, ensuring timely updates of new name additions, transfers, and expirations. Resolvers using WebSocket subscriptions may operate with a longer polling interval as a fallback, since real-time updates are received via subscription.

Individual ANT record caches should honor the record's `ttl_seconds` field, with a resolver-enforced minimum of 900 seconds.

#### Observation Compliance

The resolver is expected to perform the above actions in a timely manner to meet the standards defined in the AR.IO Observation and Incentive protocol:

- At launch, observers test resolution of Arweave targets. IPFS observation support will be added in a future phase once gateways support IPFS content retrieval.
- A future protocol capability bitmask on the Gateway struct (in the GAR) will allow gateways to declare which storage protocols they support. Until then, all gateways are expected to support Arweave resolution; IPFS resolution is opt-in at the gateway level.
- It is the responsibility of each ArNS Name owner to ensure their ANT records contain valid, reachable content targets for the declared protocol.
