* Name: Version_Detection
* Date: 2021-09-20
* Pull Request: [#23](https://github.com/opencomputeproject/Security/pull/23)

# Objective

Cerberus does not currently expose a particularly precise interface for
discovering the version of a remote device being challenged or otherwise
interacted with. However, Cerberus does make reference to these versions as part
of the `CHALLENGE` message.

Cerberus also has significant "optional" components, though there is no way to
discover whether a remote device supports them. In theory this would be provided
to a PA-RoT via a manifest, but it some cases it may be useful to query this
information dynamically.

This RFC describes:
- A formal versioning scheme for the overall protocol, as well as its optional
  subcomponents.
- A protocol command for querying this version information.
- A way for vendors to allow for unambiguous querying of their vendor
  extensions.
- A story for deprecation of commands.

# Proposal

Although Cerberus has some minor prior art around version numbers, we will be
starting over from a clean slate.

## Protocol Version Numbers

A Cerberus version is a sixteen-bit opaque value, mapped to a commit hash of
this repository. Protocol version N is described by the repository at that
commit.

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

After this RFC is merged and implemented, a new version, version 0, must be
created immediately.

## Subcomponents

A subcomponent is a set of optional Cerberus commands that need to be provided
together, such as the PFM-related commands. Each subcomponent has a byte
associated with it.

The subcomponents, their identifying bytes, and the list of commands in each
should be listed in a table, just past the list of all commands. All optional
commands must be part of a subcomponent, and a command may be part of multiple
components.

## The `Protocol Version` Command

We define a new command, `Protocol Version`, using command byte `0x05`, and
marked as *required*. Its purpose is to negotiate a shared version for the
devices to communicate over.

A request has the following structure:

```c
struct ProtocolVersionRequest {
  uint8_t reserved;  // Must be zero.

  uint16_t min_version;
  uint16_t max_version;

  uint8_t bad_versions_len;
  uint16_t bad_versions[bad_versions_len];

  uint8_t extns_len;
  struct {
    uint8_t len;
    uint8_t name[len];
  } extns[extns_len];
}
```

The requester provides the minimum and maximum versions (inclusive) it
understands, as well as a list of versions it refuses to use (for security
or other reasons). It also provides a list of vendor-defined extensions it
knows how to speak.

Vendor-defined extensions are defined by strings, to
avoid running into the usual "private use area" problems in code-point
allocation. Vendors should choose strings that incorporate their name into them
to avoid chances of collision. Period-separated names are ideal:
- `widgetsinc.unsealing-with-chacha20`
- `acmeco.fancy-pcie-update-scheme`

The response looks like this:

```c
struct ProtocolVersionResponse {
  uint16_t version;

  uint8_t extns_len;
  uint16_t extn_versions[extns_len];
};
```

This provides the responder-chosen version, and the versions of the requested
vendor-defined extensions. A version of `0xffff` is used as a sentinel to
indicate that the extension was unrecognized.

All messages that follow must use the chosen version number. This number not
only specifies which messages are supported, but also which format to use for
parsing commands, since that may vary across versions. The `Protocol Version`
command, however, is unversioned. A reserved byte is included at the top of
the command in case we ever need to change it.

A new error code, "Unnegotiated Version" (code `0x05`), should be returned by
devices if no version has been negotiated yet with the requesting device.
Requesters which had previously negotiated a version, but which recieve this
message, should re-negotiate a version.

## Updates to `CHALLENGE`

Because only a single protocol version is negotiated, we can replace the version
range in the `CHALLENGE` with the single negotiated version. This doesn't change
the layout of the `CHALLENGE` in a meaningful way, since we are replacing two
eight-bit fields with one sixteen-bit field.

## Evolution and Deprecation Process

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

# Specification Changelist

The following changes are required:
- The creation of `PROTOCOL_VERSIONS.md` as described above.
- Prose in a section immediately before the `RoT Commands` section that
  describes the Cerberus Protocol versioning scheme, including protocol
  subcomponents and vendor-defined extensions.
- A table immediately after the table of all commands, which defines the
  protocol subcomponents.
- Add the new error code to the error codes table.
- A new message definition, after `Device Information`, for the new `Protocol
  Version` command. This shall include prose of the negotiation process.
- The min/max version fields in the `CHALLENGE` should be replaced with the
  single negotiated version.

# Implementation Guidance

Implementers which support a range of Cerberus versions will need to maintain a
"currently negotiated version" for each bus they can service requests from.
This should not be a significant cost, given they already need to maintain
similar information for sessions.

Because requesters must know how to re-establish a negotiated version, an
implementer can choose to record only a single version at a time.

# Alternatives Considered

An alternative is to have requester-chosen, rather than responder-chosen,
versions. This doesn't have any specific benefits for us, but it does have the
downside that we need to deal with different versions having different layouts
for commands, meaning that the requester still needs to inform the responder
about which protocol version it wishes to use.

This could be worked around by being careful about how commands are evolved, but
it's likely to be rare enough that command formats change that the complexity
would be worth it.

# Future Work

This RFC does not describe norms and practices around when to mint versions nor
when to make the actual decision of deprecation; these are left up to the
maintainers or a potential future RFC.
