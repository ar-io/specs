# Arweave Name System Specifications (ArNSS)

## Status:

**In-Review**

## Version:

| Version | Description                                                         | Date       |
| ------- | ------------------------------------------------------------------- | ---------- |
| 1.0.0   | Initial version of the Arweave Name System Specifications overview. | 2024-09-01 |

## Abstract

These specifications outline the overview, structure, and operations of the Arweave Name System. They are used to create Arweave Name Tokens (ANTs), which are AO Processes utilized within the Arweave Name System (ArNS), and resolve them to Arweave identities and data. By defining both the standard and optional interfaces for these processes, this document aims to ensure consistency and interoperability for developers integrating ANTs into their own processes or applications.

## Motivation

The building blocks in the Arweave Name Specifications ensure that ArNS can serve a broad spectrum of use cases, from simple name resolution to complex, multi-layered digital services and AO processes. This could include custom logic or interfaces suited to particular user groups or technical requirements. Developers can utilize the open protocols and APIs in these specifications to create resolvers and Arweave Name Processes tailored to specific applications or networks.

For instance, a resolver could be built to integrate with decentralized applications on other blockchain platforms or to support enhanced security features like cryptographic validation of name queries.

By extending the utility of ArNS across different technologies, developers can further enhance its overall interoperability and flexibility.

## Arweave Name System

### Motivation

Arweave Transaction IDs, Wallet IDs, and AO Process IDs, characterized by their length and complexity, present usability challenges for everyday applications. They are cumbersome to remember, share, and are often erroneously flagged as spam by filters.

The Arweave Name System (ArNS) addresses these issues by introducing a decentralized, censorship-resistant naming system built on the Arweave network. ArNS allows for the assignment of human-readable names to Arweave resources such as dApps, web pages, digital identities, or even AO processes.

Using IO tokens for transactions and compatible with AR.IO gateway domains, ArNS simplifies access to Permaweb content, making it more navigable and user-friendly. It serves as a practical tool for enhancing user interactions on the network by replacing opaque identifiers with memorable names, thereby streamlining access and communication within the Permaweb.

### Overview

The ArNS system functions similarly to traditional DNS services, where users can purchase a name in a registry and DNS name servers resolve these names to IP addresses.

- **Permanent and Lease-Based Name Registration**: Users can purchase names permanently or lease them for a defined period, depending on their needs. The registry is stored permanently on Arweave via AO, making it immutable and globally resilient.
- **Wayback Machine for Data**: This permanent storage enables apps and infrastructure to not only read the latest state of the registry but also check any point in time in the past, creating a "Wayback Machine" of the permaweb.

- **Name Registration Process**: Users can register a name, like `ardrive`, within the ArNS Registry. Before owning a name, they must create an Arweave Name Process or Token (ANT)—an AO Compute-based token and open-source protocol used by ArNS to track ownership and control over the name. ANTs allow the owner to set a mutable pointer to any type of Permaweb data, such as a page, dApp, or file, via its Arweave Transaction ID.

- **ArNS Resolvers**: Each AR.IO gateway acts as an ArNS Name resolver. They fetch the latest state of both the ArNS Registry and its associated ANTs from an AO Compute Unit (CU) and serve this information rapidly for apps and users. AR.IO gateways will also resolve that name as one of their own subdomains, e.g., `https://ardrive.arweave.net`, and proxy all requests to the associated Arweave Transaction ID. This ensures ANTs work across all AR.IO gateways or ArNS Resolvers that support them: `https://ardrive.g8way.io/`, `https://ardrive.ar.io`, etc.

- **User Accessibility**: Users can easily reference these friendly names in their browsers, and other applications and infrastructure can build rich solutions on top of these ArNS primitives.

### Arweave Name System Registry

The ArNS Registry is a list of all registered names and their associated Arweave Name Process IDs. There are two different types of name registrations that can be utilized based on the needs of the user:

- **Lease**: A name may be leased on a yearly basis. A leased name can have its lease extended or renewed, but only up to a maximum active lease of 5 years at any time.
- **Permanent (Permabuy)**: A name may be purchased for an indefinite duration.

Registering a name requires spending IO tokens corresponding to the name’s character length and purchase type. Key registry rules embedded within the smart contract include:

- **Genesis Prices**: Set within the contract as starting conditions.
- **Dynamic Pricing**: Varies based on name length, purchase type (lease vs. buy), lease duration, and current Demand Factor.
- **Name Records**: Include a pointer to the Arweave Name Process identifier, lease end time (if applicable), and undername allocation.
- **Reassignment**: Name registrations can be reassigned from one Arweave Name Process to another.
- **Lease Extension**: Anyone with available IO Tokens can extend any name’s active lease.
- **Lease to Permanent Buy**: Anyone with available IO Tokens can convert a name’s lease to a permanent buy.
- **Undername Capacity**: Additional undername capacity can be purchased for any actively registered name.
- **Name Removal**: Name records can only be removed from the registry if a lease expires, or a permanent name is returned to the protocol.

### Arweave Name Tokens (ANTs)

