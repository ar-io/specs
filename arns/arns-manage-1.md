# ARNS-MANAGE-1

## Status:

**In-Review**

## Version:

| Version | Description                                             | Date       |
| ------- | ------------------------------------------------------- | ---------- |
| 1.0.0   | Initial version of the **ARNS-MANAGE-1** specification. | 2024-09-01 |
| 1.1.0   | Added AR.IO Network handlers, Priority parameter for Set-Record, Initialized field, and boot handler documentation. | 2025-07-29 |
| 1.2.0   | Added undername ownership with Transfer-Record handler, record metadata fields, and delegated permission model for record owners. | 2025-08-01 |

## Abstract

The **ARNS-MANAGE-1** specification introduces additional management and control utilities for Arweave Names. It provides the necessary handlers for adding, modifying, and removing records, as well as managing controllers who can add, set or remove these records. Additionally, it supports delegated ownership of individual undernames, enabling record-level control while maintaining ANT owner authority.

## Motivation

The **ARNS-MANAGE-1** specification builds on the foundational **ARNS-CORE-1** by adding essential management capabilities. As Arweave Names require ongoing updates and adjustments, this specification ensures that process owners have the tools needed to manage records and delegate control to other users securely. This includes the ability to assign ownership of individual undernames to specific users while maintaining the ANT owner's ultimate authority. By standardizing these management operations, **ARNS-MANAGE-1** enables efficient and consistent name management across the Arweave ecosystem.

#### Language of Implementation

All examples and code snippets in this specification are written in Lua. This choice ensures compatibility with the AO Processes and the Arweave ecosystem.

However, developers are not restricted to using Lua exclusively when building new features or extending functionalities around ArNS. While Lua is recommended for direct integration with existing infrastructure, other programming languages can be used, provided they adhere to the protocols and specifications outlined in this document.

## Specification

### Overview

The **ARNS-MANAGE-1** Specification includes the following requirements:

- Must include a list of `Controllers` who act as secondary users who control the `Records` set in this ANT.
- Must have an `Add-Controller` handler to add new `Controllers`.
  - Authorized for the Process `Owner` only.
- Must have a `Remove-Controller` handler to remove `Controllers`.
  - Authorized for the Process `Owner` only.
- Must have a `Set-Record` handler to set new `Records` and modify existing ones.
  - Authorized for the Process `Owner`, `Controllers`, and individual record owners (for their own records).
  - Supports optional `Priority` parameter for undername sorting.
  - Supports optional record ownership and metadata fields: `Record-Owner`, `Display-Name`, `Logo`, `Description`, and `Keywords`.
  - Priority and explicit owner assignment require ANT owner or controller authorization.
- Must have a `Transfer-Record` handler to transfer ownership of a specific undername.
  - Authorized for the Process `Owner`, `Controllers`, and the current record owner.
- Must have a `Remove-Record` handler to remove `Records`.
  - Authorized for the Process `Owner` and `Controllers`.
- Must have a handler to read all `Controllers`.
- Handler to read entire ANT `State` is updated to also return `Controllers`.
- Must have AR.IO Network integration handlers:
  - Authorized for the Process `Owner` only.
    - `Release-Name` to release an ArNS name back to the registry.
    - `Reassign-Name` to transfer an ArNS name to another ANT.
  - Authorized for the Process `Owner` or the undername owner if the undername has one.
    - `Approve-Primary-Name` to approve primary name requests.
    - `Remove-Primary-Names` to remove primary name associations.
  

This adds flexibility for Arweave Name Process Owners to manage their records, leverage controllers for access controls, delegate undername ownership, and interact with the AR.IO Network for name management.

### Objects

The **ARNS-MANAGE-1** specification includes all of the objects contained in **ARNS-CORE-1**.

#### ARNS-MANAGE-1 Objects

