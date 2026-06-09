# ARNS-MANAGE-1

## Status:

**Draft**

## Version:

| Version | Description                                             | Date       |
| ------- | ------------------------------------------------------- | ---------- |
| 1.0.0   | Initial version of the **ARNS-MANAGE-1** specification. | 2024-09-01 |
| 1.1.0   | Added AR.IO Network handlers, Priority parameter for Set-Record, Initialized field, and boot handler documentation. | 2025-07-29 |
| 1.2.0   | Added undername ownership with Transfer-Record handler, record metadata fields, and delegated permission model for record owners. | 2025-08-01 |
| 2.0.0   | Rewritten for Solana. ANTs are Metaplex Core NFTs. Owner = NFT holder. Controllers stored in on-chain PDA. Record owner delegation model. AR.IO Network integration instructions moved to ario-arns / ario-core programs. Boot handler removed. | 2026-04-20 |
| 2.1.0   | Pluggable ANT program: management instructions live on the program named in the asset's `ANT Program` Attributes-plugin entry. Conformance is on the third-party program. | 2026-05-03 |
| 2.1.1   | Audit corrections: ADR-016 authorization model for registry-side instructions (caller matches stored `owner`, not current NFT holder), `AntRecordMetadata` PDA split, description max 256, keywords max 8, `reassign_name` parameter clarification. | 2026-05-11 |
| 2.1.2   | Documented full primary-name flow (`request_primary_name`, `remove_primary_name`); added Primary Names concept section and PDA seeds. | 2026-05-11 |
| 2.1.3   | Contract-drift corrections: controllers max 4, keywords max 3, description max 128; `version` fields are a 3-byte `SchemaVersion`; `AntControllers` carries `version`; expiry-close instruction is `close_expired_request`; primary-name fee is a lease/permabuy split. | 2026-06-09 |

## Abstract

The **ARNS-MANAGE-1** specification defines the management and control operations for AR.IO Name Tokens (ANTs). It covers undername record creation, modification, and removal; controller delegation for shared record management; and integration with the AR.IO Network for name release, reassignment, and primary name operations.

On Solana, each ANT is a Metaplex Core NFT. Ownership is determined by who holds the NFT -- there is no stored `Owner` variable. Extended state (configuration, controllers, records) is stored in Program Derived Accounts (PDAs) keyed to the NFT mint address.

The program that owns these PDAs is declared per-asset in the Metaplex Core Attributes plugin under the `ANT Program` key (set at `CreateV1` time by the SDK and migration mint paths). Implementations of this specification SHOULD treat that key as the routing target: derive every management PDA against it and, on absence, fall back to the canonical `ARIO_ANT_PROGRAM_ID`. Third-party programs that wish to plug into the AR.IO Name System MUST conform to the management instruction surface defined here AND mint their assets with `ANT Program: <their_program_id>` in the Attributes plugin so resolvers can route correctly. See ADR-016 / BD-100 for the full design.

## Motivation

The **ARNS-MANAGE-1** specification builds on **ARNS-CORE-1** by adding the management capabilities required for ongoing name administration. AR.IO Names require updates over time -- pointing records at new content, delegating control to collaborators, and managing network-level name operations. This specification standardizes these operations so that ANT owners have a consistent interface for record and controller management across the AR.IO ecosystem.

## Specification

### Overview

The **ARNS-MANAGE-1** specification requires the following capabilities:

**Controller Management:**

- Must maintain a list of `Controllers` -- addresses authorized to manage records on behalf of the NFT holder.
- Must have an `add_controller` instruction to add new controllers.
  - Authorized for the NFT holder or existing controllers only.
  - Maximum 4 controllers per ANT.
- Must have a `remove_controller` instruction to remove controllers.
  - Authorized for the NFT holder or existing controllers only.
- Controllers must be automatically cleared when the NFT is transferred to a new holder (lazy reconciliation).

**Record Management:**

