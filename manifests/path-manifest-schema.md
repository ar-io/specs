# PATH-MANIFEST-SCHEMA

## Status

**In-Review**

## Version

| Version | Description                                                    | Date       |
| ------- | -------------------------------------------------------------- | ---------- |
| 0.1.0   | Initial version of the **PATH-MANIFEST-SCHEMA** specification. | 2019-09-30 |
| 0.2.0   | Added fallback mechanism for unresolved paths.                 | 2024-08-01 |

## Abstract

The Arweave Path Manifest Schema defines a standard for creating path manifests within the Arweave blockchain protocol. These manifests are JSON objects that map subpaths to specific content, represented by Arweave transaction IDs. This schema allows for efficient content resolution, improved user experience, and ensures consistency across different implementations. Version 0.2.0 introduces a fallback mechanism for handling unresolved paths.

## Motivation

The **PATH-MANIFEST-SCHEMA** specification provides a structured method to map paths to content stored on the Arweave blockchain. By defining a standardized schema, it ensures that different applications and services can consistently interpret and serve content based on these mappings.

Key benefits include:

1. **Efficient Content Resolution**: The schema provides a clear mapping between subpaths and Arweave transaction IDs, enabling fast and accurate retrieval of content.
2. **Enhanced User Experience**: By defining a default path and fallback behavior, the schema allows gateways and clients to provide a more intuitive experience when accessing content on the Arweave network.
3. **Consistency Across Implementations**: Standardizing the structure of path manifests promotes interoperability, ensuring content is served consistently across the Arweave ecosystem.

This manifest schema enables gateways to serve content in a manner similar to web servers with specified routes, allowing developers to host web content directly from an Arweave Gateway.

## Specification

### Schema

Path manifests are JSON objects with the following keys.

| Field       | Mandatory? | Type   | Description                                                                                                                                                    |
| ----------- | ---------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| manifest    | ✓          | string | The manifest type identifier, this MUST be `arweave/paths`.                                                                                                    |
| version     | ✓          | string | The manifest specification version, currently `"0.2.0"`. This will be updated with future updates according to semver.                                         |
| index       |            | object | The behavior gateways SHOULD follow when the manifest is accessed directly.                                                                                    |
| index.path  |            | string | The default path to load. If defined, the field MUST reference a key in the paths object (it MUST NOT reference a transaction ID directly).                    |
| index.id    |            | string | A specific Arweave transaction ID that will be served when the manifest itself is accessed without any additional path.                                        |
| fallback    |            | object | Defines an Arweave transaction ID to use if a requested path cannot be resolved.                                                                               |
| fallback.id |            | string | The transaction ID to resolve to when a path cannot be found.                                                                                                  |
| paths       | ✓          | object | The path mapping between subpaths and the content they resolve to. The object keys represent the subpaths, and the values tell us which content to resolve to. |
| paths[id]   | ✓          | string | The transaction ID to resolve to for the given path.                                                                                                           |

A path manifest transaction MUST NOT contain any data other than this JSON object.

Manifests are uploaded to Arweave like any other data item, with a specific content type tag required for proper resolution by gateways. Tags must be attached to the manifest at the time of upload.

The `Content-Type` tag for manifest files MUST be `application/x.arweave-manifest+json`; users MAY add other arbitrary user-defined tags.

### Manifest Data

Being a JSON object, several attributes make up the structure of a manifest. The JSON object must be fully defined and uploaded to Arweave as a data item.

#### manifest

The manifest attribute serves as an additional validation layer. It must have the value arweave/paths for a gateway to resolve the manifest.

```json
"manifest": "arweave/paths"
```

#### version

The version attribute defines the version of the manifest schema.

```json
"version": "0.2.0"
```

#### index

The index attribute defines the base, or 'starting' data item. It acts like the / endpoint on a website. When resolving the manifest with no additional path, this is the data item returned. The index can either reference a path within the manifest or directly reference an Arweave transaction ID.

```json
"index": {
    "id": "cG7Hdi_iTQPoEYgQJFqJ8NMpN4KoZ-vH_j7pG4iP7NI"
}
```

or

```json
"index": {
    "path": "index.html"
}
```

#### fallback

The fallback attribute defines an Arweave transaction ID for the resolver to use if it fails to correctly resolve a requested path. This can act as a 404 page if a user requests manifest/non-existent-page.

```json
"fallback": {
    "id": "iXo3LSfVKVtXUKBzfZ4d7bkCAp6kiLNt2XVUFsPiQvQ"
}
```

#### paths

The paths attribute defines the URL paths that a manifest can resolve to. If a user navigates to manifest/index.html, the resolver looks for index.html in the paths object and returns the corresponding ID (`cG7Hdi_iTQPoEYgQJFqJ8NMpN4KoZ-vH_j7pG4iP7NI`).

```json
"paths": {
    "index.html": {
        "id": "cG7Hdi_iTQPoEYgQJFqJ8NMpN4KoZ-vH_j7pG4iP7NI"
    },
    "404.html": {
        "id": "iXo3LSfVKVtXUKBzfZ4d7bkCAp6kiLNt2XVUFsPiQvQ"
    },
    "js/style.css": {
        "id": "3zFsd7bkCAUtXUKBQ4XiPiQvpLVKfZ6kiLNt2XVSfoV"
    },
    "css/style.css": {
        "id": "sPiQvpAUXLVK3zF6iXSfo7bkCVQkiLNt24dVtXUKBfZ"
    },
    "css/mobile.css": {
        "id": "fZ4d7bkCAUiXSfo3zFsPiQvpLVKVtXUKB6kiLNt2XVQ"
    },
    "assets/img/logo.png": {
        "id": "or0_fRYFcQYWh-QsozygI5Zoamw_fUsYu2w8_X1RkYZ"
    },
    "assets/img/icon.png": {
        "id": "0543SMRGYuGKTaqLzmpOyK4AxAB96Fra2guHzYxjRGo"
    }
}
```

### Example Manifest

```json
{
  "manifest": "arweave/paths",
  "version": "0.2.0",
  "index": {
    "path": "index.html"
  },
  "fallback": {
    "id": "iXo3LSfVKVtXUKBzfZ4d7bkCAp6kiLNt2XVUFsPiQvQ"
  },
  "paths": {
    "index.html": {
      "id": "cG7Hdi_iTQPoEYgQJFqJ8NMpN4KoZ-vH_j7pG4iP7NI"
    },
    "404.html": {
      "id": "iXo3LSfVKVtXUKBzfZ4d7bkCAp6kiLNt2XVUFsPiQvQ"
    },
    "js/style.css": {
      "id": "3zFsd7bkCAUtXUKBQ4XiPiQvpLVKfZ6kiLNt2XVSfoV"
    },
    "css/style.css": {
      "id": "sPiQvpAUXLVK3zF6iXSfo7bkCVQkiLNt24dVtXUKBfZ"
    },
    "css/mobile.css": {
      "id": "fZ4d7bkCAUiXSfo3zFsPiQvpLVKVtXUKB6kiLNt2XVQ"
    },
    "assets/img/logo.png": {
      "id": "or0_fRYFcQYWh-QsozygI5Zoamw_fUsYu2w8_X1RkYZ"
    },
    "assets/img/icon.png": {
      "id": "0543SMRGYuGKTaqLzmpOyK4AxAB96Fra2guHzYxjRGo"
    }
  }
}
```
