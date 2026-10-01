<!--
  - Licensed to the Apache Software Foundation (ASF) under one
  - or more contributor license agreements.  See the NOTICE file
  - distributed with this work for additional information
  - regarding copyright ownership.  The ASF licenses this file
  - to you under the Apache License, Version 2.0 (the
  - "License"); you may not use this file except in compliance
  - with the License.  You may obtain a copy of the License at
  -
  -   http://www.apache.org/licenses/LICENSE-2.0
  -
  - Unless required by applicable law or agreed to in writing,
  - software distributed under the License is distributed on an
  - "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  - KIND, either express or implied.  See the License for the
  - specific language governing permissions and limitations
  - under the License.
  -->

Extension Types
===============

An extension type annotates a schema node with application-level semantics
that are layered on top of the node's ordinary Parquet type. The extension is
identified by a namespaced name and optional parameters stored in
`SchemaElement.extension_type`.

The node's ordinary Parquet type is called its **storage type**. It consists of
the node's physical type, `LogicalType` and `ConvertedType` annotations, type
parameters, repetition and subtree structure. An extension type never changes
the storage type or how values are decoded. A reader that does not recognize an
extension type reads the node exactly as if the extension were absent.

Extension types complement the logical types defined in
[LogicalTypes.md](LogicalTypes.md). A logical type replaces the interpretation
of its physical type and is part of the core format. An extension type refines
the interpretation of an otherwise complete storage type and can be defined
without changing the format.

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT" and "MAY" in this
document are to be interpreted as described in [RFC 2119][rfc2119].

[rfc2119]: https://www.rfc-editor.org/rfc/rfc2119

## Extension Type Descriptor

### Metadata

```thrift
struct ExtensionType {
  1: required string name;
  2: optional string parameters;
}

struct SchemaElement {
  ...
  11: optional ExtensionType extension_type;
}
```

Readers that do not understand `extension_type` skip the field, as with any
unknown Thrift field, and read the storage type. No new `LogicalType` or
`ConvertedType` member is added.

### Names

* A name consists of two or more components separated by `.`. Each component
  MUST match `[A-Za-z0-9][A-Za-z0-9_-]*`. The complete name MUST NOT exceed 256
  bytes. Names are compared byte-wise, case-sensitively, without normalization.
* The leading components identify the authority that specifies the type:
  * names beginning with `parquet.` are reserved for canonical extension types
    specified by the Parquet project;
  * other authorities use a prefix they control, such as a reverse domain name
    (`com.example.ipv6`) or an established project prefix (`geoparquet.*`).
* A name identifies a specification. It is not a network location, a package or
  a class name.

### Parameters

* When present, `parameters` MUST be a UTF-8 encoded JSON object. An absent
  value and an empty string are equivalent and mean that there are no
  parameters. Neither is generally equivalent to `{}`; each type defines its own
  defaults.
* Each extension type specifies its keys, defaults and invalid forms. Duplicate
  keys and non-finite numbers are invalid. Unknown keys MAY be ignored only if
  the type's specification declares them ignorable.
* An incompatible change to a type's meaning MUST use a new name.

### Placement

* An extension type MAY annotate a `required` or `optional` leaf or group node,
  including a node that has a `LogicalType`. For example, an extension type may
  annotate a `TIMESTAMP`, a `STRING` or a `LIST` node.
* A node has at most one extension type.
* An extension type MUST define per-value semantics that remain valid when whole
  values are selected or reordered.

## Compatibility

### Readers

| Case | Behavior |
|---|---|
| No `extension_type` | Unchanged. |
| Unrecognized name | Read the storage type. Implementations MAY expose the name and parameters to callers as an opaque descriptor. Implementations MUST NOT fail because of the extension type. |
| Recognized and valid | Implementations MAY expose the extension type, and MAY rely on its documented constraints without validating every value. |
| Recognized but unsupported, malformed, or with invalid values detected | Implementations MUST NOT present the node or the affected values as valid instances of the extension type. They MAY offer explicit access to the storage values. |

Ignoring an extension type preserves decoding of the storage values and the
existing meaning of storage-type metadata. It does not preserve the extension
type's meaning, and existing writers are not expected to preserve
`extension_type` when rewriting a file.

### Statistics

Extension types MUST NOT change the meaning of `Statistics`, `ColumnIndex`,
`ColumnOrder`, Bloom filters, size statistics or encodings. They continue to
describe the storage leaf columns according to the storage types.

Extension-specific statistics are not defined by this specification.

## Governance

* **Canonical extension types.** Parquet can specify canonical extension types
  with names beginning with `parquet.`. The process for specifying them will be
  proposed separately.
* **Third-party extension types** are specified by their own authorities and
  need no registration. Reading a file never requires an online registry.
* **Extension or core type.** A type is suitable as an extension type when its
  storage type is valid on its own and its semantics can be defined over the
  stored values. Types that change how values are decoded, or that need new core
  statistics, remain core logical types.

## Security Considerations

* Names and parameters are untrusted input. Implementations MUST NOT execute
  them or deserialize them with mechanisms that can execute code, and MUST NOT
  resolve names over the network.
* Readers SHOULD bound the size of parameters and the effort spent parsing
  them.
* Extension types are stored in the footer. When the footer is not encrypted,
  they are visible even for encrypted columns.
