# AR.IO Network specifications.

Welcome to the AR.IO Network Specifications repository! This repository contains the official specifications for building on the AR.IO Network, a decentralized gateway protocol for interacting with Arweave. These specifications are designed to create a unified, interoperable, and resilient ecosystem of tools and applications built on top of Arweave.

## Overview

The AR.IO Network Specifications cover a range of critical functionality, from the Arweave Name System (ArNS) to routing and path manifest handling. By adhering to these specs, developers can build reliable, scalable applications that integrate seamlessly with the AR.IO Gateway network and the broader Arweave ecosystem.

### Why Specifications Matter

Specifications provide the foundational structure for the AR.IO Network. They ensure:

- **Consistency** : Clear guidelines for how things should work across the network.
- **Interoperability** : Ensures smooth communication between apps, tools, and services.
- **Innovation**: A solid, standardized foundation allows developers to focus on pushing boundaries.

## Specifications

### Arweave Name System (ArNS)

- **ARNS-CORE-01**: Core specification for resolving Arweave Names to transaction IDs.
- **ARNS-MANAGE-01**: Control and management features for Arweave Name Tokens (ANTs).
- **ARNS-TOKEN-01**: Adds token functionality, including transferability and metadata.

### Routing and URI Handling

- **ARNS-ROUTING-01**: Defines the AR.IO Wayfinder Protocol for routing Arweave content.

### Path Manifest Specification

- **PATH-MANIFEST-SCHEMA**: JSON-based schema for mapping paths to content within Arweave.

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

- **Review Existing Specs**: Explore the /specs folder to get familiar with the structure.
- **Open Issues**: If you spot inconsistencies or have suggestions, open an issue here.
- **Submit a Pull Request**: Fork the repo, create a branch, and submit a pull request. Ensure your changes follow the principles outlined above.

Check out our [contributing] file for more detailed guidelines.

[contributing]: ./CONTRIBUTING
