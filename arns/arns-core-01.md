# ARNS-CORE-01

## Overview

The core Arweave Name specification for an AO process contains the specific details required by AR.IO Gateways to resolve the corresponding ArNS names and their Arweave Transaction IDs. This is the most basic building block and is primarily meant to be layered on top of or below other functionality.

## Specification Requirements

The **ARNS-CORE-01** Specification includes the following requirements for valid, resolvable Records:

**Records Table**:

- Must have a Records table containing subDomains and their associated transactionId and ttlSeconds.
- The Records object should be sorted alphabetically.
- Each Record subdomain must be a string.
- Each Record must include a transactionId, which is where this subdomain will point to.
- Each Record must include a ttlSeconds value, which is the suggested time for caching by ArNS Name Resolvers.
- The ttlSeconds value must not be less than 900 seconds.
- Records should have a default Record, `@`, which points to an Arweave Transaction ID.
- `UyC5P5qKPZaltMmmZAWdakhlDXsBF6qmyrbWYFchRTk` can be used as a placeholder Transaction ID.
- The Record subdomain plus its ArNS name must not be longer than 63 characters.
- The Record subdomain must pass the following regular expression: `RegExp("^[a-zA-Z0-9_-]+$")`.

**Handlers**:

- Must include handlers to read a single Record and multiple Records.
- Must include a handler to read the State of the ANP, which includes all Records and the current Owner.

## Resolvability

Arweave Name owners can configure their process code and parameters as they wish, but all of the above rules are also enforced at each ArNS Name Resolver that is part of AR.IO Gateways.

Primarily, any Arweave Name that is to be resolved by the AR.IO Network must have an active registration in the ArNS Registry. Additionally, Records within the Arweave Name process must be valid, or else ArNS Resolvers or AR.IO Gateways may not resolve them.

ANTs may not be resolvable under the following scenarios:

- Poor performing code.
- Invalid records or malformed state.
- Broken handlers, incorrect handler responses, or conflicting process code.
- Subdomains exceeding the amount paid for (registered) within the ArNS Registry.

## Objects

The **ARNS-CORE-01** specification includes all the objects needed to support ArNS Name resolution.

### ARNS-CORE-01 Objects

```
Records = Records or {
    ["@"] = {
        transactionId = "UyC5P5qKPZaltMmmZAWdakhlDXsBF6qmyrbWYFchRTk",
        ttlSeconds = 3600
    }
}
```

## Handler Action Map, Parameters, and Responses

The following actions are handled in the **ARNS-CORE-01** specification:

```
ARNSCoreSpecActionMap = {
    -- read actions
    Record = "Record",
    Records = "Records",
    State = "State",
}
```

### Record

Retrieves the ID and time to live (seconds) for an undername contained in the Records table. Executable by anonymous users.

#### Parameters

| Name       | Type   | Description                                                               |
| ---------- | ------ | ------------------------------------------------------------------------- |
| Sub-Domain | string | The undername record to retrieve, e.g., `@`, `ardrive`, or `dapp_ardrive` |

#### Rules

- Must specify a valid Sub-Domain parameter (string) as a message tag.
- The name must exist in the Records table.
- Must add X-forwarded tags to the response notice.

#### Action

```
Send({
    Target = "{Process Identifier}",
    Action = "Record",
    ["Sub-Domain"] = "foo"
})
```

#### Responses

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

### State

Returns specific information about the state of the **ARNS-CORE-01** process, including all Records and its current Owner. Executable by anonymous users.

#### Parameters

No parameters necessary.

#### Rules

- Must return a state object as JSON in the data field of the response notice.
- The state object must include:
  - The entire Records table as JSON.
  - The entire Controller table as JSON.
  - The current process Owner.
- Must add X-forwarded tags to the response notice.

#### Action

```
Send({
    Target = "{Process Identifier}",
    Action = "State"
})
```

#### Responses

**Valid State**

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