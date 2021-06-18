* Name: Capabilties2
* Date: 2021-06-18
* Pull Request: [#19](https://github.com/opencomputeproject/Security/pull/19)

# Objective

The Challenge Protocol uses the "Device Capabilities" message to negotiate
various capabilities, such as packet size and support for different kinds of
cryptography. Unfortunately, it is not extensible: it is not possible to add
negotiation for new fields or drop it for no-longer-necessary fields. This is
particularly relevant in light of
[Security#16](https://github.com/opencomputeproject/Security/pull/16), which
proposes removing the payload negotiation into the transport layer.

This proposal replaces "Device Capabilities" with an extensible negotiation
flow that allow arbitrary new capabilities to be exchanged, while respecting the
maximum 64-byte size of required messages.

# Proposal

This proposal adds one new message, called "Negotiation", with the following
format:

```
struct {
  uint8_t count;
  uint8_t reserved;
  // Records...
}
```

The `reserved` byte must always be set to zero, and servers must reject
non-zero values. This may eventually be used to implement continuation bits for
negotiation, if either side of the connection cannot communicate every
capability in a single message.

After the reserved byte, an arbitrarily long sequence of `count` *records* may
follow. A record consists of:
1. A one-byte tag identifying the capability it communicates, in the range `0x00`
   to `0x7f`. Tags with the top bit set are reserved for future specification.
   Either end of the connection must reject tags with the top bit set, since its
   length may be unspecified.
2. At least one byte of contents, encoded in one of three forms:
   - As a six-bit integer, encoded as a byte with the top two bits forming the
     value `0b00`.
   - As a fourteen-bit integer, encoded as two bytes, where the value is formed
     as `contents[0] | (contents[1] << 6)`. The top two bits form the value
     `0b01`.
   - As an array of bytes, where the length is encoded in the bottom six bits of
     the first byte, with the top two bits having the value `0b10`.
   - As an array of bytes, where the length is encoded in the first two bytes as
     a fourteen-bit integer, following the scheme above. The top two bits form
     the value `0b11`.

In practice, we expect only the first two forms to be used, but since we want
implementations to skip over records they don't recognize, we specify up to
16-kilobyte records for forward-compatibility.

Records must appear in lexicographic order: implementations must reject
out-of-order or duplicate records. This simplifies the process for detecting
duplicate records.

When beginning communications with a remote RoT, an RoT must send a Negotiation
message with all of the capabilities it wishes to inform the remote device of.
The remote device may then either respond with its own capabilities, or return
an error, if the client failed to provide a critical capability required for
successful communication. The client may then, in turn, discontinue the
communication if it does not find the server's capabilities satisfactory.

The following capabilities are pre-defined. We do not currently specify a range
for vendor use.

- "Mode", `0x00`. Six bit record. Required.
  - Bits [1:0] are the device type. They are interpreted as in the Device
    Capabilities message (AC-RoT, PA-RoT, etc.).
  - Bits [3:2] are the bus role (i.e., can it act as a "client" or "server" on
    the bus). They are interpreted as in Device Capabilities, too.
  - Bits [5:4] must be zero.
- "RSA Support", `0x01`. Six bit record. Optional; if not present, implies that
  RSA encryption is not supported.
  - Bits [2:0] are the supported key strengths. The bits are interpreted as in
    the Device Capabilities message.
  - Bits [5:3] must be zero.
- "ECDSA Support", `0x02`. Six bit record. Optional; if not present, implies that
  ECDSA encryption is not supported. (NOTE: Cerberus currently does not specify
  any curves, so we don't do so here, either.)
  - Bits [2:0] are the supported key strengths. The bits are interpreted as in
    the Device Capabilities message.
  - Bits [5:3] must be zero.
- "AES Support", `0x03`. Six bit record. Optional; if not present, implies that
  AES encryption (and, thus, confidentiality) is unsupported.
  - Bits [2:0] are the supported key strengths, and are as in the Device
    Capabilities message.
  - Bits [5:3] must be zero.

The payload sizes and message timeouts are not included, since those are
intended to be relegated to the transport (per RFC-TODO). The "PFM support",
"Policy support", and "Firmware Protection" bits are currently not given
specified meaning, they are not included in the proposal.

Finally, we propose deprecating the Capabilities message. If a device receives a
Device Capabilities message, it MAY interpret it as a Negotiation message with
the relevant fields and react accordingly, but may reject it if the equivalent
Negotiation message would be unsuitable. It may also reject all Device
Capabilities messages altogether. Clients do not need to send a Device
Capabilities message at all, unless they wish to be compatible with existing
devices that require it.

# Specification Changelist

This proposal adds an entire new section to the list of challenge messages,
adding Negotiation under the current Device Capabilities message. It also adds
it to the list of commands at the top of the section.

Additionally, the Device Capabilities message is marked as deprecated, with the
suggested remediation strategy described above.

# Alternatives Considered

We could add another capabilities message every time we need to negotiate a new
capability. For example, if we were to add a new certificate format, we would
add a new message for negotiating that, or add a bit to the GET_CERTIFICATES
message. However, this would significantly complicate the handshake, and require
burning command message bytes far more often than is strictly necessary.

If we adopt transport agonosticism, several fields in the current Device
Capabilities message become meaningless, and would need to be zeroed out but
still sent over the wire.

# Future Work

The encoding described a above leaves us with significant leeway to add even
more capabilities contents. The flags byte, for example, can be used to
indicate a continuation byte, if all the capabilities don't fit in one byte.

We expect to add new capabilities as necessary as we add new features to
Cerberus. It is unlikely that we'll run out of encoding space, and if we do, all
tags with the top bit set are reserved, to give us an opportunity to expand the
encoding space.