Arweave Name Tokens (ANTs) are specialized AO Processes that manage the ownership, configuration, and data pointer records for names registered in ArNS. They facilitate the mapping of names to various types of Permaweb data—whether a webpage, dApp, or file—through their respective Arweave transaction IDs. They can be transferred as needed, like any other non-fungible token.

- **Process Control**: Each registered ArNS name points to the Process ID of an ANT. The owner of this ANT controls how the ArNS name functions. ANT owners can upgrade their processes as needed, or they can set the process Owner to null, rendering the process code immutable.
- **Resolution**: AR.IO Gateways and other ArNS Resolvers read the state from these registered ANTs to properly resolve and serve the data they reference.

### Undernames

Undernames allow Arweave Name owners to create multiple subdomains for a registered ArNS name, providing flexibility and extended functionality.

- **Naming Conventions**: These are configured using underscores (`_`) instead of dots (`.`), emphasizing the hierarchical relationship directly controlled by the primary name owner—ensuring that `dapp_ardrive` is unmistakably tied to `ardrive`. Unlike traditional DNS, names resembling undernames cannot be registered independently within ArNS, avoiding potential confusion and spoofing.

### Name Resolution

The Arweave Name System is designed so that registered names can be accessed across many top-level domain names, providing resiliency and censorship resistance.

- **Access**: Names registered on ArNS are accessed through standardized URLs formatted as subdomains of an ArNS Resolver, such as `https://ardrive.arweave.net`.
- **Resolver Functionality**: ArNS Resolvers interact directly with AO Compute Units to fetch the latest state of the ArNS Registry and associated ANTs, so they can serve the most up-to-date data pointers found for a registered name.
- **Gateway Integration**: AR.IO Gateways come with an ArNS Resolver out of the box, ensuring that ArNS names are consistently resolvable across any supporting gateway, extending their accessibility throughout the Arweave ecosystem. Additionally, they are incentivized to resolve registered ArNS names into actionable Arweave URLs via the AR.IO Observation and Incentive Protocol.

### ANT Registry

The Arweave Name Token Registry is a permissionless list that contains all ANTs and their current owner. While it is not required or used by the core ArNS protocols, it can be utilized by downstream applications to quickly surface name ownership information by looking up ANTs contained in this Registry.

- **Utility**: While ANTs do not have to be in this registry for resolution or other functionality like lease extension, it helps the performance of applications that need to quickly show users just the ANTs they own.
- **Updating**: ANTs can be added simply by passing along their `State` to the ANT Registry. This provides important information about the ANT, such as its current owners and controllers. This can be updated at any time, and it is recommended to update it on any transfer made by the ANT.

## Specification Framework

The Arweave Name System Specifications span across several specs that can be composed to make custom resolvers, Arweave Name Token Processes or custom variations of them.

- **ARNS-TOKEN-1**: Specifies the creation and structure of Arweave Name Tokens, focusing on token functionality and metadata.
- **ARNS-MANAGE-1**: Details the management and control features for Arweave Names, building on the core process specification.
- **ARNS-CORE-1**: The foundational AO process specification for ArNS Name resolution, detailing the core requirements for resolvers.
- **ARNS-RESOLVER-1**: Outlines the protocols for resolving and serving Arweave Names across the AR.IO network.
- **ARNS-ROUTING-1**: Describes the `ar://` schema and routing protocols for accessing permanent data and domain names via AR.IO Gateways.

Applications like `https://arns.ar.io` can leverage this framework via SDKs like the AR.IO SDK or extend them further to create a rich ecosystem of multi-functional AO processes and tooling that support the needs of ArNS name buying, management, and resolution.

### Language of Implementation

Many examples and references provided in the specification documents are written in Lua. This choice ensures compatibility with many AO Specs and Processes while making it easier for developers working within the AO and AR.IO ecosystems to implement and extend the functionalities defined by the ArNS specifications. The use of Lua also aligns with the existing infrastructure and toolsets used in the AOS environment.

However, developers are not restricted to using Lua exclusively when building new features or extending functionalities around ArNS. While Lua is recommended for direct integration with existing AO infrastructure, other programming languages can be used, provided they adhere to the protocols and specifications outlined in these documents.

### Implementations

The following implementations act as blueprints for deploying Arweave Name Processes and Tokens in AOS or other platforms. Developers can utilize these files to layer on the specification they need onto their new or existing process.

- ARNS-RESOLVER-1 https://github.com/ar-io/arns-resolver (Typescript, Node.js, Docker)
- ARNS-CORE-1 https://github.com/ar-io/ao-pilot/blob/develop/scripts/anp-resolve-01.lua (Lua)
- ARNS-MANAGE-1 https://github.com/ar-io/ao-pilot/blob/develop/scripts/anp-control-01.lua (Lua)
- ARNS-TOKEN-1 https://github.com/ar-io/ar-io-ant-process (Lua)
- AR.IO SDK https://github.com/ar-io/ar-io-sdk (Typescript)

## References

- The AO Token and Subledger Specification, which Arweave Name Tokens inherit https://hackmd.io/8DiMkhuNThOb_ooTWKqxaw#ao-Token-and-Subledger-Specification
- AO Token implementation in lua https://github.com/permaweb/aos/blob/main/blueprints/token.lua
- AR.IO White Paper https://whitepaper_ar-io.arweave.net/
