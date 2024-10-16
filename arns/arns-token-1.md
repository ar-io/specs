# ARNS-TOKEN-1

## Status:

**In-Review**

## Version:

| Version | Description                                               | Date       |
| ------- | --------------------------------------------------------- | ---------- |
| 1.0.0   | Initial version of the **ARNS-TOKEN-1** specification.    | 2024-09-01 |
| 1.0.1   | Fixed Credit/Debit response notices for Transfer handler. | 2024-09-24 |
| 1.1.0   | Added 'description' and 'keywords' metadata.              | 2024-10-14 |

## Abstract

The **ARNS-TOKEN-1** specification defines the framework for creating and managing Arweave Name Tokens (ANTs) with token transferability, balances, and metadata. It builds on the core and management specifications to introduce a non-fungible token model specific to ArNS.

## Motivation

The **ARNS-TOKEN-1** specification builds off of the **ARNS-MANAGE-1** and **ARNS-CORE-1** specifications and provides a standardized approach for adding token-like features to Arweave Name Tokens (ANTs), enabling them to be transferred, tracked, and managed as single, indivisible assets.

By implementing this specification, developers can extend the functionality of Arweave Names to include ownership transfer, metadata management, and integration with external services like block explorers and marketplaces. This allows for more dynamic and flexible use cases within the Arweave ecosystem, such as tokenizing digital identities or assets on the Permaweb.

The **ARNS-TOKEN-1** Specification merges the protocols set in the AO Token and Subledger Specification except it is designed to act as a single, non-fungible token. This means that there is a total, indivisible token supply of 1 for each ANT, and transferring it changes both the balance and ownership.

For more information on the AO Token and Subledger Specification see https://hackmd.io/8DiMkhuNThOb_ooTWKqxaw#ao-Token-and-Subledger-Specification

### Language of Implementation

All examples and code snippets in this specification are written in Lua. This choice ensures compatibility with the AO Processes and the Arweave ecosystem.

However, developers are not restricted to using Lua exclusively when building new features or extending functionalities around ArNS. While Lua is recommended for direct integration with existing infrastructure, other programming languages can be used, provided they adhere to the protocols and specifications outlined in this document.

## Specification

### Overview

The **ARNS-TOKEN-1** Specification includes the following requirements:

- Must have a `Name` which is a friendly nickname of this ANT.
  - Must be a string, eg. `ArDrive`.
- Must have a `Ticker` which is a short token symbol, shown in block explorers and marketplaces.
  - Must be a string, eg. `ANT-ARDRIVE`.
- Must have a `Logo` image icon used by downstream apps.
  - Must be a string eg. `Sie_26dvgyok0PZD_-iQAFOhOd5YxDTkczOLoqTTL_A`.
  - Must be an Arweave transaction.
- Must have a `Description` that is a brief description of this ANT and its purpose or use.  
  - Must be a string eg. `ArDrive is a permaweb app that lets you upload, download and share your files easily.`
  - Must not be longer than 512 characters.
- Must have a table of `Keywords` used to further describe this ANT.
  - `Keywords` must not contain more than 16 keywords.
  - Each `Keyword` must be a string eg. `File-sharing`
  - Each `Keyword` must consist of alphanumeric characters, dashes and underscores.
  - Each `Keyword` must not be longer than 32 characters.
  - Each `Keyword` must not include spaces.
  - Each `Keyword` must be unique in the table of `Keywords`.
- Must have a `Denomination`, in compliance with the AO Token Blueprint, indicating the decimal places used by the token.
  - Must be an integer.
  - Must be set to 0, indicating no decimal places used by the Arweave Name Token.
- Must have a `TotalSupply`, in compliance with the AO Token Blueprint, indicating the total amount of tokens minted.
  - Must be an integer.
  - Must be set to 1, indicating a single token in the supply.
- Must have a `Balances` table which stores the current balance of minted Tokens.
  - The Process `Owner` should be the only Token holder with a balance of 1.
- Must have a `Balance` handler to read an individual user’s Token Balance.
- Must have a `Balances` handler to read all Token Balances.
- Must have a handler to Transfer the ANT.
  - All Transfers move the single Token balance and Process Owner to the Recipient.
  - Authorized for the Process `Owner` only.
  - Transfer should have a `State` call-back message to the ANT Registry.
- Must have an `Info` handler to read the Token metadata including `Name`, `Ticker`, `Total-Supply`, `Logo`, `Denomination`, `Description`, `Keywords` and `Owner`.
  - `Total-Supply` must be returned as an integer string.
  - `Denomination` must be returned as an integer string.
