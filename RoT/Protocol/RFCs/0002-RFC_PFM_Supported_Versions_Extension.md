* Name: Extension for the PFM Supported Versions Command
* Date: 2021-05-11
* Pull Request: [#18](https://github.com/opencomputeproject/Security/pull/18)

# Objective

In the original design, there was only a single firmware component supported in
a PFM.  The command to get the list of supported versions in a PFM didn't require
any additional granularity.  With the introduction of version 2 of the PFM
structure, there can now be multiple independent firmware components, each with
their own list of supported firmware versions.  It is not always useful to get
the full list of all versions for all components within a PFM.  Returning
supported versions in this way would make it more challenging to search for a
specific version of a specific component.  There needs to be a way to target a
specific component's supported versions.

# Proposal

Update the Get Platform Firmware Manifest Supported Firmware command to have
optional fields added to the request to specify the firmware component ID to
query.  If these optional fields are not provided, every version for every
component will be returned.

# Specification Changelist

The request structure for Get Platform Firmware Manifest Supported Firmware
(https://github.com/opencomputeproject/Security/blob/master/RoT/Protocol/Challenge_Protocol.md#get-platform-firmware-manifest-supported-firmware)
will be updated to include:
1. A single byte length field that will specify the length of the firmware ID.
2. A variable length, null-terminated firmware ID string.

# Implementation Guidance

When dealing with v1 PFMs, specifying any firmware ID would return an error, as
there are no component IDs specified in this format.  The command without the
optional parameters will behave as it always has.

# Alternatives Considered

1.	Since the current command simply returns ASCII strings, the response payload
can just be filled with a list that includes a heading for each type of firmware
using the FW ID followed by a list of supported versions.  This would work cleanly
when the purpose is just to display what is currently supported, but would not be
so good in scenarios where the list is being queried to make decisions about the
supported list (e.g. when checking for support during host FW updates).  In this
case, you only want to check support against a specific FW type, not all FW types.
This would require a lot of assumptions on the client side about the response
contents and might not be cleanly backwards compatible.
2.	Deprecate the existing command and replace it with a new one that provides
the necessary semantics to support this flow.  This would clearly delineate
between devices that support the new information vs. devices that don’t.
Ultimately, this level of demarcation of functionality was not seen as
necessary.
3.	A less obvious mapping would be to support independent ports for each
component within the manifest.  But this would be challenging to know what ports
map to whay component.  Since there is a variable number of firmware types in
each PFM, the port mapping wouldn’t be static.  This would require no changes to
the current command structure, but is a potentially confusing interface.

# Future Work

Optimizations necessary to read back large lists of supported firmware versions.
