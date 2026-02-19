# Struct data type

This document defines `struct`, a data type for arrays whose elements are
fixed-size records composed of named, typed fields — commonly referred to as
"structured arrays" or "record arrays".

The `struct` data type closely models NumPy's
[structured arrays](https://numpy.org/doc/stable/user/basics.rec.html), where
each element consists of multiple named fields, each with its own data type.

## Background

Structured arrays allow a single array to represent tabular or record-like data
without using separate arrays per column. For example, a geospatial dataset
might store `(latitude: float64, longitude: float64, elevation: float32)` as a
single structured array rather than three separate arrays.

Each element of a structured array is a scalar of a fixed size in bytes. The
size is the sum of the sizes of all fields.

## Data type representation

### Name

The name of this data type is the string `"struct"`.

### Configuration

This data type requires a configuration object. The configuration object must
have exactly one key, `"fields"`, whose value is a JSON array of fields.

Each field is a 2-element JSON array `[field_name, field_dtype]`, where:

- `field_name` is a non-empty string that identifies the field.
- `field_dtype` is a valid Zarr v3 data type representation whose size in
  bytes is fixed and known at the time the array is opened:
  - For [core data types](https://zarr-specs.readthedocs.io/en/latest/v3/data-types/index.html#core-data-types),
    this MUST be a string (e.g. `"float32"`, `"int32"`, `"uint8"`).
  - For extension data types that require configuration (e.g. `numpy.datetime64`),
    this MUST be an object with a `"name"` key and a `"configuration"` key.
  - Variable-length data types (e.g. `"string"`) MUST NOT be used as field
    types, as they do not have a fixed encoded size.

The `"fields"` array must contain at least one field. Field names must be
unique within a given `struct` data type.

The `struct` data type may be used recursively: a field's data type may
itself be `"struct"`, enabling nested record types.

### Examples

The following is an example of array metadata for an array of 2D point
records, each with an `x` and a `y` coordinate stored as 32-bit floats:

```json
{
  "zarr_format": 3,
  "node_type": "array",
  "shape": [100],
  "data_type": {
    "name": "struct",
    "configuration": {
      "fields": [
        ["x", "float32"],
        ["y", "float32"]
      ]
    }
  },
  "chunk_grid": {
    "name": "regular",
    "configuration": {"chunk_shape": [100]}
  },
  "chunk_key_encoding": {"name": "default"},
  "fill_value": {"x": 0.0, "y": 0.0},
  "codecs": [{"name": "bytes", "configuration": {"endian": "little"}}]
}
```

The following is an example with heterogeneous field types: a 32-bit integer
identifier, a single byte of bit flags, and a 64-bit floating-point value:

```json
{
  "name": "struct",
  "configuration": {
    "fields": [
      ["id",    "int32"],
      ["flags", "uint8"],
      ["value", "float64"]
    ]
  }
}
```

The following is an example where one field uses a parametrized data type.
The `timestamp` field uses [`numpy.datetime64`](../numpy.datetime64/README.md),
which requires a `configuration` object specifying `unit` and `scale_factor`:

```json
{
  "name": "struct",
  "configuration": {
    "fields": [
      [
        "timestamp",
        {
          "name": "numpy.datetime64",
          "configuration": {
            "unit": "s",
            "scale_factor": 1
          }
        }
      ],
      ["value", "float32"]
    ]
  }
}
```

The following is an example with a nested structured field. The outer record
has a `point` field that is itself a structured type with `x` and `y`
sub-fields, plus a scalar `value` field:

```json
{
  "name": "struct",
  "configuration": {
    "fields": [
      [
        "point",
        {
          "name": "struct",
          "configuration": {
            "fields": [
              ["x", "float32"],
              ["y", "float32"]
            ]
          }
        }
      ],
      ["value", "float64"]
    ]
  }
}
```

## Bytes codec encoding

When the `bytes` codec is used (as required), each structured scalar is encoded
as the packed concatenation of the encoded bytes of each field's value, in
field declaration order. No padding bytes are inserted between fields,
regardless of alignment considerations.

The total encoded size of a structured scalar in bytes is the sum of the
encoded sizes of all fields.

As a concrete example, the structured type `[("id", int32), ("flags", uint8),
("value", float64)]` has an encoded element size of 4 + 1 + 8 = 13 bytes,
with field byte offsets of 0, 4, and 5 respectively.

## Fill value representation

The `fill_value` for arrays with the `struct` data type MUST be a JSON
object mapping each field name to its fill value. Each field's value must be
a valid fill value for that field's data type. Fields omitted from the object
are treated as zero-valued.

```json
"fill_value": {"x": 1.23, "y": 4.56}
```

For nested structured fields, the value must itself be an object mapping the
nested field names to their fill values:

```json
"fill_value": {"point": {"x": 1.0, "y": 2.0}, "value": 3.14}
```

> **Legacy compatibility:** Existing arrays may encode the fill value as a
> [base64](https://en.wikipedia.org/wiki/Base64)-encoded string of the raw
> packed bytes (e.g. `"AAAAAAAAAAA="` for 8 zero bytes). Implementations
> SHOULD support reading this form for backward compatibility, but MUST NOT
> write it for new arrays.

## Codec compatibility

This data type is compatible with any codec that supports a fixed number of
bytes per array element. The `bytes` codec MUST be used as the array-to-bytes
codec. Other codecs (e.g. `gzip`, `zstd`, `blosc`) MAY be applied on top of
the `bytes` codec for compression.

### Endianness handling

When a structured type contains multi-byte numeric fields, the `bytes` codec
MUST be configured with an explicit `endian` setting
(e.g. `{"name": "bytes", "configuration": {"endian": "little"}}`). All
multi-byte fields MUST use the byte order specified by the `endian` parameter.

Structured types composed entirely of single-byte fields (e.g. `uint8`,
`int8`) have no byte-order ambiguity and MAY omit the `endian` configuration.

> **Legacy compatibility:** Arrays where the `bytes` codec has no `endian`
> configuration (i.e. `{"name": "bytes"}` with no `configuration` key) SHOULD
> be treated as little-endian by implementations. Implementations SHOULD warn
> when `endian` is absent for structured types with multi-byte numeric fields.

Variable-length codecs (e.g. `vlen-utf8`) are not compatible with the
`struct` data type.

## Notes

> **Note:** Fields are packed contiguously with no inter-field padding. This
> matches the default NumPy structured dtype layout. NumPy's "aligned"
> structured dtype layout (created with `align=True`), which inserts padding
> for memory alignment, is NOT supported by this specification.

> **Note:** Field names MUST be unique within a `struct` data type.
> Implementations MUST reject `struct` types with duplicate field names.

> **Note:** The order of fields in the `"fields"` array is significant and
> MUST be preserved. Fields are encoded in declaration order.

## Change log

No changes yet.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
