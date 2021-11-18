<!--
  Style Guide
  - Make sure to wrap everything to 80 columns when possible.
  - Sentence-ending periods are followed by two spaces.
  - Hexadecimal integers should be formatted as `0xff`.
-->

Project Cerberus: Table Format Specification
=====

Authors:
- Miguel Young de la Sota, Software Engineer, Google
- Christopher Weimer, Senior Firmware Engineer, Microsoft

# Summary

This document defines an interface definition language (IDL) used by Cerberus
specifications for describing the layout of structures in memory and on the
wire. The format is called the *Cerberus Table Format* which has the following
properties:

- It can be included inline in Markdown documents as consistently-formatted
  tables.
- It can be parsed by a computer program, for use in generating parsers and
  other Cerberus-related code.

This document is a normative reference for the Cerberus Table Format.

# Message and Namespaces

A message is similar to a C struct, consisting of a list of named fields of
various types. A message represents any kind of structured data, such as a
command on the wire or a configuration file in flash.

Each field in a message has a name, a type, and a short description. Field names
should be `snake_case` names consisting of letters, underscores, and digits
(although not starting with a digit). This information is arranged into a table
thus (the backticks for code rendering are mandatory):

`message Namespace.Name`
| Type       | Name         | Description                        |
|------------|--------------|------------------------------------|
| `SomeType` | `field_name` | A prose description for the field. |

Each field is given a row in the table. A table is denoted as being a message
definition by the tokens `` `message `` on the line immediately before it, which
parsers can use to determine that they can start parsing a table. The name of
the message follows after the `message` token, which consists of any number of
`CamelCase` components (again, letters, digits, and underscores, but no starting
digit) separated by periods. The following are valid message names:
- `Foo`
- `Unsealing.Request`
- `Challenge.Response.KeyType`

The list of all components in a message name, except for the last, is called the
*namespace* of the name; namespaces are names too: the namespace of
`Challenge.Response.KeyType` is `Challenge.Response`, which may or may not name
a message, and the namespace of `Challenge.Response` is `Challenge`.

All of the messages in a file together form a *specification*, and can refer to
each other by name, possibly leaving off a shared prefix (when it would not be
ambiguous). For example, `Challenge.Response` can refer to
`Challenge.Response.KeyType` as just `KeyType`, since they both share the same
prefix, `Challenge.Response`. Formally, if a message `A.B.C` refers to `D.E`,
then `D.E` canonicalizes to the first of the following that is a valid message:
- `D.E`
- `A.B.C.D.E`
- `A.B.D.E`
- `A.D.E`

Fields may use the special name `_` to indicate that they are reserved or
otherwise must have a specific fixed value; they must have one of the hex or
binary literal types defined below.

## Formatting

Tables must be formatted such that each column has the same width on each row,
and the vertical bars separating columns are aligned. Every table in this
specification is correctly formatted, so they can be taken as normative
examples.

There is an exception around the 80-column width limit for Markdown text. When a
table row's `Description` would cause it to be longer than 80 columns, it may
extend past the limit, but other rows that would not overflow must end at the 80
column line. The following diagram illustrates the possibilities:

```
               . <- Column limit at this dot.
