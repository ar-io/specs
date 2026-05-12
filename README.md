# AR.IO Network Specifications

Welcome to the AR.IO Network Specifications repository! This repository contains the official specifications for building on the AR.IO Network, a decentralized gateway network for accessing Arweave and other decentralized storage protocols. These specifications are designed to create a unified, interoperable, and resilient ecosystem of tools and applications that leverage the various protocols and services offered by the AR.IO Network.

## Overview

The AR.IO Network Specifications cover a range of critical functionality, from the AR.IO Name System (ArNS) to path manifest handling. By adhering to these specs, developers can build reliable, scalable applications that integrate seamlessly with the AR.IO Gateway network and the broader permaweb ecosystem.

### Why Specifications Matter

Specifications provide the foundational structure for the AR.IO Network. They ensure:

- **Consistency** : Clear guidelines for how things should work across the network.
- **Interoperability** : Ensures smooth communication between apps, tools, and services.
- **Innovation**: A solid, standardized foundation allows developers to focus on pushing boundaries.

## Specifications

### AR.IO Name System (ArNS)

The ArNS specs define how human-readable names map to content across Arweave, IPFS, and future storage protocols. The reference implementation is a set of Solana programs (`ario-arns`, `ario-ant`, `ario-core`); AR.IO Name Tokens (ANTs) are Metaplex Core NFTs. Start with the overview if you're new to the system.

- **[ARNS-OVERVIEW](./arns/arns-overview.md)**: Umbrella reference — system architecture, name lifecycle, primary names, name validation, program IDs lookup, account serialization, and event surface.
- **[ARNS-CORE-1](./arns/arns-core-1.md)**: Record format and read interface for resolving names to multi-protocol content addresses (Arweave TX IDs, IPFS CIDs).
- **[ARNS-MANAGE-1](./arns/arns-manage-1.md)**: Record creation, controller delegation, record-level ownership, and AR.IO Network integration (release, reassign, primary names).
- **[ARNS-TOKEN-1](./arns/arns-token-1.md)**: AR.IO Name Tokens as Metaplex Core NFTs with on-chain configuration, controllers, and undername records.
- **[ARNS-RESOLVER-1](./arns/arns-resolver-1.md)**: Resolver/gateway protocol — direct on-chain reads, multi-protocol routing, URI schemes (`ar://`, `ipfs://`), response headers, caching, and observation compliance.

### Path Manifest Specification

- **[PATH-MANIFEST-SCHEMA](./manifests/path-manifest-schema.md)**: JSON-based schema for mapping paths to content within Arweave.

### Reference Implementations

- [`ar-io-solana-contracts`](https://github.com/ar-io/ar-io-solana-contracts) — the four Solana programs (`ario-core`, `ario-gar`, `ario-arns`, `ario-ant`) that back the ArNS specs. The published Anchor IDLs are the authoritative source for program IDs, account layouts, instruction signatures, and event schemas.
- [`@ar.io/sdk`](https://github.com/ar-io/ar-io-sdk) — TypeScript SDK with kit-native Solana transport and (legacy) AO support. Exports `ARIO_*_PROGRAM_ID` constants mirroring the IDLs.
- [`ar-io-node`](https://github.com/ar-io/ar-io-node) — reference AR.IO Gateway implementation; ships with an ArNS resolver per ARNS-RESOLVER-1.

## Specification Principles and Practices

When writing or contributing to AR.IO network specifications, it is essential to follow these key principles and practices to ensure the quality and consistency of the specifications across the network.

### Principles:

- **Clarity**: Specifications must be clear, concise, and free from unnecessary complexity. Avoid ambiguity and ensure that each element of the specification is well-explained and understandable.
- **Modularity**: Encourage modular design by ensuring that individual specifications can be extended or composed with other specs without breaking existing functionality.
- **Extensibility**: Specifications should allow for future updates and extensions, ensuring that the network can evolve without causing disruption.
- **Consistency**: Maintain consistency in terminology, structure, and style across all specifications to ensure a coherent and accessible documentation library.
- **Minimalism**: Focus on defining only what is necessary. Keep the specifications simple, focusing on the core functionality needed without overengineering.

### Practices:

- **Precise Data Modeling**: Ensure all data structures and interactions are well-defined, with clear constraints and behavior. This includes detailing valid and invalid states for objects.
- **Testability**: Write specifications that are easy to test and validate. Provide examples and scenarios that can be used as a basis for implementation testing.
- **Backward Compatibility**: Maintain backward compatibility where possible and clearly define any breaking changes. Use semantic versioning to indicate changes to the specification.
- **Version Control**: Specifications must use clear versioning, and each new version should include a changelog outlining updates, bug fixes, or improvements.

## Versioning

All specifications follow semantic versioning, where major versions indicate breaking changes, minor versions add functionality in a backward-compatible manner, and patch versions include backward-compatible bug fixes. Each spec includes a version history that tracks changes and new features.

For example:

- **v1.0.0**: Major changes or breaking updates.
- **v0.2.0**: New features, backward-compatible changes.

## Contributing

We welcome contributions! Here’s how you can get involved:

- **Review Existing Specs**: Explore the `arns/` and `manifests/` folders to get familiar with the structure.
- **Open Issues**: If you spot inconsistencies or have suggestions, open an issue here.
- **Submit a Pull Request**: Fork the repo, create a branch, and submit a pull request. Ensure your changes follow the principles outlined above.

Check out our [contributing] file for more detailed guidelines.

[contributing]: ./CONTRIBUTING.md