```
-- ARNS-MANAGE-1 Objects
Owner = Owner or ao.env.Process.Owner
Controllers = Controllers or { Owner }
Initialized = Initialized or false

-- ARNS-CORE-1 Objects
Records = Records or {
  ["@"] = {
    transactionId = "UyC5P5qKPZaltMmmZAWdakhlDXsBF6qmyrbWYFchRTk",
    ttlSeconds = 3600,
    priority = 0,
    -- Optional ownership and metadata fields
    owner = nil,         -- Record owner address (optional)
    displayName = nil,   -- Display name (max 61 chars, optional)
    logo = nil,          -- Arweave TX ID for logo (optional)
    description = nil,   -- Description (max 512 chars, optional)
    keywords = nil       -- Array of keywords; up to 16, each max 32 chars (optional)
  }
}
```

### Handlers

#### Action Map

The following actions are handled in the **ARNS-MANAGE-1** specification and include the actions contained in **ARNS-CORE-1**.

```
ARNSCoreSpecActionMap = {
  -- read actions
  Record = "Record",
  Records = "Records",
  State = "State",
}

ARNSManageSpecActionMap = {
  -- read actions
  Controllers = "Controllers",
  -- write actions
  AddController = "Add-Controller",
  RemoveController = "Remove-Controller",
  SetRecord = "Set-Record",
  TransferRecord = "Transfer-Record"
  RemoveRecord = "Remove-Record",
  -- AR.IO Network integration
  ReleaseName = "Release-Name",
  ReassignName = "Reassign-Name",
  ApprovePrimaryName = "Approve-Primary-Name",
  RemovePrimaryNames = "Remove-Primary-Names",
}
```

#### Controllers

Gets the entire `Controllers` table, including each wallet that has controller permission on this process.

Executable by anonymous users.

##### Parameters

No parameters needed.

##### Rules

- Must return entire `Controllers` object as JSON in the data field of the response notice.
- Must add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Controllers"
})
```

##### Responses

**Valid `Controllers`**

```
{
  Target = msg.From,
  Action = "Controllers-Notice",
  Data = json.encode(Controllers),
  ... other forwarded tag name and value pairs
}
```

#### Add-Controller

Adds a new controller into the `Controllers` table, giving them access to modify the Arweave Name's `Records`.

Executable by the process `Owner` or an authorized user in the `Controllers` table.

##### Parameters

| Name       | Type   | Description                                                                                            |
| ---------- | ------ | ------------------------------------------------------------------------------------------------------ |
| Controller | string | The controller being added to the Controllers table, e.g., iKryOeZQMONi2965nKz528htMMN_sBcjlhc-VncoRjA |

##### Rules

- Must be an authorized process `Owner` or `Controller`.
- Must specify a valid `Controller` parameter (string) as a message tag.
- The `Controller` must not already exist in the `Controllers` table.
- Must add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Add-Controller",
  Controller = "{Wallet Address}"
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Add-Controller-Notice",
  Error = "Add-Controller-Error",
  ["Message-Id"] = msg.Id,
  Data = permissionErr
}
```

**Invalid `Controller`**

```
{
  Target = msg.From,
  Action = "Invalid-Add-Controller-Notice",
  Error = "Add-Controller-Error",
  ["Message-Id"] = msg.Id,
  Data = controllerRes
}
```

**Valid `Controller`**

```
{
  Target = msg.From,
  Action = "Add-Controller-Notice",
  Data = json.encode(Controllers),
  ... other forwarded tag name and value pairs
}
```

#### Remove-Controller

Removes an existing controller in the `Controllers` table, removing their access to modify the Arweave Name's `Records.

Executable by the process `Owner` or an authorized user in the `Controllers` table.

##### Parameters

| Name       | Type   | Description                                                                                                           |
| ---------- | ------ | --------------------------------------------------------------------------------------------------------------------- |
| Controller | string | The controller wallet address to remove from the Controllers table, e.g., iKryOeZQMONi2965nKz528htMMN_sBcjlhc-VncoRjA |

##### Rules

