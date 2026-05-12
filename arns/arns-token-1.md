# ARNS-TOKEN-1

## Status:

**Draft**

## Version:

| Version | Description                                               | Date       |
| ------- | --------------------------------------------------------- | ---------- |
| 1.0.0   | Initial version of the **ARNS-TOKEN-1** specification.    | 2024-09-01 |
| 1.0.1   | Fixed Credit/Debit response notices for Transfer handler. | 2024-09-24 |
| 1.1.0   | Added 'description' and 'keywords' metadata.              | 2024-10-14 |
| 1.2.0   | Added Set-Logo to action map, fixed Set-Keywords examples, and documented ANT Registry callback. | 2025-07-29 |
| 1.3.0   | Added record ownership and metadata fields for undernames. | 2025-08-01 |
| 2.0.0   | Rewritten for Solana: ANTs are Metaplex Core NFTs with on-chain PDA state. Removed Balances, Denomination, TotalSupply, Mint, Burn, Info handler, and AO messaging. Added lazy controller reconciliation, explicit reconcile instruction, and schema-versioned migration. | 2026-04-20 |
| 2.0.1   | Audit corrections: description max 256, keywords max 8, `AntRecordMetadata` PDA documented as a separate account. | 2026-05-11 |

## Abstract

The **ARNS-TOKEN-1** specification defines the framework for creating and managing AR.IO Name Tokens (ANTs) as non-fungible assets on Solana. Each ANT is a Metaplex Core NFT whose ownership is determined by the NFT holder. Extended state -- metadata, controllers, and undername records -- is stored in Program Derived Address (PDA) accounts managed by the `ario-ant` program. This specification builds on the **ARNS-CORE-1** (records) and **ARNS-MANAGE-1** (controllers) specifications to provide a complete, self-contained token model for ArNS names.

## Motivation

The **ARNS-TOKEN-1** specification provides a standardized approach for representing AR.IO Name Tokens as Metaplex Core NFTs on Solana, enabling them to be transferred, tracked, and managed as single, indivisible assets within the Solana ecosystem.

By implementing this specification, developers can extend the functionality of AR.IO Names to include ownership transfer via any Solana wallet or marketplace, metadata management, and integration with block explorers and NFT tooling. The Metaplex Core standard ensures broad compatibility with the Solana ecosystem while the `ario-ant` program provides the ArNS-specific state (records, controllers, metadata) that makes each ANT functional as a name token.

Unlike the previous AO-based model -- which embedded a single-supply fungible token inside an AO Process -- the Solana model leverages the native NFT primitive. Ownership is determined by who holds the Metaplex Core asset. There is no Balances table, no Denomination, and no TotalSupply. Transfer is a standard Metaplex Core NFT transfer, compatible with any Solana marketplace or wallet.

## Specification

### Overview

The **ARNS-TOKEN-1** Specification includes the following requirements:

**Metadata (stored in an `AntConfig` PDA account, seeds: `["ant_config", mint]`):**

- Must have a `name` field which is a friendly name for this ANT.
  - Must be a non-empty string, e.g., `"ArDrive"`.
  - Must not exceed 61 characters.
- Must have a `ticker` field which is a short token symbol, shown in block explorers and marketplaces.
  - Must be a string, e.g., `"ANT-ARDRIVE"`.
  - Must not exceed 16 characters.
  - Defaults to `"ANT"` if not specified at initialization.
- Must have a `logo` field which is an image icon used by downstream applications.
  - Must be a valid Arweave transaction ID (43 base64url characters), e.g., `"Sie_26dvgyok0PZD_-iQAFOhOd5YxDTkczOLoqTTL_A"`.
  - Logos reference Arweave transactions exclusively, preserving the permanence guarantee.
  - Defaults to the AR.IO logo (`"AnYvLJTWcG9lr2Ll5MwYWZR2o5uTE39WbpYB0zCxwKM"`) if not specified at initialization.
- Must have a `description` field that is a brief description of this ANT and its purpose or use.
  - Must be a string, e.g., `"ArDrive is a permaweb app that lets you upload, download and share your files easily."`.
  - Must not exceed 256 characters.