| Foo | Bar    |
|-----|--------|
| 1   | Bazzz. |
| 2   | Bazzzz.|
| 3   | Bazzzz. |
```

Note that column (2) is a special case. If the column would overflow by one
character, we delete the space separating the prose from the pipe.

# Types

The `Type` column of a message table defines how each field should be serialized
and deserialized on the wire. To serialize a message, we recursively serialize
each of its fields in sequence; to serialize each message, we deserialize each
field in declaration order from a sequence of bytes (i.e, octets).

Although most sources of Cerberus messages work on byte streams, the table types
themselves are defined as bits; in general, all fields will be aligned on
eight-bit boundaries, but some bytes may be split into smaller-than-a-byte
fields.

The following basic types are defined, out of which more complex types may be
built:
- `bN`, where `N` is a literal decimal integer, is a string of that many bits.
  For example, `b8` is a single octet, `b1` is a single bit, and `b256` would be
  32 bytes. `b0` is well-formed, and can be used where an empty message would be
  useful.

  Fixed bitstrings are often interpreted as unsigned integers or signed two's
  complement integers, as indicated by prose or other types.

- `N`, where `N` is a literal hex or binary integer. Hex literals begin with
  `0x`, while binary literals begin with `0b0`. They are encoded as their
  literal value, and when decoding, that exact value must be encountered for a
  successful parse.

  Hex and binary literals have an intrinsic bit-width, unlike decimal integers;
  each digit of a hex literal is four bits, and each digit of a binary literal
  is one bit; leading zeros are significant. `0x0ab` is 12 bits, while `0b000`
  is three.

- Any message name, used as a type, is as-if all of that message's fields were
  included inline; that is, to parse a field with message type `T`, we parse
  recurse into `T`.

  Messages cannot recursively refer to themselves if that would make it
  impossible to parse a message `T` without always parsing another message `T`
  (it could be permissible to have `T` contain a variable-length array of `T`,
  though).

- `T[N]`, where `T` is any self-delimited type (see below) and `N` is:
  - A literal binary, hex, or decimal integer. This is a fixed-length array
    with `N` values of `T` back to back. The values of `T` don't need to be of
    equal size if they are messages with variable-length arrays; they are simply
    deserialized one-by-one.
  - The name of a bit-string type. This is a variable-length array which has a
    field of type `N` preceeding it acting as an element-count length prefix.
    For example, `b32[b8]` is a byte-sized length prefix followed by that many
    32-bit integers.
  - The name of another field of bit-string type.. This is just like `T[bN]`,
    but the field is declared explicitly, where the only constraint is that it
    occurs earlier in the message.
  When `T` is `b8`, it may be omitted, allowing for constructions like `[32]`
  and `[b16]`.

- `T...`, where `T` is any self-delimited type (see below). This is serialized
  as an unprefixed array of `T`s, which is delimited by the EOF of the overall
  message. `...` is shorthand for `b8...`.

Type are of *fixed length* if either:
- It is a `bN` type.
- It is `T[N]` for `T` fixed-length an `N` an integer.
- It is a message consisting entirely of fixed-length types.

Types are not *self-delimited*, meaning it is necessary to appeal to the overall
length of the message to parse it, if either:
- It is `T...` for some `T`.
- It is a message type which has a non-self-delimited field.

If a message is to have a non-self-delimited field, all of the fields that
follow it (if any) must be fixed-length, so that parsing the end of the
non-self-delimited field is unambiguous.

## Examples

The following are examples taken from the Challenge Protocol Specification.

`message Challenge.Request`
| Type   | Name    | Description                                               |
|--------|---------|-----------------------------------------------------------|
| `b8`   | `slot`  | Slot Number (0 to 7) of the target Certificate Chain to read. |
| `0x00` | `_`     | Reserved.                                                 |
| `[32]` | `nonce` | Random nonce.                                             |

The challenge request message consists of a byte, a fixed 0x00 byte, and a
32-element array of bytes.

`message Challenge.Response`
| Type     | Name              | Description                                   |
|----------|-------------------|-----------------------------------------------|
| `b8`     | `slot`            | Slot Number of the Certificate Chain used.    |
| `b8`     | `mask`            | Certificate slot mask.                        |
| `b8`     | `min_version`     | MinProtocolVersion supported by device.       |
| `b8`     | `max_version`     | MaxProtocolVersion supported by device.       |
| `0x0000` | `_`               | Reserved.                                     |
| `[32]`   | `nonce`           | Random nonce.                                 |
| `b8`     | `pmr0_components` | Number of components used to generate the PMR0 measurement. |
| `[b8]`   | `pmr0`            | Value of PMR0 (Aggregated Firmware Digest).   |
| `...`    | `signature`       | Signature over concatenated request and response message payloads. |

The challenge response message is more complex. It has four data bytes, then
two fixed bytes in a reserved field, then a 32-element array of bytes, another
byte, a byte array with a single-byte length prefix, and finally, a byte array
spanning to the end of the message.

Types may be arbitrarily complicated. Here are some examples:
- `b32...`: read four-byte chunks until the end of the message (with no trailing
  bytes).
- `[b8][b16]`: a two-byte length prefix, followed by that many byte-prefixed
  byte arrays.
- `0x55[field]`: The byte `0x55` repeated `field` times.
- `[b8]...`: Byte-prefixed arrays until the end of the message.
- `b16[field][2]`: An array of sixteen-bit integers of length `field * 2`.

# Enums and Mappings

In addition to messages, a table can define an *enum*; a new type consisting of a
finite set of equally-sized constants. The `message` token is replaced with
`enum`, and the `Type` column becomes `Value`. The items in this column must be
hex or binary literals, all of which have equal bit width; these are called the
*variants* of the enum. For example:

`enum Namespace.Enum`
| Value    | Name            | Description                                |
|----------|-----------------|--------------------------------------------|
| `0xabcd` | `variant_name`  | A prose description for the variant.       |
| `0xef01` | `variant_name2` | A prose description for the other variant. |

The width of the variant values is the enum's *length*.

Enums may appear as types in a message. To deserialize an enum field, a bit
string of the length of the enum is parsed, which must match one of the variant
constants. To serialize an enum field, the constant associated with the variant
is serialized.

Enums may be *mapped* by defining enum mappings, allowing fields in a message to
select their type or length based on a previous enum field's value.

A value map looks exactly like an enum, except that the name is followed by the
enum it maps wrapped in parentheses, and the description column is optional.
Value maps must have the same set of variant names as an enum, and the constants
may also be decimal literals, too. For example:

`enum Namespace.Mapping(Enum)`
| Value | Name            |
|-------|-----------------|
| `100` | `variant_name`  |
| `200` | `variant_name2` |

Value maps can be used to construct array lengths: if `enum_field` is an enum
field in a message, and `Mapping` is a mapping over `enum_field`'s type, then
`T[Mapping(enum_field)]` is an array of `T`s with length given by the constant
in `Mapping` with the same variant name as the value of `enum_field`. In other
words, `Mapping` acts like a function mapping an enum type to integer literals.

For example, if `enum_field` was a `Namespace.Enum`, and was the variant
`variant_name2`, then `b32[Mapping(enum_field)]` would be 200 `b32`s, or 800
bytes.

A type map is identical to a value map, except it has a `Type` column instead of
a `Value` column:

`enum Namespace.TMapping(Enum)`
| Type    | Name            |
|---------|-----------------|
| `[100]` | `variant_name`  |
| `[b8]`  | `variant_name2` |

Type mappings can be used directly as types for message fields with the
`TMapping(enum_field)` syntax as above. For example, if `enum_field` was again
`variant_name2`, then a field of type `TMapping(enum_field)` would be parsed
like a `[b8]`: a byte-sized length prefix, followed by that many bytes.

## Examples

The Key Exchange message in the Challenge Protocol Specification makes extensive
use of enums and mappings, so it's a good place to start when trying to
understand them.