- Must have a `set_record` instruction to create new records and update existing ones.
  - New records: authorized for the NFT holder and controllers only.
  - Existing records: authorized for the NFT holder, controllers, or the record's delegated owner.
  - Supports `target` (content address), `target_protocol` (storage protocol identifier), `ttl_seconds`, `priority`, and optional metadata fields.
  - Supports optional per-record owner delegation.
- Must have a `remove_record` instruction to remove records.
  - Authorized for the NFT holder and controllers only.
  - The root `@` record cannot be removed.
- Must have a `transfer_record` instruction to transfer delegated ownership of a record.
  - Authorized for the NFT holder, controllers, or the current record owner.

**Read Operations:**

- Must support reading the controller list.
- The State read operation (from ARNS-CORE-1) is extended to include the controller list.

**AR.IO Network Integration:**

The following operations interact with the AR.IO Network programs (ario-arns and ario-core), not the ANT program itself. Authorization for name-owner operations is by the stored `owner` field on the relevant on-chain record (set at name purchase or record delegation), not by current Metaplex Core NFT possession -- see [Registry-Side Authorization](#registry-side-authorization) below.

- `release_name` -- releases an ArNS name back to the registry (on ario-arns program).
- `reassign_name` -- reassigns an ArNS name to a different ANT (on ario-arns program).
- `request_primary_name` -- requests that an ArNS name become the caller's primary identity (on ario-core program).
- `approve_primary_name` -- approves a pending primary name request (on ario-core program).
- `remove_primary_name` -- the bound wallet removes its own primary-name binding (on ario-core program).
- `remove_primary_name_for_base_name` -- the name owner revokes a primary-name binding using their name (on ario-core program).

### Ownership Model

Ownership of an ANT is determined solely by who holds the Metaplex Core NFT. There is no stored owner field that can be set independently of NFT possession.

**Lazy ownership reconciliation:** The `AntConfig` account stores a `last_known_owner` field. On every write operation, the program reads the current NFT holder from the Metaplex Core asset account. If the holder differs from `last_known_owner`, the controller list is cleared and `last_known_owner` is updated. This ensures that transferring the NFT via any Metaplex-compatible marketplace or wallet automatically revokes all controller access without requiring a separate transaction.

A permissionless `reconcile` instruction is also available to explicitly trigger reconciliation (e.g., immediately after a marketplace transfer) without performing any other operation.

### Objects

The **ARNS-MANAGE-1** specification includes all objects from **ARNS-CORE-1**, plus the following:

#### AntControllers

Stores the controller list for an ANT. Derived as a PDA with seeds `["ant_controllers", mint]`.

| Field       | Type      | Description                                        |
| ----------- | --------- | -------------------------------------------------- |
| mint        | PublicKey | The Metaplex Core asset (NFT mint) this belongs to |
| controllers | PublicKey[] | Controller addresses (max 4)                    |
| bump        | u8        | PDA bump seed                                      |
| version     | SchemaVersion | Schema version for migrations (3-byte: major/minor/patch) |

#### AntRecord (extended from ARNS-CORE-1)

Each undername record is stored as two PDAs: a required `AntRecord` (resolution-critical fields) and an optional `AntRecordMetadata` (descriptive fields). The metadata account is created lazily when an optional field is first written.

PDA seeds: `["ant_record", mint, hash(undername.lowercase())]`

| Field                 | Type            | Description                                                              |
| --------------------- | --------------- | ------------------------------------------------------------------------ |
| mint                  | PublicKey       | The Metaplex Core asset this record belongs to                           |
| undername             | String          | The undername (e.g. `@`, `blog`, `docs`) -- lowercase, max 61 chars      |
| target                | String          | Content address -- Arweave TX ID (43 chars), IPFS CID, or future protocol. Max 128 chars |
| target_protocol       | u8              | Storage protocol: `0` = Arweave (default), `1` = IPFS.                                  |
| ttl_seconds           | u32             | Cache TTL in seconds (60 -- 86400)                                        |
| priority              | Option\<u32\>   | Sort priority for undername resolution. Must be `0` (or unset) for `@`. Non-`@` records accept any non-negative value. Unset records sort lexicographically |
| owner                 | Option\<PublicKey\> | Delegated record owner. When set, this address can update content fields (target, ttl, metadata) but cannot change priority or record ownership |
| last_reconciled_owner | PublicKey       | ANT owner at last record modification -- used to detect stale record owners after NFT transfer |
| bump                  | u8              | PDA bump seed                                                            |
| version               | SchemaVersion | Schema version for per-record migrations                                 |

#### AntRecordMetadata

PDA seeds: `["ant_record_meta", mint, hash(undername.lowercase())]`

| Field              | Type                | Description                                            |
| ------------------ | ------------------- | ------------------------------------------------------ |
| mint               | PublicKey           | The Metaplex Core asset this metadata belongs to       |
| display_name       | Option\<String\>    | Optional display name (max 61 chars)                   |
| record_logo        | Option\<String\>    | Optional logo (Arweave TX ID, 43 chars)                |
| record_description | Option\<String\>    | Optional description (max 128 chars)                   |
| record_keywords    | Option\<String[]\>  | Optional keywords (max 3, each max 32 chars)           |
| bump               | u8                  | PDA bump seed                                          |
| version            | SchemaVersion | Schema version for per-record-metadata migrations      |

### Instructions

#### Instruction Summary

The following instructions are defined by the **ARNS-MANAGE-1** specification, in addition to the read instructions from **ARNS-CORE-1**.

| Instruction               | Program    | Authorization                          |
| ------------------------- | ---------- | -------------------------------------- |
| `add_controller`          | ario-ant   | NFT holder or controller               |
| `remove_controller`       | ario-ant   | NFT holder or controller               |
| `set_record`              | ario-ant   | NFT holder, controller, or record owner (limited) |
| `remove_record`           | ario-ant   | NFT holder or controller               |
| `transfer_record`         | ario-ant   | NFT holder, controller, or record owner |
| `reconcile`               | ario-ant   | Permissionless                         |
| `release_name`            | ario-arns  | `ArnsRecord.owner`                     |
| `reassign_name`           | ario-arns  | `ArnsRecord.owner`                     |
| `request_primary_name`    | ario-core  | Requestor (signs and pays the fee)     |
| `approve_primary_name`    | ario-core  | `AntRecord.owner` (requested undername)|
| `remove_primary_name`     | ario-core  | Owner of the primary-name binding      |
| `remove_primary_name_for_base_name` | ario-core | `AntRecord.owner` (`@` undername) |

---

#### add_controller

Adds an address to the controller list, granting it permission to manage records for this ANT.

**Authorization:** NFT holder or existing controller.

##### Parameters

| Name       | Type      | Description                                  |
| ---------- | --------- | -------------------------------------------- |
| controller | PublicKey | The address to add as a controller           |

##### Rules

- Caller must be the NFT holder or an existing controller (after lazy reconciliation).
- The `controller` address must not already exist in the controller list.
- The controller list must not exceed 4 entries.

##### Errors

| Error                    | Condition                                  |
| ------------------------ | ------------------------------------------ |
| Unauthorized             | Caller is not the NFT holder or controller |
| ControllerAlreadyExists  | Address is already a controller            |
| MaxControllersReached    | Controller list already has 4 entries     |

---

#### remove_controller

Removes an address from the controller list.

**Authorization:** NFT holder or existing controller.

##### Parameters

| Name       | Type      | Description                                  |
| ---------- | --------- | -------------------------------------------- |
| controller | PublicKey | The controller address to remove             |

##### Rules

- Caller must be the NFT holder or an existing controller (after lazy reconciliation).
- The `controller` address must exist in the controller list.

##### Errors

| Error              | Condition                                  |
| ------------------ | ------------------------------------------ |
| Unauthorized       | Caller is not the NFT holder or controller |
| ControllerNotFound | Address is not in the controller list      |

---

#### set_record

Creates a new undername record or updates an existing one. This is the primary instruction for managing what content an AR.IO Name points to.

**Authorization:**
- Creating a new record: NFT holder or controller only.
- Updating an existing record: NFT holder, controller, or the record's delegated owner.
- Record owners can update `target`, `ttl_seconds`, and metadata fields, but cannot change `priority` or `record_owner`.

##### Parameters

| Name               | Type              | Description                                                                                   |
| ------------------ | ----------------- | --------------------------------------------------------------------------------------------- |
| undername           | String            | The undername to set (e.g. `@`, `blog`, `dapp_ardrive`). Lowercased before storage. Max 61 chars. Must start with alphanumeric (or be exactly `@`) |
| target              | String            | Content address. For Arweave (protocol 0): a 43-character base64url TX ID. For IPFS (protocol 1): a CIDv0 (`Qm...`, 46 chars) or CIDv1 (multibase-prefixed) string. Max 128 chars. |
| target_protocol     | u8                | Storage protocol: `0` = Arweave, `1` = IPFS, `2+` = reserved for future protocols.           |
| ttl_seconds         | u32               | Cache TTL in seconds. Must be between 60 and 86400 (1 minute to 1 day)                       |
| priority            | Option\<u32\>     | Optional sort priority. For `@` records, must be `0` or unset. Only NFT holder / controllers can set this |
| record_owner        | Option\<PublicKey\> | Optional delegated owner address. Only NFT holder / controllers can set or clear this         |
| display_name        | Option\<String\>  | Optional display name for this record (max 61 chars)                                          |
| record_logo         | Option\<String\>  | Optional logo (Arweave TX ID, 43 chars)                                                       |
| record_description  | Option\<String\>  | Optional description (max 128 chars)                                                          |
| record_keywords     | Option\<String[]\> | Optional keywords (max 3 keywords, each max 32 chars, no duplicates)                          |

##### Rules

- Caller must be authorized (see Authorization above).
- `undername` must be a valid format: `@`, or starting with an alphanumeric character followed by alphanumeric, dash, or underscore characters.
- `target` must be a valid content address for the declared `target_protocol` (Arweave: 43 base64url characters; IPFS: CIDv0 `Qm`-prefixed 46-char string OR CIDv1 multibase-prefixed string).
- `ttl_seconds` must be in range [60, 86400].
- For `@` records, `priority` must be `0` or unset (it always resolves to `0`).
- For non-`@` records, if `priority` is provided, the caller must be the NFT holder or a controller.
- Only the NFT holder or a controller can assign or clear `record_owner`.
- On NFT transfer (detected via `last_reconciled_owner` mismatch), the record's `owner` field is automatically cleared before permission checks.
- Keywords must be unique, alphanumeric with dash, underscore, `#`, and `@` allowed, no spaces.

##### Errors

| Error                              | Condition                                                    |
| ---------------------------------- | ------------------------------------------------------------ |
| InvalidUndername                   | Undername format is invalid                                  |
| InvalidTarget                      | Content target is not valid for the declared protocol        |
| InvalidTtl                         | TTL is outside [60, 86400] range                             |
| CannotChangePriorityOfRoot         | Attempted to set non-zero priority on `@` record             |
| PriorityRequiresOwnerOrController  | Record owner tried to change priority                        |
| OnlyOwnerOrControllerCanCreate     | Record owner or unauthorized user tried to create new record |
| UnauthorizedRecordAccess           | Caller has no permission for this record                     |

---

#### remove_record

Removes an undername record, closing its PDA account and returning rent to the caller.

**Authorization:** NFT holder or controller only.

##### Parameters

No instruction parameters. The record to remove is identified by the PDA derivation (the record account is passed as part of the account context).

##### Rules

- Caller must be the NFT holder or a controller (after lazy reconciliation).
- The `@` (root) record cannot be removed.
- The record PDA account is closed and rent is returned to the caller.

##### Errors

| Error                  | Condition                                  |
| ---------------------- | ------------------------------------------ |
| Unauthorized           | Caller is not the NFT holder or controller |
| CannotRemoveRootRecord | Attempted to remove the `@` record         |

---

#### transfer_record

Transfers delegated ownership of a record to a new address. This changes who the `record_owner` is without modifying the record's content.

**Authorization:** NFT holder, controller, or the current record owner.

##### Parameters

| Name      | Type      | Description                                          |
| --------- | --------- | ---------------------------------------------------- |
| new_owner | PublicKey | The address to transfer record ownership to          |

##### Rules

- If the caller is the NFT holder or a controller, they can assign record ownership to any address (even if the record currently has no owner).
- If the caller is the current record owner (not the NFT holder or controller), they can transfer ownership to a different address.
- Cannot transfer to the current record owner (no-op prevention).
- On NFT transfer (detected via `last_reconciled_owner` mismatch), the record's `owner` field is automatically cleared before permission checks.

##### Errors

| Error                    | Condition                                                 |
| ------------------------ | --------------------------------------------------------- |
| Unauthorized             | Caller is not authorized                                  |
| UnauthorizedRecordAccess | Caller is neither owner/controller nor record owner       |
| RecordHasNoOwner         | Non-owner/controller caller tried to transfer unowned record |
| RecordTransferToSelf     | `new_owner` is the same as the current record owner       |

---

#### reconcile

Forces ownership reconciliation. If the NFT has been transferred, this clears all controllers and updates `last_known_owner`. This is useful to call immediately after a marketplace transfer to ensure clean state without waiting for the next write operation.

**Authorization:** Permissionless -- anyone can trigger reconciliation.

##### Parameters

No parameters.

##### Rules

- Reads the current NFT holder from the Metaplex Core asset account.
- If the holder differs from `last_known_owner`, clears all controllers and updates `last_known_owner`.
- If the holder matches, this is a no-op.

---

### AR.IO Network Integration Instructions

The following instructions live on the `ario-arns` and `ario-core` programs, not on the ANT program. They are the primary way an ArNS name owner interacts with the AR.IO Name System at the network level.

#### Registry-Side Authorization

Per ADR-016 / BD-100, `ario-arns` and `ario-core` are MPL-agnostic — they do **not** read the Metaplex Core asset account to verify authorization. Instead, each instruction below requires:

- **`release_name`, `reassign_name`** (`ario-arns`): caller must match `ArnsRecord.owner` (set at name purchase via `buy_name` / `buy_returned_name`).
- **`approve_primary_name`, `remove_primary_name_for_base_name`** (`ario-core`): caller must match `AntRecord.owner` for the relevant undername, read from the program named in the asset's `ANT Program` Attributes-plugin entry (or the canonical `ARIO_ANT_PROGRAM_ID` when absent).

The ANT program (or any pluggable equivalent) is responsible for keeping the on-chain `owner` field in sync with the current NFT holder via the standard record-delegation flow. SDKs SHOULD ensure `owner` reflects the intended signer before invoking these instructions; an NFT transferred via a marketplace alone does **not** automatically convey registry-side authority.

---

#### release_name

Releases a permanently purchased (permabuy) ArNS name back to the AR.IO Network, making it available for re-registration after a return auction period.

**Program:** ario-arns

**Authorization:** `ArnsRecord.owner` (see [Registry-Side Authorization](#registry-side-authorization)).

##### Parameters

No instruction-level parameters. The ArNS record is passed as part of the account context.

##### Rules

- Caller must match `ArnsRecord.owner`.
- Only permabuy names can be released (leased names expire naturally).
- The name must be currently active (not expired).
- Creates a `ReturnedName` entry and removes the name from the on-chain `NameRegistry`.
- The ArNS record account is closed.

##### Errors

| Error             | Condition                                       |
| ----------------- | ----------------------------------------------- |
| NotAntHolder      | Caller does not match `ArnsRecord.owner`        |
| CannotReleaseLease| Name is a lease, not a permabuy                 |
| RecordExpired     | Name has already expired                        |

---

#### reassign_name

Reassigns an ArNS name from its current ANT to a different ANT. The name registration (lease or permabuy) stays intact; only the ANT association changes.

**Program:** ario-arns

**Authorization:** `ArnsRecord.owner` (see [Registry-Side Authorization](#registry-side-authorization)).

##### Parameters

| Name    | Type      | Description                                                         |
| ------- | --------- | ------------------------------------------------------------------- |
| new_ant | PublicKey | The Metaplex Core asset (mint) of the new ANT to assign the name to |

##### Rules

- Caller must match `ArnsRecord.owner`.
- The name must be currently active (not expired or in grace period).
- `new_ant` is stored without on-chain validation. Callers SHOULD verify it is a valid Metaplex Core asset client-side; subsequent ANT-side instructions will reject mismatches.

##### Errors

| Error         | Condition                                       |
| ------------- | ----------------------------------------------- |
| NotAntHolder  | Caller does not match `ArnsRecord.owner`        |
| RecordExpired | Name has expired                                |
| InGracePeriod | Name is in its grace period                     |

---

#### Primary Names

A **primary name** is a wallet's display identity within the AR.IO ecosystem — when wallet `W` sets `ardrive_alice` as its primary, gateways and apps can render `W` as that name. The binding is two-way (forward + reverse lookup) and requires consent from both the requesting wallet and the name owner.

Flow:

1. **Request** — Wallet `W` calls `request_primary_name(name)` on `ario-core`, paying a fee. A `PrimaryNameRequest` PDA is created with a 7-day expiry.
2. **Approve** — The name owner (the `AntRecord.owner` for the relevant undername) calls `approve_primary_name`, consuming the request. This creates `PrimaryName` (forward: wallet → name) and `PrimaryNameReverse` (reverse: name → wallet) PDAs.
3. **Remove** — Either `W` (via `remove_primary_name`) or the name owner (via `remove_primary_name_for_base_name`) can dissolve the binding at any time. Both `PrimaryName` and `PrimaryNameReverse` accounts are closed.

PDA seeds (under `ario-core`):

```
PrimaryNameRequest  seeds = ["primary_name_request", requestor_pubkey]
PrimaryName         seeds = ["primary_name",          owner_pubkey]
PrimaryNameReverse  seeds = ["primary_name_reverse",  sha256(name.toLowerCase())]
```

A `PrimaryNameRequest` older than the configured expiry (default 7 days) is closed permissionlessly by `close_expired_request`. Approving an expired request fails with `PrimaryNameRequestExpired`.

---

#### request_primary_name

Initiates a primary name request. The requestor (a normal user wallet) wishes to bind a name to their address; the name owner must subsequently approve.

**Program:** ario-core

**Authorization:** Any wallet may request (signs the transaction and pays the fee).

##### Parameters

| Name | Type   | Description                                       |
| ---- | ------ | ------------------------------------------------- |
| name | String | The ArNS name being requested (no undername; the binding is for `<wallet> → name`). |

##### Rules

- The ArNS name must exist in the registry and be active (lease not expired).
- The fee is charged in ARIO tokens and depends on the name's purchase type: `PRIMARY_NAME_REQUEST_BASE_FEE_LEASE` (0.2 ARIO at genesis) for a leased name, or `PRIMARY_NAME_REQUEST_BASE_FEE_PERMABUY` (1.0 ARIO at genesis) for a permabuy — each scaled by the current ArNS demand factor.
- Any existing `PrimaryNameRequest` for the requestor is overwritten.
- Creates a `PrimaryNameRequest` with `expires_at = now + 7 days` (default).

---

#### approve_primary_name

Approves a pending primary name request. When a user requests to use an ArNS name as their primary name, the name owner must approve the request.

**Program:** ario-core

**Authorization:** `AntRecord.owner` for the requested undername (see [Registry-Side Authorization](#registry-side-authorization)).

##### Parameters

| Name                | Type     | Description                                         |
| ------------------- | -------- | --------------------------------------------------- |
| reverse_lookup_hash | [u8; 32] | Hash of the name (for reverse lookup PDA derivation) |

##### Rules

- Caller must match the `owner` field of the relevant `AntRecord`, read from the program named in the asset's `ANT Program` Attributes-plugin entry (or `ARIO_ANT_PROGRAM_ID` when absent).
- The primary name request must not be expired.
- The ArNS record (passed via `remaining_accounts`) must exist and be active (lease not expired).
- If the requestor already has a different primary name set, they must remove it first.
- Creates or updates the `PrimaryName` and `PrimaryNameReverse` accounts.

##### Errors

| Error                          | Condition                                              |
| ------------------------------ | ------------------------------------------------------ |
| NotAntHolder                   | Caller does not match `AntRecord.owner`                |
| PrimaryNameRequestExpired      | The request has expired                                |
| MustRemoveExistingPrimaryName  | Requestor already has a different primary name          |
| PrimaryNameAlreadySet          | Another user already has this name as their primary     |

---

#### remove_primary_name

Removes the caller's own primary-name binding. Self-service equivalent of `remove_primary_name_for_base_name`.

**Program:** ario-core

**Authorization:** The wallet that owns the `PrimaryName` binding (i.e., the wallet whose name is being unset).

##### Parameters

No parameters. The binding to remove is identified by the caller's pubkey via PDA derivation.

##### Rules

- Closes the caller's `PrimaryName` and the corresponding `PrimaryNameReverse` PDA.

---

#### remove_primary_name_for_base_name

Removes a primary name association for a base ArNS name. This allows the name owner to revoke a user's primary name that was set using their ArNS name.

**Program:** ario-core

**Authorization:** `AntRecord.owner` for the `@` undername (see [Registry-Side Authorization](#registry-side-authorization)).

##### Parameters

| Name                | Type     | Description                                         |
| ------------------- | -------- | --------------------------------------------------- |
| reverse_lookup_hash | [u8; 32] | Hash of the name (for reverse lookup PDA derivation) |

##### Rules

- Caller must match the `owner` field of the base name's `AntRecord` (`@` undername), read from the asset's declared ANT program (or `ARIO_ANT_PROGRAM_ID` fallback).
- The ArNS record must be active.
- Closes the `PrimaryName` and `PrimaryNameReverse` accounts.

### Record Owner Delegation Model

The `set_record` instruction supports an optional `record_owner` field that enables delegated control over individual undername records. This is the "record owner" tier of the three-tier permission model:

1. **NFT holder** -- full control over all records, controllers, and metadata.
2. **Controllers** -- can manage all records (create, update, remove) and are set by the NFT holder or other controllers.
3. **Record owners** -- can update content fields (`target`, `ttl_seconds`, metadata) on their assigned record only. Cannot change `priority`, cannot assign or transfer `record_owner`, and cannot create or remove records.

**Stale owner cleanup:** When an NFT is transferred, record-level owners become stale (they were granted by the previous NFT holder). The program detects this by comparing the record's `last_reconciled_owner` against `AntConfig.last_known_owner`. On any write to a record where these differ, the record's `owner` is cleared and `last_reconciled_owner` is updated. This prevents inherited record-level permissions from surviving NFT transfers.

### Controller Auto-Clear on NFT Transfer

When the NFT changes hands (via marketplace sale, direct transfer, or any Metaplex Core transfer mechanism), the controller list is automatically cleared on the next write operation that touches the `AntConfig` and `AntControllers` accounts. This is implemented via lazy reconciliation:

1. Every write instruction reads the current NFT holder from the Metaplex Core asset account.
2. If the holder differs from `AntConfig.last_known_owner`, the `AntControllers.controllers` vector is cleared and `last_known_owner` is updated.
3. The new NFT holder can then add their own controllers.

This design ensures that transferring an ANT NFT is a clean handoff -- the new holder starts with no controllers and no stale record-level owners, while the NFT holder always retains full access by virtue of holding the NFT.