- Must have a `keywords` field used to further describe this ANT.
  - Must be an array of strings.
  - Must not contain more than 8 keywords.
  - Each keyword must not exceed 32 characters.
  - Each keyword must consist of alphanumeric characters, dashes (`-`), underscores (`_`), `@`, or `#`.
  - Each keyword must not include spaces.
  - Each keyword must be unique within the array.
- Must have a `last_known_owner` field that tracks the most recently observed NFT holder address.
  - Used for lazy controller reconciliation (see Transfer section).
- Must have a `version` field (unsigned 8-bit integer) indicating the schema version of this ANT's on-chain data.
  - Used for per-ANT schema migrations via the `migrate_ant` instruction.

**Ownership:**

- Ownership is determined by the Metaplex Core NFT holder. The `ario-ant` program reads the owner directly from the Metaplex Core asset account data (AssetV1 layout: byte 0 is the Key discriminator, bytes 1-32 are the owner public key).
- There is no Balances table, Denomination, or TotalSupply. These concepts from the AO token model do not apply to Metaplex Core NFTs.
- There is no Mint or Burn capability within the `ario-ant` program. ANT minting is handled by the Metaplex Core program during asset creation; the `ario-ant` program's `initialize` instruction creates the associated PDA state after minting.

**Transfer:**

- Transfer is a standard Metaplex Core NFT transfer, executed via the Metaplex Core program (not a custom `ario-ant` instruction). This makes ANTs compatible with any Solana wallet or marketplace that supports Metaplex Core assets.
- Upon the first `ario-ant` instruction executed after a transfer, the program detects that the NFT holder has changed (by comparing the on-chain NFT owner to `last_known_owner` in the `AntConfig` PDA). This triggers lazy reconciliation:
  - All controllers are cleared from the `AntControllers` PDA.
  - The `last_known_owner` field in `AntConfig` is updated to the new NFT holder.
  - Record-level owners (`owner` field on `AntRecord` accounts) are cleared when their `last_reconciled_owner` differs from the updated `last_known_owner`.
- A dedicated `reconcile` instruction is also available to explicitly trigger this reconciliation without performing any other operation. It is permissionless: anyone can call it for any ANT.
- There are no Credit-Notice or Debit-Notice messages. Solana does not use AO-style inter-process messaging.

**Metadata management instructions:**

- Must have a `set_name` instruction to update the ANT's name.
  - Authorized for the NFT holder or an authorized controller.
- Must have a `set_ticker` instruction to update the ANT's ticker.
  - Authorized for the NFT holder or an authorized controller.
- Must have a `set_logo` instruction to update the ANT's logo.
  - Must be a valid Arweave transaction ID (43 base64url characters).
  - Authorized for the NFT holder or an authorized controller.
- Must have a `set_description` instruction to update the ANT's description.
  - Authorized for the NFT holder or an authorized controller.
- Must have a `set_keywords` instruction to update the ANT's keywords.
  - Authorized for the NFT holder or an authorized controller.

**State reads:**

- There is no `Info` handler or `State` handler. All ANT state is read directly from Solana accounts via RPC:
  - `AntConfig` PDA -- metadata (name, ticker, logo, description, keywords, last_known_owner, version).
  - `AntControllers` PDA -- controller list.
  - `AntRecord` PDAs -- individual undername records (resolution-critical fields).
  - `AntRecordMetadata` PDAs -- optional per-record metadata (lazy; absent when unset).
  - Metaplex Core asset account -- current NFT holder (owner).
- The complete state of an ANT is the union of these accounts.

**ANT enumeration:**

- The previous AO-based ANT Registry is replaced by on-chain account enumeration. Clients use Solana's `getProgramAccounts` RPC method with `memcmp` filters on the `ario-ant` program to discover all `AntConfig`, `AntControllers`, `AntRecord`, and `AntRecordMetadata` accounts.

**Schema migration:**