- Must have a `Total-Supply `handler to calculate the total supply of outstanding Token balances.
- Should have a `Set-Ticker` handler to change the ANT’s ticker.
  - Authorized for the Process `Owner` and `Controllers`.
- Should have a `Set-Name` handler to change the ANT’s name.
  - Authorized for the Process `Owner` and `Controllers`.
- Should have a `Set-Logo` handler to change the ANT’s logo.
  - Authorized for the Process `Owner` and `Controllers`.
- Should have a `Set-Description` handler to change the ANT’s description.
  - Authorized for the Process `Owner` and `Controllers`.
- Should have a `Set-Keywords` handler to change the ANT’s keywords.
  - Authorized for the Process `Owner` and `Controllers`.
- Handler to read entire ANT State must be updated to also return `Balances`, `Name`, `Ticker`, `Logo`, `Denomination`, `Description`, `Keywords` and `TotalSupply`.

The **ARNS-TOKEN-1** Specification does not include the ability to `Mint` or `Burn`; however, developers can extend ANTs to new functionality if needed.

### Objects

The **ARNS-TOKEN-1** specification leverages the general AO Token and Subledger specification, and includes all of the objects in **ARNS-MANAGE-1** and **ARNS-CORE-1**

#### ARNS-TOKEN-1 Objects

```
-- ARNS-TOKEN-1 Objects
Name = Name or "Arweave Name Token"
Ticker = Ticker or "ANT"
Description = Description or "This is an Arweave Name Token."
Keywords = Keywords or {}
Logo = Logo or "Sie_26dvgyok0PZD_-iQAFOhOd5YxDTkczOLoqTTL_A"
Denomination = Denomination or 0
TotalSupply = TotalSupply or 1

-- ARNS-MANAGE-1 Objects
Owner = Owner or ao.env.Process.Owner
Controllers = Controllers or { Owner }

-- ARNS-CORE-1 Objects
Records = Records or {
  ["@"] = {
    transactionId = "UyC5P5qKPZaltMmmZAWdakhlDXsBF6qmyrbWYFchRTk",
    ttlSeconds = 3600
  }
}
```

### Handlers

#### Action Map

The following actions are handled in the **ARNS-TOKEN-1** specification, and include all of the actions contained in **ARNS-MANAGE-1**, **ARNS-CORE-1**, and AO Token specifications.

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
  RemoveRecord = "Remove-Record",
}

ARNSTokenSpecActionMap = {
  -- write
  SetName = "Set-Name",
  SetTicker = "Set-Ticker",
  SetDescription = "Set-Description"
  SetKeywords = "Set-Keywords"
}

TokenSpecActionMap = {
  Info = "Info",
  Balances = "Balances",
  Balance = "Balance",
  Transfer = "Transfer",
  TotalSupply = "Total-Supply",
  -- not implemented
  Mint = "Mint",
  Burn = "Burn",
}
```

#### Set-Name

Updates the `Name` or the nickname for the ANT.

Executable by the process `Owner` or an authorized user in the `Controllers` table.

##### Parameters

| Name | Type   | Description               |
| ---- | ------ | ------------------------- |
| Name | string | The new Name for the ANT. |

##### Rules

- Must be an authorized process `Owner` or `Controller`.
- Must specify a valid `Name` parameter (string) as a message tag.
- Should add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Set-Name",
  Name = "ArDrive"
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Set-Name-Notice",
  Data = permissionErr,
  Error = "Set-Name-Error",
  ["Message-Id"] = msg.Id
}
```

**Invalid parameters**

```
{
  Target = msg.From,
  Action = "Invalid-Set-Name-Notice",
  Data = nameRes,
  Error = "Set-Name-Error",
  ["Message-Id"] = msg.Id
}
```

**Valid parameters**

```
{
  Target = msg.From,
  Action = "Set-Name-Notice",
  Data = json.encode({ Name = Name }),
  ... other forwarded tag name and value pairs
}
```

#### Set-Ticker

Updates the `Ticker` symbol for this ANT.

Executable by the process `Owner` or an authorized user in the `Controllers` table.

##### Parameters

| Name   | Type   | Description                 |
| ------ | ------ | --------------------------- |
| Ticker | string | The new Ticker for the ANT. |

##### Rules

