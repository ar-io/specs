# ARNS-RESOLVER-01

## Overview

An ArNS Resolver is responsible for resolving registered ArNS Names to their corresponding data and process ID. The **ARNS-RESOLVER-01** specification includes the set of patterns and protocols to retrieve the latest state of names and their mapped processes from AO and serve them to users.

## Resolver Operations

### 1. Pulling Records from the ArNS Registry

The resolver must pull all Records from the ArNS Registry by connecting to an AO Compute Unit. This provides the latest details such as:

- The associated process.
- Lease or permabuy status.
- The number of undernames allowed.
- The expiration date of the name (if applicable).

### 2. Requesting the State of the Process

For each Record, the resolver must request the State of its associated process. This includes important details such as:

- Ownership information.
- Data pointers.
- Time-to-live (TTL) settings for the given ArNS name.

If the process does not respond to the State request, the resolver will be unable to fully resolve its Records.

### 3. Local Storage of Data

The resolver should store all retrieved information locally to facilitate easy lookups by users or downstream infrastructure, such as an AR.IO Gateway.

## Resolution Protocols

The resolver must adhere to the following protocols to correctly identify, resolve, and serve ArNS names:

- **Expired Names**: Must not serve names that have expired from the ArNS Registry.
- **Undernames**:
  - Must not serve undernames (i.e., subdomains) beyond the amount purchased.
  - Must not serve undernames that exceed the maximum DNS label limit.
  - The maximum length of each label is 63 characters, and a full domain name can have a maximum of 253 characters.
- **Root Record**: The Record `"@"` is used for the root of the ArNS Name. All undernames for a given name must be sorted alphabetically, with the root `@` being at the top.
- **Regex Compliance**: Must not serve undernames that don't match the ArNS standard regex pattern.
- **Caching**:
  - Must cache each undername for the time-to-live (TTL) associated with it.
  - The minimum TTL is 900 seconds (15 minutes).
- **Response Headers**: Must respond to requests with additional headers, `x-arns-process-id` and `x-arns-resolved-id`, which indicate the process ID and the ID that the name is resolving to. For example:
  - `x-arns-process-id: tVBd6CHJD1L46sjFBb-Vmr2WHu6FzNyWKJnO42RT_lc`
  - `x-arns-resolved-id: wHt7VINjV_oKGtb-f7yB-oajzNb9qTKzkVoFoXDFStI`

## Performance and Compliance

The resolver is expected to perform the above actions in a timely manner to best serve users and/or to meet the standards defined in the AR.IO Observation and Incentive protocol.

For example, by default, the AR.IO ArNS Resolver fetches the ArNS Registry every 15 minutes, ensuring timely updates of new name additions and expirations. It is the responsibility of each ArNS Name owner to ensure their ANT code is functional and performant enough to be resolved.