- Must have a `migrate_ant` instruction that upgrades an ANT's on-chain data layout to the latest schema version.
  - Permissionless: anyone can pay to migrate any ANT. The instruction only upgrades the data layout and never changes user data.
  - Uses Solana's `realloc` to handle account size changes between schema versions.

### Objects

The **ARNS-TOKEN-1** specification stores state across multiple PDA accounts derived from the Metaplex Core asset's mint address. It includes all of the objects in **ARNS-MANAGE-1** (controllers) and **ARNS-CORE-1** (records).

#### AntConfig

Per-ANT configuration and metadata. One account per ANT.

PDA seeds: `["ant_config", <mint>]`

| Field              | Type           | Description                                                |
| ------------------ | -------------- | ---------------------------------------------------------- |
| `mint`             | PublicKey      | The Metaplex Core asset (NFT mint) this config belongs to. |
| `name`             | String         | ANT display name (max 61 characters).                      |
| `ticker`           | String         | Ticker symbol (max 16 characters), e.g., `"ANT-ARDRIVE"`. |
| `logo`             | String         | Arweave transaction ID (43 base64url characters).          |
| `description`      | String         | Brief description (max 256 characters).                    |
| `keywords`         | Vec\<String\>  | Up to 8 keywords, each max 32 characters.                  |
| `last_known_owner` | PublicKey      | Last observed NFT holder, for lazy reconciliation.         |
| `bump`             | u8             | PDA bump seed.                                             |
| `version`          | u8             | Schema version for per-ANT data migrations.                |

#### AntControllers

Controller list for an ANT. One account per ANT.

PDA seeds: `["ant_controllers", <mint>]`

| Field         | Type             | Description                                                  |
| ------------- | ---------------- | ------------------------------------------------------------ |
| `mint`        | PublicKey        | The Metaplex Core asset this controller list belongs to.     |
| `controllers` | Vec\<PublicKey\> | Controller addresses (max 10).                               |
| `bump`        | u8               | PDA bump seed.                                               |

#### AntRecord

A single undername record for an ANT. One account per undername per ANT. Contains resolution-critical fields only; optional descriptive metadata lives in a separate `AntRecordMetadata` PDA (below).

PDA seeds: `["ant_record", <mint>, <hash(undername.lowercase())>]`

| Field                  | Type                    | Description                                                          |
| ---------------------- | ----------------------- | -------------------------------------------------------------------- |
| `mint`                 | PublicKey               | The Metaplex Core asset this record belongs to.                      |
| `undername`            | String                  | The undername (e.g., `"@"`, `"blog"`, `"docs"`), stored lowercase.   |
| `target`               | String                  | Content address -- Arweave TX ID (43 chars), IPFS CID, or future protocol (max 128 chars). |
| `target_protocol`      | u8                      | Storage protocol: `0` = Arweave, `1` = IPFS, `2+` = reserved.       |
| `ttl_seconds`          | u32                     | TTL in seconds (60--86400).                                          |
| `priority`             | Option\<u32\>           | Ordering priority. `Some(0)` required for `@`; `None` = unset.      |
| `owner`                | Option\<PublicKey\>     | Optional record-level owner (delegated control).                     |
| `last_reconciled_owner`| PublicKey               | ANT owner at last record modification; stale records are cleared.    |
| `bump`                 | u8                      | PDA bump seed.                                                       |
| `version`              | u8                      | Schema version for per-record data migrations.                       |

#### AntRecordMetadata

Optional descriptive metadata for a record. Created lazily when an optional field is first written. Resolvers MAY skip reading this account when only resolution data is needed.

PDA seeds: `["ant_record_meta", <mint>, <hash(undername.lowercase())>]`

| Field                | Type                    | Description                                                       |
| -------------------- | ----------------------- | ----------------------------------------------------------------- |
| `mint`               | PublicKey               | The Metaplex Core asset this metadata belongs to.                 |
| `display_name`       | Option\<String\>        | Optional display name (max 61 characters).                        |
| `record_logo`        | Option\<String\>        | Optional logo (Arweave TX ID, 43 characters).                     |
| `record_description` | Option\<String\>        | Optional description (max 256 characters).                        |
| `record_keywords`    | Option\<Vec\<String\>\> | Optional keywords (same validation rules as ANT-level keywords).  |
| `bump`               | u8                      | PDA bump seed.                                                    |
| `version`            | u8                      | Schema version for per-record-metadata migrations.                |