- Must be an authorized process `Owner` or `Controller`.
- Must specify a valid `Controller` parameter (string) as a message tag.
- The `Controller` must already exist in the `Controllers` table.
- Must add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Remove-Controller",
  Controller = "{Wallet Address}"
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Remove-Controller-Notice",
  Error = "Remove-Controller-Error",
  ["Message-Id"] = msg.Id,
  Data = permissionErr,
}
```

**Invalid `Controller`**

```
{
  Target = msg.From,
  Action = "Invalid-Remove-Controller-Notice",
  Error = "Remove-Controller-Error",
  ["Message-Id"] = msg.Id,
  Data = removeRes,
}
```

**Valid `Controller`**

```
{
  Target = msg.From,
  Action = "Remove-Controller-Notice",
  Data = json.encode(Controllers),
  ... other forwarded tag name and value pairs
}
```

#### Set-Record

Updates an existing undername's Sub-Domain in the Records table, including modifying the transaction id, time to live, ownership, and optional metadata.

Executable by the process Owner, an authorized user in the Controllers table, or the record owner (for their own record).

##### Parameters

| Name                   | Type   | Description                                                                                 |
| ---------------------- | ------ | ------------------------------------------------------------------------------------------- |
| Sub-Domain             | string | The undername record to update, e.g., `@`, `ardrive`, or `dapp_ardrive`                     |
| Transaction-Id         | string | The Arweave transaction ID that this undername points to.                                   |
| TTL-Seconds            | string | The time to live for this record, indicating how long an ArNS Resolver should cache it for. |
| Priority               | string | Optional. The priority for ArNS resolution of undernames. Must be integer > 0 (or 0 for `@`). Only settable by ANT owner/controllers. |
| Record-Owner           | string | Optional. The address to assign as the owner of this record. Only settable by ANT owner/controllers. |
| Display-Name           | string | Optional. A display name for this undername (max 61 characters). |
| Logo                   | string | Optional. An Arweave transaction ID for the logo image. |
| Description            | string | Optional. A description of this undername (max 512 characters). |
| Keywords               | string | Optional. A JSON-encoded array of keywords (max 16 keywords, each max 32 characters). |
| Allow-Unsafe-Addresses | string | Optional. If set to `true`, allows setting addresses that may not be valid AO addresses.    |

##### Rules

- Must be an authorized process `Owner`, `Controller`, or the record owner (for existing records they own).
- New records can only be created by the ANT owner or controllers.
- Must specify a valid `Sub-Domain` parameter (string) as a message tag.
- Must specify a valid `Transaction-Id` parameter (string) as a message tag.
- Must specify a valid `TTL-Seconds` parameter (string) as a message tag, which is an integer between 60 and 86400.
- `Priority` and `Record-Owner` assignment require ANT owner or controller authorization.
- `Priority` must be an integer greater than 0 for undernames, or exactly 0 for the `@` record.
- `Display-Name` must not exceed 61 characters.
- `Description` must not exceed 512 characters.
- `Keywords` must be a valid JSON array with up to 16 keywords, each consisting of alphanumeric characters, dashes, underscores, @ or #, and not exceeding 32 characters.
- Must add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Set-Record",
  ["Sub-Domain"] = "foo",
  ["Transaction-Id"] = "{Base64 URL}",
  ["TTL-Seconds"] = "900",
  Priority = "10", -- optional, ANT owner/controllers only
  ["Record-Owner"] = "{Wallet Address}", -- optional, ANT owner/controllers only
  ["Display-Name"] = "Foo's Site", -- optional
  Logo = "Sie_26dvgyok0PZD_-iQAFOhOd5YxDTkczOLoqTTL_A", -- optional
  Description = "This is Foo's personal site", -- optional
  Keywords = '["personal","blog"]' -- optional, JSON array
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Set-Record-Notice",
  Data = permissionErr,
  Error = "Set-Record-Error",
  ["Message-Id"] = msg.Id
}
```

**Invalid parameters**

```
{
  Target = msg.From,
  Action = "Invalid-Set-Record-Notice",
  Data = setRecordResult,
  Error = "Set-Record-Error",
  ["Message-Id"] = msg.Id,
}
```

**Valid parameters**

```
{
  Target = msg.From,
  Action = "Set-Record-Notice",
  Data = json.encode({
    transactionId = transactionId,
    ttlSeconds = ttlSeconds,
    priority = priority,
    owner = owner, -- if set
    displayName = displayName, -- if set
    logo = logo, -- if set
    description = description, -- if set
    keywords = keywords, -- if set
  }),
  ... other forwarded tag name and value pairs
}
```

