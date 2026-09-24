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

The node's ordinary Parquet type is called its **backing type**. It consists of
the node's physical type, `LogicalType` and `ConvertedType` annotations, type
parameters, repetition and subtree structure. An extension type never changes
the backing type or how values are decoded. A reader that does not recognize an
extension type reads the node exactly as if the extension were absent.

Extension types complement the logical types defined in
[LogicalTypes.md](LogicalTypes.md). A logical type replaces the interpretation
of its physical type and is part of the core format. An extension type refines
the interpretation of an otherwise complete backing type and can be defined
without changing the format.

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT" and "MAY" in this
document are to be interpreted as described in [RFC 2119][rfc2119].

[rfc2119]: https://www.rfc-editor.org/rfc/rfc2119

## Terminology

* **Backing schema**: the schema obtained by removing all extension type
  descriptors. It MUST be a valid Parquet schema.
* **Extension-aware implementation**: an implementation that understands the
  `extension_type` field, whether or not it recognizes a particular name.
* **Type-aware implementation**: an implementation that recognizes a particular
  extension type name and supports its parameters and backing representations.

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

### Names

* A name consists of two or more components separated by `.`. Each component
  MUST match `[A-Za-z0-9][A-Za-z0-9_-]*`. The complete name MUST NOT exceed 256
  bytes. Names are compared byte-wise, case-sensitively, without normalization.
* Names beginning with `parquet.` are reserved for the canonical extension types
  defined in this document.
* Other extension types SHOULD use a prefix controlled by their owner, such as a
  reverse domain name (`com.example.ipv6`) or an established project prefix.
* A name identifies a specification. It is not a network location, a package or
  a class name. Implementations MUST NOT fetch resources or load code because a
  file contains a particular name.

### Parameters

* `parameters` is UTF-8 text interpreted according to the named type's
  specification. An absent value and an empty string are equivalent. Neither is
  generally equivalent to `{}` or JSON `null`; each type defines its own
  defaults.
* Each extension type MUST specify its parameter encoding, defaults, invalid
  forms and the handling of unknown content.
* Canonical extension types with parameters MUST use a UTF-8 JSON object with a
  documented schema. Duplicate keys and non-finite numbers are invalid. Unknown
  keys MAY be ignored only if the type's specification declares them ignorable.
* An incompatible change to a type's meaning MUST use a new name.

### Placement

* An extension type MAY annotate a `required` or `optional` data-bearing leaf or
  group, including a node that has a `LogicalType`.
* An extension type MUST NOT annotate the schema root, a `repeated` field, or the
  repeated middle level of a `LIST` or `MAP`.
* A node has at most one extension type. Descendant nodes MAY have their own
  extension types if the outer type's specification allows it.
* An extension type MUST define per-value semantics that remain valid when whole
  values are selected or reordered. Its validity MUST NOT depend on the external
  field name, row position or field id.

## Compatibility

### Readers

| Case | Behavior |
|---|---|
| No `extension_type` | Unchanged. |
| Unrecognized name | Read the backing type. Implementations SHOULD expose the name and parameters to callers as an opaque descriptor. Implementations MUST NOT fail only because the name is unrecognized. |
| Recognized, with valid parameters and backing schema | Implementations MAY expose the extension type, and MAY rely on its documented value constraints without validating every value. |
| Recognized, but unsupported, malformed, or invalid values detected | Implementations MUST NOT present the node or the affected values as valid instances of the extension type. They MAY offer explicit access to the backing values. |

Readers that do not understand `extension_type` skip the field, as with any
unknown Thrift field, and read the backing type. The meaning of an extension is
lost, but the backing values are not.

### Writers

A writer that emits `extension_type` MUST:

1. Write data and metadata that are valid for the backing schema, including the
   `ConvertedType` annotations that the backing logical types require.
2. Emit extension types only in allowed positions, with valid parameters.
3. Ensure that the written values satisfy the extension type's constraints.
4. Keep any parallel representation of the same information (for example an
   embedded Arrow schema) consistent with the extension type.

### Rewriting and Schema Operations

Existing writers are not required to preserve `extension_type`; a rewrite that
drops it produces a valid file of the backing type. Extension-aware writers
MUST follow these rules:

* Selecting, reordering or losslessly copying whole values MAY preserve the
  extension type.
* Projecting part of an annotated group, changing the backing schema, or
  changing values MUST either revalidate the result with a type-aware
  implementation or drop the extension type. An unchanged backing schema alone
  does not show that changed values still satisfy the type's constraints.
* When combining data from several files, extension types MUST match exactly
  (name, parameters and backing schema) unless a type-aware implementation
  defines their equivalence. Otherwise the operation MUST be rejected or MUST
  explicitly fall back to the backing type.

### Statistics

Extension types MUST NOT change the meaning of `Statistics`, `ColumnIndex`,
`ColumnOrder`, Bloom filters, size statistics or encodings. They continue to
describe the backing leaf columns according to the backing types.

A type-aware reader MAY use these statistics for predicates on extension values
only through a translation that cannot exclude matching data. A reader MUST
NOT assume that the semantic order or equality of an extension type matches
that of its backing type unless the type's specification says so.

An extension type's specification MAY state that recognized, valid instances
guarantee a property that is otherwise expressed by an optional statistic. Such
a guarantee only relaxes the handling of an absent statistic where the core
specification explicitly allows it (see `nan_count` and `nan_counts` in
`parquet.thrift`). Written statistics MUST still be correct and MUST still be
written wherever the core specification requires them.