### Instructions

#### Instruction Map

The following instructions are provided by the `ario-ant` program. This list includes all instructions from the **ARNS-CORE-1** and **ARNS-MANAGE-1** specifications.

**From ARNS-CORE-1 (record management):**

| Instruction       | Description                                       |
| ----------------- | ------------------------------------------------- |
| `set_record`      | Create or update an undername record.              |
| `remove_record`   | Remove an undername record (cannot remove `@`).    |
| `transfer_record` | Transfer record-level ownership to another address.|

**From ARNS-MANAGE-1 (controller management):**

| Instruction          | Description                        |
| -------------------- | ---------------------------------- |
| `add_controller`     | Add a controller address.          |
| `remove_controller`  | Remove a controller address.       |

**ARNS-TOKEN-1 (metadata management):**

| Instruction        | Description                        |
| ------------------ | ---------------------------------- |
| `set_name`         | Update ANT display name.           |
| `set_ticker`       | Update ANT ticker symbol.          |
| `set_description`  | Update ANT description.            |
| `set_keywords`     | Update ANT keywords.               |
| `set_logo`         | Update ANT logo.                   |

**Lifecycle:**

| Instruction           | Description                                                    |
| --------------------- | -------------------------------------------------------------- |
| `initialize`          | Initialize ANT state (config, controllers, `@` record).       |
| `reconcile`           | Explicitly trigger lazy ownership reconciliation.              |
| `migrate_ant`         | Upgrade ANT schema to latest version (permissionless).         |

#### initialize

Initializes an ANT's on-chain PDA state after a Metaplex Core asset has been minted. Creates the `AntConfig`, `AntControllers`, and root `@` record accounts.

Executable only by the current NFT holder.

##### Parameters

| Name             | Type           | Required | Description                                              |
| ---------------- | -------------- | -------- | -------------------------------------------------------- |
| `name`           | String         | Yes      | ANT display name (non-empty, max 61 characters).         |
| `ticker`         | Option\<String\> | No    | Ticker symbol (max 16 characters). Defaults to `"ANT"`.  |
| `target`           | String         | Yes      | Content target for the `@` record (Arweave TX ID, IPFS CID, etc., max 128 chars). |
| `target_protocol`  | Option\<u8\>  | No       | Storage protocol (`0` = Arweave, `1` = IPFS). Defaults to `0` (Arweave). |
| `logo`           | String         | No       | Arweave TX ID for logo. Empty string uses default logo.  |
| `description`    | String         | No       | Description (max 256 characters). Can be empty.          |
| `keywords`       | Vec\<String\>  | No       | Keywords (max 8, each max 32 chars). Can be empty.       |

##### Rules