#### Transfer-Record

Transfers ownership of a specific undername record to another address.

Executable by the process `Owner`, an authorized user in the `Controllers` table, or the current record owner.

##### Parameters

| Name                   | Type   | Description                                                                                                   |
| ---------------------- | ------ | ------------------------------------------------------------------------------------------------------------- |
| Sub-Domain             | string | The undername record to transfer, e.g., `@`, `ardrive`, or `dapp_ardrive`                                     |
| Recipient              | string | The wallet address to transfer record ownership to, e.g., iKryOeZQMONi2965nKz528htMMN_sBcjlhc-VncoRjA         |
| Allow-Unsafe-Addresses | string | Optional. If set to `true`, allows transferring to addresses that may not be valid AO addresses.              |

##### Rules

- Must be an authorized process `Owner`, `Controller`, or the current record owner.
- Must specify a valid `Sub-Domain` parameter (string) as a message tag.
- Must specify a valid `Recipient` parameter (string) as a message tag.
- The undername's `Sub-Domain` must already exist in the `Records` table.
- The record must have an existing owner to be transferable.
- Must add `X-`forwarded tags to the response notice.
- Sends an additional `Transfer-Record-Notice` to the new owner.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Transfer-Record",
  ["Sub-Domain"] = "foo",
  Recipient = "{Wallet Address}"
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Transfer-Record-Notice",
  Data = permissionErr,
  Error = "Transfer-Record-Error",
  ["Message-Id"] = msg.Id
}
```

**Invalid parameters**

```
{
  Target = msg.From,
  Action = "Invalid-Transfer-Record-Notice",
  Data = transferRecordResult,
  Error = "Transfer-Record-Error",
  ["Message-Id"] = msg.Id
}
```

**Valid parameters**

```
-- Notice sent to the caller
{
  Target = msg.From,
  Action = "Transfer-Record-Notice",
  Data = json.encode({
    ["Sub-Domain"] = subdomain,
    Recipient = recipient,
    ["Previous-Owner"] = previousOwner,
  }),
  ... other forwarded tag name and value pairs
}

-- Additional notice sent to the new owner
{
  Target = recipient,
  Action = "Transfer-Record-Notice",
  ["Sub-Domain"] = subdomain,
  ["Previous-Owner"] = previousOwner,
  Data = json.encode({
    ["Sub-Domain"] = subdomain,
    Recipient = recipient,
    ["Previous-Owner"] = previousOwner,
  })
}
```

#### Remove-Record

Removes an existing undername’s `Sub-Domain` in the Records table.

##### Parameters

| Name       | Type   | Description                                                             |
| ---------- | ------ | ----------------------------------------------------------------------- |
| Sub-Domain | string | The undername record to remove, e.g., `@`, `ardrive`, or `dapp_ardrive` |

##### Rules

- Must be an authorized process `Owner` or `Controller`.
- Must specify a valid `Sub-Domain` parameter (string) as a message tag.
- The undername’s `Sub-Domain` must already exist in the `Records` table.
- Must add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Remove-Record",
  ["Sub-Domain"] = "foo"
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Remove-Record-Notice",
  Data = permissionErr,
  Error = "Remove-Record-Error",
  ["Message-Id"] = msg.Id
}
```

**Invalid parameters**

```
{
  Target = msg.From,
  Action = "Invalid-Remove-Record-Notice",
  Data = removeRecordResult,
  Error = "Remove-Record-Error",
  ["Message-Id"] = msg.Id
}
```

**Valid parameters**

```
{
  Target = msg.From,
  Action = "Remove-Record-Notice",
  Data = json.encode(Records),
  ... other forwarded tag name and value pairs
}
```

#### State

The State handler is updated in **ARNS-MANAGE-1** to return specific information about the state of the process, including all `Records`, its current `Owner`, and all `Controllers`.

Executable by anonymous users.

##### Parameters

No parameters necessary.

##### Rules

- Must return a state object as JSON in the data field of the response notice. The state object must include:
  - Entire `Records` table.
  - Entire `Controllers` table.
  - Process `Owner`.