- Must be an authorized process `Owner` or `Controller`.
- Must specify a valid `Ticker` parameter (string) as a message tag.
- Should add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Set-Ticker",
  Ticker = "ANT-ARDRIVE"
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Set-Ticker-Notice",
  Data = permissionErr,
  Error = "Set-Ticker-Error",
  ["Message-Id"] = msg.Id
}
```

**Invalid parameters**

```
{
  Target = msg.From,
  Action = "Invalid-Set-Ticker-Notice",
  Data = tickerRes,
  Error = "Set-Ticker-Error",
  ["Message-Id"] = msg.Id,
}
```

**Valid parameters**

```
{
  Target = msg.From,
  Action = "Set-Ticker-Notice",
  Data = json.encode({ Ticker = Ticker }),
  ... other forwarded tag name and value pairs
}
```

#### Set-Description

Updates the `Description` of this ANT.

Executable by the process `Owner` or an authorized user in the `Controllers` table.

##### Parameters

| Name   | Type   | Description                 |
| ------ | ------ | --------------------------- |
| Description | string | The new Description for the ANT. |

##### Rules

- Must be an authorized process `Owner` or `Controller`.
- Must specify a valid `Description` parameter (string) as a message tag.
- `Description` must not be longer than 512 characters.
- Should add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Set-Description",
  Description = "ArDrive is an app that makes it easy to upload, download and share your public or private files on Arweave"
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Set-Description-Notice",
  Data = permissionErr,
  Error = "Set-Description-Error",
  ["Message-Id"] = msg.Id
}
```

**Invalid parameters**

```
{
  Target = msg.From,
  Action = "Invalid-Set-Description-Notice",
  Data = descriptionRes,
  Error = "Set-Description-Error",
  ["Message-Id"] = msg.Id,
}
```

**Valid parameters**

```
{
  Target = msg.From,
  Action = "Set-Description-Notice",
  Data = json.encode({ Description = Description }),
  ... other forwarded tag name and value pairs
}
```

#### Set-Keywords

Updates the `Keywords` of this ANT.

Executable by the process `Owner` or an authorized user in the `Controllers` table.

##### Parameters

| Name   | Type   | Description                 |
| ------ | ------ | --------------------------- |
| Keywords | table | The new Keywords for the ANT. |

##### Rules

- Must be an authorized process `Owner` or `Controller`.
- Must specify a valid `Keywords` parameter (table) as a message tag.
- Each `Keyword` must not be longer than 32 characters.
- Each `Keyword` must consist of alphanumeric characters, dashes, underscores, @ or #.
- Each `Keyword` must not include spaces.
- Each `Keyword` must be unique in the table of `Keywords`.
- There must not be more than 16 total `Keywords`.
- Should add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Set-Description",
  Description = "ArDrive is an app that makes it easy to upload, download and share your public or private files on Arweave"
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Set-Description-Notice",
  Data = permissionErr,
  Error = "Set-Description-Error",
  ["Message-Id"] = msg.Id
}
```

**Invalid parameters**

```
{
  Target = msg.From,
  Action = "Invalid-Set-Description-Notice",
  Data = descriptionRes,
  Error = "Set-Description-Error",
  ["Message-Id"] = msg.Id,
}
```

**Valid parameters**

```
{
  Target = msg.From,
  Action = "Set-Description-Notice",
  Data = json.encode({ Description = Description }),
  ... other forwarded tag name and value pairs
}
```

#### Balance

Returns the balance of a target; if a target is not supplied, then the balance of the sender of the message must be returned.

Executable by anonymous users.

AO Token Specification reference: https://hackmd.io/8DiMkhuNThOb_ooTWKqxaw#BalanceTarget--string

##### Parameters

| Name      | Type   | Description                       |
| --------- | ------ | --------------------------------- |
| Recipient | string | The recipient to get balance for. |

##### Rules

- Must return entire `Balances` table as JSON in the data field of the response notice.
- Should add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Balances"
})
```

##### Responses

**Valid `Balances`**

```
{
  Target = msg.From,
  Action = "Balances-Notice",
  Data = json.encode(Balances),
  ... other forwarded tag name and value pairs
}
```

#### Balances

Gets the entire Balances table, including each token holder and their current balance.

Executable by anonymous users.

AO Token Specification reference: https://hackmd.io/8DiMkhuNThOb_ooTWKqxaw#Balances

##### Rules