## Security Considerations

* Names and parameters are untrusted input. Implementations MUST NOT execute
  them or deserialize them with mechanisms that can execute code, and MUST NOT
  resolve names over the network.
* Implementations SHOULD bound the size and nesting depth of parameters they
  parse, and SHOULD validate parameters that control allocation before
  allocating.
* Extension types are stored in the footer. When the footer is not encrypted,
  they are visible even for encrypted columns.

## Canonical Extension Types

Canonical extension types use names beginning with `parquet.` and are
specified in this section. Each canonical extension type is proposed, discussed
and voted on separately, and requires two interoperable implementations as
described in [CONTRIBUTING.md](CONTRIBUTING.md). A specification MUST define:

* the name and the parameter schema;
* the allowed backing schemas, including required child names;
* the value constraints;
* the handling of invalid and unsupported instances;
* the meaning of the backing statistics for the type;
* its mapping to Apache Arrow, or the reason why there is none.

Once adopted, a canonical extension type is stable. Revisions MAY only add
parameters that its specification declares ignorable. Any other change requires
a new name.

### VECTOR

`parquet.vector` annotates fixed-length sequences of non-null, finite numeric
elements, such as embeddings.

#### Parameters

The parameters MUST be a JSON object with exactly one member:

```json
{"num_elements": 768}
```

* `num_elements` is an integer in the range 1 to 2147483647 (inclusive). It is
  the number of elements in every non-null vector.
* Absent or empty parameters, other members, and values that are not integers
  in range are invalid.

Readers SHOULD check `num_elements` against their resource limits before
allocating memory based on it.

#### Backing Schema

`parquet.vector` annotates the outer group of the canonical three-level `LIST`
structure:

```
<vector-repetition> group <name> (LIST) {
  repeated group list {
    required <element-type> element;
  }
}
```

* The outer group MUST have `LogicalType` `LIST` and `ConvertedType` `LIST`, and
  carries the extension type. Its repetition MUST be `optional` or `required` and
  determines whether a vector may be null.
* The middle level MUST be a repeated group named `list` with exactly one field
  named `element`.
* `element` MUST be a `required` primitive field. Group elements are not
  allowed.
* The two-level `LIST` structures accepted by the `LIST` backward-compatibility
  rules are not valid for `parquet.vector`.

The allowed element types are:

| Physical type | Logical type |
|---|---|
| `BOOLEAN`, `FLOAT`, `DOUBLE` | none |
| `INT32`, `INT64` | none, `INTEGER` or `DECIMAL` |
| `FIXED_LEN_BYTE_ARRAY` | `FLOAT16` or `DECIMAL` |

An element annotated only with the corresponding `ConvertedType` (`INT_8` to
`UINT_64`, or `DECIMAL`) is treated as having the matching `LogicalType`. Other
element types are not allowed.

A node that does not satisfy these rules is not a valid `parquet.vector`, but
remains readable as its backing `LIST`.

#### Values

* Every non-null vector MUST contain exactly `num_elements` elements.
* Elements MUST NOT be null. This follows from `element` being `required`.
* Every `FLOAT`, `DOUBLE` and `FLOAT16` element MUST be finite: NaN, positive
  infinity and negative infinity are not allowed. Both signed zeros are allowed.
* A vector MAY be null only if the outer group is `optional`.

For example, with `num_elements = 3` and a `required float element`:

| Value | Valid |
|---|---|
| `[1.0, 2.0, 3.0]` | yes |
| `null` | only if the outer group is `optional` |
| `[]`, `[1.0, 2.0]` | no: wrong number of elements |
| `[1.0, NaN, 3.0]`, `[1.0, +Infinity, 3.0]` | no: non-finite element |

Writers that cannot guarantee these constraints MUST write a plain `LIST`
instead. A type-aware reader that detects a violation MUST NOT return the
affected vector as a valid vector.

#### Encoding and Statistics

Definition and repetition levels, encodings, compression, encryption and page
boundaries are those of the backing `LIST`. A vector MAY span data pages.

Statistics, column indexes, Bloom filters and size statistics are those of the
`element` leaf column and describe individual elements. Vectors have no defined
sort order, and no statistics over whole vectors are defined. Because elements
are not null, nulls counted for the leaf column represent null vectors.

For floating-point elements, `nan_count` and every entry of `nan_counts` MUST be
zero when written. The rules of `ColumnOrder` that require `nan_count` to be
written still apply. A reader that recognizes `parquet.vector` and has
validated its parameters and backing schema MAY assume that the column contains
no NaN values when `nan_count` or `nan_counts` is absent.

#### Mapping to Apache Arrow

This section is informative.

* A `parquet.vector` maps to an Arrow `FixedSizeList` of `num_elements` values
  with a non-nullable child field of the corresponding element type. The
  nullability of the Arrow field follows the repetition of the outer group.
* An Arrow `FixedSizeList` can be written as `parquet.vector` only if its child
  contains no nulls, its element type is allowed, and all floating-point values
  are finite. Otherwise it is written as a plain `LIST`.

#### Example

A nullable vector of 768 `FLOAT` elements:

```
optional group embedding (LIST) {
  repeated group list {
    required float element;
  }
}
```

The `SchemaElement` of `embedding` has `logicalType` `LIST`, `converted_type`
`LIST` and:

```
extension_type = ExtensionType {
  name: "parquet.vector"
  parameters: "{\"num_elements\":768}"
}
```

A reader that does not recognize `parquet.vector` reads a nullable list of
non-null floats with the same values.
