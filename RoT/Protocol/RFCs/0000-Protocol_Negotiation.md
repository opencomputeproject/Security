* Name: Protocol_Negotiation
* Date: 2021-10-06
* Pull Request: [#25](https://github.com/opencomputeproject/Security/pull/25)

# Objective

This proposal is a merger of the
[Capabilities2](https://github.com/opencomputeproject/Security/pull/21) and
[Version_Detection](https://github.com/opencomputeproject/Security/pull/23)
RFCs, which are found to have sufficient overlap that a single message should be
able to cover both use-cases.

This proposal defines the following:
1. A message for negotiating a collection of key-value pairs between a host and
   a device.
2. A versioning definition mechanism for the Cerberus Challenge Protocol, which
   is defined in terms of the protocol's Git history. It is negotiated using the
   aforementioned message.

These two mechanisms are closely related: we want versioning to determine the
layout of messages, which means a version must be negotiated. Originally, the
two proposals were essentially disjoint negotiations. However, the following
observations imply they are the same idea:
- The version negotiation message cannot itself be versioned via the usual
  versioning mechanism (self-evident: this is a bootstrapping problem).
- The format of capabilities negotiation is, by its nature, self-describing: a
  host can ask a device any number of questions about what it supports, so it is
  desireable to make it easy to ask the device questions it can choose to
  ignore.

Thus, this proposal combines them into one.

## Requirements for Negotiation

1. Extensible: the host must be able to ask the device questions it does not
   understand, and the device must be able to ignore them.
    - As a corollary, hosts must be able to ask vendor-specific questions in a
      way that does not encourage collision.
2. Multi-part: because this message occurs very early in communication with a
   device, the message must fit within the 64 byte minimum that devices are
   required to support; the current `Device Capabilities` message has this
   explicit property.
3. Future-compatible: This message is special: it is not versioned with the rest
   of the Challenge protocol, since it is used for picking a version. Therefore,
   there should be a way to nonetheless change how questions and answers are
   encoded.
4. Reusable. Negotiation results must be reusable across many interactions with
   a device, given that neither the host nor the device has reset. For example,
   any number of challenges, key exchanges, or manifest updates must be possible
   without explicit renegotiation. Reset of either device must discard the
   negotiation.

## Requirements for Versioning

1. Future-proof: it should be unlikely that we run out of version numbers.
2. Git-compatible: it should make use of the existing change-by-change
   versioning of the protocol text itself, to make it absolutely clear what the
   text for a specific version is.
3. Deprecation: it must be easy to deprecate messages, and even re-use things
   like command bytes. Old versions should restrict new versions as little as
   possible.

## Non-goals

Currently, negotiated capabilities in Cerberus are not included in any
authentication signatures. Although this is something we need to address, it is
beyond the scope of this RFC. We will outline a way to achieve this in [Future
Work](#future-work).

# Proposal

This proposal has two parts: the negotiation message, and the versioning scheme.

## The `Protocol Negotiation` Message

Negotiation, fundamentally, is about getting the host and the device to agree on
a set of key-value pairs. We will define a new message, `Protocol Negotiation`,
for this purpose. Using it is as follows:

1. Host (H) sends Device (D) a `Protocol Negotiation` request, which is a map of
   keys ("questions") to lists of values (potential "answers"). Each are
   represented by arbitrary byte strings.
2. D sends H a `Protocol Negotiation` response, which is a map of a subset of
   the questions to one answer for each.
3. Repeat from step (1) with more questions.

The format of keys and values is defined below.

### Encoding

A question/answer pair, or "record", consists of a 32-bit integer followed by a
buffer of at most 64 bytes. Records are encoded thus:
- The first byte, the "header", has two fields: `0bxxyyyyyy`. The six low bits
  are the length of the record, not including the first byte. The two high bits
  describe the number of bytes in the "question".
- The "question" is encoded as a little-endian integer. The value of the high
  bits above defines the number of bytes in the integer: `00` for one, `01` for
  2, and `10` for 4. `11` is not used and must be rejected.
    - Integers must be minimally-encoded. The integer `42` must be encoded as
      one byte; parsers that recognize this and reject them.
- The remaining bytes are the "answers", which are interpreted in a per-question
  manner.

Because each record is length-prefixed, they can be skipped by a device that
does not understand them. The 64-byte maximum length is more than enough space,
since negotiation messages should not be longer than 64 bytes long, anyways.

Above, we described requests as containing many possible answers, and responses
containing only one. How this is encoded is defined by each question, but the
host must reject any responses that contain multiple answers for a question.
Hosts must also reject responses that contain answers for questions they did not
ask. Devices must reject requests that ask questions they understand, but where
the answer list fails to parse; this list must be non-empty.

The `Protocol Negotiation` request and response are identical, and is described
as the following pseudo-C struct:

```c
struct Negotiation {
  uint8_t reserved;
  uint8_t flags;
  record records[flags & 0x7f];
}
```

In other words, a negotiation message consists of a list of records, some flags,
and a reserved byte for forward compatibility. This byte must always be set to
zero; future versions may introduce new values for it that update how the
questions are encoded.

The flags byte consists of the length (in records) of the record list, and the
"renegotiate" bit (the high bit), described below. Responses with the
"renegotiate" bit set must be rejected as invalid.

### Negotiation State

Because negotiation defines much of the interactions that are to follow, it must
occur very early in an interaction between a host and a device. In particular,
it must be the first message sent to a device.

Hosts and devices maintain *negotiation state*: once negotiation occurs, the
agreed-upon set of key-value pairs is referenced by messages that follow, in
particular for determining the version of the protocol (and, therefore, the
layout of other messages).

Operations that can be performed on the shared set of key-value pairs are
appending (by way of `Protocol Negotiation` messages) and clearing (by way of
sending a `Protocol Negotiation` request with the "renegotiate" bit set). When
clearing, clearing occurs before the records in the same message are processed.

Appending new key-value pairs via a `Protocol Negotiation` message must not
introduce a repeated key: devices must reject messages that try to do so without
requesting renegotiation first.

If a device receives a `Protocol Negotiation` request from a host, but the
device believes the negotiation state is empty, and the request does not
indicate renegotiation, the device must reject the request. This is to prevent
the state from desyncing if either party resets unexpectedly: the host must
always set the renegotiation bit for the first negotiation request, since the
device will reject the request if it is not set for the first one.

Negotiation state must not be stored across resets.

### Example Negotiation

The following is an example of an exchange between a host H and device D, and
the state of the negotiation state at each step. Questions and answers are
represented in an ad-hoc JSON-like format.

0. Negotiation state is `{}`.
1. H sends D the request `{0: [1, 2, 3], 1: [4, 5]}`; renegotiation is set.
    a. D replies with `{0: 2, 1: 4}`.
    b. Negotiation state is `{0: 2, 1: 4}`.
2. H sends D the request `{1: [3, 4, 5]}`.
    a. D rejects it, since `1` was already negotiated.
    b. Negotiation state is unchanged.
3. H sends D the request `{2: [0, 2]}`.
    a. D replies with `{}`; it does not understand the question `2`.
    b. Negotiation state is unchanged.
4. H sends D the request `{3: [0], 4: [1, 2]}`.
    a. D replies with `{4: 1}`; it undersands `3`, but does not like any of the
    suggested answers.
    b. Negotiation state is `{0: 2, 1: 4, 4: 1}`.
5. H sends D the request `{3: []}`.
    a. D rejects it, since no answers for `3` are provided, a mistake on H's
    part.
    b. Negotiation state is unchanged.
6. H sends D the request `{0: [1, 2, 3], 3: [0, 2]}`; renegotiation is set.
    a. D replies with `{0: 2, 3: 2}`.
    b. Negotiation state is `{0: 2, 3: 2}`.

2, 3, and 5 represent a misbehaving host, and should not occur in a normal
exchange. Hosts should try to maximize the number of questions sent in each
message.

### Legacy Capabilities Negotiation

Because the old `Device Capabilities` message cannot negotiate a version, it is
completely incompatible with this scheme; we propose removing it altogether and
reallocating its command byte to `Protocol Negotiation`.

## Versioning

We define a Cerberus version to be a sixteen-bit opaque value, which maps to a
commit on this repository. Protocol version N is described by the repository at
that commit.

Versions are recorded as Git tags with the name `protocol-v{#}`, where
`{#}` is replaced with the version number. There will also be a file,
`PROTOCOL_VERSIONS.md`, which contains a list of all versions, their dates,
and a changelog, in the following format:

```
# `protocol-v{#}`
Date: YYYY-MM-DD
<changelog>
```

To create a new version:
1. Create a commit that adds the new version to `PROTOCOL_VERSIONS.md`.
2. The pull request created from that commit acts as an opportunity to discuss
   the new version and changelog.
3. Once approved, the author must ensure the date on the new version matches.
4. A maintainer will merge the PR, and then create a lightweight tag with the
   correct name pointing to the merged commit. This can be done via
   `git tag protocol-v{#} && git push origin --tags`. This must be a push to
   the upstream repository.

This proposal does not stipulate guidelines under which creating a new version
is recommended; maintainers should create new versions as they deem necessary.

After this RFC is merged and implemented, a tag for `protocol-v0` will be
created pointing to the implementation commit.

The version to use should be negotiated via the `Protocol Negotiation` message;
see [Cerberus-Defined Questins](#cerberus-defined-questions) below.
The version must be negotiated before any commands that come after; devices
should reject any other requiests until a version is negotiated.

### Updates to `CHALLENGE`

Because only a single protocol version is negotiated, we can replace the version
range in the `CHALLENGE` with the single negotiated version. This doesn't change
the layout of the `CHALLENGE` in a meaningful way, since we are replacing two
eight-bit fields with one sixteen-bit field.

### Evolution and Deprecation Process

This versioning scheme gives us a way to freely evolve the protocol without
fear of subtle incompatibility: because one version is chosen, there is no
ambiguity about different layouts of commands.

We also get deprecation for free: if we remove a message, devices can, given
sufficiently wide range of advertised versions, select a version both are
can work with. We can even re-use command bytes across versions, if that ever
becomes a problem.

There is no particular reason to mark messages as deprecated in the spec itself,
although it may be worthwhile to do so to indicate that they will eventually be
removed. Whether to leave messages deprecated for a version or two, or to remove
them immediately, is up to the maintainers.

This introduces the risk that two devices may refuse to interoperate due to
incompatible versions. It is up to implementers to chose a sufficiently large
range of versions to interoperate with other vendors' devices.

Note that the `Protocol Negotiation` message is exempt: due to the bootstrapping
problem mentioned in [Objectives](#objectives), it cannot be versioned along
with the rest of the protocol. Instead, the `reserved` byte in its structure can
be used for this, instead.

## Cerberus-Defined Questions

Cerberus defines a number of questions, some coming from either the old `Device
Capabilities` message. They are as follows:

| Question | Name              | Description                                          |
|----------|-------------------|------------------------------------------------------|
| `0x00`   | `version`         | Negotiates the overall protocol version to use.      |
| `0x01`   | `has_sessions`    | Subcomponent: secure sessions.                       |
| `0x02`   | `has_logging`     | Subcomponent: log-extraction commands.               |
| `0x03`   | `has_pfms`        | Subcomponent: PFM operations.                        |
| `0x03`   | `has_manifests`   | Subcomponent: CFM/PCD operations.                    |
| `0x06`   | `has_fw_updates`  | Subcomponent: firmware updates.                      |
| `0x07`   | `has_fw_recovery` | Subcomponent: firmware recovery.                     |
| `0x08`   | `has_pmrs`        | Subcomponent: PMR operations.                        |
| `0x09`   | `has_unsealing`   | Subcomponent: unsealing.                             |
| `0x10`   | `rsa`             | Negotiates which RSA key sizes are supported.        |
| `0x11`   | `ecdsa`           | Negotiates which ECDSA key sizes are supported.      |
| `0x12`   | `aes`             | Negotiates which AES key sizes are supported.        |

`version` negotiates the sixteen-bit version. The answer list is encoded as a
sequence of sixteen bit little-endian integers. The first two are an inclusive
version range of supported versions, followed by versions explicitly not
supported. For example, the sequence `[5, 10, 7, 8]` indicates that the answer
set is `5, 6, 9, 10`. This encoding is chosen because most devices will support
a contiguous range of versions with very few explicitly forbidden versions.

`has_*` messages are "subcomponent" messages, which specify whether or not
a device supports an optional portion of Cerberus. They have an empty body,
since it's a yes/no question.

`rsa`, `ecdsa`, and `aes` are used to ask which key sizes are supported for
different algorithms. They are each encoded as single-byte bit flags, with
values according to those in the "RSA", "ECC", and "AES" fields of `Device
Capabilities`. Not answering these questions indicates that that particular
algorithm is unsupported.

## Vendor-Defined Questions

Vendors may also want to define their own questions to negotiate; these are
identified by very large question IDs (greater than 65535). Vendors *must*
choose question IDs at random.

This is more than enough entropy: given the birthday paradox rule of thumb that
the expected number of choices before a collision is `sqrt(pi/2 * N)`, where N
is the ID space, we get a value of `~80000` for `~2^32`; it is extremely
unlikely we will see anywhere close to that number of extensions.

Vendors are free to define the encoding of their answers as they see fit.

# Specification Changelist

- A section describing Challenge Protocol versioning early in the spec, probably
  after the Summary.
- A section describing the negotiation mechanism. This should go right before
  the Certificates section.
- New text describing the new `Protocol Negotiation` command encoding in the
  Command Format section.
- A table after the table of command bytes, which maps commands onto
  subcomponents (as described in [Cerberus-Defined
  Questions](#cerberus-defined-questions)).
- The min/max fields in the `CHALLENGE` should be replaced with the negotiated
  version.
- The `Device Capabilities` message should be deleted.
- `PROTOCOL_VERSIONS.md` must be created, and the first version `protocol-v0`
  minted.

# Implementation Guidance

The main implementation concern is keeping track of negotiation state. In
general, not all state needs to be kept around; only which keys were already
negotiated, and answers that one or the other side intends to use.
Implementations must also record whether or not negotiation state with a host
exists at all, for enforcing when renegotiation is required. This cost is
bounded by the number of questions a device knows how to answer.

# Alternatives Considered

See:
- [Capabilities2](https://github.com/opencomputeproject/Security/pull/21)
- [Version_Detection](https://github.com/opencomputeproject/Security/pull/23)

# Future Work

Negotiation becomes somewhat more load-bearing by containing a version, and
including it in the challenge signature will be necessary for protection against
downgrade attacks. A followup RFC should describe how to incorporate the
negotiation process (not just the negotiated key-value pairs) into the
signature.

This RFC does not describe norms and practices around when to mint versions nor
when to make the actual decision of deprecation; these are left up to the
maintainers or a potential future RFC.