- Caller must be the current NFT holder (verified by reading the Metaplex Core asset's owner bytes).
- The Metaplex Core asset account must be owned by the Metaplex Core program (`CoREENxT6tW1HoK8ypY1SxRMZTcVPm7R94rH4PZNhX7d`).
- All PDA accounts (`AntConfig`, `AntControllers`, root `AntRecord`) must not already exist (enforced by Anchor's `init` constraint).
- The NFT holder is added as the initial controller.
- The root `@` record is created with `priority = Some(0)` and `ttl_seconds = 900` (default).

##### Errors

| Error Code       | Condition                                                    |
| ---------------- | ------------------------------------------------------------ |
| `NotNftHolder`   | Caller is not the NFT holder.                                |
| `InvalidAsset`   | Asset account is not a valid Metaplex Core AssetV1.          |
| `NameEmpty`      | Name is an empty string.                                     |
| `NameTooLong`    | Name exceeds 61 characters.                                  |
| `TickerTooLong`  | Ticker exceeds 16 characters.                                |
| `InvalidTarget`  | Content target is not valid for the declared protocol.       |
| `InvalidLogo`    | Logo is not a valid Arweave transaction ID.                  |
| `DescriptionTooLong` | Description exceeds 256 characters.                      |
| `InvalidKeyword` | Keywords fail validation (count, length, format, uniqueness).|

#### set_name

Updates the `name` field on the `AntConfig` PDA.

Executable by the NFT holder or an authorized controller.

##### Parameters

| Name   | Type   | Description                              |
| ------ | ------ | ---------------------------------------- |
| `name` | String | The new name for the ANT (max 61 chars). |

##### Rules

- Caller must be the NFT holder or an authorized controller.
- Name must be a non-empty string.
- Name must not exceed 61 characters.
- Lazy reconciliation is performed before the permission check.

##### Errors

| Error Code     | Condition                                          |
| -------------- | -------------------------------------------------- |
| `Unauthorized` | Caller is not the NFT holder or a controller.      |
| `NameEmpty`    | Name is an empty string.                           |
| `NameTooLong`  | Name exceeds 61 characters.                        |

#### set_ticker

Updates the `ticker` field on the `AntConfig` PDA.

Executable by the NFT holder or an authorized controller.

##### Parameters

| Name     | Type   | Description                                  |
| -------- | ------ | -------------------------------------------- |
| `ticker` | String | The new ticker for the ANT (max 16 chars).   |

##### Rules

- Caller must be the NFT holder or an authorized controller.
- Ticker must be a non-empty string.
- Ticker must not exceed 16 characters.
- Lazy reconciliation is performed before the permission check.

##### Errors

| Error Code     | Condition                                          |
| -------------- | -------------------------------------------------- |
| `Unauthorized` | Caller is not the NFT holder or a controller.      |
| `TickerEmpty`  | Ticker is an empty string.                         |
| `TickerTooLong`| Ticker exceeds 16 characters.                      |

#### set_description

Updates the `description` field on the `AntConfig` PDA.

Executable by the NFT holder or an authorized controller.

##### Parameters

| Name          | Type   | Description                                        |
| ------------- | ------ | -------------------------------------------------- |
| `description` | String | The new description for the ANT (max 256 chars).   |

##### Rules

- Caller must be the NFT holder or an authorized controller.
- Description must not exceed 256 characters. An empty string is permitted (clears the description).
- Lazy reconciliation is performed before the permission check.

##### Errors

| Error Code          | Condition                                          |
| ------------------- | -------------------------------------------------- |
| `Unauthorized`      | Caller is not the NFT holder or a controller.      |
| `DescriptionTooLong`| Description exceeds 256 characters.                |

#### set_keywords

Updates the `keywords` field on the `AntConfig` PDA.

Executable by the NFT holder or an authorized controller.

##### Parameters

| Name       | Type          | Description                             |
| ---------- | ------------- | --------------------------------------- |
| `keywords` | Vec\<String\> | The new keywords for the ANT.           |

##### Rules

- Caller must be the NFT holder or an authorized controller.
- Must not contain more than 8 keywords.
- Each keyword must not exceed 32 characters.
- Each keyword must consist of alphanumeric characters, dashes (`-`), underscores (`_`), `@`, or `#`.
- Each keyword must not include spaces.
- Each keyword must be unique within the array.
- An empty array is permitted (clears all keywords).
- Lazy reconciliation is performed before the permission check.

##### Errors

| Error Code       | Condition                                                |
| ---------------- | -------------------------------------------------------- |
| `Unauthorized`   | Caller is not the NFT holder or a controller.            |
| `InvalidKeyword` | Keywords fail validation (count, length, format, or uniqueness). |

#### set_logo

Updates the `logo` field on the `AntConfig` PDA.

Executable by the NFT holder or an authorized controller.

##### Parameters

| Name   | Type   | Description                                              |
| ------ | ------ | -------------------------------------------------------- |
| `logo` | String | Arweave transaction ID (43 base64url characters).        |

##### Rules

- Caller must be the NFT holder or an authorized controller.
- Logo must be a valid Arweave transaction ID: exactly 43 characters, each character alphanumeric or `-` or `_` (base64url encoding).
- Lazy reconciliation is performed before the permission check.

##### Errors

| Error Code     | Condition                                          |
| -------------- | -------------------------------------------------- |
| `Unauthorized` | Caller is not the NFT holder or a controller.      |
| `InvalidLogo`  | Logo is not a valid 43-character Arweave TX ID.    |

#### Transfer (Metaplex Core)

ANT ownership transfer is a standard Metaplex Core NFT transfer. It is **not** an `ario-ant` program instruction. Any Solana wallet, marketplace, or application that supports Metaplex Core transfers can transfer an ANT.

##### Behavior

1. The NFT is transferred using the Metaplex Core program's transfer instruction.
2. The `ario-ant` program is not invoked during the transfer itself.
3. On the next `ario-ant` instruction for this ANT (any instruction that reads the asset account), the program detects the ownership change by comparing the NFT holder to `AntConfig.last_known_owner`.
4. Lazy reconciliation occurs automatically:
   - `AntControllers.controllers` is cleared (empty array).
   - `AntConfig.last_known_owner` is updated to the new NFT holder.
5. On the next interaction with each `AntRecord`, if `record.last_reconciled_owner` differs from `config.last_known_owner`, the record's `owner` field is cleared and `last_reconciled_owner` is updated.

##### Explicit Reconciliation

The `reconcile` instruction can be called to explicitly trigger ownership reconciliation without performing any other operation. This is useful after a marketplace transfer to ensure the ANT's state is clean before the new owner interacts with it.

- Permissionless: anyone can call `reconcile` for any ANT.
- Idempotent: calling `reconcile` when no ownership change has occurred is a no-op.

#### reconcile

Explicitly triggers lazy ownership reconciliation. Clears controllers if NFT ownership has changed since the last recorded owner. Does not modify records (record-level reconciliation happens per-record on next interaction).

Permissionless: anyone can call this for any ANT.

##### Parameters

No parameters.

##### Rules

- If `AntConfig.last_known_owner` differs from the current NFT holder, controllers are cleared and `last_known_owner` is updated.
- If no ownership change is detected, the instruction is a no-op.

#### migrate_ant

Upgrades an ANT's on-chain data layout to the latest schema version. Uses Solana's `realloc` to handle account size changes between schema versions.

Permissionless: anyone can pay the rent differential to migrate any ANT. The instruction only upgrades the data layout and never changes user data.

##### Parameters

No parameters.

##### Rules

- The ANT's current `version` must be less than the program's `ANT_CONFIG_VERSION` constant.
- The payer covers any rent increase from account reallocation.

##### Errors

| Error Code            | Condition                                       |
| --------------------- | ----------------------------------------------- |
| `AlreadyLatestVersion`| ANT is already at the latest schema version.    |

#### State (Account Reads)

There is no dedicated `State` or `Info` instruction. The full state of an ANT is read directly from Solana accounts via standard RPC calls:

| Account            | PDA Seeds                                                | Contents                                                       |
| ------------------ | -------------------------------------------------------- | -------------------------------------------------------------- |
| `AntConfig`         | `["ant_config", mint]`                                   | name, ticker, logo, description, keywords, last_known_owner, version |
| `AntControllers`    | `["ant_controllers", mint]`                              | controller addresses                                           |
| `AntRecord`         | `["ant_record", mint, hash(undername.lowercase())]`      | undername, target, target_protocol, ttl_seconds, priority, owner, last_reconciled_owner |
| `AntRecordMetadata` | `["ant_record_meta", mint, hash(undername.lowercase())]` | display_name, record_logo, record_description, record_keywords (lazy; absent when no optional fields set) |
| Metaplex Core Asset | (Asset mint address)                                     | Current NFT holder (owner)                                     |

To enumerate all records for a given ANT, use `getProgramAccounts` with a `memcmp` filter matching the `mint` field in `AntRecord` accounts.

To enumerate all ANTs, use `getProgramAccounts` with the `ario-ant` program ID and a `memcmp` filter on the account discriminator for `AntConfig`.