- Must add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "State"
})
```

##### Responses

**Valid `State`**

```
{
  Target = msg.From,
  Action = "State-Notice",
  Data = json.encode({
    Records = Records,
    Controllers = Controllers,
    Owner = ao.env.Process.Owner,
  }),
  ... other forwarded tag name and value pairs
}
```

### AR.IO Network Integration Handlers

The following handlers enable ANTs to interact with the AR.IO Network Process for managing ArNS name registrations.

#### Release-Name

Releases an ArNS name back to the AR.IO Network, making it available for registration by others.

Executable by the process `Owner` only.

##### Parameters

| Name           | Type   | Description                                                      |
| -------------- | ------ | ---------------------------------------------------------------- |
| IO-Process-Id  | string | The AR.IO Network Process ID to send the release request to.    |
| Name           | string | The ArNS name to release, e.g., `ardrive` or `my-name`.         |

##### Rules

- Must be the process `Owner`.
- Must specify a valid `IO-Process-Id` parameter (Arweave address) as a message tag.
- Must specify a valid `Name` parameter (string) as a message tag.
- Sends a message to the AR.IO Network Process to release the name.
- Must add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Release-Name",
  ["IO-Process-Id"] = "{AR.IO Network Process ID}",
  Name = "my-arns-name"
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Release-Name-Notice",
  Data = "Caller is not the process owner",
  Error = "Release-Name-Error",
  ["Message-Id"] = msg.Id
}
```

**Invalid parameters**

```
{
  Target = msg.From,
  Action = "Invalid-Release-Name-Notice",
  Data = "Invalid Arweave ID",
  Error = "Release-Name-Error",
  ["Message-Id"] = msg.Id
}
```

**Valid parameters**

```
{
  Target = msg.From,
  Action = "Release-Name-Notice",
  Initiator = msg.From,
  Name = name,
  ... other forwarded tag name and value pairs
}
```

#### Reassign-Name

Reassigns an ArNS name from this ANT to another ANT Process.

Executable by the process `Owner` only.

##### Parameters

| Name           | Type   | Description                                                      |
| -------------- | ------ | ---------------------------------------------------------------- |
| IO-Process-Id  | string | The AR.IO Network Process ID to send the reassign request to.   |
| Process-Id     | string | The new ANT Process ID to assign the name to.                   |
| Name           | string | The ArNS name to reassign, e.g., `ardrive` or `my-name`.        |

##### Rules

- Must be the process `Owner`.
- Must specify a valid `IO-Process-Id` parameter (Arweave address) as a message tag.
- Must specify a valid `Process-Id` parameter (Arweave address) as a message tag.
- Must specify a valid `Name` parameter (string) as a message tag.
- Sends a message to the AR.IO Network Process to reassign the name.
- Must add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Reassign-Name",
  ["IO-Process-Id"] = "{AR.IO Network Process ID}",
  ["Process-Id"] = "{New ANT Process ID}",
  Name = "my-arns-name"
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Reassign-Name-Notice",
  Data = "Caller is not the process owner",
  Error = "Reassign-Name-Error",
  ["Message-Id"] = msg.Id
}
```

**Invalid parameters**

```
{
  Target = msg.From,
  Action = "Invalid-Reassign-Name-Notice",
  Data = "Invalid Arweave ID",
  Error = "Reassign-Name-Error",
  ["Message-Id"] = msg.Id
}
```

**Valid parameters**

```
{
  Target = msg.From,
  Action = "Reassign-Name-Notice",
  Initiator = msg.From,
  Name = name,
  ["Process-Id"] = antProcessIdToReassign,
  ... other forwarded tag name and value pairs
}
```

#### Approve-Primary-Name

Approves a primary name request for a specific recipient address.

Executable by the process `Owner`, or the record owner (for their own undername).

##### Parameters

| Name                   | Type   | Description                                                      |
| ---------------------- | ------ | ---------------------------------------------------------------- |
| IO-Process-Id          | string | The AR.IO Network Process ID to send the approval to.           |
| Recipient              | string | The address to approve for primary name usage.                  |
| Name                   | string | The ArNS name or undername to approve, e.g., `ardrive`, `my-name`, or `alice`. |
| Allow-Unsafe-Addresses | string | Optional. If set to `true`, allows addresses that may not be valid AO addresses. |

##### Rules

- Must be the process `Owner`, or the record owner of the specified undername.
- If the caller is a record owner (not ANT owner), the `Recipient` must equal the caller's address (record owners can only approve primary names for themselves).
- Must specify a valid `IO-Process-Id` parameter (Arweave address) as a message tag.
- Must specify a valid `Recipient` parameter (AO address) as a message tag.
- Must specify a valid `Name` parameter (string) as a message tag.
- Sends a message to the AR.IO Network Process to approve the primary name request.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Approve-Primary-Name",
  ["IO-Process-Id"] = "{AR.IO Network Process ID}",
  Recipient = "{Address to approve}",
  Name = "alice" -- can be base name or undername
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Approve-Primary-Name-Notice",
  Data = "Caller is not the process owner",
  Error = "Approve-Primary-Name-Error",
  ["Message-Id"] = msg.Id
}
```

