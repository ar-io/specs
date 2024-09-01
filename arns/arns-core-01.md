# ARNS-CORE-1

## Status:

**Draft**

## Version:

| Version | Description                                           | Date       |
| ------- | ----------------------------------------------------- | ---------- |
| 1.0.0   | Initial version of the **ARNS-CORE-1** specification. | 2024-09-01 |

## Abstract

The **ARNS-CORE-1** specification defines the foundational processes required for resolving Arweave Name System (ArNS) names to their corresponding Arweave Transaction IDs and Owners. It provides the essential framework upon which other functionalities and extensions are built, ensuring consistent name resolution across the AR.IO network.

## Motivation

The **ARNS-CORE-1** specification provides the essential building blocks for resolving Arweave Names within the Arweave Name System (ArNS). It defines the minimal structure and processes needed to map human-readable names to Arweave Transaction IDs, ensuring consistent and reliable name resolution across AR.IO gateways.

This specification establishes a standardized record format and a set of core handlers that are crucial for any ArNS implementation. By adhering to these foundational requirements, developers can build additional functionalities on top of a stable and predictable resolution framework, ensuring interoperability and ease of use within the Arweave ecosystem.

### Language of Implementation

All examples and code snippets in this specification are written in Lua. This choice ensures compatibility with the AO Processes and the Arweave ecosystem.

However, developers are not restricted to using Lua exclusively when building new features or extending functionalities around ArNS. While Lua is recommended for direct integration with existing infrastructure, other programming languages can be used, provided they adhere to the protocols and specifications outlined in this document.

## Specification

### Overview

The **ARNS-CORE-1** Specification includes the following requirements for valid, resolvable Records:

**Records Table**:

- Must have a `Records` table containing `subDomains` and their associated `transactionId` and `ttlSeconds`.
- The `Records` object should be sorted alphabetically.
- Each Record subdomain must be a string.
- Each Record must include a `transactionId`, which is where this subdomain will point to.
- Each Record must include a `ttlSeconds` value, which is the suggested time for caching by ArNS Name Resolvers.
- The `ttlSeconds` value must not be less than 900 seconds.
- `Records` must have a default Record, `@`, which points to an Arweave Transaction ID.
- `UyC5P5qKPZaltMmmZAWdakhlDXsBF6qmyrbWYFchRTk` can be used as a placeholder Transaction ID.
- The Record subdomain plus its ArNS name must not be longer than 63 characters.
- The Record subdomain must pass the following regular expression: `RegExp("^[a-zA-Z0-9_-]+$")`.

**Handlers**:

- Must include handlers to read a single `Record` and multiple `Records`.
- Must include a handler to read the `State` of the ANP, which includes all `Records` and the current `Owner`.

**Resolvability**:

Arweave Name owners can configure their process code and parameters as they wish, but all of the above rules are also enforced at each ArNS Name Resolver that is part of AR.IO Gateways.

Primarily, any Arweave Name that is to be resolved by the AR.IO Network must have an active registration in the ArNS Registry. Additionally, Records within the Arweave Name process must be valid, or else ArNS Resolvers or AR.IO Gateways may not resolve them.

Arweave Names may not be resolvable under the following scenarios:

- Poor performing code.
- Invalid records or malformed state.
- Broken handlers, incorrect handler responses, or conflicting process code.
- Subdomains exceeding the amount paid for (registered) within the ArNS Registry.

### Objects

The **ARNS-CORE-1** specification includes all the objects needed to support ArNS Name resolution.

#### ARNS-CORE-1 Objects

```
Records = Records or {
    ["@"] = {
        transactionId = "UyC5P5qKPZaltMmmZAWdakhlDXsBF6qmyrbWYFchRTk",
        ttlSeconds = 3600
    }
}
```

### Handlers

#### Action Map

The following actions are handled in the **ARNS-CORE-1** specification:

```
ARNSCoreSpecActionMap = {
    -- read actions
    Record = "Record",
    Records = "Records",
    State = "State",
}
```

#### Record

Retrieves the ID and time to live (seconds) for an undername contained in the `Records` table. Executable by anonymous users.

##### Parameters

| Name       | Type   | Description                                                               |
| ---------- | ------ | ------------------------------------------------------------------------- |
| Sub-Domain | string | The undername record to retrieve, e.g., `@`, `ardrive`, or `dapp_ardrive` |

##### Rules

- Must specify a valid `Sub-Domain` parameter (string) as a message tag.
- The name must exist in the `Records` table.
- Must add `X-`forwarded tags to the response notice.

##### Action

```
Send({
    Target = "{Process Identifier}",
    Action = "Record",
    ["Sub-Domain"] = "foo"
})
```

##### Responses

**Invalid Record**

```
{
    Target = msg.From,
    Action = "Invalid-Record-Notice",
    Data = nameRes,
    Error = "Record-Error",
    ["Message-Id"] = msg.Id,
}
```

**Valid Record**

```
{
    Target = msg.From,
    Action = "Record-Notice",
    Name = msg.Tags["Sub-Domain"],
    Data = json.encode(Records[name]),
    ... other forwarded tag name and value pairs
}
```

#### State

Returns specific information about the state of the **ARNS-CORE-1** process, including all `Records` and its current `Owner`. Executable by anonymous users.

##### Parameters

No parameters necessary.

##### Rules

- Must return a state object as JSON in the data field of the response notice.
- The state object must include:
  - The entire `Records` table as JSON.
  - The current process `Owner`.
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
        Owner = ao.env.Process.Owner,
    }),
    ... other forwarded tag name and value pairs
}
```
