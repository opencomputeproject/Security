<!--
  Style Guide
  - Make sure to wrap everything to 80 columns when possible.
  - Sentence-ending periods are followed by two spaces.
  - Hexadecimal integers should be formatted as `0xff`.
-->

Project Cerberus: Attestation Specification
=====

Authors:
- Christopher Weimer, Principal Firmware Engineer, Microsoft
- Akram Hamdy, Senior Firmware Engineer, Microsoft

## Revision History

- v1.00 (2022-07-01)
  - Initial version.

# Summary

The goal of the [Cerberus Challenge Protocol](https://github.com/opencomputeproject/Security/blob/master/RoT/Protocol/Challenge_Protocol.md)
is to ensure only authenticated and compliant firmware is running on all
components in a system.  This is achieved through both direct measurements of
protected firmware and measurement challenges of remote devices.  The Cerberus
Challenge Protocol provides the primitives necessary to ensure the integrity and
authenticity of a platform but does not provide many specifics about how this is
achieved.  This specification provides those details.

# Device Measurements

The basis for any attestation flow is the ability to maintain and report
measurements for firmware and configuration that is loaded by the device.  In
Cerberus devices, measurements are reported through Platform Measurement
Registers (PMR), which have similar behavior and attributes as [TPM PCRs](https://trustedcomputinggroup.org/work-groups/trusted-platform-module/).

Each PMR represents a single measurement, extended by multiple individual
components.  The extend operation follows the same formula as for TPM PCRs, such
that the PMR value is represented as `PMRnew = HASH (PMRold || Data)`.  The
hashing algorithm used to generate the PMR and the initial PMR values (i.e. the
PMR value before any extend operation has been performed) may be different
between each Cerberus device.

## Platform Measurement Register (PMR)

The Cerberus Challenge Protocol allows for 5 PMR values (PMR0 - 4), each of
which can be used to measure different types and amounts of data depending on
the specific implementation.  It is not required that a device support all 5
PMRs, but it is mandatory for every device to at least support PMR0.  Beyond
that, there is a lot of flexibility regarding how they are used.  What is
documented here are considered 'best practices' for different classes of
Cerberus device.

### PMR0

As the only required PMR, every device must generate this measurement for
attestation and there is an expectation regarding the minimum set of elements
that will be included in this measurement.  There can be other entries, as
required by specific implementations, but PMR0 must contain at least the
following list of measurements.

- All layers of mutable firmware that were loaded.  Following a [TCG DICE](https://trustedcomputinggroup.org/work-groups/dice-architectures/)
  model of boot, all layers 0 through N need to be measured, where layer N is
  the running application firmware.
- Secure boot keys.  This includes both the hardware-backed root key and the key
  manifest used to load subsequent stages of boot.
- Current anti-rollback counters.

Simple RoT devices that are only attesting to their own firmware state may
reasonably not have any PMRs other than PMR0.

### External RoT

An external Root of Trust is a device that will be interposed on a processor's
non-volatile firmware storage (e.g. SPI flash) and will authenticate and attest
to the firmware state on behalf of the processor.  Since an external RoT is
attesting to more than just its own state, it will need to use more of the
available PMRs.

For these devices, PMR0 should contain firmware measurements of the RoT device
itself.  It should not include measurements of any manifests or other
configuration applied to the device.  This provides a consistent measurement for
attesting the RoT state.

PMR1 and PMR2 should contain measurements of any configurable state.  This will
include any PFMs that the device is using for authenticating the processor
firmware as well as the result of the authentication.  How the measurements are
actually organized in these two PMRs is less important.  For example, in the
case of a RoT protecting two processors, the device could equally put all
measurements in a single PMR or have one processor in PMR1 and the other in
PMR2.

PMR3 and PMR4 are provided to offer similar functionality as a TPM.  A
processor could extend its own data into the PMRs managed by Cerberus.  The
state of these PMRs could then be used as part of attestation claims.

### Platform RoT

The PA-RoT (Platform Active Root of Trust) is responsible for attesting to the
health of all components within a system.  It may or may not also be an
External RoT, but it has similar expectations around PMR usage.

PMR0 will be the firmware measurements for the PA-RoT device and PMR1 and PMR2
will be use to report the configuration and attestation state of system
components.  This includes any PCDs or CFMs used to attest components and the
results of the attestation.

PMR3 and PMR4 may or may not be necessary in this configuration.  Depending on
the implementation, it may still be desirable to have a measurement available
for a processor in the system to extend.

### Intrinsic RoT

An intrinsic Root of Trust is device within a SoC package and on on-die, but
separate from the main application processor.  The intrinsic RoT enforces secure
boot and measurement of all firmware loaded into the application processor and
may optionally provide management of additional measurements from the
application processor run-time.  Since the intrinsic RoT is embedded within the
device, it likely will not have a use for PFMs as an external RoT does.

PMR0 would contain the measurements of just the intrinsic RoT firmware.  The
other PMRs (1-4) could be used in any combination to report measurements of
firmware and security state for the application processor.  The same type of
information that is measured for the RoT in PMR0 should be measured for the
application processor.  If the RoT exposes measurements that can be extended by
the application processor, it would use PMR3 or PMR4 for this purpose.

If the intrinsic RoT is also being used used as a PA-RoT or to attest other
components within the system, it would need to allocate at least one PMR to
measuring the PCD, CFM, and attestation results.

# Attesting Measurements

Cerberus Challenge Protocol provides both a stateful (`Challenge`/`Get PMR`) and
stateless (`Message Unseal`) mechanism for attesting device measurements.  Of
the attestation commands, only Challenge is required of every device since this
is likely the only command needed to attest a simple AC-RoT that only contains
measurements in PMR0.  Devices that support additional PMRs are highly
encouraged to implement `Get PMR`.  Implementation of `Message Unseal` is
encouraged for PA-RoT devices to provide flexibility for distributed attestation
workflows.

This section just describes the mechanism for securely attesting measurements.
It does not address how it is known if the reported measurements are correct.
Within a system, this is handled by the [CFM](#component-firmware-manifest),
described in detail later.

## Device Certificate

All attestation workflows utilize a device-unique key that is endorsed by the
DICE identity certificate for the device.  Which device key is used depends on
the specific attestation flow being executed and device implementation details,
but the process by which this key is retrieved and validated is the same.
Cerberus supports up to 8 different certificate chains (slots), but it is only
required for a device to support a single certificate chain to support
attestation.

Getting and validating the certificate chain for a device involves executing a
`Get Digests` request followed by multiple `Get Certificate` requests in the
following sequence.

1. Issue `Get Digests` for the target certificate chain.  This provides a hash
   of every certificate in the chain, which can be used for two purposes.
   First, it tells the requestor exactly how many certificates are in this
   chain. Second, it provides a way for a requestor to cache information from
   previous sessions and detect if certificates have changed.  When caching
   certificate information, the requestor can cache the full certificate, a
   digest of each certificate, or a digest of the entire chain and be able to
   verify if the reported certificate chain matches what was previously
   validated.
2. Issue `Get Certificate` for each certificate in the chain.  The last
   certificate will be an end entity certificate for the key that will be used
   for attestation requests.  If a valid cache exists that matches the current
   digests, any number of these requests may be skipped.
3. Verify that the root CA is trusted.  How this trust is established depends on
   the requestor.  Another Cerberus device will use information in the CFM to
   determine if the root CA is trusted.
4. Once the root CA is known to be good, X.509 path validation is executed over
   the entire certificate chain, proving the attestation certificate has been
   endorsed by the root CA.
5. If validation of the certificate chain fails, the device is not trusted.  No
   further attestation is required for this device.

```
                Requestor                           Device
                    |  ----------Get Digests------->  |
                    |  <----------Response----------  |
                    |                                 |
                   /|  --------Get Certificate----->  |
    For Each Cert | |                                 |
                   \|  <----------Response----------  |
                    |                                 |
                    | --                              |
                    |   | Check Root CA               |
                    | <-                              |
                    |                                 |
                    | --                              |
                    |   | Validate Cert Chain         |
                    | <-                              |
```

Once a trusted device certificate has been obtained, it can be used as part of
the secure exchanges to attest the device state.

## Challenge / Get PMR

Attesting a Cerberus device using the `Challenge` or `Get PMR` commands is the
simplest mechanism to check PMR values.  The response for both of these commands
just returns the raw value of a PMR.  In the case of `Challenge`, the PMR that
is returned is PMR0.  With `GetPMR`, any PMR supported by the device can be
queried.  Freshness nonces in the request and signatures using the previously
acquired attestation key provide binding to the specific device and anti-replay.

## Message Unseal

`Message Unseal` can also attest to the state of device PMRs, but without
necessarily getting the PMR values from the device directly.  This flow works if
the Attestor has some secret or other artifact that is required for the system
to function normally.  An external Attestor can use knowledge of a device's
certificate and the expected PMR values to Seal this system secret to the
expected PMR values.  The target system will only be able to Unseal this secret
if the device with the matching identity certificate chain has the expected PMR
values.  The following sequence is used.

1. Receive and validate the device certificate chain.
2. Determine the set of expected PMR values for the device.
3. Generate a random seed.
   - If the Seal/Unseal will use ECDH, this seed is the ECDH shared secret using
     a random ECC keypair and the Cerberus ECC public key.
   - If the Seal/Unseal will use RSA, this seed is a random number, which is
     then encrypted with the Cerberus RSA public key.
4. From the seed, derive a 256-bit HMAC signing key using NIST800-108 Counter
   Mode.
   - Ki = seed
   - Label = "signing key"
   - No Context
   - r = 32
   - `key = HMAC (seed, 0x00000001 || signing key || 0x00 || 0x00000100)`
5. From the seed, derive a 256-bit symmetric encryption key using NIST800-108
   Counter Mode.
   - Ki = seed
   - Label = "encryption key"
   - No Context
   - r = 32
   - `key = HMAC (seed, 0x00000001 || encryption key || 0x00 || 0x00000100)`
6. Use the encryption key to encrypt the system secret.  The encryption mode and
   algorithm can be defined by the platform.  If a different key length is
   required, the 256-bit encryption key can can be further morphed through
   additional KDFs to derive the final encryption key.
7. Generate the sealing policy.  This is an array of five 64 byte values, one
   for each PMR.  Unused bytes due to measurements not using the full 64 byte
   allotment should be set to 0.  Unused PMRs, either because the device doesn't
   support it or because the value of the PMR does not matter, should be set
   to 0.  This array maps directly to the `PMRx Sealing` values in the
   `Message Unseal` request structure.
8. Calculate the HMAC signature for the sealed payload, such that
   `sig = HMAC (signing key, ciphertext || sealing policy)`
9. Send the encrypted system secret, the sealing policy, HMAC signature, and the
   seed to the target system.
   - If sealed using ECDH, the seed is the random ECC public key generated by
     the attestor.
   - If sealed using RSA, the seed is the RSA encrypted random number.
10. Once the target system receives the complete package, it must execute the
    `Message Unseal` request, providing the four pieces of data to the Cerberus
    device.
11. The Cerberus device will regenerate the same signing and encryption keys
    using the provided seed and validate the HMAC signature over the payload.
    If the HMAC signature is valid and the PMR values match those specified in
    the sealing policy, the encryption key will be returned to the requestor.
12. The target system can then use the encryption key to decrypt the system
    secret or execute any other necessary actions needing that key.

Note:  The encryption key generated by this flow can be sent in plain text from
the device.  The device will also regenerate the same encryption key based on
the same seed, as long as the PMR values remain the same, with no understanding
of who is sending the request or how many times this payload has been Unsealed.
For these reasons, the encryption key must be considered a one-time use key and
the system secret that is Sealed with the key also needs to be one-time use.  If
either of these is violated, the system is exposed to compromise through replay.

```
   Attestor                         System                          Device
      |                               |  -------Get Cert Chain----->  |
      |                               |  <---------Response---------  |
      |  <-----Send Cert Chain------  |                               |
      |                               |                               |
      | --                            |                               |
      |   | Validate Cert Chain       |                               |
      | <-                            |                               |
      | --                            |                               |
      |   | Determine Expected PMR    |                               |
      | <-                            |                               |
      | --                            |                               |
      |   | Seal System Secret to PMR |                               |
      | <-                            |                               |
      |                               |                               |
      |  ----Send Sealed Secret---->  |                               |
      |                               |  -------Message Unseal----->  |
      |                               |  <---------Response---------  |
      |                               |                               |
      |                               |  ---Message Unseal Result-->  |
      |                               |  <---------Response---------  |
      |                               |                               |
      |                               | --                            |
      |                               |   | Decrypt System Secret     |
      |                               | <-                            |
```

One important aspect of the `Message Unseal` flow is that the Attestor must know
the PMR values of the device.  Additionally, it must know exactly what these
values are and not just a range of acceptable values.  The Cerberus Challenge
Protocol provides several mechanisms to determine the PMR state.  Alternatively,
the Attestor could be provided with sufficient offline knowledge to know what
these values should be.

- **Get PMR**<br>
The first and most obvious way to get the device PMR values would be to issue
`Get PMR` requests.

- **Attestation Log**<br>
Cerberus devices have the option to provide an Attestation Log of all
measurements recorded for each PMR.  Using the `Get Log` request, the
Attestation Log can be retrieved for the device and sent to the Attestor along
with the certificate chain.  The Attestation Log will not only have the final
value for each PMR, but each measurement that contributed to that value.  This
gives the Attestor more granularity with which to validate the PMR state.

  If the Attestation Log alone is not sufficient to validate the PMR state, it
is possible to retrieve the actual data measured for each measurement in the
Attestation Log.  If a device supports the `Get Attestation Data` request, this
can be issued for each entry in the Attestation Log and also sent to the
Attestor.

- **TCG Log**<br>
If the Cerberus device supports it, the `Get Log` request can be used to
retrieve a TCG-compliant measurement log that can be sent to the Attestor.  This
log will contain the same information as the combination of the Cerberus
Attestation Log and `Get Attestation Data`.

# Manifests

Cerberus manifests are configuration files that get applied to devices to drive
the authentication and attestation of firmware.  Different manifest types apply
in different scenarios, but are based in the same fundamental structure.
Manifests are designed with the expectation they would be stored on flash
external to the device and there would not be sufficient memory in the device to
have the entire manifest in RAM at any one time.  It would obviously be
functional to have the entire manifest in RAM, but there may be more efficient
ways of structuring the required data for this use case.

## General Manifest Structure

Every manifest contains a minimum of three main sections: the manifest header,
table of contents, and signature.  Every manifest starts with a fixed length
header and ends with a signature, whose length is specified in the header.
Using only the header information, the complete manifest can be verified.  The
table of contents immediately follows the manifest header and describes all the
information contained within the manifest.

The manifest itself is composed of a variable number of elements, whose type and
order are specified in the table of contents.  The table of contents is a
variable length structure containing a header, one entry for each element within
the manifest, a list of optional hashes for elements, and an overall table hash
that can be used for independent validation of the table data.

Reserved fields in manifests should be set to 0, but the parser does not
validate this value.  It is possible future versions of the manifest will use
the reserved fields for some data, and to maintain backwards compatibility, the
parser must not reject manifests that have non-zero values in these fields.  The
same holds for reserved bits within specified fields.

All fields larger than one byte are stored in little endian.  This also includes
fields that do not fall on byte boundaries.  To ease parsing, fields that are
not an even multiple of bytes are broken up to align with memory byte
boundaries.  For example:
- A 12-bit field [11:0], starting on a memory byte boundary is stored as <br>
  `byte0[7:0] = data[7:0]`<br>
  `byte1[7:4] = data[11:8]`
- A 12-bit field [11:0], starting in the middle of a byte is stored as <br>
  `byte0[3:0] = [3:0]`<br>
  `byte1[7:0] = [11:4]`

The exception to this are fields that are an array of bytes, particularly
digests.  Unless otherwise specified, all digests and other byte arrays are
stored in big endian.

All lengths are represented in bytes.

### Manifest Header

`bitfield Manifest.Header`
| Type            | Name               | Description                           |
|-----------------|--------------------|---------------------------------------|
| `16`            | `total_length`     | Total length of all data in the manifest, including the signature. |
| `16`            | `manifest_type`    | Identifier indicating the type of manifest. |
| `32`            | `version_id`       | Version identifier for the manifest.  The version ID is a monotonically increasing value, preventing rollback to earlier manifest versions. |
| `16`            | `signature_length` | Length of the signature appended to the end of the manifest data. |
| `PublicKeyType` | `public_key_type`  | The type of public key used to generate the manifest signature. |
| `KeyStrength`   | `key_strength`     | Key strength used to generate the manifest signature. |
| `HashType`      | `hash_type`        | Hash algorithm used to generate the manifest signature. |
| `0x00`          | `_`                | Reserved.                             |

`enum Manifest.PublicKeyType`
| Value   | Name    | Description            |
|---------|---------|------------------------|
| `00b`   | `rsa`   | The public key is RSA. |
| `01b`   | `ecc`   | The public key is ECC. |

`enum Manifest.KeyStrength`
| Value   | Name              | Description                             |
|---------|-------------------|-----------------------------------------|
| `000b`  | `rsa_2k_ecc_256`  | The key is 2048-bit RSA or 256-bit ECC. |
| `001b`  | `rsa_3k_ecc_384`  | The key is 3072-bit RSA or 384-bit ECC. |
| `010b`  | `rsa_4k_ecc_521`  | The key is 4096-bit RSA or 521-bit ECC. |

`enum Manifest.HashType`
| Value   | Name        | Description                             |
|---------|-------------|-----------------------------------------|
| `000b`  | `sha2_256`  | The hash was calculated using SHA2-256. |
| `001b`  | `sha2_384`  | The hash was calculated using SHA2-384. |
| `010b`  | `sha2_512`  | The hash was calculated using SHA2-512. |

### Manifest Table of Contents

`bitfield Manifest.TOC.Header`
| Type        | Name           | Description                                   |
|-------------|----------------|-----------------------------------------------|
| `8`         | `entry_count`  | The number of entries in the table.           |
| `8`         | `hash_count`   | The number of entries in the table that have an associated hash.  It is not required that all entries have a hash. |
| `00000b`    | `_`            | Padding for the unused HashType bits.  This must be 0. |
| `HashType`  | `hash_type`    | Specifies the type of hash added for table validation.  This is also the type of hash used to generate individual element hashes. |
| `0x00`      | `_`            | Reserved.                                     |

Each element within the manifest must have an entry in the Table of Contents.
The entries must be in order based on position within the manifest.

`bitfield Manifest.TOC.Entry`
| Type   | Name       | Description                                            |
|--------|------------|--------------------------------------------------------|
| `8`    | `type_id`  | An identifier indicating the element type.             |
| `8`    | `parent`   | A Type ID representing the parent for this element.  The first element preceding this element with the matching Type ID is the parent element.  If the element has no parent, this will be set to `0xff`. |
| `8`    | `format`   | Format version of the data contained in the element.  New format versions must be backwards compatible with older versions.  Fields cannot be moved or reordered. |
| `8`    | `hash_id`  | Identifier specifying the index in the hash table for this element.  An element hash is optional.  A hash ID equal to or greater than the total number of hashes in the table indicates there is no hash for the element.  It is recommended that a hash ID of `0xff` should be used for this purpose. |
| `16`   | `offset`   | The offset from the start of the manifest where the element data is located. |
| `16`   | `length`   | Length of the element data.                            |

After the element entries, there is a hash table for element validation.  This
table is an array of equal sized hashes, whose length is determined by the hash
algorithm specified in the Table of Contents header.  All hashes must be
calculated using the same algorithm.  Elements that have a hash must have the
hash validated prior to using the data, even if only a portion of the element
data is required in memory.  The hash is calculated over the entire length
specified in the Table of Contents entry.

The Table of Contents can be consumed in a couple of different ways, depending
on performance requirements and memory capabilities.

1. To optimize for memory consumption, the Table of Contents header and hash
   will be cached during manifest verification.  The rest of the Table of
   Contents and manifest elements remain on external flash.  Every element read
   would require a sequence of actions.
   - Read each element entry from flash to find the desired entry.
   - Read the hash entry, if available, for the desired entry.
   - Ensure all Table of Contents data on flash is hashed as part of these
     reads.
   - Compare the calculated Table of Contents hash against the cached hash to
     ensure validity.
   - Read and hash the element data.  Compare against the hash entry from the
     Table of Contents to ensure validity.

   <br>This flow is less performant, but only requires a small amount of data
   storage to handle any manifest.
2. If there is enough memory in the device, the Table of Contents could be read
   and cached in its entirety during manifest validation so that future element
   accesses are quicker.  Element accesses would not need to revalidate the
   Table of Contents and would only need to validate element data.

   This approach may not work on memory constrained devices since a Table of
   Contents with 255 entries and 255 hashes using SHA2-512 would require a cache
   of about 18k of memory.

### Manifest Elements

While the order and number elements in most cases is variable, elements can be
restricted to specific contexts.  Certain elements may only be part of the
top-level structure (i.e., it will not have any parent element), and others
require a parent of a specific type.  Elements in the wrong context will be
ignored by the parser.  Additionally, elements can be defined as singletons.
Duplicate entries of singleton elements will be ignored.  All identifier strings
are limited to 255 bytes and do not have any terminator stored in the manifest.

There are a base set of elements defined that are common across all manifest
types.  Specific manifests can define additional types.

| Type ID        | Element Type        | Format | Top-Level | Singleton | Description |
|----------------|---------------------|--------|-----------|-----------|-------------|
| `0x00`         | `Platform ID`       |   1    |     X     |     X     | An identifier for the platform that uses this manifest. |
| `0x01 - 0x0f`  | `Reserved`          |        |           |           | Reserved for future use. |
| `0xe0 - 0xfe`  | `Vendor Extensions` |        |           |           | Reserved for vendor-specific element types. |
| `0xff`         | `Unavailable`       |        |           |           | Cannot be used as an element type since it is used as part of parent/child relationships. |

#### Platform ID Element

The Platform ID element is used to associate a specific manifest to a particular
type of platform.  It also serves as a secondary way to identify a specific
manifest.  Since manifest IDs are just integers and are likely to have overlap
across manifests of different types and for different systems, the combination
of manifest ID and platform ID provide a way to uniquely identify a specific
manifest.  Additionally, the platform ID is used during manifest verification to
ensure only a manifest for the same platform is used used to replace an existing
manifest.

While the manifest structure does not require the presence of any particular
element, manifest parsers will likely reject a manifest as not valid if it
doesn't contain a platform ID element.

`bitfield Manifest.PlatformID`
| Type                | Name              | Description                        |
|---------------------|-------------------|------------------------------------|
| `8`                 | `plat_id_length`  | Length of the platform ID string.  |
| `0x000000`          | `_`               | Reserved.                          |
| `plat_id_length`    | `plat_id_string`  | Platform ID string.  This is an ASCII string that is not null terminated. |
| `ALIGN (32)`        | `_`               | Zero padding.                      |

## Platform Firmware Manifest

The Platform Firmware Manifest (PFM) is used by external RoT devices to validate
the contents of a processor's firmware storage.  The PFM structure is tailored
for SPI flash devices, though it is possible the same structure could apply to
other storage types.

A PFM contains a list of allowed versions of firmware that can be loaded by the
processor and provides the information necessary to authenticate the image.  The
structure is flexible to account for many different contents and layouts of
flash.  In the simple case, there is a single firmware image contained in a
contiguous block of flash, but there can be scenarios where a single flash
device has multiple firmware components that are each loaded independently by
the processor.  All of these scenarios can be described in a PFM.

A PFM will report a `manifest_type` of `0x706d` in the manifest header.  The PFM
defines the following additional manifest element types.

| Type ID  | Element Type       | Format | Top-Level | Singleton | Description |
|----------|--------------------|--------|-----------|-----------|-------------|
| `0x10`   | `Flash Device`     |   0    |     X     |     X     | Defines global information relevant to the entire flash device. |
| `0x11`   | `Firmware`         |   1    |     X     |           | Defines a single piece of firmware that is present on the flash.  This will only be valid if there is a Flash Device element in the manifest. |
| `0x12`   | `Firmware Version` |   1    |           |           | Description for single version of firmware.  This will only be used as a child to a Firmware element. |

### Flash Device Element

Each PFM is designed to describe the contents of a single flash device.  The
Flash Device element provides overall information to the RoT about the physical
flash that may be necessary for authentication of the contents.

Note:  In configurations that use two redundant flashes for a single processor,
there would be a single PFM for both flash devices.  It is expected that the RoT
is managing access to one flash or the other at any point in time.  The layout
and validation of either device should be the same.

`bitfield PFM.FlashDevice`
| Type     | Name         | Description                                        |
|----------|--------------|----------------------------------------------------|
| `8`      | `blank_byte` | The value that represents an unused byte of flash.  During validation, all unused regions of flash will be checked against this value to ensure no unexpected data is stored. |
| `8`      | `fw_count`   | The number of firmware components stored in the flash. |
| `0x0000` | `_`          | Reserved.                                          |

### Firmware Element

A Firmware element describes a logical partition of a flash device, representing
a single, updatable piece.  Most processors will only require a single Firmware
element since there is only one image loaded from flash.  Some more complicated
SoCs, however, may have multiple independent cores that each load firmware.  In
this case, it is possible to partition the flash so each component is validated
individually.

`bitfield PFM.Firmware`
| Type            | Name              | Description                            |
|-----------------|-------------------|----------------------------------------|
| `8`             | `version_count`   | The number of allowed versions for this firmware component. |
| `8`             | `fw_id_length`    | Length of the firmware component identifier. |
| `0000000b`      | `_`               | Reserved.                              |
| `UpdateApplied` | `run_time_update` | Flag to indicate if updates can be applied without a reboot. |
| `0x00`          | `_`               | Reserved.                              |
| `fw_id_length`  | `fw_id_string`    | ASCII string identifier for the firmware component.  This is not null terminated. |
| `ALIGN(32)`     | `_`               | Zero padding.                          |

`enum PFM.Firmware.UpdateApplied`
| Value  | Name       | Description                                            |
|--------|------------|--------------------------------------------------------|
| `0b`   | `reboot`   | Updates are applied only on reboot of the processor.   |
| `1b`   | `run_time` | Updates can be applied without notifying the RoT of a reboot. |

### Firmware Version Element

The Firmware Version element describes all the information necessary to
authenticate a single version of firmware.  Each firmware component can support
multiple versions in one PFM.  If the firmware on flash does not match the
information contained in a Firmware Version element, it is considered to be
untrusted.

`bitfield PFM.FirmwareVersion`
| Type                     | Name             | Description                    |
|--------------------------|------------------|--------------------------------|
| `8`                      | `img_count`      | The number of signed images contained in this version. |
| `8`                      | `rw_count`       | The number of R/W data regions used by this version. |
| `8`                      | `version_length` | Length of the version identifier string. |
| `0x00`                   | `_`              | Reserved.                      |
| `32`                     | `version_addr`   | Address on flash where the version string is stored.  This must be within a signed image that is validated every time. |
| `version_length`         | `version_string` | ASCII string identifier for the version.  This is not null terminated. |
| `ALIGN(32)`              | `_`              | Zero padding.                  |
| `RWRegion(rw_count)`     | `rw_regions`     | List of flash regions that contain R/W data and therefore cannot be statically authenticated. |
| `SignedImage(img_count)` | `images`         | List of hashes that must be used to authenticate the contents of flash. |

`bitfield PFM.FirmwareVersion.RWRegion`
| Type          | Name           | Description                                 |
|---------------|----------------|---------------------------------------------|
| `000000b`     | `_`            | Reserved.                                   |
| `RWOperation` | `auth_fail_op` | The operation that the RoT should take on this R/W region when an update fails authentication. |
| `24`          | `_`            | Reserved.                                   |
| `Region`      | `region`       | The flash region that contains R/W data.    |

`bitfield PFM.FirmwareVersion.SignedImage`
| Type                   | Name           | Description                        |
|------------------------|----------------|------------------------------------|
| `00000b`               | `_`            | Padding for the unused HashType bits.  This must be 0. |
| `Manifest.HashType`    | `hash_type`    | Specifies the type of hash used for image authentication. |
| `8`                    | `region_count` | The number of flash regions to hash for this image. |
| `0000000b`             | `_`            | Reserved.                          |
| `MustValidate`         | `validate`     | A flag to indicate if an image should be validated on each boot or not. |
| `0x00`                 | `_`            | Reserved.                          |
| `hash_type`            | `img_hash`     | Expected hash for the image data.  |
| `Region(region_count)` | `_`            | The list of flash regions that should be hashed. |

`bitfield PFM.FirmwareVersion.Region`
| Type  | Name         | Description                                    |
|-------|--------------|------------------------------------------------|
| `32`  | `start_addr` | First valid address within the defined region. |
| `32`  | `end_addr`   | Last valid address within the defined region.  |

`enum PFM.FirmwareVersion.MustValidate`
| Value  | Name        | Description                                           |
|--------|-------------|-------------------------------------------------------|
| `0b`   | `updates`   | The firmware image is chain loaded from an image validated on boot, so only validate it when there is an update. |
| `1b`   | `each_boot` | Validate the firmware image on every system boot.     |

`enum PFM.FirmwareVersion.RWOperation`
| Value  | Name       | Description                                            |
|--------|------------|--------------------------------------------------------|
| `00b`  | `nothing`  | Do nothing with the R/W region.                        |
| `01b`  | `restore`  | Restore the region from backup, if possible.           |
| `10b`  | `erase`    | Erase the entire R/W region.                           |
| `11b`  | `_`        | Reserved (do nothing).                                 |

### XML Representation for PFM Generation

Each version of each firmware component needs to have a metadata file created
that can be used to generate a PFM.  This is an example using XML to store that
metadata that can be consumed by generation scripts to create the binary format.

```xml
<Firmware type= “Identifier” version="Identifier" platform=”Identifier”>
	<VersionAddr>0x00000000</VersionAddr>
	<UnusedByte>0xff</UnusedByte>
	<RuntimeUpdate>false</RuntimeUpdate>
	<ReadWrite>
		<Region>
			<StartAddr>0x00000000</StartAddr>
			<EndAddr>0x00000000</EndAddr>
			<OperationOnFailure>Nothing</OperationOnFailure>
		</Region>
		<Region>
			<StartAddr>0x00000000</StartAddr>
			<EndAddr>0x00000000</EndAddr>
			<OperationOnFailure>Restore</OperationOnFailure>
		</Region>
	</ReadWrite>
	<SignedImage>
		<Hash>
			0x000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f
		</Hash>
		<HashType>SHA256</HashType>
		<Region>
			<StartAddr>0x00000000</StartAddr>
			<EndAddr>0x00000000</EndAddr>
		</Region>
		<Region>
			<StartAddr>0x00000000</StartAddr>
			<EndAddr>0x00000000</EndAddr>
		</Region>
		<ValidateOnBoot>true</ValidateOnBoot>
	</SignedImage>
	<SignedImage>
		<Hash>
			0x000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f202122232425262728292a2b2c2d2e2f
		</Hash>
		<HashType>SHA384</HashType>
		<Region>
			<StartAddr>0x00000000</StartAddr>
			<EndAddr>0x00000000</EndAddr>
		</Region>
		<Region>
			<StartAddr>0x00000000</StartAddr>
			<EndAddr>0x00000000</EndAddr>
		</Region>
		<ValidateOnBoot>false</ValidateOnBoot>
	</SignedImage>
</Firmware>
```

| Field                | Description                                           |
|----------------------|-------------------------------------------------------|
| `type`               | Firmware component string identifier.                 |
| `version`            | Firmware version string identifier.                   |
| `platform`           | Platform string identifier.                           |
| `VersionAddr`        | Address on flash of the version string identifier.    |
| `UnusedByte`         | Byte to check for in unused flash regions.            |
| `RuntimeUpdate`      | Flag to indicate if the firmware can update at run-time. |
| `ReadWrite`          | Defines a single R/W data region.                     |
| `Region`             | Defines a region of contiguous flash.                 |
| `StartAddr`          | First valid address within the region.                |
| `EndAddr`            | Last valid address within the region.                 |
| `OperationOnFailure` | Operation to perform on a R/W data region after authentication failures. |
| `SignedImage`        | Defines a single image that needs authentication.     |
| `Hash`               | The expected hash for the image.                      |
| `HashType`           | The type of hash used.                                |

#### Allowed Values for XML Enums

Enums that have default values indicated mean the tag is optional.  If it is not
present in the XML, the default value will be used.

**OperationOnFailure**
- Nothing (default)
- Restore
- Erase

**HashType**
- SHA256 (default)
- SHA384
- SHA512

## Platform Configuration Data

The Platform Configuration Data (PCD) manifest is used to customize a Cerberus
device and specify policy information.  Additionally, the PCD is used to inform
the RoT of the platform it's in and how to reach other components to attest on
the platform.  This manifest is unique to each platform and is applied
independently of PFMs and CFMs.

A PCD will report a `manifest_type` of `0x1029` in the manifest header.  The PCD
defines the following additional manifest element types.

| Type ID | Element Type                            | Format | Top-Level | Singleton | Description |
|---------|-----------------------------------------|--------|-----------|-----------|-------------|
| `0x40`  | `RoT`                                   |   2    |     X     |     X     | Provides general configuration for the device. |
| `0x41`  | `SPI Flash Port`                        |   1    |           |           | A single protected firmware store on SPI flash.  This is a child of a RoT element. |
| `0x42`  | `I2C Power Management Controller`       |   1    |     X     |           | Defines a power management device utilized by the RoT. |
| `0x43`  | `Component with Direct I2C Connection`  |   1    |     X     |           | Defines an AC-RoT connected directly to the device over I2C. |
| `0x44`  | `Component with MCTP Bridge Connection` |   1    |     X     |           | Defines an AC-RoT connected through an MCTP bridge. |

### RoT Element

The RoT element contains configuration options for the Cerberus device and the
firmware it directly protects.  The RoT communicates to the BMC and other
devices over I2C.

`bitfield PCD.RoT`
| Type       | Name                                     | Description          |
|------------|------------------------------------------|----------------------|
| `0000000b` | `_`                                      | Reserved.            |
| `RoTType`  | `rot_type`                               | Indicates the type of RoT in the attestation hierarchy. |
| `8`        | `port_count`                             | The number of external processors directly protected by an external RoT. |
| `8`        | `components_count`                       | The number of remote components the RoT must attest. |
| `8`        | `rot_address`                            | The physical bus address for the RoT device. |
| `8`        | `rot_default_eid`                        | The default MCTP EID the RoT should use.  An external MCTP bridge can assign a different EID at run-time. |
| `8`        | `bridge_address`                         | Physical bus address for a connected MCTP bridge. |
| `8`        | `bridge_eid`                             | EID for the MCTP bridge. |
| `0x00`     | `_`                                      | Reserved.            |
| `32`       | `attestation_success_retry`              | Duration in milliseconds after device succeeds attestation to wait before reattesting. |
| `32`       | `attestation_fail_retry`                 | Duration in milliseconds after device fails attestation to wait before reattesting. |
| `32`       | `discovery_fail_retry`                   | Duration in milliseconds after device fails a discovery step to wait before retrying. |
| `32`       | `mctp_ctrl_timeout`                      | MCTP control protocol response timeout period in milliseconds. |
| `32`       | `mctp_bridge_get_table_wait`             | Duration in milliseconds to wait after RoT boots before sending a MCTP Get Routing Table Request to MCTP bridge, if MCTP bridge does not support dynamic EID assignments. If set to 0, RoT will only wait for EID assignment. |
| `32`       | `mctp_bridge_additional_timeout`         | Time in milliseconds to wait in addition to device timeout period due to MCTP bridge. |
| `32`       | `attestation_rsp_not_ready_max_duration` | Whan an SPDM device responds with a ResponseNotReady error, this is the maximum duration in milliseconds the RoT is permitted to wait before retrying the request. |
| `8`        | `attestation_rsp_not_ready_max_retry`    | The maximum number of SPDM ResponseNotReady retries allowed per request.  If this is exceeded, the RoT will fail attestation for the device. |
| `0x000000` | `_`                                      | Reserved.            |

`enum PCD.RoT.RoTType`
| Value | Name     | Description                        |
|-------|----------|------------------------------------|
| `0b`  | `PA-RoT` | A Platform Active Root of Trust.   |
| `1b`  | `AC-RoT` | An Active Component Root of Trust. |


### SPI Flash Port Element

The SPI flash port element represents a single external processor directly
protected by the RoT.  The firmware for this processor is stored on SPI flash.

`bitfield PCD.SPIFlashPort`
| Type                  | Name                   | Description                 |
|-----------------------|------------------------|-----------------------------|
| `8`                   | `port_id`              | Port identifier.  This maps to Port ID arguments in protocol messages. |
| `0b`                  | `_`                    | Reserved.                   |
| `HostResetAction`     | `host_reset_action`    | Action taken by the RoT on a host reset. |
| `WatchdogMonitoring`  | `watchdog_monitoring`  | Indication if the port supports watchdog monitoring to trigger recovery flows. |
| `RuntimeVerification` | `runtime_verification` | Indication if the port supports run-time verification of firmware updates. |
| `FlashMode`           | `flash_mode`           | Configuration of the flash device(s) containing firmware. |
| `ResetControl`        | `reset_control`        | The reset control behavior that should be used with the external processor. |
| `0000000b`            | `_`                    | Reserved.                   |
| `Policy`              | `policy`               | Action that should be taken on authentication failures. |
| `8`                   | `pulse_interval`       | Pulse width of the reset signal if the port used a pulsed reset control.  This value is in multiples of 10 ms. |
| `32`                  | `spi_frequency_hz`     | SPI clock frequency to use for flash communication.  This value is represented in Hz. |

`enum PCD.SPIFlashPort.HostResetAction`
| Value | Name           | Description                                         |
|-------|----------------|-----------------------------------------------------|
| `0b`  | `none`         | On host reset, no additional actions will be taken by the RoT.  Normal reset handling will still be executed. |
| `1b`  | `reset_flash`  | On host reset, always reset host flash.  This is in addition to any normal reset handling and means the RoT will be involved in every host reset. |

`enum PCD.SPIFlashPort.WatchdogMonitoring`
| Value | Name       | Description                        |
|-------|------------|------------------------------------|
| `0b`  | `disabled` | Watchdog monitoring not supported. |
| `1b`  | `enabled`  | Watchdog monitoring supported.     |

`enum PCD.SPIFlashPort.RuntimeVerification`
| Value | Name       | Description                          |
|-------|------------|--------------------------------------|
| `0b`  | `disabled` | Run-time verification not supported. |
| `1b`  | `enabled`  | Run-time verification supported.     |

`enum PCD.SPIFlashPort.FlashMode`
| Value | Name                     | Description                               |
|-------|--------------------------|-------------------------------------------|
| `00b` | `dual`                   | The SPI bus is connected to two flash devices for storing firmware. |
| `01b` | `single`                 | There is a single flash device for processor firmware. |
| `10b` | `dual_filtered_bypass`   | Dual flash chips for firmware.  In this mode, RoT protection can never be fully disabled.  Even in bypass mode, the interposed RoT will block some set of unwanted commands. |
| `11b` | `single_filtered_bypass` | Same as `dual_filtered_bypass` but with only a single flash device for firmware. |

`enum PCD.SPIFlashPort.ResetControl`
| Value | Name       | Description                                             |
|-------|------------|---------------------------------------------------------|
| `00b` | `notify`   | The reset control signal does not directly cause reset of the processor.  Instead, it notifies the processor of pending firmware authentication that will take place on the next reset.  During host reset, the reset control signal is released to indicate when authentication has completed. |
| `01b` | `reset`    | The reset control signal directly causes reset of the processor.  The processor is held in reset during validation. |
| `10b` | `pulse`    | The reset control signal directly causes reset of the processor.  The processor is left running without flash access during validation, and reset is pulsed after validation. |
| `11b` | `_`        | Reserved.                                               |

`enum PCD.Policy`
| Value    | Name      | Description                                           |
|----------|-----------|-------------------------------------------------------|
| `0b`     | `passive` | Indicate an attestation failure in RoT measurements but allow the failing device to continue to boot. |
| `1b`     | `active`  | Prevent a device that fails attestation from booting. |

### I2C Power Management Controller Element

The Power Management Controller element contains information about a system
power management device utilized by the RoT to gate power to devices that fail
attestation.  If no Power Management Controller element is present, Component
power control is not available.

The I2C mux information in the element is ordered starting with the mux directly
connected to the RoT and ending with the one directly connected to the power
management device.  If communication with Power Management Controller does not
utilize MCTP, the EID field can be set to `0x00`.

`bitfield PCD.I2CPowerManagementController`
| Type                 | Name          | Description                           |
|----------------------|---------------|---------------------------------------|
| `I2CDevice`          | `device`      | Communication information for the Power Management Controller. |

`bitfield PCD.I2CDevice`
| Type                | Name        | Description                              |
|---------------------|-------------|------------------------------------------|
| `4`                 | `mux_count` | Number of muxes in the I2C path between the RoT and device.  |
| `000b`              | `_`         | Reserved.                                |
| `I2CMode`           | `i2c_mode`  | I2C operational mode of the device.  Muxes are always Master-Slave. |
| `8`                 | `bus`       | Identifier for the I2C bus connected to the device. |
| `8`                 | `address`   | I2C address of the target device.        |
| `8`                 | `eid`       | Device MCTP EID.  If the device doesn't use MCTP, set this to `0x00`. |
| `I2CMux(mux_count)` | `mux`       | List of I2C muxes.                       |

`bitfield PCD.I2CMux`
| Type        | Name          | Description                                  |
|-------------|---------------|----------------------------------------------|
| `8`         | `mux_address` | I2C address of the mux.                      |
| `8`         | `mux_channel` | Mux channel that the device is connected to. |
| `0x0000`    | `_`           | Reserved.                                    |

`enum PCD.I2CMode`
| Value    | Name           | Description                     |
|----------|----------------|---------------------------------|
| `0b`     | `multi_master` | Multi-master I2C communication. |
| `1b`     | `master_slave` | Master-Slave I2C communication. |

### Component Elements

Each Component element in the PCD represents a remote RoT in the system that
must be attested.  Different AC-RoT devices will have different connection
properties, so there are different element types to account for the different
methods an attestor will use to communicate with the AC-RoT.  The Component type
identifier will match a type specified in the CFM.

`bitfield PCD.Component`
| Type     | Name              | Description                                   |
|----------|-------------------|-----------------------------------------------|
| `Policy` | `policy`          | Action that should be taken on authentication failures.  Active policies are only possible if there is power controller present. |
| `8`      | `power_ctrl_reg`  | Register address in the power controller that manages this component. |
| `8`      | `power_ctrl_mask` | A bitmask indicating the bit(s) used to control power to this component. |
| `0x00`   | `_`               | Reserved.                                     |
| `32`     | `component_id`    | Identifier for the component type.  This ID will be used to match against an attestation policy in the CFM.  If there are multiple components in the PCD with the same component ID, they will all use the same policy in the CFM. |

#### Component with Direct I2C Connection Element

This element contains information about an AC-RoT that is directly connected to
the RoT over I2C.  An AC-RoT is considered to be directly connected even if
there are I2C muxes in the I2C path.

The mux information in the element is ordered starting with the mux directly
connected to the attestor and ending with the one directly connected to the
AC-RoT.

`bitfield PCD.ComponentWithDirectI2CConnection`
| Type         | Name              | Description                               |
|--------------|-------------------|------------------------------------------ |
| `Component`  | `component`       | AC-RoT attestation configuration.         |
| `I2CDevice`  | `device`          | I2C communication details for the AC-RoT. |

#### Component with MCTP Bridge Connection Element

This element contains information about an AC-RoT that is connected to a device
that will route MCTP requests based on the destination EID.  In a server, this
is typically the BMC.  The RoT is expected to have a direct connection to this
bridge and understand how to communicate with it.  In cases where there are
multiple identical devices in a system and the EID can be dynamically discovered
from the bridging device, a single element entry can be used to identify
multiple devices for attestation.

`bitfield PCD.ComponentWithMCTPBridgeConnection`
| Type         | Name                  | Description                           |
|--------------|-----------------------|---------------------------------------|
| `Component`  | `component`           | AC-RoT attestation configuration.     |
| `16`         | `device_id`           | Device identifier used to identify this device during the discovery phase of attestation. |
| `16`         | `vendor_id`           | Vendor identifier used to identify this device during the discovery phase of attestation. |
| `16`         | `subsystem_device_id` | Subsystem device identifier used to identify this device during the discovery phase of attestation. |
| `16`         | `subsystem_vendor_id` | Subsystem vendor identifier used to identify this device during the discovery phase of attestation. |
| `8`          | `components_count`    | Number of identical components this element describes. |
| `8`          | `eid`                 | Default EID to use if the EID is not dynamically discoverable from the MCTP bridge. |
| `0x0000`     | `_`                   | Reserved.                             |

### XML Representation for PCD Generation

Each RoT is expected to have PCD that defines the behavior and components
relevant for that particular configuration.  This is particularly relevant for
a common RoT device that gets used in multiple different platforms.  This is an
example using XML to store the platform configuration that can be consumed by
generation scripts to create the binary format.

```xml
<PCD sku="Identifier" version="Hex integer">
	<RoT type="PA-RoT" mctp_ctrl_timeout="Decimal Integer"
		mctp_bridge_get_table_wait="Decimal Integer">
		<Ports>
			<Port id="Decimal integer">
				<SPIFreq>Decimal integer</SPIFreq>
				<ResetCtrl>Reset</ResetCtrl>
				<FlashMode>Dual</FlashMode>
				<Policy>Passive</Policy>
				<PulseInterval>0</PulseInterval>
				<RuntimeVerification>Enabled</RuntimeVerification>
				<WatchdogMonitoring>Enabled</WatchdogMonitoring>
			</Port>
			<Port id="Decimal integer">
				<SPIFreq>Decimal integer</SPIFreq>
				<ResetCtrl>Notify</ResetCtrl>
				<FlashMode>Single</FlashMode>
				<Policy>Active</Policy>
				<PulseInterval>0</PulseInterval>
				<RuntimeVerification>Disabled</RuntimeVerification>
				<WatchdogMonitoring>Disabled</WatchdogMonitoring>
			</Port>
		</Ports>
		<Interface type="I2C">
			<Address>Hex integer</Address>
			<RoTEID>Hex integer</RoTEID>
			<BridgeEID>Hex integer</BridgeEID>
			<BridgeAddress>Hex integer</BridgeAddress>
		</Interface>
	</RoT>
	<PowerController>
		<Interface type="I2C">
			<Bus>Decimal integer</Bus>
			<EID>Hex integer</EID>
			<Address>Hex integer</Address>
			<I2CMode>MultiMaster</I2CMode>
			<Muxes>
				<Mux level="Decimal integer">
					<Address>Hex integer</Address>
					<Channel>Decimal integer</Channel>
				</Mux>
			</Muxes>
		</Interface>
	</PowerController>
	<Components>
		<Component type="Identifier" connection="Direct"
			attestation_success_retry="Decimal Integer"
			attestation_fail_retry="Decimal Integer"
			attestation_rsp_not_ready_max_retry="Decimal Integer"
			attestation_rsp_not_ready_max_duration="Decimal Integer">
			<Policy>Passive</Policy>
			<Interface type="I2C">
				<Bus>Decimal integer</Bus>
				<Address>Hex integer</Address>
				<I2CMode>MultiMaster</I2CMode>
				<EID>Hex integer</EID>
				<Muxes>
					<Mux level="Decimal integer">
						<Address>Hex integer</Address>
						<Channel>Decimal integer</Channel>
					</Mux>
				</Muxes>
			</Interface>
			<PwrCtrl>
				<Register>Hex integer</Register>
				<Mask>Hex integer</Mask>
			</PwrCtrl>
		</Component>
		<Component type="Identifier" connection="MCTPBridge"
			count="Decimal integer"
			attestation_success_retry="Decimal Integer"
			attestation_fail_retry="Decimal Integer"
			discovery_fail_retry="Decimal Integer"
			mctp_bridge_additional_timeout="Decimal Integer"
			attestation_rsp_not_ready_max_retry="Decimal Integer"
			attestation_rsp_not_ready_max_duration="Decimal Integer">
			<Policy>Passive</Policy>
			<DeviceID>Hex integer</DeviceID>
			<VendorID>Hex integer</VendorID>
			<SubsystemDeviceID>Hex integer</SubsystemDeviceID>
			<SubsystemVendorID>Hex integer</SubsystemVendorID>
			<EID>Hex Integer</EID>
			<PwrCtrl>
				<Register>Hex Integer</Register>
				<Mask>Hex integer</Mask>
			</PwrCtrl>
		</Component>
	</Components>
</PCD>
```

| Field                                    | Description                       |
|------------------------------------------|-----------------------------------|
| `sku`                                    | String identifier for this platform configuration.  This becomes the platform ID. |
| `version`                                | Version of the PCD.  This becomes the manifest ID. |
| `RoT`                                    | Container for the RoT configuration. |
| `type`                                   | RoT type identifier.              |
| `mctp_ctrl_timeout`                      | MCTP control protocol response timeout period in milliseconds. |
| `mctp_bridge_get_table_wait`             | Duration in milliseconds to wait after RoT boots before sending a MCTP Get Routing Table Request to MCTP bridge, if MCTP bridge does not support dynamic EID assignments. If set to 0, RoT will only wait for EID assignment. |
| `Ports`                                  | Collection of ports an external RoT protects. |
| `Port`                                   | A single protected port.          |
| `id`                                     | Integer identifier for the port.  |
| `SPIFreq`                                | SPI flash frequency in Hz.        |
| `ResetCtrl`                              | Reset control setting when validation. |
| `FlashMode`                              | Flash configuration for the protected device. |
| `Policy`                                 | Attestation policy.               |
| `PulseInterval`                          | Reset pulse width if the port uses a pulsed reset. |
| `RuntimeVerification`                    | Port run-time verification setting. |
| `WatchdogMonitoring`                     | Port watchdog monitoring setting. |
| `Interface`                              | Communication interface to the RoT. |
| `type`                                   | Communication interface type or component type identifier string. |
| `Address`                                | 7-bit I2C address.                |
| `RoTEID`                                 | Default MCTP EID of the root of trust. |
| `BridgeEID`                              | MCTP bridge EID.                  |
| `BridgeAddress`                          | MCTP bridge 7-bit I2C address.    |
| `Bus`                                    | Identifier for the I2C bus the device is on. |
| `EID`                                    | Device MCTP EID.                  |
| `I2CMode`                                | I2C communication mode.           |
| `Muxes`                                  | A series of I2C muxes connected to a device. |
| `Mux`                                    | A single I2C mux.                 |
| `level`                                  | The mux level in the I2C path.  0 if the first mux, 1 is second, etc. |
| `Address`                                | 7-bit I2C address of the mux.     |
| `Channel`                                | Channel to activate on mux.       |
| `Components`                             | Collection of components to attest. |
| `Component`                              | A single component's attestation information. |
| `connection`                             | Component connection type.        |
| `attestation_success_retry`              | Duration in milliseconds after device succeeds attestation to wait before reattesting. |
| `attestation_fail_retry`                 | Duration in milliseconds after device fails attestation to wait before reattesting. |
| `attestation_rsp_not_ready_max_retry`    | Maximum number of SPDM ResponseNotReady retries permitted for the device. |
| `attestation_rsp_not_ready_max_duration` | Maximum wait time between retries after receiving an SPDM ResponseNotReady error. |
| `PwrCtrl`                                | Component power control information. |
| `Register`                               | Power control register address.   |
| `Mask`                                   | Power control bitmask.            |
| `count`                                  | Number of identical components this element describes. |
| `discovery_fail_retry`                   | Duration in milliseconds after device fails a discovery step to wait before retrying. |
| `mctp_bridge_additional_timeout`         | Time in milliseconds to wait in addition to device timeout period due to MCTP bridge. |
| `DeviceID`                               | Device ID.  See the device discovery description. |
| `VendorID`                               | Vendor ID.  See the device discovery description. |
| `SubsystemDeviceID`                      | Subsystem device ID.  See the device discovery description. |
| `SubsystemVendorID`                      | Subsystem vendor ID.  See the device discovery description. |

#### Allowed Values for XML Enums

Enums that have default values indicated mean the tag is optional.  If it is not
present in the XML, the default value will be used.

**RoT/type**
 - PA-RoT
 - AC-RoT

**Port/ResetCtrl**
 - Notify
 - Reset
 - Pulse

**Port/FlashMode**
 - Single
 - Dual
 - SingleFilteredBypass
 - DualFilteredBypass

**Policy/RuntimeVerification**
 - Enabled
 - Disabled

**Policy/WatchdogMonitoring**
 - Enabled
 - Disabled

**Interface/type**
 - I2C

**Interface/I2CMode**
 - MultiMaster
 - MasterSlave

**Component/connection**
 - Direct
 - MCTPBridge

**Policy/FailureAction**
 - Passive
 - Active

## Component Firmware Manifest

The Component Firmware Manifest (CFM) describes a list of allowable measurements
or policies for each component in the system.  This is similar to a PFM but for
the purpose of attesting remote AC-RoT devices.  Only Cerberus devices that will
attest other AC-RoTs will consume a CFM.  Elements in the CFM describe how to
attest a device using either the Cerberus Challenge Protocol or the DMTF SPDM
protocol.  Each component in a CFM must have at least one corresponding entry in
the PCD.  Multiple PCD entries for the same type of component device will map to
a single component in the CFM.

A CFM will report a `manifest_type` of `0xa592` in the manifest header.  The CFM
defines the following additional manifest element types.

| Type ID | Element Type       | Format | Top-Level | Singleton | Description  |
|---------|--------------------|--------|-----------|-----------|--------------|
| `0x70`  | `Component Device` |   0    |     X     |           | Defines a single type of AC-RoT to attest. |
| `0x71`  | `PMR`              |   0    |           |           | Provides information necessary when regenerating PMR. This is a child of a Component Device element. |
| `0x72`  | `PMR Digest`       |   0    |           |           | A list of allowable values for a single PMR.  This is a child of a Component Device element. |
| `0x73`  | `Measurement`      |   0    |           |           | A list of allowable values for a measurement digest.  This is a child of a Component Device element. |
| `0x74`  | `Measurement Data` |   0    |           |           | Provides information about raw measurement data checks.  This is a child of a Component Device element. |
| `0x75`  | `Allowable Data`   |   0    |           |           | Provides information about a single check for raw measurement data.  This is a child of a Measurement Data element. |
| `0x76`  | `Allowable PFM `   |   0    |           |           | A list of allowable PFM IDs for a single port.  This is a child of a Component Device element. |
| `0x77`  | `Allowable CFM`    |   0    |           |           | A list of allowable CFM IDs.  This is a child of a Component Device element. |
| `0x78`  | `Allowable PCD`    |   0    |           |           | A list of allowable PCD IDs.  This is a child of a Component Device element. |
| `0x79`  | `Allowable ID`     |   0    |           |           | Provides information about a single check for manifest IDs.  This is a child of an Allowable PFM, CFM, or PCD element. |
| `0x7A`  | `Root CAs`         |   0    |           |           | The Root CAs that can be used for certificate chain authentication.  This is a child of a Component Device element. |

### Component Device Element

The Component Device element is the top-level container for all the information
necessary to attest a single type of AC-RoT.  Child elements to the Component
Device specify what attestation actions and checks must be taken.  In order for
attestation of the AC-RoT to succeed, all attestation checks must pass.  Each
AC-RoT described in the PCD is matched with a Component Device element using the
`type` attribute.  Each Component Device element needs to have at least one
child element.

`bitfield CFM.ComponentDevice`
| Type                  | Name                     | Description               |
|-----------------------|--------------------------|---------------------------|
| `8`                   | `cert_slot`              | Slot number of the certificate chain to use for attestation challenges. |
| `AttestationProtocol` | `attestation_protocol`   | Protocol to use for attestation requests to the component. |
| `Manifest.HashType`   | `transcript_hash_type`   | The type of hash used for SPDM transcript hashing. |
| `Manifest.HashType`   | `measurement_hash_type`  | The type of hash used to generate measurement, PMR, and root CA digests. |
| `00b`                 | `_`                      | Padding for the unused HashType bits.  This must be 0. |
| `0x00`                | `_`                      | Reserved.                 |
| `32`                  | `component_id`           | Identifier for the component type that will be attested.  This must be a unique identifier within the CFM. |

`enum CFM.AttestationProtocol`
| Value  | Name                | Description                      |
|--------|---------------------|----------------------------------|
| `0x00` | `cerberus_protocol` | The Cerberus challenge protocol. |
| `0x01` | `dmtf_spdm`         | The DMTF SPDM protocol.          |

`bitfield CFM.ComponentDevice.Check`
| Type             | Name               | Description                          |
|------------------|--------------------|--------------------------------------|
| `CheckType`      | `check`            | The type of comparison to execute.   |
| `0000b`          | `_`                | Unused bits.  This must be 0.        |
| `1`              | `endianness`       | Endianness of multi-byte data values.  This will be 0 if the data is presented in little endian or 1 if the data is big endian.  If the data is only a single byte, this value does not matter. |

`enum CFM.ComponentDevice.CheckType`
| Value  | Name               | Description                                    |
|--------|--------------------|------------------------------------------------|
| `000b` | `equal`            | Ensure the reported data is equal to the specified value. |
| `001b` | `not_equal`        | Ensure the reported data is not equal to the specified value. |
| `010b` | `less_than`        | Ensure the reported data is less than the specified value. |
| `011b` | `less_or_equal`    | Ensure the reported data is less than or equal to the specified value. |
| `100b` | `greater_than`     | Ensure the reported data is greater than the specified value. |
| `101b` | `greater_or_equal` | Ensure the reported data is greater than or equal to the specified value. |

### PMR Element

The PMR element contains information necessary to regenerate a PMR digest.  If
this element in not present for a PMR ID, the initial value of zeroes is used
for any flows that are required to calculate the PMR.

`bitfield CFM.ComponentDevice.PMR`
| Type                    | Name            | Description                      |
|-------------------------|-----------------|----------------------------------|
| `measurement_hash_type` | `initial_value` | Initial value to use when generating PMR. |

### PMR Digest Element

The PMR Digest element contains a list of all allowable digests for a single
PMR.  These digests represent the final measurement reported by the PMR, as
would be reported by `Challenge` or `Get PMR` requests.

`bitfield CFM.ComponentDevice.PMRDigest`
| Type                                  | Name           | Description         |
|---------------------------------------|----------------|---------------------|
| `8`                                   | `pmr_id`       | Identifier for the PMR to attest.  The Cerberus Challenge Protocol allows this to be between 0 and 4. |
| `8`                                   | `digest_count` | Number of allowable digests for this PMR. |
| `0x0000`                              | `_`            | Reserved.           |
| `measurement_hash_type(digest_count)` | `pmr_digest`   | List of allowed digests for the PMR. |

### Measurement Element

The Measurement element contains a list of allowable digests for a single
measurement.  For a Cerberus PMR, this will be an entry within a specific PMR
reported by the `Get Log` request when the Attestation or TCG log is provided.
In SPDM, this will be the digest for a single measurement block and can be
aggregated with all other measurement blocks when reported by `SPDM Challenge`
requests or reported separately by the `SPDM Get Measurement` request.

`bitfield CFM.ComponentDevice.Measurement`
| Type                                      | Name                     | Description |
|-------------------------------------------|--------------------------|-------------|
| `8`                                       | `pmr_id`                 | Identifier for the PMR that contains the entry to attest.  The Cerberus Challenge Protocol allows this to be between 0 and 4. This will be zero for devices using SPDM, since SPDM does not provide an equivalent to PMRs. |
| `8`                                       | `measurement_id`         | Index of the specific entry within the PMR to attest if utilizing Cerberus Challenge Protocol.  If using SPDM, this is the measurement block index. |
| `8`                                       | `allowable_digest_count` | Number of allowable digests for this measurement. |
| `0x00`                                    | `_`                      | Reserved. |
| `AllowableDigest(allowable_digest_count)` | `digests_list`           | List of allowable digests across all versions of expected firmware. |

`bitfield CFM.ComponentDevice.Measurement.AllowableDigest`
| Type                                  | Name           | Description         |
|---------------------------------------|----------------|---------------------|
| `16`                                  | `version_set`  | Identifier for a set of measurements associated with the same version of firmware on the device.  This will be 0 if the same set of measurements applies to all versions of firmware. |
| `8`                                   | `digest_count` | The number of allowable digests for this version set. |
| `0x00`                                | `_`            | Reserved.           |
| `measurement_hash_type(digest_count)` | `digest`       | List of expected measurements digests for this version set. |

When attesting individual measurements, it is necessary to ensure that the
measurements reported by the device represent an expected state across all
measurements.  Attesting individual measurements alone does not ensure this,
since each would be checked independently from other checks.  This would allow a
device reporting one measurement from firmware A and another measurement from
firmware B to be viewed as healthy, even if it is required that both
measurements represent state for either firmware A or B, exclusively.  The
`version_set` identifier provides a way to ensure that reported measurements can
be checked within a broader attestation context, providing coupling between
separate measurement checks.

### Measurement Data Element

The Measurement Data element contains a list of checks to perform against the
raw data measured for a single measurement block.  For a Cerberus PMR, this will
be an entry within a specific PMR reported by the `Get Attestation Data`
request.  In SPDM, this will be the raw form for a single measurement block and
can be reported separately by the `SPDM Get Measurement` request.  This allows
for more sophisticated policy checks than simply comparing digests and has the
potential for more efficient use of space within the CFM.  There are as many
checks executed against this data as there are child Allowable Data elements,
and all need to succeed for successful attestation.

The use of `version_set` within individual checks for Measurement Data elements
is the same as with Measurement elements.  Additionally, the coupling between
Measurement elements that exists due to `version_set` also extends to
Measurement Data elements.  This means that all Measurement and Measurement Data
checks must attest within same `version_set` for attestation to pass.

`bitfield CFM.ComponentDevice.MeasurementData`
| Type                        | Name             | Description                 |
|-----------------------------|------------------|-----------------------------|
| `8`                         | `pmr_id`         | Identifier for the PMR that contains the entry to attest.  The Cerberus Challenge Protocol allows this to be between 0 and 4. This will be zero for devices supporting SPDM. |
| `8`                         | `measurement_id` | Index of the specific entry within the PMR to attest if utilizing Cerberus Challenge Protocol.  If using SPDM, this is the measurement block index. |
| `0x0000`                    | `_`              | Reserved.                   |

### Allowable Data Element

The Allowable Data element contains a list of allowable values for a single
measurement data check.  For `equal` and `not equal` checks, there can be as
many data entries with the same `version_set` identifier in a single Allowable
Data element as required to cover all allowed or disallowed values.  For any
other type of check, there should be only a single data entry in the Allowable
Data element per `version_set` identifier.  In all cases, any number of data
entries with different `version_set` identifiers are allowed.

To combine multiple measurement data checks (e.g. `not equal` and
`greater than`), multiple Allowable Data elements must be used, one for each
check.

`bitfield CFM.ComponentDevice.MeasurementData.AllowableData`
| Type              | Name               | Description                         |
|-------------------|--------------------|-------------------------------------|
| `Check`           | `check`            | The type of comparison to execute on the data. |
| `8`               | `num_data`         | The total number of values that will be checked. |
| `16`              | `bitmask_length`   | Length of the bitmask to apply to the measurement data before applying any checks.  If this is 0, no bitmask will be applied and the raw data will be checked. |
| `bitmask_length`  | `data_bitmask`     | A bitmask to apply to the received data.  This allows the comparison to ignore certain bits or bytes when necessary.  This bitmask must be created accounting for the endianness of the data.  The same bitmask applies to all data entries.  If the bitmask is longer than the measurement data, unused mask bits are ignored. |
| `ALIGN(32)`       | `_`                | Zero padding.                       |
| `Data(num_data)`  | `data_list`        | List of supported data.             |

`bitfield CFM.ComponentDevice.MeasurementData.AllowableData.Data`
| Type          | Name          | Description                                  |
|---------------|---------------|----------------------------------------------|
| `16`          | `version_set` | Identifier for a set of measurements associated with the same version of firmware on the device.  This will be 0 if the same measurement data check applies to all versions of firmware. |
| `16`          | `data_length` | Length of the data to use for comparison.    |
| `data_length` | `data`        | Data to use for comparison.  This must be stored in the format indicated by the `Check.endianness` flag. |
| `ALIGN(32)`   | `_`           | Zero padding.                                |

In order to group a set of data into a single Allowable Data element, each data
check must use the same bitmask.  Each unique bitmask must be a separate
Allowable Data element.  Since the raw data can be different lengths between
different versions of firmware, it is not required that the data length be the
same.  However, using bitmasks with data of different lengths presents some
challenges, so the following properties must hold to allow for efficient
grouping of checks.

1. The `bitmask_length` must be at least equal to the largest `data_length`
   across all supported data.
2. If `bitmask_length` is greater than `data_length` for any given `data`, any
   `data_bitmask` bytes that exceed `data_length` will be ignored by the
   comparison.
3. `data_bitmask` is always applied starting with the least significant byte of
   data.  If the data and bitmask lengths do not match, the most significant
   bytes of the bitmask will be ignored.  The `Check.endianness` flag indicates
   how that data and bitmask are stored and in which order they need to be
   processed.


### Allowable PFM Element

The Allowable PFM element provides a mechanism to attest an AC-RoT based on the
active PFM configuration.  Each allowable PFM element will compare against the
active PFM for a single port.  The data that is compared would be reported by
the `Get Configuration Ids` request.  Child Allowable ID elements will provide
the checks to run against the PFM ID.

`bitfield CFM.ComponentDevice.AllowablePFM`
| Type                | Name      | Description                                |
|---------------------|-----------|--------------------------------------------|
| `8`                 | `port_id` | Index in the list of returned PFM IDs that should be checked.  This is the same as the port ID. |
| `AllowableManifest` | `allowed` | Attestation to perform for the PFM IDs.    |

`bitfield CFM.ComponentDevice.AllowableManifest`
| Type                | Name             | Description                         |
|---------------------|------------------|-------------------------------------|
| `8`                 | `plat_id_length` | Length of the expected platform ID. |
| `plat_id_length`    | `plat_id_string` | Platform ID string.  This is an ASCII string that is not null terminated. |
| `ALIGN(32)`         | `_`              | Zero padding.                       |

### Allowable ID Element

The Allowable ID element contains a list of allowable IDs for a single manifest
check.  This element can be a child to the Allowable PFM, CFM, or PCD elements.
For `equal` and `not equal` checks, there can be as many data entries in a
single Allowable ID element as required to cover all allowed or disallowed
values.  For any other type of check, there should be only a single data entry
in the Allowable ID element.

To combine multiple manifest ID checks (e.g. `not equal` and `greater than`),
multiple Allowable ID elements must be used, one for each check.

`bitfield CFM.ComponentDevice.AllowableID`
| Type            | Name     | Description                                   |
|-----------------|----------|-----------------------------------------------|
| `Check`         | `check`  | The type of comparison to execute.            |
| `8`             | `num_id` | The total number of IDs that will be checked. |
| `16`            | `_`      | Reserved.                                     |
| `32 (num_data)` | `ids`    | Manifest identifiers to use for comparison.   |

### Allowable CFMs Element

The Allowable CFM element provides a mechanism to attest an AC-RoT based on the
active CFM configuration.  Each allowable CFM element will compare against a
single CFM.  The data that is compared would be reported by the
`Get Configuration Ids` request.  Child Allowable ID elements will provide the
checks to run against the PFM ID.

`bitfield CFM.ComponentDevice.AllowableCFM`
| Type                | Name        | Description                              |
|---------------------|-------------|------------------------------------------|
| `8`                 | `cfm_index` | Index in the list of returned CFM IDs that should be checked. |
| `AllowableManifest` | `allowed`   | Attestation to perform for the CFM IDs.  |

### Allowable PCDs Element

The Allowable PCD element provides a mechanism to attest an AC-RoT based on the
active PCD configuration.  The data that is compared would be reported by the
`Get Configuration Ids` request.  Child Allowable ID elements will provide the
checks to run against the PFM ID.

`bitfield CFM.ComponentDevice.AllowablePCD`
| Type                | Name      | Description                             |
|---------------------|-----------|-----------------------------------------|
| `0x00`              | `_`       | Reserved.                               |
| `AllowableManifest` | `allowed` | Attestation to perform for the PCD IDs. |

### Root CAs Element

The Root CAs element contains a list of trusted root certificates that
can be used to validate the attestation certificate chain of an AC-RoT.  If a
component does not contain a Root CAs element, the certificate chain of the
AC-RoT must share a root CA with the requestor.

`bitfield CFM.ComponentDevice.RootCAs`
| Type                              | Name       | Description                 |
|-----------------------------------|------------|-----------------------------|
| `8`                               | `ca_count` | Number of allowable root CA digests. |
| `0x000000`                        | `_`        | Reserved.                   |
| `measurement_hash_type(ca_count)` | `root_ca`  | List of hashes for trusted root certificates.  This represents the hash over the entire certificate, including the signature. |

### XML Representation for CFM Generation

Each RoT that must attest AC-RoT devices needs to have at least one CFM with
information about the components to attest.  Unlike a PFM that only deals in one
flash device, the CFM is intended to hold attestation information for multiple
components.  XML files with a CFMComponent element are used to define attestable
components.  A single CFM XML file is used to select attestable components to
include in the CFM binary.  Each component will have a single XML per firmware
version, and the resulting CFM binary will differentiate measurements belonging
to different firmware versions using the `version_set` field.  Each component
XML shall have at least one Measurement or MeasurementData element that is
unique for each firmware version.  This unique entry must be the first
Measurement or MeasurementData child element to CFMComponent in the XML.  If
that entry is a MeasurementData element, it must have a single AllowableData
child.  The generated CFM binary cannot have any child elements with version set
0 for this entry.  If multiple AllowableData elements are required for this
MeasurementData, they must be broken into two different MeasurementData
elements.  The first with only the single AllowableData containing unique
information and a second with the remaining checks.

This is an example of both XML formats that can be consumed by generation
scripts to create the binary format.

```xml
<CFM sku="identifier">
	<Component>
		"Component type string"
	</Component>
	<Component>
		"Component type string"
	</Component>
</CFM>

<CFMComponent type="identifier" attestation_protocol="Cerberus"
	slot_num="Decimal integer" transcript_hash_type="SHA256"
	measurement_hash_type="SHA384">
	<RootCADigest>
		<!-- Digest type is determined by measurement_hash_type. -->
		<Digest>
			Root CA Digest
		</Digest>
		<Digest>
			Root CA Digest
		</Digest>
	</RooTCADigest>
	<PMR pmr_id="Decimal integer">
		<SingleEntry>True</SingleEntry>
		<!-- InitialValue type is determined by measurement_hash_type. -->
		<InitialValue>
			Initial value digest
		</InitialValue>
	</PMR>
	<PMRDigest pmr_id="Decimal integer">
		<!-- Digest type is determined by measurement_hash_type. -->
		<Digest>
			PMR Digest
		</Digest>
		<Digest>
			PMR Digest
		</Digest>
	</PMRDigest>
	<Measurement pmr_id = "Decimal integer" measurement_id="Decimal integer">
		<!-- Digest type is determined by measurement_hash_type. -->
		<Digest>
			Measurement Digest
		</Digest>
		<Digest>
			Measurement Digest
		</Digest>
	</Measurement>
	<MeasurementData pmr_id="Decimal integer" measurement_id="Decimal integer">
		<AllowableData>
			<Endianness>BigEndian</Endianness>
			<Check>Equal</Check>
			<Data>
				Allowable Data 1
			</Data>
			<Data>
				Allowable Data 2
			</Data>
			<Bitmask>
				Measurement Data Bitmask
			</Bitmask>
		</AllowableData>
		<AllowableData>
			<Endianness>LittleEndian</Endianness>
			<Check>GreaterOrEqual</Check>
			<Data>
				Allowable Data
			</Data>
		</AllowableData>
	</MeasurementData>
	<MeasurementData pmr_id="Decimal integer" measurement_id="Decimal integer">
		<AllowableData>
			<Endianness>LittleEndian</Endianness>
			<Check>Equal</Check>
			<Data>
				"String"
			</Data>
			<Bitmask>
				Measurement Data Bitmask
			</Bitmask>
		</AllowableData>
	</MeasurementData>
	<AllowablePFM port="Decimal integer" platform="Identifier">
		<ManifestID>
			<Check>Equal</Check>
			<ID>Hex Integer</ID>
			<ID>Hex Integer</ID>
		</ManifestID>
		<ManifestID>
			<Check>GreaterThan</Check>
			<ID>Hex Integer</ID>
		</ManifestID>
	</AllowablePFM>
	<AllowableCFM index="Decimal integer" platform="Identifier">
		<ManifestID>
			<Check>Equal</Check>
			<ID>Hex Integer</ID>
		</ManifestID>
	</AllowableCFM>
	<AllowablePCD platform="Identifier">
		<ManifestID>
			<Check>Equal</Check>
			<ID>Hex Integer</ID>
		</ManifestID>
	</AllowablePCD>
</CFMComponent>
```

| Field                  | Description                                         |
|------------------------|-----------------------------------------------------|
| `sku`                  | String identifier for the platform configuration. This becomes the platform ID. |
| `Component`            | Defines a single component device to include in CFM. ID of component is used here. |
| `CFMComponent`         | Defines a single component device.                  |
| `type`                 | Component type string identifier.                   |
| `attestation_protocol` | The protocol used by the AC-RoT for attestation.    |
| `slot_num`             | Certificate chain slot number to be used for component attestation. |
| `transcript_hash_type` | Hashing algorithm used for transcript hashing.      |
| `measurement_hash_type`| Hashing algorithm used to compute PMR, measurement, and root CA digests. |
| `RootCADigest`         | Defines trusted root CAs to be used for certificate chain validation.  This is an optional tag. |
| `Digest`               | The expected digest.                                |
| `PMR`                  | Defines information needed to regenerate a PMR.  This is an optional tag. |
| `SingleEntry`          | Boolean indicating if PMR has a single entry measurement. |
| `InitialValue`         | Initial value to utilize when generating a PMR.     |
| `PMRDigest`            | Defines allowable digests for a single PMR.         |
| `pmr_id`               | Identifier for the PMR.                             |
| `Measurement`          | Defines group of allowable measurements.            |
| `measurement_id`       | Measurement ID.                                     |
| `MeasurementData`      | Defines group of allowable measurement data.        |
| `AllowableData`        | Defines allowable values for single measurement data check. |
| `Endianness`           | Multi-byte format of data.                          |
| `Check`                | Type of comparison to perform on data.              |
| `Data`                 | The expected data for the measurement data entry.   |
| `Bitmask`              | The bitmask to apply to the data during comparison. |
| `AllowablePFM`         | Defines attestation to be performed on PFM IDs.  This tag is not supported for attesting SPDM devices. |
| `port`                 | Port PFM is applied to.                             |
| `platform`             | The expected manifest platform ID.                  |
| `ManifestID`           | A single manifest ID comparison.                    |
| `ID`                   | A manifest ID to compare.                           |
| `AllowableCFM`         | Defines attestation to be performed on CFM IDs.  This tag is not supported for attesting SPDM devices. |
| `index`                | Index of the target CFM.                            |
| `AllowablePCD`         | Defines attestation to be performed on PCD IDs.  This tag is not supported for attesting SPDM devices. |

#### Allowed Values for XML Enums

Enums that have default values indicated mean the tag is optional.  If it is not
present in the XML, the default value will be used.

**Check**
 - Equal
 - NotEqual
 - LessThan
 - LessOrEqual
 - GreaterThan
 - GreaterOrEqual

**Component/attestation_protocol**
 - Cerberus
 - SPDM

**Endianness**
- LittleEndian
- BigEndian

# Attestation Flows

Depending on the device configuration, attestation workflows involve either
authentication of flash contents, challenging remote devices, or both.

## Authentication of Flash Contents

Flash authentication is typically used by external RoT devices connected to SPI
flash.  The PCD is used as part of this flow to provide configuration for the
SPI clock frequency to use when accessing the flash device and other parameters
related to interactions with the external processor.  The rest of the
authentication flow only requires a PFM.

### Authentication of Firmware Updates

Whenever a firmware update is executed against the SPI flash device, the RoT
will need to validate the contents before it can be attested as valid.
Validation takes the following flow:

1. Read the PFM to determine how many firmware components exist on the flash.
2. For each firmware component
   - Read the flash device to match against a version string in the PFM.  This
     will determine which version entry should be used for authentication.
   - For each signed image defined in the matched version, calculate and compare
     the hash for the flash contents.
3. Once each component has been authenticated, look for unused regions of flash.
   An unused region is one that is neither a R/W region nor a signed image in
   any firmware component.  These unused regions must be filled with the static
   value defined in the Flash Device PFM element.
4. If any step produces unexpected results, validation of the image has failed.

### Authentication on System Boot Without Updates

If firmware on flash has not been updated, the RoT will still authenticate all
necessary images on the flash device, but there will be a couple of
modifications to the flow.

1. The unused regions of flash will not be blank checked.
2. Any signed images that do not assert the `PFM.FirmwareVersion.MustValidate`
   flag will be skipped.

## Attestation of Remote Devices

Attestation of an AC-RoT follows this general flow:

1. Determine the type of device that is being attested.
2. Retrieve the device certificate chain, validate it, and establish that the
   device is trusted.
3. Get a report of measurements from the device representing the current state.
4. Compare the measurements to the attestation policy in the CFM.
5. Store the attestation result in the requesting RoT's attestation log.

Cerberus devices can support attestation using the Cerberus Challenge Protocol
and/or the DMTF SPDM attestation protocol.  While there are differences in the
execution details between protocols, the overall flow is the same.

All Cerberus Protocol commands used in the attestation flow are described in the
Cerberus Challenge Protocol specification.  MCTP and SPDM commands are described
in those respective specifications.

### Cerberus to AC-RoT Communication

As a prerequisite to attestation of AC-RoT components, communication to the
AC-RoT needs to be enabled.  Cerberus and AC-RoT component devices will all act
as MCTP endpoints and will either be connected directly on the same physical bus
or will communicate through an MCTP bridge.  A Cerberus instance that needs to
attest other AC-RoTs needs to be aware of the addressing details and the
platform configuration as a whole.  The Platform Configuration Data manifest
provides these details to the Cerberus RoT.

#### AC-RoT Discovery with an MCTP Bridge

To communicate with an AC-RoT through an MCTP bridge, Cerberus will first need
to fetch the target device's EID by sending a `Get Routing Table Entries` MCTP
control request to the MCTP bridge.  Cerberus will then issue a
`Get Message Type Support` MCTP control request to each EID in the received
routing table to discover the attestation protocol supported by the AC-RoT.

##### AC-RoT Discovery using Cerberus Challenge Protocol

Support for the Cerberus Challenge Protocol is determined when both of the
following conditions are true:

1. The AC-RoT supports a vendor defined message type.
2. Support for the Microsoft PCI Vendor ID (0x1414) is advertised in reponse to
   a `Get Vendor Defined Message Support` MCTP control request.

Once it is determined that an AC-RoT supports the Cerberus Challenge Protocol,
the type of device is identified by sending the Cerberus Protocol `Device ID`
request.  The response to this request is used to search component entries in
the PCD for a match.

##### AC-RoT Discovery using SPDM

Since SPDM has a defined message type in MCTP, support for SPDM is determined by
simply looking for support of this message type.  Once it is determined that the
AC-RoT supports SPDM, it is necessary to match this device to a component entry
in the PCD, just as it is when using Cerberus Challenge Protocol.  However, as
of SPDM 1.2, there is no protocol-defined method to achieve this.  In the
absence of any standardized SPDM workflow, the following will be used by
Cerberus, which must be implemented by any SPDM-based AC-RoT that needs to be
attested.  If a later version of the SPDM specification is updated to include
a way to determine this information, that method will be used for AC-RoTs that
support it.

To provide an SPDM-compliant way for devices to identify themselves for
attestation, the same PCD IDs that are reported by the Cerberus Protocol
`Device ID` command will be reported in a SPDM measurement block.  For AC-RoTs
that support SPDM 1.1.x, this information will be reported in the last
measurement block index supported by the device.  For other AC-RoTs, this
information will be at the fixed measurement index of `0xef`.

Retrieval of the measurement block is achieved through this sequence of SPDM
commands:

1. `Get Version`
2. `Get Capabilities`
3. `Negotiate Algorithms`
4. If the AC-RoT supports SPDM 1.1.x, Cerberus will get the total number of
   measurement blocks supported by the device using `Get Measurements` with no
   signature requested and measurement operation set to `0`.  Cerberus follows
   this with a `Get Measurements` request with no signature requested for the
   last measurement index.
5. If the AC-RoT supports SPDM 1.2.x or later, Cerberus will send a
   `Get Measurements` request with no signature requested for the raw bit stream
   at measurement index `0xef`.

In all cases, the response is expected to contain the raw bit stream of the
measurement block.  The format of this measurement block is based on the
`QueryDeviceIdentifiers` command from
[DMTF DSP0267 1.1.0, PLDM for Firmware Update](https://www.dmtf.org/sites/default/files/standards/documents/DSP0267_1.1.0.pdf).

`bitfield DeviceIds`
| Type                           | Name                | Description           |
|--------------------------------|---------------------|-----------------------|
| `8`                            | `completion_code`   | PLDM_BASE_CODES completion code. |
| `32`                           | `descriptor_length` | Total length of all descriptors provided. |
| `8`                            | `descriptor_count`  | Total number of descriptors. |
| `Descriptor(descriptor_count)` | `descriptors`       | List of device descriptors. |

`bitfield Descriptor`
| Type        | Name       | Description                                       |
|-------------|------------|---------------------------------------------------|
| `16`        | `type`     | Type identifier for the descriptor.               |
| `16`        | `length`   | Length of the descriptor data.                    |
| `length`    | `data`     | The raw descriptor data.  What this represents will depend on the type of descriptor. |

While the descriptors included in the response may include more than the PCI
IDs, at least descriptors for `PCI Vendor ID`, `PCI Device ID`,
`PCI Subsystem Vendor ID`, and `PCI Subsystem ID` must be included.  Included
here is a quick reference for the values to assign to `Descriptor` fields for
this information.

| `type`   | `length` | `data`                   |
|----------|----------|--------------------------|
| `0x0000` | `2`      | PCI Vendor ID.           |
| `0x0100` | `2`      | PCI Device ID.           |
| `0x0101` | `2`      | PCI Subsystem Vendor ID. |
| `0x0102` | `2`      | PCI Subsystem ID         |

For more details regarding descriptor types and formats, refer to Table 8 in
[DSP0267 1.1.0](https://www.dmtf.org/sites/default/files/standards/documents/DSP0267_1.1.0.pdf).

#### AC-RoT Discovery Without an MCTP Bridge

When the system does not use an MCTP bridge, the Cerberus device would be
directly connected the required AC-RoTs.  In this scenario, the discovery
procedure is the same as the one followed with an MCTP bridge.  The only
difference is there is no `Get Routing Table Entries` command sent.  Instead,
AC-RoT information is determined from PCD component entries.

### AC-RoT Attestation

To perform attestation of an AC-RoT, remote Cerberus instances send back flash
measurements or policy information to a Cerberus device performing the
challenge.  The Cerberus device performing the challenge will then compare the
information received with allowable values contained in the corresponding entry
in the Component Firmware Manifest for the target device.

Cerberus devices can support the Cerberus Challenge protocol, the DMTF SPDM
protocol, or both protocols to perform attestation of an AC-RoT.  It is expected
that PA-RoT devices would support both protocols to allow it to attest any type
of component.

### Attestation Procedure using Cerberus Protocol

After successful receipt and validation of the Alias certificate chain, the
attestor will then ensure all digests and measurement data listed in the CFM
entry matches values returned by the component device.

If the CFM contains only a Platform Measurement Register (PMR) digest for
register 0, only the `Challenge` request will be issued.  The PMR0 digest
value from the response will be compared to the CFM element contents.  This
process is illustrated below:

```
                 Attestor                           Device
                    |  -------Challenge Request---->  |
                    |                              -- |
                    |   Generate signed challenge |   |
                    |   response for PMR 0        |   |
                    |                              -> |
                    |  <-----Challenge Response-----  |
                    | --                              |
                    |   | Verify challenge signature  |
                    |   | and measurement             |
                    | <-                              |
```

If the CFM contains multiple PMR digest elements, the
`Get Platform Measurement Register` request will be issued to the device for
each PMR that is required.  All listed digests will be compared with the CFM
element contents, and if any of them mismatches, then the attestation will have
failed.  This flow is shown below:

```
                 Attestor                           Device
                   /|  --------Get PMR Request----->  |
     For Each PMR | |                              -- |
                  | |   Generate signed response  |   |
                  | |   for requested PMR         |   |
                  | |                              -> |
                  | |  <------Get PMR Response------  |
                  | | --                              |
                  | |   | Verify response signature   |
                  | |   | and PMR digest              |
                   \| <-                              |
```

If the CFM contains measurement elements, the attestor will inspect the
attestation log to compare expected measurement entries to those in the log.
All listed digests will be compared with the CFM element contents, and if any of
them mismatches, then the attestation will have failed. Then, the attestor will
issue a `Get Platform Measurement Register` request for all PMR IDs that have
CFM measurement entries. The PMR values returned will be compared to those in
the attestation log. This process takes the following flow.

1. The attestor issues the `Get Log` request to get the attestation log from the
   device.
2. The digests reported in the attestation log are compared to the CFM elements.
3. The attestor issues the `Get Platform Measurement Register` request to get
   the PMR value containing this entry.
4. The expected PMR value is calculated from the retrieved attestation log and
   compared against the value reported by the device.

If any comparison fails, the process terminates and attestation of the device
has failed.  If the attestor has several entries in a single PMR to validate,
the flow could be optimized to get and check all measurements first, then
proceed to validate up the layers to the PMR value.

This process is show below:

```
                 Attestor                           Device
                    |  -------Get Log Request------>  |
                    |                              -- |
                    |   Respond with attestation  |   |
                    |   log                       |   |
                    |                              -> |
                    |  <-------Get Log Response-----  |
                    | --                              |
                    |   | Compare CFM measurement to  |
                    |   | digest in attestation log   |
                    | <-                              |
                    |  -------Get PMR Request------>  |
                    |                              -- |
                    |   Respond with requested    |   |
                    |   PMR digest                |   |
                    |                              -> |
                    |  <------Get PMR Response------  |
                    | --                              |
                    |   | Compare PMR digest to one   |
                    |   | from attestation log        |
                    | <-                              |
```

If the CFM contains measurement data elements, it is required that the attestor
check the actual measured data to determine if the device is operating in an
allowed configuration.  This process takes the following flow.

1. The attestor issues the `Get Attestation Data` request to get the measurement
   data from the device.
2. The returned data is compared against the expected data in the CFM element.
3. The attestor issues the `Get Log` request to get the attestation log from the
   device.
4. The measurement data received earlier is hashed and compared against the
   digest reported in the attestation log.
5. The attestor issues the `Get Platform Measurement Register` request to get
   the PMR value containing this entry.
6. The expected PMR value is calculated from the retrieved attestation log and
   compared against the value reported by the device.

If any comparison fails, the process terminates and attestation of the device
has failed.  If the attestor has several entries in a single PMR to validate,
the flow could be optimized to get and check all data first, then proceed to
validate up the layers to the PMR value.

This process is show below:

```
                 Attestor                           Device
                    |  Get Attestation Data Request-> |
                    |                              -- |
                    |   Respond with requested    |   |
                    |   attestation data          |   |
                    |                              -> |
                    | <-Get Attestation Data Response |
                    | --                              |
                    |   | Compare to data in CFM      |
                    | <-                              |
                    |  -------Get Log Request------>  |
                    |                              -- |
                    |   Respond with attestation  |   |
                    |   log                       |   |
                    |                              -> |
                    |  <-------Get Log Response-----  |
                    | --                              |
                    |   | Compare digest of           |
                    |   | measurement data to digest  |
                    |   | in attestation log          |
                    | <-                              |
                    |  -------Get PMR Request------>  |
                    |                              -- |
                    |   Respond with requested    |   |
                    |   PMR digest                |   |
                    |                              -> |
                    |  <------Get PMR Response------  |
                    | --                              |
                    |   | Compare PMR digest to one   |
                    |   | from attestation log        |
                    | <-                              |
```

If the CFM contains a PFM, CFM, or PCD element, then the `Get Configuration IDs`
command is used to get active manifest IDs from the device.  The attestor will
compare the received IDs against the values contained in the CFM.  This process
is shown below:

```
                 Attestor                           Device
                    |  ---Get Config IDs Request--->  |
                    |                              -- |
                    |   Respond with IDs for all  |   |
                    |   manifests                 |   |
                    |                              -> |
                    |  <---Get Config IDs Response--  |
                    | --                              |
                    |   | Compare all IDs to CFM      |
                    |   | element                     |
                    | <-                              |
```

### Attestation Procedure using SPDM Attestation Protocol

After successful version and capabilities negotiation, then receipt and
validation of the Alias certificate chain, the attestor will issue a `Challenge`
request to the device if the device supports the command.  This will always be
done irrespective of whether the CFM contains a PMR digest element or not.  In
SPDM, only PMR 0 is considered valid, and if the CFM contains an element for a
PMR digest for register 0, the value returned by the `Challenge` response will
be compared to the CFM element contents.  If not, the `Challenge` is still
performed as recommended by the SPDM specification, and to ensure the
authenticity of the prior transactions.  This process is illustrated below:

```
                 Attestor                           Device
                    |  -------Challenge Request---->  |
                    |                              -- |
                    |   Generate signed challenge |   |
                    |   response for PMR 0        |   |
                    |                              -> |
                    |  <-----Challenge Response-----  |
                    | --                              |
                    |   | Verify challenge signature  |
                    |   | and measurement             |
                    | <-                              |
```

If the device supports SPDM 1.2 or higher, and does not support the `Challenge`
command, the `Get Measurement` command is used instead if applicable.  If the
CFM contains a PMR digest element but the device does not support the
`Challenge` command, the `Get Measurement` command is used and all measurement
blocks are requested.  The attestor then combines all measurement blocks into a
single digest and compares it to the PMR0 digest CFM element.  This process
takes the following flow:

```
                 Attestor                           Device
                    |  ---Get Measurement Request-->  |
                    |                              -- |
                    |   Generate measurement      |   |
                    |   response with hashes of   |   |
                    |   all measurement blocks    |   |
                    |                              -> |
                    |  <--Get Measurement Response--  |
                    | --                              |
                    |   | Aggregate all measurement   |
                    |   | block hashes and compare    |
                    |   | result to CFM PMR digest    |
                    |   | element                     |
                    | <-                              |
```

If the CFM contains measurement elements, and the device supports SPDM 1.2+, the
attestor will request the specific measurement blocks as indicated by
measurement IDs as digests from the device using the `Get Measurement` command
with the RawBitStreamRequested request attribute set to 0.  This process takes
the following flow:

```
                 Attestor                           Device
                    |  ---Get Measurement Request-->  |
                    |                              -- |
                    |   Generate measurement      |   |
                    |   response with hash of     |   |
                    |   requested measurement     |   |
                    |   block                     |   |
                    |                              -> |
                    |  <--Get Measurement Response--  |
                    | --                              |
                    |   | Compare response content    |
                    |   | to CFM measurement element  |
                    | <-                              |
```

If the CFM contains measurement data elements, the attestor will request the
specific measurement blocks as indicated by measurement IDs from the device
using the `Get Measurement` command.  This process takes the following flow:

```
                 Attestor                           Device
                    |  ---Get Measurement Request-->  |
                    |                              -- |
                    |   Generate measurement      |   |
                    |   response with requested   |   |
                    |   measurement block         |   |
                    |                              -> |
                    |  <--Get Measurement Response--  |
                    | --                              |
                    |   | Compare response content    |
                    |   | to CFM measurement data     |
                    |   | element                     |
                    | <-                              |
```

The SPDM protocol does not explicitly support verification of configuration IDs.
To verify a device's configuration IDs, a measurement block can be generated
with the configuration IDs as the expected content, and the aforementioned flows
for measurement or measurement data attestation can be utilized.

### Handling Measurement Version Sets

Regardless of whether Cerberus Challenge protocol or SPDM is used to attest the
AC-RoT, the attestor must ensure that all measurements represent the same
version of firmware, as required by the CFM.  In these scenarios, there are
additional checks that take place when attesting Measumement and Measurement
Data elements.

The first Measurement or Measurement Data element in the CFM must correspond to
a measumerent that has an entry for all supported versions of firmware and is
unique between each version.  For example, a measumerent of the version number
or string could be used here.  Once the first measurement is attested
successfully, the attestor must determine the version set identifier specified
in the CFM for the matching entry.  All subsequent Measurement and Measurement
Data elements will be attested with these additional conditions, using the
selected version set identifier from this first check.

1. If the element contains an entry with version set identifier equal to 0, this
   check must always succeed, irrespective of any selected version set
   identifier.
2. If the element contains an entry matching the selected version set
   identifier, attestation of the device state must match only that element
   entry.
3. If the element contains no entries for the selected version set identifier
   and no entries with a version set identifier of 0, that element is ignored as
   part of the current attestation.

## References

1. Cerberus Challenge Protocol:<br>
https://github.com/opencomputeproject/Security/blob/master/RoT/Protocol/Challenge_Protocol.md
2. TCG DICE:<br>
https://trustedcomputinggroup.org/work-groups/dice-architectures/
3. TCG TPM:<br>
https://trustedcomputinggroup.org/work-groups/trusted-platform-module/
4. Security Protocol and Data Model Specification:<br>
https://www.dmtf.org/sites/default/files/standards/documents/DSP0274_1.2.0.pdf
5. Platform Level Data Model (PLDM) Base Specification:<br>
https://www.dmtf.org/sites/default/files/standards/documents/DSP0267_1.1.0.pdf
6. Management Component Transport Protocol (MCTP) Base Specification:<br>
https://www.dmtf.org/sites/default/files/standards/documents/DSP0236_1.1.0.pdf