- Must return entire `Balances` table as JSON in the data field of the response notice.
- Should add `X-`forwarded tags to the response notice.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Balances"
})
```

##### Responses

**Valid `Balances`**

```
{
  Target = msg.From,
  Action = "Balances-Notice",
  Data = json.encode(Balances),
  ... other forwarded tag name and value pairs
}
```

#### Transfer

If the sender is authorized, transfer ANT Ownership to the `Recipient`, issuing a `Credit-Notice` to the recipient and a `Debit-Notice` to the sender. If the sender is not authorized, fail and notify the sender.

Additionally, the ARNS-TOKEN-1 specification modifies the AO Token transfer specification to accommodate its non-fungible, single token supply.

It is executable only by the single Token `Balance` holder, process `Owner`, or Process itself.

`Quantity` is no longer needed, since the ARNS-TOKEN-1 specification calls for a single token in the supply. `Transfer` will move the single token to the `Recipient`.

Upon transfer, it also sets the `Recipient` of the token transfer to be the new `Owner` of the process, as well as clears out any previous `Controllers`.

AO Token Specification reference: https://hackmd.io/8DiMkhuNThOb_ooTWKqxaw#TransferTarget-Quantity

##### Parameters

| Name      | Type   | Description                                                                                                   |
| --------- | ------ | ------------------------------------------------------------------------------------------------------------- |
| Recipient | string | The wallet address to transfer the ANT ownership and token, e.g., iKryOeZQMONi2965nKz528htMMN_sBcjlhc-VncoRjA |

##### Rules

- Must be the ANT `Balance` holder, process `Owner`, or Process itself.
- Must add `X-`forwarded tags to the `Credit-Notice` and `Debit-Notice` responses.
- Should send a `State` notice to any associated ANT Registry Processes.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Transfer",
  Recipient = "{Wallet Address}"
})
```

##### Responses

**Permission error, not authorized**

```
{
  Target = msg.From,
  Action = "Invalid-Transfer-Notice",
  Data = permissionErr,
  Error = "Transfer-Error",
  ["Message-Id"] = msg.Id
}
```

**Invalid parameters**

```
{
  Target = msg.From,
  Action = "Invalid-Transfer-Notice",
  Data = transferResult,
  Error = "Transfer-Error",
  ["Message-Id"] = msg.Id,
}
```

**Valid parameters**

```
-- Debit-Notice message template, that is sent to the Sender of the transfer
{
  Target = msg.From,
  Action = "Debit-Notice",
  Recipient = msg.Tags.Recipient,
  Quantity = msg.Tags.Quantity,
  Data = "You transferred " .. msg.Tags.Quantity .. " to " .. msg.Tags.Recipient,
}
-- Credit-Notice message template, that is sent to the Recipient of the transfer
{
  Target = msg.Tags.Recipient,
  Action = "Credit-Notice",
  Sender = msg.From,
  Quantity = msg.Tags.Quantity,
  Data = "You received " .. msg.Tags.Quantity .. " from " .. msg.From,
}
```

#### Info

Returns all Token metadata associated with this process including its `Name`, `Ticker`, `Total-Supply`, `Logo`, `Denomination`, and process `Owner`.

Executable by anonymous users.

##### Parameters

No parameters necessary.

##### Rules

- Must return general process information as encoded JSON in the data field and as tags of the response notice. Info JSON must include:
  - Process `Owner`
  - `Name`, `Ticker`, `Logo`, `Description`, and `Keywords`
  - `Denomination` and `Total-Supply` as integer strings.

##### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Info"
})
```

##### Responses

**Valid Info**

```
{
  Target = msg.From,
  Action = "Info-Notice",
  Tags = {
    Name = Name,
    Ticker = Ticker,
    ["Total-Supply"] = tostring(TotalSupply),
    Logo = Logo,
    Denomination = tostring(Denomination),
    Owner = Owner,
  },
  Data = json.encode({
    Name = Name,
    Ticker = Ticker,
    ["Total-Supply"] = tostring(TotalSupply),
    Logo = Logo,
    Denomination = tostring(Denomination),
    Owner = Owner,
    ["Source-Code-TX-ID"] = SourceCodeTxId,
  })
}
```

#### State

The State handler is updated in **ARNS-TOKEN-1** to return all information about the state of the token, including all `Records`, `Owner`, `Controllers`, `Balances`, `Name`, `Ticker`, `Logo`, `Description`, `Keywords`, `Denomination`, and `TotalSupply`.

Executable by anonymous users.

##### Parameters

No parameters necessary.

##### Rules

- Must return the full state of the process as encoded JSON in the data field of the response notice. State information must include:
  - Entire `Records` table
  - Entire `Controllers` table
  - Entire `Balances` table
  - Process `Owner`
  - `Name`, `Ticker`, `Logo`, `Description` and `Keywords`
  - `Denomination` and `TotalSupply`
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
    Balances = Balances,
    Owner = Owner,
    Name = Name,
    Ticker = Ticker,
    Logo = Logo,
    Description = Description,
    Keywords = Keywords,
    Denomination = Denomination,
    TotalSupply = TotalSupply,
  }),
  ... other forwarded tag name and value pairs
}
```
