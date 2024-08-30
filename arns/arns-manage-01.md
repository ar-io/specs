# ARNS-MANAGE-01

## Overview

The first layer added onto the ARNS-CORE-01 Specification includes the utilities needed to manage and control the ANT. This includes adding new records, modifying existing records, removing records, and configuring who else can control the ANT.

The **ARNS-MANAGE-01** Specification includes the following requirements:

- Must include a list of `Controllers` who act as secondary users who control the `Records` set in this ANT.
- Must have an `Add-Controller` handler to add new `Controllers`.
  - Authorized for the Process `Owner` only.
- Must have a `Remove-Controller` handler to remove `Controllers`.
  - Authorized for the Process `Owner` only.
- Must have a `Set-Record` handler to set new `Records` and modify existing ones.
  - Authorized for the Process `Owner` and `Controllers`.
- Must have a `Remove-Record` handler to remove `Records`.
  - Authorized for the Process `Owner` and `Controllers`.
- Must have a handler to read all `Controllers`.
- Handler to read entire ANT `State` is updated to also return `Controllers`.

This adds flexibility for Arweave Name Process Owners to manage their records, and leverage controllers for other handlers that need additional access controls.

## Objects

The **ARNS-MANAGE-01** specification includes all of the objects contained in **ARNS-CORE-01**.

### ARNS-MANAGE-01 Objects

```
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

The following actions are handled in the **ARNS-MANAGE-01** specification and include the actions contained in **ARNS-CORE-01**.

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
```

### Controllers

Gets the entire `Controllers` table, including each wallet that has controller permission on this process.

Executable by anonymous users.

#### Parameters

No parameters needed.

#### Rules

- Must return entire `Controllers` object as JSON in the data field of the response notice.
- Must add `X-`forwarded tags to the response notice.

#### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Controllers"
})
```

#### Responses

**Valid `Controllers`**

```
{
  Target = msg.From,
  Action = "Controllers-Notice",
  Data = json.encode(Controllers),
  ... other forwarded tag name and value pairs
}
```

### Add-Controller

Adds a new controller into the `Controllers` table, giving them access to modify the Arweave Name's `Records`.

Executable by the process `Owner` or an authorized user in the `Controllers` table.

#### Parameters

| Name       | Type   | Description                                                                                            |
| ---------- | ------ | ------------------------------------------------------------------------------------------------------ |
| Controller | string | The controller being added to the Controllers table, e.g., iKryOeZQMONi2965nKz528htMMN_sBcjlhc-VncoRjA |

#### Rules

- Must be an authorized process `Owner` or `Controller`.
- Must specify a valid `Controller` parameter (string) as a message tag.
- The `Controller` must not already exist in the `Controllers` table.
- Must add `X-`forwarded tags to the response notice.

#### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Add-Controller",
  Controller = "{Wallet Address}"
})
```

#### Responses

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

### Remove-Controller

Removes an existing controller in the `Controllers` table, removing their access to modify the Arweave Name's `Records.

Executable by the process `Owner` or an authorized user in the `Controllers` table.

#### Parameters

| Name       | Type   | Description                                                                                                           |
| ---------- | ------ | --------------------------------------------------------------------------------------------------------------------- |
| Controller | string | The controller wallet address to remove from the Controllers table, e.g., iKryOeZQMONi2965nKz528htMMN_sBcjlhc-VncoRjA |

#### Rules

- Must be an authorized process `Owner` or `Controller`.
- Must specify a valid `Controller` parameter (string) as a message tag.
- The `Controller` must already exist in the `Controllers` table.
- Must add `X-`forwarded tags to the response notice.

#### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Remove-Controller",
  Controller = "{Wallet Address}"
})
```

#### Responses

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

### Set-Record

Updates an existing undername’s Sub-Domain in the Records table, including modifying the transaction id and time to live.

Executable by the process Owner or an authorized user in the Controllers table.

#### Parameters

| Name           | Type   | Description                                                                                 |
| -------------- | ------ | ------------------------------------------------------------------------------------------- |
| Sub-Domain     | string | The undername record to update, e.g., `@`, `ardrive`, or `dapp_ardrive`                     |
| Transaction-Id | string | The Arweave transaction ID that this undername points to.                                   |
| TTL-Seconds    | string | The time to live for this record, indicating how long an ArNS Resolver should cache it for. |

#### Rules

- Must be an authorized process `Owner` or `Controller`.
- Must specify a valid `Sub-Domain` parameter (string) as a message tag.
- Must specify a valid `Transaction-Id` parameter (string) as a message tag.
- Must specify a valid `TTL-Seconds` parameter (string) as a message tag.
- The undername’s `Sub-Domain` must already exist in the `Records` table.
- Must add `X-`forwarded tags to the response notice.

#### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Set-Record",
  ["Sub-Domain"] = "foo",
  ["Transaction-Id"] = "{Base64 URL}",
  ["TTL-Seconds"] = "900"
})
```

#### Responses

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
  }),
  ... other forwarded tag name and value pairs
}
```

### Remove-Record

Removes an existing undername’s `Sub-Domain` in the Records table.

#### Parameters

| Name       | Type   | Description                                                             |
| ---------- | ------ | ----------------------------------------------------------------------- |
| Sub-Domain | string | The undername record to remove, e.g., `@`, `ardrive`, or `dapp_ardrive` |

#### Rules

- Must be an authorized process `Owner` or `Controller`.
- Must specify a valid `Sub-Domain` parameter (string) as a message tag.
- The undername’s `Sub-Domain` must already exist in the `Records` table.
- Must add `X-`forwarded tags to the response notice.

#### Action

```
Send({
  Target = "{Process Identifier}",
  Action = "Remove-Record",
  ["Sub-Domain"] = "foo"
})
```

#### Responses

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

### State

The State handler is updated in **ARNS-MANAGE-01** to return specific information about the state of the process, including all `Records`, its current `Owner`, and all `Controllers`.

Executable by anonymous users.

#### Parameters

No parameters necessary.

#### Rules

- Must return a state object as JSON in the data field of the response notice. The state object must include:
  - Entire `Records` table.
  - Entire `Controllers` table.
  - Process `Owner`.
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
    Owner = ao.env.Process.Owner,
  }),
  ... other forwarded tag name and value pairs
}
```