**Invalid parameters**

```
{
  Target = msg.From,
  Action = "Invalid-Approve-Primary-Name-Notice",
  Data = "Invalid Arweave ID",
  Error = "Approve-Primary-Name-Error",
  ["Message-Id"] = msg.Id
}
```

#### Remove-Primary-Names

Removes one or more primary name associations.

Executable by the process `Owner`, or record owners (for their own undernames).

##### Parameters

| Name           | Type   | Description                                                      |
| -------------- | ------ | ---------------------------------------------------------------- |
| IO-Process-Id  | string | The AR.IO Network Process ID to send the removal request to.    |
| Names          | string | Comma-separated list of names to remove primary status from.    |

##### Rules

- Must be the process `Owner`, or the record owner for each name being removed.
- If the caller is a record owner (not ANT owner), they can only remove their own record's primary name status.
- Must specify a valid `IO-Process-Id` parameter (Arweave address) as a message tag.
- Must specify a valid `Names` parameter (comma-separated string) as a message tag.
- Each name in the list must be a valid undername format.
- Each name is validated to ensure the caller has permission (either ANT owner or the specific record owner).
- Sends a message to the AR.IO Network Process to remove primary name associations.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Remove-Primary-Names",
  ["IO-Process-Id"] = "{AR.IO Network Process ID}",
  Names = "name1,name2,name3"
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Remove-Primary-Names-Notice",
  Data = "Caller is not the process owner",
  Error = "Remove-Primary-Names-Error",
  ["Message-Id"] = msg.Id
}
```

**Invalid parameters**

```
{
  Target = msg.From,
  Action = "Invalid-Remove-Primary-Names-Notice",
  Data = "Invalid Arweave ID",
  Error = "Remove-Primary-Names-Error",
  ["Message-Id"] = msg.Id
}
```

### Boot Handler

The ANT process includes a special `_boot` handler that is triggered when the process is initialized or loaded. This handler is particularly important for WASM module implementations.

#### Behavior

When the process receives a boot message (Type = "Process" from the Owner):

1. **State Initialization**: If the message includes Data containing a valid JSON state, the ANT will initialize its state from this data, setting:
   - Records
   - Controllers
   - Owner
   - Name, Ticker, Logo, Description, Keywords (if ARNS-TOKEN-1 is implemented)
   - Balances (if ARNS-TOKEN-1 is implemented)

2. **Credit Notice**: Sends a Credit-Notice to the Owner (if ARNS-TOKEN-1 is implemented).

3. **State Notification**: 
   - Sends a State-Notice to the ANT Registry (if configured with ANT-Registry-Id tag).
   - Sends a patch notice for state caching.
   - Sends a self State-Notice to enable caching.

4. **Initialized Flag**: Sets the `Initialized` flag to true after successful initialization.

#### Rules

- Only the process Owner can trigger the boot handler.
- Invalid JSON data will result in an Invalid-Boot-Notice error.
- The boot handler has highest priority (prepended to handler list).
- This handler is automatically configured and should not be modified.
