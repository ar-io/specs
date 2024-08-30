# ARNS-TOKEN-01

## Overview

The Arweave Name Token is created when layering the utilities needed to support token transferability, balances, and other helpful metadata on top of the **ARNS-CORE-01** and **ARNS-MANAGE-01** Specifications.

The **ARNS-TOKEN-01** Specification merges the specifications set in the AO Token Blueprint and is designed to act as a true non-fungible token. This means that there is a total, indivisible token supply of 1 for each ANT, and transferring it changes both the balance and ownership.

## Specification Requirements

The **ARNS-TOKEN-01** Specification includes the following requirements:

- Must have a `Name` which is a friendly nickname of this ANT.
- Must have a `Ticker` which is a short token symbol, shown in block explorers and marketplaces.
- Must have a `Logo` that is an Arweave Transaction ID that contains an icon used by downstream apps.
- Must have a `Denomination` of 0 indicating the decimal places used by the token.
- Must have a `TotalSupply` of 1 indicating the total amount of tokens minted.
- Must have a `Balances` object which stores the current balance of minted Tokens.
  - The Process `Owner` should be the only Token holder with a balance of 1.
- Must have a `Balance` handler to read an individual user’s Token Balance.
- Must have a `Balances` handler to read all Token Balances.
- Must have a handler to Transfer the ANT.
  - All Transfers move the single Token balance and Process Owner to the Recipient.
  - Authorized for the Process `Owner` only.
  - Transfer should have a `State` call-back message to the ANT Registry.
- Must have an `Info` handler to read the Token metadata including `Name`, `Ticker`, `TotalSupply`, `Logo`, `Denomination`, and `Owner`.
- Must have a `Total-Supply `handler to calculate the total supply of outstanding Token balances.
- Should have a `Set-Ticker` handler to change the ANT’s ticker.
  - Authorized for the Process `Owner` and `Controllers`.
- Should have a `Set-Name` handler to change the ANT’s name.
  - Authorized for the Process `Owner` and `Controllers`.
- Should have a `Set-Logo` handler to change the ANT’s logo.
  - Authorized for the Process `Owner` and `Controllers`.
- Handler to read entire ANT State must be updated to also return `Balances`, `Name`, `Ticker`, `Logo`, `Denomination`, and `TotalSupply`.

The **ARNS-TOKEN-01** Specification does not include the ability to `Mint` or `Burn`; however, developers can extend ANTs to new functionality if needed.

## Objects

The **ARNS-TOKEN-01** specification includes all of the objects in **ARNS-MANAGE-01**, **ARNS-CORE-01**, and an adapted AO Token blueprint.

More information on the AO Token and Subledger specification: https://hackmd.io/8DiMkhuNThOb_ooTWKqxaw#ao-Token-and-Subledger-Specification

### ARNS-TOKEN-01 Objects

```
-- ARNS-TOKEN-01 Objects
Name = Name or "Arweave Name Token" -- optional
Ticker = Ticker or "ANT" -- optional
Logo = Logo or "Sie_26dvgyok0PZD_-iQAFOhOd5YxDTkczOLoqTTL_A" -- optional
Denomination = Denomination or 0
TotalSupply = TotalSupply or 1

-- ARNS-MANAGE-01 Objects
Owner = Owner or ao.env.Process.Owner
Controllers = Controllers or { Owner }

-- ARNS-CORE-01 Objects
Records = Records or {
  ["@"] = {
    transactionId = "UyC5P5qKPZaltMmmZAWdakhlDXsBF6qmyrbWYFchRTk",
    ttlSeconds = 3600
  }
}

```

## Handlers and Responses

The following actions are handled in the **ARNS-TOKEN-01** specification, and include all of the actions contained in **ARNS-MANAGE-01**, **ARNS-CORE-01**, and AO Token specifications.

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

### Set-Name

Updates the `Name` or the nickname for the ANT.

Executable by the process `Owner` or an authorized user in the `Controllers` table.

#### Parameters

| Name | Type   | Description               |
| ---- | ------ | ------------------------- |
| Name | string | The new Name for the ANT. |

#### Rules

- Must be an authorized process `Owner` or `Controller`.
- Must specify a valid `Name` parameter (string) as a message tag.
- Should add `X-`forwarded tags to the response notice.

#### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Set-Name",
  Name = "ArDrive"
})
```

#### Responses

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

### Set-Ticker

Updates the `Ticker` symbol for this ANT.

Executable by the process `Owner` or an authorized user in the `Controllers` table.

#### Parameters

| Name   | Type   | Description                 |
| ------ | ------ | --------------------------- |
| Ticker | string | The new Ticker for the ANT. |

#### Rules

- Must be an authorized process `Owner` or `Controller`.
- Must specify a valid `Ticker` parameter (string) as a message tag.
- Should add `X-`forwarded tags to the response notice.

#### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Set-Ticker",
  Ticker = "ANT-ARDRIVE"
})
```

