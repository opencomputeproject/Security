* Name: `Command_Definition_Syntax`
* Date: 2021-09-20
* Pull Request: [#24](https://github.com/opencomputeproject/Security/pull/24)

# Objective

Currently, the byte layouts of Cerberus protocol commands (hereafter "protocol
structs") are defined ad-hoc by way of tables containing offsets. These tables
present a number of problems:
- They are painful to edit, because they are offset-based.
- They are unnecessarily verbose in some cases.
- They are not machine readable.

This RFC proposes a formal syntax for the tables that is machine readable and
concise in the common case.

# Proposal

Each table looks like the following in Markdown:
```
`message <Name>`
| Type | Name | Description |
| ---- | ---- | ----------- |
...
```
Each table defines a message type. Each message type consists of a name and
some number of fields. Each field consists of a type, a name, and an optional
description.

Message names must be be in `CamelCase`, with optional periods to separate
components; field names must be in `snake_case`. All table cells, save the
descriptions, are to be wrapped in backticks.

The special field name `_` can be used for reserved/unused fields.

Descriptions should be complete sentences. Although ideally descriptions should
fit in one line, this isn't always possible; if a description would surpass the
80 column limit, it should go on as far as possible, ending in ` |` after the
final sentence's period.

When editing tables, a formatter like http://markdowntable.com/ is recommended.

## Common Types

The vast majority of Cerberus messages' fields are just buffers of bytes, which
are sometimes interpreted as integers. Some buffers are variable length (with
a length prefix) and some are variable length to the end of the message.

The following types are intended to capture these use-cases:
- Fixed-width bit strings, specified as `b` followed by a literal integer.
  For example, `b1` is a single bit, `b32` is four bytes, and `b256` is 32
  bytes, enough for a SHA2-256 digest. The length of `bN` is `N` bits.
  `b0` is well-formed, and specifies that the field is not encoded at all.
  The primary value is in conjunction with `align`, discussed below.

- Literal bit strings, specified as C-style hex or binary literals, such as
  `0xabcd` or `0b101010`. Binary literals' width is the number of binary digits:
  `0b0001` is four bits; leading zeros are significant. Hex literals' width is
  four times the number of hex digits: `0x0abcd` is 20 bits. Again, leading
  zeros are significant.

- Other message types, specified by that type's name. The type `MyMessage`
  indicates the entire representation of `MyMessage` inline. When referring
  to another message in a document, if the message contains a common
  dot-separated prefix, it may be omited. For example, `Foo.Bar.Baz` may be
  referred to as `Baz` inside of `Foo.Bar`, and `Foo.Bar.Baz` may refer to
  `Foo.Bar` as `Bar`.

- An enum type mapping, specified as `Map(field)`, where `field` is a previous
  field of enum type and `Map` is a type mapping (see below) for that field.
  The field is encoded as the type specified by the mapping.

- Fixed-length arrays: any other type followed by `[N]`, where `N` a literal
  integer value, consisting of encodings of that type concatenated. For
  example, `b8[32]` is equivalent to `b256`, and `MyMessage[2]` is two
  `MyMessages` back to back, which may be variable-length if `MyMessage` is
  variable-length.

  If the type is `b8`, it may be ommited: `[32]` is 32 bytes.

- Variable-length arrays: any other type followed by `[field]`, or
  `[Map(field)]` where `field` is a previous field and `Map` is an enum
  value mapping (see below). `field`'s contents, as bits, are interpreted as a
  little-endian integer, which specifies the number of copies of the type
  to follow. For example, if `foo` is a `b16`, then `MyMessage[foo]` is as
  many `MyMessage`s as `foo` specifies when viewed as an unsigned, 16-bit
  integer.

  If the type is `b8`, it may be ommited: `[field]` is `b8[field]`

- Variable-length prefixed arrays: any other type followed by `[bN]`, where
  `N` is a literal integer value. `T[bN]` is equivalent to encoding a
  `bN` called `length` followed by a `T[length]`. For example, `b8[b16]` is
  a sixteen-bit prefix followed by that many bytes.

  If the type is `b8`, it may be ommited: `[b16]` is `b8[b16]`

- Variable-length unprefixed arrays: any other type followed by `...`. This
  is encoded as as many copies of that type until the end of the message.
  For example, `b8...` is all bytes to the end of the message, and
  `MyMessage...` means to parse `MyMessage`s until you run out of bytes.
  A `...` field must be the last field in the message, and such messages
  may only occur as the last field in other messages, and cannot be used to
  construct array types.

  If the type is `b8`, it may be ommited: `...` is `b8...`

Unless stated otherwise, bytes form integers in little-endian order, and bits
are ordered within bytes according to the underlying transport; messages' sizes
are rounded up to a byte. For example, a message consisting of two `b1` fields
is one byte long, and the fields represent the least- and second-least-significant
bits of that byte.

The following are representative examples taken from the current Challenge
Protocol spec:

`message Challenge.Response`
| Type     | Name          | Description                                       |
|----------|---------------|---------------------------------------------------|
| `b8`     | `slot`        | Slot number of the Certificate Chain.             |
| `b8`     | `slot_mask`   | Certificate slot mask.                            |
| `b8`     | `min_version` | Minimum protocol version supported by device.     |
| `b8`     | `max_version` | Maximum protocol version supported by device.     |
| `0x0000` | `_`           | Reserved.                                         |
| `b256`   | `nonce`       | Random 256-bit nonce.                             |
| `[b8]`   | `pmr0`        | The contents of PMR0 (aggregated firmware digest).|
| `...`    | `signature`   | Signature over concatenated request and response payloads. |

`message KeyExchange.Request.PairedKeyHmac`
| Type   | Name               | Description                                |
|--------|--------------------|--------------------------------------------|
| `0x01` | `key_type`         | The type of key data being sent.           |
| `b16`  | `pairing_key_len`  | Length of the pairing key, in bytes.       |
| `...`  | `pairing_key_hmac` | HMAC of the pairing key: `HMAC(K_M, K_P)`. |

`message AttestationLogFormat`
| Type     | Name                | Description                                 |
|----------|---------------------|---------------------------------------------|
| `0x0b`   | `header_format`     | Header format version.                      |
| `b16`    | `entry_length`      | Total length of the entry.                  |
| `b32`    | `unique_id`         | A unique identifier for the entry.          |
| `b32`    | `tcg_type`          | The associated TCG event type.              |
| `b8`     | `measurement_index` | Index of the measurement within the PMR.    |
| `b8`     | `pmr_index`         | Index of the PMR being extended.            |
| `0x0000` | `_`                 | Reserved.                                   |
| `b8`     | `digest_count`      | Number of digests.                          |
| `0x0000` | `_`                 | Reserved.                                   |
| `0x0b`   | `digest_algo_id`    | Digest algorithm ID, fixed to SHA-256.      |
| `b256`   | `digest`            | SHA-256 digest used to extend the measurement. |
| `[b32]`  | `measurement`       | The measurement value.                      |

### Alignment

Field types may be followed by `align(n)`, where `n` is a literal integer. This
specifies that the field must be aligned to an `n`-byte boundary relative to the
start of the message. The padding must be all-zeroes, and may be variable-length
depending on the length of fields that came before. For example:

`message AlignedBuf`
| Type             | Name   | Description                        |
|------------------|--------|------------------------------------|
| `b16`            | `len`  | The buffer length.                 |
| `[len]`          | `buf`  | The buffer.                        |
| `[len] align(4)` | `buf2` | Another buffer but 4-byte-aligned. |

If `len` were `3`, then `buf` would take up bytes `2` through `5`. The next
multiple of `4` is `8`, so there would be three bytes of zero-padding before the
first byte of `buf2`.

`b0 align(n)` may be used as the last field to indicate tail padding.

Alignment is most useful for types that are stored in-memory rather than
deserialized from a byte-stream.

## Enums

Some fields are of a fixed size and take on a small set of values. Just like
`message`s, `enum` names are `CamelCase` with periods, and their values must be
`snake_case`. Values may be in hex or binary, and must all be of the same
width.

`enum GetCertState.CertState`
| Value  | Name                | Description                                   |
|--------|---------------------|-----------------------------------------------|
| `0x00` | `chain_provisioned` | A valid chain has been provisioned.           |
| `0x01` | `chain_missing`     | A valid chain has yet to be provisioned.      |
| `0x02` | `validating`        | The stored chain is currently being validated.|

This can then be used directly in the "Type" column:

`message GetCertState`
| Type        | Name            | Description                                  |
|-------------|-----------------|----------------------------------------------|
| `CertState` | `cert_state`    | The current certificate state.               |
| `b32`       | `error_details` | Details of an error in certificate validation, if one has occurred. |

An `enum` may be *mapped* to provide an alternative encoding of its values.
If we have an enum like

`enum HashType`
| Value  | Name       | Description |
|--------|------------|-------------|
| `0b00` | `sha2_256` | SHA2-256.   |
| `0b01` | `sha2_324` | SHA2-384.   |
| `0b10` | `sha2_512` | SHA2-512.   |

We can *map* it like so:
`enum HashLength(HashType)`
| Value | Name       |
|-------|------------|
| `32`  | `sha2_256` |
| `48`  | `sha2_324` |
| `64`  | `sha2_512` |

Note the name of the mapped enum in parenthesis. It satisfies the same namespace
rules as a field of a message.

Because the width is not important, decimal integers may be used in addition to
hex or binary. This can then be used to specify a variable-length array:
`b8[HashLength(hash_type)]`. 

Maps may also produce types:

`enum Digest(HashType)`
| Type   | Name       |
|--------|------------|
| `b256` | `sha2_256` |
| `b384` | `sha2_324` |
| `b512` | `sha2_512` |

These may be used as fields directly: `Digest(hash_type)`.

# Specification Changelist

In addition to updating all tables to use the new syntax, a new specification,
the "Cerberus Schema Table Specification", would be added, giving a description
of the above formal language.

# Open Questions

There are a handful of messages this scheme cannot handle, and we need to decide
whether to make those messages conform to it or introduce new options into
the scheme:
- "Get Configuration Ids" needs to sum two length prefixes together.
  We could resolve this by having two back-to-back `T[field]`s with the value of
  each respective field. Multiplication of length prefixes can be achieved by
  `T[field1][field2]`.
- The "Get $Manifest Id" messages have an optional field, as does "Get Recovery
  Image Id".
  We could resolve this by using `T...` for them, with prose to specify the
  default value. We could also add syntax for "until the end but with a maximum
  length", e.g. `T[...n]`, but that seems overkill to me.