#### Responses

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

### Balance

Returns the balance of a target; if a target is not supplied, then the balance of the sender of the message must be returned.

Executable by anonymous users.

AO Token Specification reference: https://hackmd.io/8DiMkhuNThOb_ooTWKqxaw#BalanceTarget--string

#### Parameters

| Name      | Type   | Description                       |
| --------- | ------ | --------------------------------- |
| Recipient | string | The recipient to get balance for. |

#### Rules

- Must return entire `Balances` object as JSON in the data field of the response notice.
- Should add `X-`forwarded tags to the response notice.

#### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Balances"
})
```

#### Responses

**Valid `Balances`**

```
{
  Target = msg.From,
  Action = "Balances-Notice",
  Data = json.encode(Balances),
  ... other forwarded tag name and value pairs
}
```

### Balances

Gets the entire Balances table, including each token holder and their current balance.

Executable by anonymous users.

AO Token Specification reference: https://hackmd.io/8DiMkhuNThOb_ooTWKqxaw#Balances

#### Rules

- Must return entire `Balances` object as JSON in the data field of the response notice.
- Should add `X-`forwarded tags to the response notice.

#### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Balances"
})
```

#### Responses

**Valid `Balances`**

```
{
  Target = msg.From,
  Action = "Balances-Notice",
  Data = json.encode(Balances),
  ... other forwarded tag name and value pairs
}
```

### Transfer

If the sender is authorized, transfer ANT Ownership to the `Recipient`, issuing a `Credit-Notice` to the recipient and a `Debit-Notice` to the sender. If the sender is not authorized, fail and notify the sender.

Additionally, the ARNS-TOKEN-01 specification modifies the AO Token transfer specification to accommodate its non-fungible, single token supply.

It is executable only by the single Token `Balance` holder, process `Owner`, or Process itself.

`Quantity` is no longer needed, since the ARNS-TOKEN-01 specification calls for a single token in the supply. `Transfer` will move the single token to the `Recipient`.

Upon transfer, it also sets the `Recipient` of the token transfer to be the new `Owner` of the process, as well as clears out any previous `Controllers`.

AO Token Specification reference: https://hackmd.io/8DiMkhuNThOb_ooTWKqxaw#TransferTarget-Quantity

#### Parameters

| Name      | Type   | Description                                                                                                   |
| --------- | ------ | ------------------------------------------------------------------------------------------------------------- |
| Recipient | string | The wallet address to transfer the ANT ownership and token, e.g., iKryOeZQMONi2965nKz528htMMN_sBcjlhc-VncoRjA |

#### Rules

- Must be the ANT `Balance` holder, process `Owner`, or Process itself.
- Must add `X-`forwarded tags to the `Credit-Notice` and `Debit-Notice` responses.
- Should send a `State` notice to any associated ANT Registry Processes.

#### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Transfer",
  Recipient = "{Wallet Address}"
})
```

#### Responses

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
{
  Target = msg.From,
  Action = "Transfer-Notice",
  Data = json.encode({ Ticker = Ticker }),
  ... other forwarded tag name and value pairs
}
```

### Info

Returns all Token metadata associated with this process including its `Name`, `Ticker`, `Total-Supply`, `Logo`, `Denomination`, and process `Owner`.

Executable by anonymous users.

#### Parameters

No parameters necessary.

#### Rules

- Must return an info object as JSON in the data field and as tags of the response notice. Info object must include:
  - Process `Owner`
  - `Name`, `Ticker`, `Logo`
  - `Denomination` and `Total-Supply`

#### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Info"
})
```

#### Responses

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

### State

The State handler is updated in **ARNS-TOKEN-01** to return all information about the state of the token, including all `Records`, `Owner`, `Controllers`, `Balances`, `Name`, `Ticker`, `Logo`, `Denomination`, and `TotalSupply`.

Executable by anonymous users.

#### Parameters

No parameters necessary.

#### Rules

- Must return a state object as JSON in the data field of the response notice. State object must include:
  - Entire `Records` table
  - Entire `Controllers` table
  - Entire `Balances` table
  - Process `Owner`
  - `Name`, `Ticker`, `Logo`
  - `Denomination` and `TotalSupply`
- Must add `X-`forwarded tags to the response notice.

#### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "State"
})
```

#### Responses

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
    Denomination = Denomination,
    TotalSupply = TotalSupply,
  }),
  ... other forwarded tag name and value pairs
}
```
