<!--
  Style Guide
  - Make sure to wrap everything to 80 columns when possible.
  - Sentence-ending periods are followed by two spaces.
  - Hexadecimal integers should be formatted as `0xff`.
-->

Project Cerberus: Firmware Challenge Specification
=====

Authors:
- Bryan Kelly, Principal Firmware Engineering Manager, Microsoft
- Christopher Weimer, Senior Firmware Engineer, Microsoft
- Akram Hamdy, FirmwareEngineer, Microsoft

## Revision History

- v0.01 (2017-08-28)
    - Initial draft.
- v0.02 (2017-09-28)
    - Add References section.
- v0.03 (2017-10-28)
    - Move message exchange from protocol to register based.
- v0.04 (2018-12-02)
    - Add MCTP Support and update session
- v0.05 (2018-30-04)
    - Incorporate Supplier feedback.
- v0.06 (2018-15-10)
    - Update Authentication flow.
    - Change measurement to PMR and attestation integration.
- v0.07 (2019-10-01)
    - Change PMR naming to PM due to static requirements on extension.
- v0.08 (2019-15-02)
    - Add Firmware Recovery image update commands.
    - Clarify Error Response.
- v0.09 (2019-26-06) 
    - Add Reset Configuration command.
    - Identify commands subject to the cryptographic timeout.
- v0.10 (2019-05-08)
    - Update Cerberus-defined MCTP message definition.
- v0.11 (2019-21-10)
    - Add detail on Mfg pairing for devices.
    - Add commands to get RIoT, chip, and host reset information.
- v0.12 (2019-27-12)
    - Clarification regarding required and optional commands.
- v0.13 (2020-17-03)
    - Add commands to get manifest platform IDs and PMR measured data.
    - Update unseal and device capabilities commands.
    - Clarifications around command packet format.
    - Add log formats.
- v0.14 (2020-30-04)
    - Update format of several commands, add extended update status.
    - Clarifications around certificates.
    - Add details about encrypted messages.
- v0.15 (2020-22-05)
    - Add unseal ECDH seed parameters.
    - Define a range of reserved commands.
- v1.00 (2020-19-08)
    - Update session establishment and secure device binding.
    - Add back Rq bit.

(c) 2017 Microsoft Corporation.

## Legal

As of November 1, 2017, the following persons or entities have made this
Specification available under the Open Web Foundation Final Specification
Agreement (OWFa 1.0), which is available at
<http://www.openwebfoundation.org/legal/the-owf-1-0-agreements/owfa-1-0>.

- Microsoft Corporation.

You can review the signed copies of the Open Web Foundation Agreement Version
1.0 for this Specification at
<http://www.opencompute.org/participate/legal-documents/> which may also include
additional parties to those listed above.

Your use of this Specification may be subject to other third party rights.  THIS
SPECIFICATION IS PROVIDED "AS IS." The contributors expressly disclaim any
warranties (express, implied, or otherwise), including implied warranties of
merchantability, non-infringement, fitness for a particular purpose, or title,
related to the Specification.  The entire risk as to implementing or otherwise
using the Specification is assumed by the Specification implementer and user.  IN
NO EVENT WILL ANY PARTY BE LIABLE TO ANY OTHER PARTY FOR LOST PROFITS OR ANY
FORM OF INDIRECT, SPECIAL, INCIDENTAL, OR CONSEQUENTIAL DAMAGES OF ANY CHARACTER
FROM ANY CAUSES OF ACTION OF ANY KIND WITH RESPECT TO THIS SPECIFICATION OR ITS
GOVERNING AGREEMENT, WHETHER BASED ON BREACH OF CONTRACT, TORT (INCLUDING
NEGLIGENCE), OR OTHERWISE, AND WHETHER OR NOT THE OTHER PARTY HAS BEEN ADVISED
OF THE POSSIBILITY OF SUCH DAMAGE.

CONTRIBUTORS AND LICENSORS OF THIS SPECIFICATION MAY HAVE MENTIONED CERTAIN
TECHNOLOGIES THAT ARE MERELY REFERENCED WITHIN THIS SPECIFICATION AND NOT
LICENSED UNDER THE OWF CLA OR OWFa.  THE FOLLOWING IS A LIST OF MERELY REFERENCED
TECHNOLOGY: INTELLIGENT PLATFORM MANAGEMENT INTERFACE (IPMI); I2C IS
A TRADEMARK AND TECHNOLOGY OF NXP SEMICONDUCTORS ; EPYC IS A TRADEMARK AND
TECHNOLOGY OF ADVANCED MICRO DEVICES INC.; ASPEED AST 2400/2500 FAMILY
PROCESSORS IS A TECHNOLOGY OF  ASPEED TECHNOLOGY INC.; MOLEX NANOPITCH, NANO
PICOBLADE, AND MINI-FIT JR AND ASSOCIATED CONNECTORS ARE TRADEMARKS AND
TECHNOLOGIES OF MOLEX LLC;  WINBOND IS A TRADEMARK OF WINBOND ELECTRONICS
CORPORATION; NVLINK IS A TECHNOLOGY OF NVIDIA; INTEL XEON SCALABLE PROCESSORS,
INTEL QUICKASSIST TECHNOLOGY, INTEL HYPER-THREADING TECHNOLOGY, ENHANCED INTEL
SPEEDSTEP TECHNOLOGY, INTEL VIRTUALIZATION TECHNOLOGY, INTEL SERVER PLATFORM
SERVICES, INTEL MANAGABILITY ENGINE, AND INTEL TRUSTED EXECUTION TECHNOLOGY ARE
TRADEMARKS AND TECHNOLOGIES OF INTEL CORPORATION; SITARA ARM CORTEX-A9 PROCESSOR
IS A TRADEMARK AND TECHNOLOGY OF TEXAS INSTRUMENTS; GUIDE PINS FROM PENCOM;
BATTERIES FROM PANASONIC.  IMPLEMENTATION OF THESE TECHNOLOGIES MAY BE SUBJECT
TO THEIR OWN LEGAL TERMS.


# Summary

Throughout this document, the term "Processor" refers to all Central Processing
Unit (CPU), System On Chip (SOC), Micro Control Unit (MCU), and Microprocessor
architectures.  The document details the required Challenge Protocol required
for Active Component and Platform RoTs.  The Processor must implement all
required features to establish a hardware based Root of Trust.  Processors that
intrinsically fail to meet these requirements must implement the flash
protection Cerberus RoT described Physical Flash Protection Requirements
document.

Active Components are add-in cards and peripherals that contain Processors,
Microcontrollers or devices that run soft-logic.

This document describes the protocol used for attestation measurements of
firmware for the Platform’s Active RoT.  The specification encompasses the
pre-boot, boot and runtime challenge and verification of platform firmware
integrity.  The hierarchical architecture extends beyond the typically UEFI
measurements, to include integrity measurements of all Active Component
firmware.  The document describes the APIs needed to support the attestation
challenge for Project Cerberus.


# Physical Communication Channel

The typically cloud server motherboard layout has I2C buses routed to all Active
Components.  These I2C buses are typically used by the Baseboard Management
Controller (BMC) for the thermal monitoring of Active Components.  In the
Cerberus board layout, the I2C lanes are first used by the platform Cerberus
microcontroller during boot and pre-boot, then later mux switched back to the
BMC for thermal management.  Cerberus can at any time request for the BMC to
yield control for runtime challenge and attestation.  Cerberus controls the I2C
mux position, and coordinates access during runtime.  It is also possible for
Cerberus to proxy commands through the BMC at runtime, with the option for link
encryption and asymmetric key exchange, making the BMC blind to the
communications.  The Cerberus microcontroller on the motherboard is referred to
as the Platform Active Root-of-Trust (PA-RoT).  This microcontroller is head of
the hierarchical root-of-trust platform design and contains an attestable hash
of all platform firmware kept in the Platform Firmware Manifest (PFM) and
Component Firmware Manifest (CFM).

Most cloud server motherboards route I2C to Active Components for thermal
monitoring, the addition of the mux logic is the only modification to the
motherboard.  An alternative to adding the additional mux, is to tunnel a secure
challenge channel through the BMC over I2C.  Once the BMC has been loaded and
attested by Cerberus, it can act as an I2C proxy.  This approach is less
desirable, as it limits platform attestation should the BMC ever fail
attestation.  In either approach, physical connectors to Active Component
interfaces do not need to change as they already have I2C.

Active Components with the intrinsic security attributes described in the
"Processor Secure Boot Requirements" document do not need to place the physical
Cerberus microcontroller between their Processor and Flash.  Active Components
that do not meet the requirements described in the "Processor Secure Boot
Requirements" document are required to implement the Cerberus micro-controller
between their Processor and Flash to establish the needed Root-of-Trust.  Figure
1 Motherboard I2C lane diagram, represents the pre-boot and post-boot
measurement challenge channels between the motherboard PA-RoT and Active
Component RoTs (AC-RoT).

> TODO: Figure 1

The Project Cerberus firmware attestation is a hierarchical architecture.  Most
Active Components in the modern server boot to an operational level before the
platform’s host processors complete their initialization and become capable of
challenging the devices.  In the Cerberus design, the platform is held in
pre-power-on or reset state, whereby Active Components are quarantined and
challenged for their firmware measurements.  Active Components must respond to
challenges from the PA-RoT confirming the integrity of their firmware before
they are taken out of quarantine.

In this version of the Cerberus platform design, the PFM and CFMs are static.
The manifest is programmable through the PA-RoT’s communication interface.
Auto-detection of Active Components and computation of the PFM/CFM will be
considered in future version of the specification.  The PFM and CFM are
manifests of allowed firmware versions and their corresponding firmware
measurements.  The manifests contain a monotonic identifier used to restrict
rollbacks.

The PA-RoT uses the measurements in the CFM to challenge the Active Components
and compare their measurements.  The PA-RoT then uses the digest of these
measurements as the platform level measurement, creating a hierarchical platform
level digest that can attest the integrity of the platform and active component
firmware.

The PA-RoT will support Authentication, Integrity and Confidentiality of
messages.  Active Components RoT’s (AC-RoT) will support Authentication and
Integrity of messages and challenges.  To facilitate this, AC-RoT are required
to support certificate authentication.  The Active Component will support a
component unique CA signed challenge certificate for authentication.

> Note:  I2C is a low speed link, there is a performance tradeoff between
> optimizing the protocol messages and strong cryptographic hashing algorithms
> that carry higher bit counts.  RoT’s that cannot support certificate
> authentication are required to support hashing algorithms and either RSA or
> ECDSA signatures of firmware measurements.


## Power Control

In the Cerberus motherboard design, power and reset sequencing
is orchestrated by the PA-RoT.  When voltage is applied to the motherboard, it
passes through in-rush circuity to a CPLD that performs time sensitive
sequencing of power rails to ensure stabilization.  Once a power good level is
established, the platform is considered powered on.  Upon initial powering of
the platform in the Cerberus design, the only active component powered-on is the
PA- RoT.  The RoT first securely loads and decompresses its internal firmware,
then verifies the integrity of Baseboard Management Controller (BMC) firmware by
measuring the BMC flash.  When the BMC firmware has been authenticated, the
Active RoT enables power to be applied to the BMC.  Once the BMC has been
powered, the Active RoT authenticates the firmware for the platform UEFI, during
which time the RoT sequences power to the PCIe slots and begins Active Component
RoT challenge.  When the UEFI has been authenticated, the platform is held in
system reset and the Active RoT will keep the system in reset until AC-RoTs have
responded to the measurement challenge.  Any PCIe ports that do not respond to
their measurement challenge will be subsequently unpowered.  Should any of the
expected Active Components fail to respond to the measurement challenge,
Cerberus policies determine whether the system should boot with the Active
Component powered off, or the platform should remain on standby power, while
reporting the measurement failure to the Data Center Management Software through
the OOB path.


# Communication 

The Cerberus PA-RoT communicates with the AC-RoT’s over I2C.  The protocol
supports an authentication and measurement challenge.  The Cerberus PA-RoT
generates a secure asymmetric key pair unique to the microcontroller closely
following the DICE architecture.  Private keys are inaccessible outside of the
secure region of the Cerberus RoT.  Key generation and chaining follows the
RIoT specification, described in section: 9.3 DICE and RIoT Keys and
Certificates.  Derived platform alias public keys are available for use in
attestation and communication from the AC-RoT’s during the challenge handshake
for establishing communication.


## RSA/ECDSA Key Generation

The Cerberus platform Active RoT should support the Device Identifier
Composition Engine (DICE) architecture.  In DICE, the Compound Device Identifier
(CDI) is used as the foundation for device identity and attestation.  Keys
derived from the Unique Device Secret (UDS) and measurement of first mutable
code are used for data protection within the microcontroller and attestation of
upper firmware layers.  The Device Id asymmetric key pair is derived
cryptographically from the CDI and associated with the Device Id Certificate.
The CDI uses the UDS derived from PUF and other random entropy including the
microcontroller unique id and Firmware Security Descriptors.

Cerberus implements the RIoT Core architecture for certificate generation and
attestation.  For details on key generation on DICE and RIoT, review section:
9.3 DICE and RIoT Keys and Certificates.

Note:  The CDI is a compound key based on the UDS, Microcontroller Security
Descriptors and second stage bootloader (mutable) measurement.  The second
stage bootloader is mutable code, but not typically updated with firmware
updates.  Changes to the second stage bootloader will result in a different
CDI, resulting in different asymmetric Device Id key generation.  Certificates
associated with the previous Device key will be invalidated and a new
Certificate would need to be signed.

A second asymmetric key pair certificate is created in the RIoT Core layer of
Cerberus and passed to the Cerberus Application firmware.  This key pair forms
the Alias Certificate and is derived from the CDI, Cerberus Firmware Security
Descriptor and measurement of the next stage Cerberus Firmware.

Proof-of-knowledge of the CDI derived private key known as the Device Id private
key is used as a building-block in a cryptographic protocol to identify the
device.  The Device Id private key is used to sign the Alias Certificate, thus
verifying the integrity of the key.  During initial provisioning, the Device Id
Certificate is CA signed by the Microsoft Certificate Authority.  When
provisioned, the Device Id keys must match the previously signed public key.

> TODO: Figure 2

> Note:  The CDI and Device Id private key are security erased before exiting
> RIoT Core.

Each layer of the software can use its private key certificate to sign and issue
a new certificate for the next layer, each successive layer continues this
chain.  The certificate in the application layer (Alias Certificate) can be
used when authenticating with devices and establishing a secure channel.  The
certificate can also establish authenticity.  Non-application layer private keys
used to sign certificates must be accessible only to the layer they are
generated.  The Public keys are persisted or passed to upper layers, and
eventually exchanged with upstream and downstream entities.


## Chained Measurements

The Cerberus firmware measurements are based on the Device Identifier
Composition Engine (DICE) architecture:
<https://trustedcomputinggroup.org/work-groups/dice-architectures>

The first mutable code on the RoT is the Second Bootloader (SBL).  The CDI is a
measurement of the `HMAC(Device Secret Key + Entropy, H(SBL)`).  This
measurement then passes to the second stage boot loader, that calculates the
digest of the Third Bootloader (TBL).  On the Cerberus RoT this is the
Application Firmware: `HMAC(CDI, H(TBL))`.

> TODO: Figure 4

The Third Stage Bootloader (TBL) which runs the Cerberus Application Firmware
will take additional area measurements of the SPI/QSPI flash for the Processor
it protects, measuring both active and inactive areas.  The TBL measurements
are verified and extended with an attestation freshness seed.  The final
measurement is signed, sealed, and made available to the challenge software.

Seeds for attesting firmware by the application firmware can be extended from
Cerberus Firmware measurements, or using the Alias Certificates a dedicated
freshness seed can be provided for measuring the protected processor firmware.

The measurements are stored in either firmware or hardware register values
within the PA-RoT.  The seed is typically transferred to the device using the
Device Cert, Alias Cert or Attestation Cert.


# Protocol and Hierarchy

The following section describes the capabilities and required protocol and
Application Programming Interface (API) of the motherboard’s Platform Active RoT
(PA-RoT) and Active Component to establish a platform level RoT.  The Cerberus
Active RoT and Active Component RoTs are required to support the following I2C
protocol.

The protocol is derived from the MCTP SMBus/I2C Transport Binding Specification.
A limited version of the protocol is defined for devices that do not support
MCTP.  If an AC-RoT implements the Attestation Protocol over MCTP, it may also
optionally implement the minimum attestation protocol over native SMBus/I2C.


## Attestation Message Interface

The Attestation Message Interface uses the MCTP over I2C message protocol for
transporting the Attestation payloads.  The AC-RoT MCTP Management Endpoint
should implement the required behaviors detailed in the Management Component
Transport Protocol (MCTP) Base Specification, in relation to the MCTP SMBus/I2C
Transport Binding Specification.  The following section outlines additional
requirements upon the expected behavior of the Management Endpoint:

*   The Message Interface Request and Response Messages are transported with a
    custom message type.

*   MCTP messages will be transmitted in a synchronous Request and Response
    manner only.  An Endpoint (AC-RoT) should never initiate a Request Message
    to the Controller (PA-RoT).

*   MCTP Endpoints must strictly adhere to the response timeout defined in this
    specification.  When an Endpoint receives a standard message, it should be
    transmitting the response within 100ms.  If the Endpoint has not begun
    transmitting the Response Message within 100ms, it should drop the message
    and not respond.

*   MCTP Endpoints must strictly adhere to the response timeout advertised for
    cryptographic commands.  Cryptographic commands include transmission of
    messages signature generation and verification.  The cryptographic command
    timeout multiplier is negotiated in the Device Capabilities command.

*   MCTP leaves Authentication to the application implementation.  This
    specification partially follows the flow of USB Authentication Specification
    flow, when authentication has been established attestation seeds can be
    exchanged.

*   It is not required that the Management Endpoint response to ARP messages.
    AC-RoT Endpoints should not generate any ARP messages to Notify Master.
    Devices should be aware they are normally behind I2C muxes and should not
    master the I2C bus outside of the allotted time they are provided to
    response to an MCTP Request Message.

*   MCTP Endpoint devices should be response only.

*   Irrespective as to whether Endpoints are ARP capable, they should operate in
    a Non-ARP-capable manner.

*   MCTP specifications use big endian byte ordering while this specification
    uses little endian byte ordering.  This ordering does not change the payload
    order in which bytes are sent out on the physical layer.

*   Endpoints should support Fixed Addresses; Endpoint IDs are supported to
    permit multiple MCTP Endpoints behind a single physical address.

*   As defined in the MCTP SMBus/I2C Transport Binding Specification Endpoints
    should support fast-mode 400KHz.

*   Endpoint devices that do not support multi-master should operate in slave
    mode.  The PA-RoT PCD will identify the mode of the device.  The Master
    will issue SMBUS Write Block in MCTP payload format, the master will then
    issue an I2C Read for the response.  The Master read the initial 12 bytes
    and use byte 3 and the MCTP EOM header bits to determine if additional SMBUS
    read commands are required to collect the remainder of the response message.

*   Endpoints should support EID assignment using the MCTP Set Endpoint ID
    control message.

*   Endpoints should support the MCTP Get Vendor Defined Message Support control
    message to indicate support level for the Cerberus protocol.

The Platform Cerberus Active RoT is always the MCTP master.  Active Component
RoT’s can be configured as Endpoint, or Endpoint and Master.  An Active
Component RoT Endpoint and Master should interface to separate physical busses.
There is no requirement for master arbitration as master and slave definitions
are hierarchically established.  The only hierarchy whereby the Active Component
RoT becomes both Endpoint and Master is when there is a downstream sub-device,
such as the Host Bus Adapter (HBA) depicted in the following block diagram:

> TODO: Figure 6

In this diagram, the HBA RoT is an Endpoint to the Platform Active RoT and
Master to the downstream HBA Expanders.  To the Platform’s Active RoT, the HBA
is an Endpoint RoT.  To the HBA Expanders, the HBA Controller is a Master RoT.

The messaging protocol encompasses Management Component Transport Protocol
(MCPT) Base Specification, in relation to the MCTP SMBus/I2C Transport Binding
Specification, whereby the Active Component RoT is Endpoint and the Platform’s
Active RoT as Master.

## Protocol Format

All MCTP transactions are based on the SMBus Block Write bus protocol.  The
following table shows an MCTP encapsulated message.

`message MCTP.Message`
| Type           | Name             | Description                            |
|----------------|------------------|----------------------------------------|
| `b8`           | `i2c_dest`       | I2C destination address.               |
| `0x0f`         | `command_code`   | Command code.                          |
| `b8`           | `packet_len`     | Number of bytes in the packet payload. |
| `b8`           | `i2c_src`        | I2C source address.                    |
| `0b0000`       | `_`              | Reserved.                              |
| `b4`           | `header_verison` | MCTP Header version.                   |
| `b8`           | `dest_eid`       | Destination EID.                       |
| `b8`           | `src_eid`        | Source EID.                            |
| `b1`           | `som`            | Start of message (SOM) flag.           |
| `b1`           | `eom`            | End of message (EOM) flag.             |
| `b2`           | `seq_num`        | Sequence number.                       |
| `b1`           | `tag_owner`      | Tag owner.                             |
| `b3`           | `tag`            | Message tag.                           |
| `[packet_len]` | `payload`        | Packet payload.                        |
| `[8]`          | `pec`            | PEC                                    |

A package should contain a minimum of 1 byte of payload, with the maximum not to
exceed the negotiated MCTP Transmission Unit Size.  The byte count indicates the
number of subsequent bytes in the transaction, excluding the PEC.

The PEC at the end of each transaction is calculated using a standard CRC-8,
which uses polynomial `x^8 + x^2 + x + 1`, initial value of 0,
and final XOR of 0.  This CRC is independent of the message integrity check
defined by the MCTP protocol.  The CRC is calculated over the entire
encapsulated MCTP packet, which includes all headers and the destination I2C
address as it was sent over the I2C bus (bytes 1 through N in Figure 7 MCTP
Encapsulated Message).  Since the I2C slave address is not typically included as
part of the transaction data, the CRC is equal to CRC8 (`7_bit_address << 1 ||
i2c_payload`).  For example, an MCTP packet sent to a device with 7-bit I2C
address `0x41` would have a CRC calculated as CRC8 (`0x82 || packet`).


## Packet Format

The Physical Medium-Specific Header and Physical Medium-Specific Trailer are
defined by the MCTP transport binding specification utilized by the port.
Refer to the MCTP transport binding specifications.

A compliant Management Endpoint shall implement all MCTP required features
defined in the MCTP base specification.

The base protocol’s common fields include message type field that identifies the
higher layer class of message being carried within the MCTP protocol.


## Transport Layer Header

The Management Component Transport Protocol (MCTP) Base Specification defines
the MCTP packet header (refer to DSP0236 for field descriptions) as follows.
Note that offsets are in bits, rather than bytes. This table corresponds to the
data after bit `31` in the table above.

<!-- NOTE: the `Variable` fields currently cannot be expressed in the table
format. -->

| Payload  | Description                  |
|----------|------------------------------|
| 3:0      | Reserved (should be zero).   |
| 7:4      | MCTP Header version.         |
| 15:8     | Destination EID.             |
| 23:16    | Source EID.                  |
| 24:24    | Start of message (SOM) flag. |
| 25:25    | End of message (EOM) flag.   |
| 27:26    | Sequence number.             |
| 28:28    | Tag owner.                   |
| 31:29    | Message tag.                 |
| 32:32    | Integrity check flag.        |
| 39:33    | MCTP message type.           |
| Variable | MCTP header.                 |
| Variable | MCTP message data.           |
| Variable | Integrity check.             |


Null (`0x00`) Source and Destination EIDs are typically supported, however
AC-RoT devices that have multiple MCTP Endpoints may specify an EID value
greater than `0x07` and less than `0xff`.  The PA-RoT does not broadcast any
MCTP messages.

Note that the last three fields above are dependent on the MCTP message type.


## MCTP Messages

An MCTP message consists of one or more MCTP packets.  There are typically two
types of Messages, MCTP Control Messages and MCTP Cerberus Messages.  Control
Messages are standard MCTP messages with a maximum message body of 64 bytes.
Cerberus Messages are those defined in this specification.  The maximum message
body for these messages is 4096 bytes, but this size can be negotiated to be
smaller based on device capabilities.


### Message Type

The message type should be `0x7e` as per the Management Component Transport
Protocol (MCTP) Base Specification.  The message type is used to support Vendor
Defined Messages where the Vendor is identified by the PCI based Vendor ID.  The
initial message header is specified in the Management Component Transport
Protocol (MCTP) Base Specification, and detailed below for completeness:


    Table 2 Vendor Defined Message


| Message Header | Byte | Description                                                                         |
|----------------|------|-------------------------------------------------------------------------------------|
| Request Data   | 1:2  | PCI/PCIe Vendor ID.  The MCTP Vendor Id formatted per 0x00 Vendor ID format offset. |
|                | 3:N  | Vendor-Defined Message Body.  0 to N bytes.                                         |
| Response Data  | 1:2  | PCI/PCIe Vendor ID, the value is formatted per 0x00 Vendor ID offset.               |
|                | 3:M  | Vendor-Defined Message Body.  0 to M bytes.                                         |


The Vendor ID is a 16-bit unsigned integer, described in the PCI 2.3
specification.  The value identifies the device manufacturer.

The message body and content are described in Table 4 MCTP Message Format.


### Message Fields

The format of the MCTP message consists of a message header in the first two
bytes, followed by the message data, and ending with the Message Integrity
Check.

The Message header contains a Message Type (MT) field and Integrity Check (IC)
that are defined by the MCTP Base Specification.  The Message Type field
indicate 


### Message Integrity Check

The Message Integrity Check field contains a 32-bit CRC computed over the
contents of the message.


### Packet Assembly into Messages

An MCTP message may be split into multiple MCTP Packet Payloads and sent as a
series of packets.  Refer to the MCTP Base Specification for packetization and
message assembly rules.


### Request Messages

Request Messages are messages that are generated by a Master MTCP Controller and
sent to an MCTP Endpoint.  Request Messages specify an action to be performed by
the Endpoint.  Request Messages are either Control Messages or Cerberus Messages.


### Response Messages

Response Messages are messages that are generated when an MCTP Endpoint
completes processing of a previously issued Request Message.  The Response
Message must be completed within the allocated time or discarded.


## EID Assignment

The BMC will assign EIDs to the different Active Component RoT devices.  All
Active Component RoT devices should support the MCTP Set Endpoint ID control
request and response messages.  The Platform Active RoT will have a static EID of
0x0b.


# Certificates

The PA-RoT and AC-Rot will have a minimum of two certificates: Device Id
Certificate (typically CA signed by offline CA) and the Alias Certificate
(signed by the Device Id Certificate).  The PA-RoT may also have an additional
Attestation Certificate signed by the Device Id Certificate.

Certificates follow the 9.3 DICE and RIoT Keys and Certificates.


## Format

All Certificates shall use the X509v3 ASN.1 structure.  All Certificates shall
use binary DER encoding for ASN.1.  All Certificates shall be compliant with
RFC5280.  To facilitate certificate chain authentication, Authority and Subject
Key Identifier extensions must be present.  Further certificate requirements are
defined in the 9.3 DICE and RIoT Keys and Certificates.  Extensions beyond those
required by RFC5280 or DICE are allowed so long as they conform to the
appropriate standards.  Custom extensions must be marked non-critical.


### Textual Format

All text ASN.1 objects contained within Certificates, shall be specified as
either a `UTF8String`, `PrintableString`, or `IA5String`.  The length of any
textual object shall not exceed 64 bytes excluding the DER type and DER length
encoding.


### Distinguished Names

The distinguished name consists of many attributes that uniquely identify the
device.  Distinguished name uniqueness can be accomplished by including
attributes such as the serial number.


### Object Identifiers

Object Identifiers should follow 9.3 DICE and RIoT Keys and Certificates


### Serial Numbers

As per 9.3 DICE and RIoT Keys and Certificates, the Certificate Serial Numbers
MUST be statistically unique per-Alias Certificate.

If the security processor has an entropy source, an 8-octet random number MAY be
used.

If the security processor has does not have an entropy source, then an 8-octet
Serial Number MAY be generated using a cryptographically secure key derivation
function based on a secret key, such as those described in SP800-108 [9.6 NIST
Special Publication 800-108].  The Serial Number MUST be unique for each
generated certificate.  For the Alias Certificate, this SHOULD be achieved by
incorporating the FWID into the key derivation process (e.g. as the Context
value in SP-800-108)

If the 8-octet serial number generated is not a positive number, it may be
padded with an additional octet to maintain compliance with RFC5280.


## Certificate Chains

The maximum Certificate Chain Size is 4096 bytes.  There is no additional
encoding necessary for the certificate chain as each certificate can be
retrieved individually.

Certificate signatures should use ECDSA with the NIST P256 `secp256r1` curve in
uncompressed point form. Hashing should be performed with the `SHA2-256`
algorithm.


# Authentication

Authentication is the process used to establish trust in a specific device and
prove integrity of the device firmware.  Once a device is authenticated, a
secure session can optionally be established to provide confidentiality.  For
some devices, a secure session can also be used to enable a cryptographic
binding that can be used for additional security or to enable additional
functionality.

A device only needs to support a single session from another endpoint.  In
single session devices, the establishment of a new session will supersede and
terminate the existing session to permit the new session.  The previous session
will be terminated upon receiving an out of session (unencrypted) `GET_DIGESTS`
request specifying a requested key exchange.  `GET_DIGESTS` requests received in
session (encrypted) or without requesting key exchange will not cause the active
session to terminate.


## PA-RoT and AC-RoT Authentication

Devices are authenticated using the Alias certificate chain and signed firmware
measurements.  The certificate chain is endorsed by the Certificate Authority
signing the Device Id Certificate.  The certificate hierarchy is explained in
the TCG [Implicit Identity Based Device Attestation](https://trustedcomputinggroup.org/wp-content/uploads/TCG-DICE-Arch-Implicit-Identity-Based-Device-Attestation-v1-rev93.pdf)
Reference.

The certificate authentication flow closely follows the USB Authentication
Architecture and Authentication Messages.  The command structures defined in
this specification are derived from the USB-C Authentication Protocol.  Relevant
sections of the specification are as follows:

- Section 3: Authentication Architecture, *USB Authentication Specification*

- Section 4: Authentication Protocol, *USB Authentication Specification*

- Section 5: Authentication Messages, *USB Authentication Specification*

The authentication sequence starts with the master (e.g.  PA-RoT) issuing a Get
Digests command.  The slave (e.g.  AC-RoT) responds with a list of SHA256 hashes
for each certificate in the device’s Alias certificate chain.  If the master has
cached the certificate chain, it may optionally skip requesting these
certificates.

If the master does not have the correct certificates, it will issue a Get
Certificate request for each certificate in the slave’s Alias Certificate chain.
The slave will respond with the requested certificates, which must have
origination from a trusted CA.

The master will verify the Alias certificate chain of the slave.  If
verification fails or the certificate has been revoked, the authentication will
fail.  The PA-RoT and AC-RoT can be updated with revocation patches, see
firmware update specification for further details.

Once the certificate chain has been authenticated, the master will verify
firmware integrity through the Challenge request.  The slave will generate a
Challenge response signed with its Alias key which includes the measurement of
device firmware.  The master can verify this measurement against a Component
Firmware Manifest (CFM) or some other reference to confirm the firmware is
valid.

The authentication flow is as follows:

1.  Master issues `GET_DIGESTS` command for Slot 0
2.  Slave responds with digests for the Alias certificate chain
3.  For each certificate in the chain, Master checks if the certificate has been
    cached.
4.  If the certificate has not been cached, Master issues `GET_CERTIFICATE`
    request.
5.  Slave responds with the requested certificate.
6.  Master verifies the Alias certificate chain against a trusted root CA.
7.  Master issues Challenge command containing a nonce `RN1`.
8.  Slave generates a response containing
    *   Random nonce, `RN2`.
    *   Collective firmware measurement, `PMR0`.
    *   Signature over (`request || response`) pair using the Alias key.
9.  Master verifies the signature and firmware measurement

> TODO: Figure 9


## Secure Session Establishment

Cerberus devices may optionally support secure session establishment to provide
confidentiality between two devices or between external data center software and
the device.  Session establishment leverages the basic authentication flow with
an additional key exchange to establish the secure session.  Device
authentication uses the identity certificate chain, while the session key
exchange is based on ephemeral ECDH keys.  The Device Capabilities command will
determine if a device supports secure sessions.

After querying the for the device capabilities, the master will begin the
authentication sequence by issuing a Get Digests request.  When using
authentication to establish a secure session, the `GET_DIGESTS` request will
also indicate the type of key exchange that will be ultimately be used.  If the
device does not support the requested key exchange, it will respond with an
error.

After authentication has completed, the master will generate an ephemeral key
and send it to the slave.  The slave will generate its own ephemeral session key
and complete the key exchange.  Session keys will be derived by running ECDH and
subsequent KDFs over the session keys and nonces that were included in the
challenge.

The KDF used for deriving session and HMAC keys is NIST SP800-108 Counter Mode.
Session state is volatile.  Each end of the session must be able to handle a
loss of session context and support session re-establishment.

### Session Establishment Sequence

The session establishment flow starts with standard authentication described is
Section 5.1.1.  Once the slave has been authenticated, the key exchange is used
to establish the session.

1.  Master generates an ephemeral ECC key pair and sends a `Key Exchange`
    request with Type 0 that includes the public key (`PKreq`).
2.  Slave generates an ephemeral ECC key pair of equivalent strength (`PKresp`).
3.  Slave generates a 256-bit AES session key (`K_S`) using NIST SP800-108
    Counter Mode with the following parameters:
    *   `K_I = ECDH(PKreq, PKresp)`
    *   `label = RN1`
    *   `context = RN2`
    *   `r = 32`
4.  Slave generates a 256-bit MAC key (`K_M`) using NIST SP800-108
    Counter Mode with the following parameters (note `label` and `context` are
    swapped):
    *   `K_I = ECDH(PKreq, PKresp)`
    *   `label = RN2`
    *   `context = RN1`
    *   `r = 32`
5.  Slave sends a Key Exchange response that includes:
    1.  The slave session public key (`PKresp`).
    2.  A signature using the Alias key over `PKreq` and `PKresp`.
    3.  An HMAC of its Alias key certificate using `K_M`.
6.  Master generates the same session and MAC keys using the public key in the
    Key Exchange response.
7.  Master validates the signature and HMAC on the response.
8.  Messages can now be encrypted with 256-bit AES-GCM using the shared session
    key.

> TODO: Figure 10


## Secure Device Binding

In some scenarios, it is necessary to have an additional level of authentication
between the devices.  This additional authentication mechanism is used to pair
two devices such that replacement of either device would be detected.  Detection
of unauthorized part replacement is flagged as a security violation.

The binding comes from an additional KDF run on the session key to derive the
final session key.  Each device partaking in the tightly coupled pairing contain
a common HMAC key that is used in a KDF to device the paired session key.  This
common HMAC key is derived from session data available the first time the two
devices establish a secure session.


### Secure Device Binding Sequence

A paired session with device binding begins with a standard secure session.
Once the session has been established, an additional key exchange is used to
update the session key.  This additional key exchange must be encrypted.


1.  Master loads the pairing key `K_P`.
    1.  If this is the first pairing request, the Master generates the key using
        NIST SP800-108 Counter Mode with the following paramters:
        *   `K_I = K_S`
        *   `label = "pairing"`
        *   No `context`
        *   `r = 32`
    2.  If the Master has already been paired to the Slave, the Master loads key
        from secure storage.
2.  Master generates a new session key `K_S'` using NIST SP800-108 Counter Mode.
    *   `K_I = K_P`
    *   `label = K_S`
    *   No `context`
    *   `r = 32`
3.  Master sends a Key Exchange request with Type 1 that includes the length and
    HMAC of the pairing key using `K_M`.  This message is encrypted with the
    current session key `K_S`.
4. Slave loads `K_P`
    1.  If this is the first paring request from this Master, the Slave
        generates the key using the same inputs as the Master.
    2.  If the Slave has already paired with the Master, the Slave loads the key
        from secure storage.
5.  Slave calculates the pairing key HMAC and compares it to the received data.
    *   If this is the first paring request from this Master, Slave securely
        stores `K_P`.
7.  Slave generates a new session key (`K_S'`).
8.  Slave generates a Key Exchange response encrypted with `K_S'`.
9.  Master verifies the response using `K_S'`.  Failure to decrypt with `K_S'`
    will require the Master to decrypt with `K_S` since the Slave will not
    update the session key on failure.
10. The secure session continues using `K_S'`.  Messages encrypted with
    `K_S` will not be processed by either side.

> TODO: Figure 11


# Command Format

The following section describes the MCTP message format to support the
Authentication and Challenge and Attestation protocol.  The Request/Response
message body describes the Vendor Defined MCTP message encapsulated inside the
MCTP transport.  This section does not describe the MCTP Transport Header, which
includes the MCTP Header Version, Destination Endpoint ID and other fields as
defined by MCTP protocol.  The MCTP message encapsulation is described in
section 3.2

The MCTP Get Vendor Defined Message Support command enables discovery of what
Endpoint vendor defined messages are supported.  The discovery identifies the
vendor organization and defined messages types.  The format of this request is
described in the MCTP base protocol specification.

For the Cerberus protocol, the following information shall be returned in
response to the MCTP "Get Vendor Defined Message Support" request:

*   Vendor ID Format: `0x00`
*   PCI Vendor ID: `0x1414`
*   Command Set Version: `0x04`


## Attestation Protocol Format

Messages from a PA-RoT to an AC-RoT will have the following format.
This is embedded in the structure defined in Section 3.4.

`message CerberusHeader`
| Type      | Name                | Description                   |
|-----------|---------------------|-------------------------------|
| `b1`      | `integrity_check`   | Integrity check flag.         |
| `0x7e`    | `mctp_message_type` | MCTP message type.            |
| `0x1414`  | `mctp_pci_vendor`   | MCTP PCI Vendor ID.           |
| `b1`      | `request_type`      | Request type.                 |
| `0b0`     | `_`                 | Reserved.                     |
| `b1`      | `is_encrypted`      | Set if encrypted (see below). |
| `0b00000` | `_`                 | Reserved.                     |
| `b8`      | `command_byte`      | Cerberus command byte.        |
| `...`     | `payload`           | The command payload.          |

The protocol header fields are to be included only in the first packet of a
multiple packet MCTP message.  After reconstruction of the message body, the
protocol header will be used to interpret the message contents.  Reserved fields
must be set to 0.

The "request type" field indicates what type of request is contained in the
message.  Messages defined in this specification shall have this field set to 0.
Setting this bit to 1 provides a mechanism, besides different vendor IDs, to
support a device-specific command set.  Devices that don't have any additional
command support will return an error if this bit is 1.


## Encrypted Messages

If the crypt field is set in the protocol header, the message body contains
encrypted data.  The command code byte and full message payload, reconstructed
from individual MCTP packets, is encrypted.  The session establishment flow
described in section 5 is used to generate the encryption keys.  An encrypted
message will have a 16-byte GCM authentication tag and 12-byte initialization
vector in plaintext at the end of the message body.  The following table shows
the body of an encrypted Cerberus message, with the encryption trailer. Note
that offsets are given in bits, not bytes.

`message CerberusHeader`
| Type      | Name                | Description                                |
|-----------|---------------------|--------------------------------------------|
| `b1`      | `integrity_check`   | Integrity check flag.                      |
| `0x7e`    | `mctp_message_type` | MCTP message type.                         |
| `0x1414`  | `mctp_pci_vendor`   | MCTP PCI Vendor ID.                        |
| `b1`      | `request_type`      | Request type.                              |
| `0b0`     | `_`                 | Reserved.                                  |
| `0b1`     | `is_encrypted`      | Set if encrypted (always, in this case).   |
| `0b00000` | `_`                 | Reserved.                                  |
| `...`     | `ciphertext`        | Ciphertext; corresponds to the command byte and its payload above. |
| `b16`     | `aead_tag`          | AES-GCM tag.                               |
| `b16`     | `aead_iv`           | AES-GCM initialization vector.             |

## RoT Commands

The following table summarizes messages defined under this specification, with
the folloiwing legend:
*   "Enc" means the command must be encrypted within a session.
*   "Slow" means the command performs cryptographic operations, and should use
    the longer message timeout.
*   "Byte" is the command byte that identifies this message.
*   "Required" is whether the message *must* be implemented: `R` means
    "required in all cases"; `O` means "implement if necessary"; `M` means
    "required for an attestation master".

| Message Name               | Enc | Slow | Byte | Required | Description                                         |
|----------------------------|-----|------|------|----------|-----------------------------------------------------|
| `ERROR`                    |     |      | 0x7f | R        | Default response message.                           |
| `Firmware Version`         |     |      | 0x01 | R        | Retrieves firmware version information.             |
| `Device Capabilities`      |     |      | 0x02 | R        | Retrieves device capabilities.                      |
| `Device Id`                |     |      | 0x03 | R        | Retrieves device id.                                |
| `Device Information`       |     |      | 0x04 | R        | Retrieves device information.                       |
| `Export CSR`               |     |      | 0x20 | R        | Exports CSR for device keys.                        |
| `Import Certificate`       |     | X    | 0x21 | R        | Imports CA signed Certificate.                      |
| `Get Certificate State`    |     |      | 0x22 | R        | Checks the state of the signed Certificate chain.   |
| `GET DIGESTS`              |     | X    | 0x81 | R        | Retrieves certificate chain digests.                |
| `GET CERTIFICATE`          |     |      | 0x82 | R        | Retrieves certificate chain.                        |
| `CHALLENGE`                |     | X    | 0x83 | R        | Authenticates the collective measurement.           |
| `Key Exchange`             |     | X    | 0x84 | O        | Exchanges session keys and mfg device pairing keys. |
| `Session Sync`             | X   | X    | 0x85 | O        | Checks status of a secure session.                  |
| `Get Log Info`             |     |      | 0x4f | O        | Retrieves logging information.                      |
| `Get Log`                  |     |      | 0x50 | O        | Retrieves debug, attestation, and tamper logs.      |
| `Clear Log`                |     |      | 0x51 | O        | Clears device logs.                                 |
| `Get Attestation Data`     |     |      | 0x52 | O        | Retrieves raw data from the attestation log.        |
| `Get Host State`           |     |      | 0x40 | O        | Gets reset state of the host processor.             |
| `Get PFM Id`               |     |      | 0x59 | O        | Gets PFM information.                               |
| `Get PFM Supported`        |     |      | 0x5a | O        | Retrieves the PFM.                                  |
| `Prepare PFM`              |     |      | 0x5b | O        | Prepares an incoming PFM update.                    |
| `Update PFM`               |     |      | 0x5c | O        | Uploads part of a new PFM.                          |
| `Activate PFM`             |     |      | 0x5d | O        | Activates a newly-staged PFM.                       |
| `Get CFM Id`               |     |      | 0x5e | M        | Gets CFM information.                               |
| `Prepare CFM`              |     |      | 0x5f | M        | Prepares an incoming CFM update.                    |
| `Update CFM`               |     |      | 0x60 | M        | Uploads part of a new CFM.                          |
| `Activate CFM`             |     |      | 0x61 | M        | Activates a newly-staged CFM.                       |
| `Get CFM Supported`        |     |      | 0x8d | M        | Retrieve supported CFM IDs.                         |
| `Get PCD Id`               |     |      | 0x62 | M        | Gets PCD information.                               |
| `Prepare PCD`              |     |      | 0x63 | M        | Prepares an incoming PCD update.                    |
| `Update PCD`               |     |      | 0x64 | M        | Uploads part of a new PCD.                          |
| `Activate PCD`             |     |      | 0x65 | M        | Activates a newly-staged PCD.                       |
| `Prepare Firmware Update`  |     |      | 0x66 | O        | Prepares an incoming firmware update.               |
| `Update Firmware`          |     |      | 0x67 | O        | Uploads part of a new firmare image.                |
| `Update Status`            |     |      | 0x68 | M        | Firmware or manifest update status                  |
| `Extended Update Status`   |     |      | 0x8e | M        | Firmware or manifest extended status                |
| `Activate Firmware Update` |     |      | 0x69 | O        | Activates a newly-staged firmware update.           |
| `Reset Configuration`      |     | X    | 0x6a | O        | Resets configuration to default state.              |
| `Get Config IDs`           |     | X    | 0x70 | M        | Gets authenticated manifest IDs.                    |
| `Recovery Firmware`        |     |      | 0x71 | O        | Restores firmware index using backup.               |
| `Prepare Recovery Image`   |     |      | 0x72 | O        | Prepares an incoming recovery image update.         |
| `Update Recovery Image`    |     |      | 0x73 | O        | Uploads part of a new recovery image.               |
| `Activate Recovery Image`  |     |      | 0x74 | O        | Activate newly-staged recovery image.               |
| `Get Recovery Image Id`    |     |      | 0x75 | O        | Get recovery image information.                     |
| `Get PMR`                  |     | X    | 0x80 | O        | Gets a Platform Measurement Register.               |
| `Update PMR`               | X   | X    | 0x86 | O        | Extends a Platform Measurements Register.           |
| `Reset Counter`            |     |      | 0x87 | R        | Reset Counter.                                      |
| `Unseal Message`           |     | X    | 0x89 | O        | Begin an unsealing challenge.                       |
| `Unseal Message Result`    |     |      | 0x8a | O        | Get unsealing status and result.                    |

Command bytes `0xf0` through `0xff` will never be allocated by Cerberus, and may
be treated as a "private use" area.

A note on required messages:
1. `Key Exchange` and `Session Sync` are required if any "Enc" messages are supported.
2. `Get Attestation Data` is required if an Attestation Log is supported.
3. If any update command (e.g., `Prepare`/`Update`/`Activate` commands) are
   supported, `Update Status` and `Extended Update Status` are required.
4. If any manifest types are supported, `Get Config IDs` is required.


## Message Body Structures

The following section describes the structures of the MCTP message body.

<!-- Style note: structures pertinent to one message should have that message's
  name as a namespace prefix; .Request and .Response should come first. -->

### Error Message

The error command is returned for command responses when the command was not
completed as proposed, it also acts as a generic status for commands without
response whereby "No Error" code would indicate success.  The Message Tag,
Sequence Number and Command match the response to the corresponding request.
The Message Body is returned as follows:

`message Error.Response`
| Type   | Name   | Description |
|--------|--------|-------------|
| `Code` | `code` | Error code  |
| `[4]`  | `data` | Error data  |

`enum Error.Code`
| Value  | Name               | Description                              |
|--------|--------------------|------------------------------------------|
| `0x00` | `success`          | Success.                                 |
| `0x01` | `invalid_data`     | Invalidated data in the request          |
| `0x03` | `busy`             | Device is busy processing other commands |
| `0x04` | `unspecified`      | Vendor-defined error occurred            |
| `0xf0` | `bad_checksum`     | Invalid checksum                         |
| `0xf1` | `eom_before_som`   | EOM before SOM                           |
| `0xf2` | `no_auth`          | Authentication not established           |
| `0xf3` | `out_of_order`     | Message received out of Sequence Window  |
| `0xf4` | `bad_packet_size`  | Packet received with unexpected size     |
| `0xf5` | `bad_message_size` | Message exceeded maximum length          |

<!-- Note: we do not use an enum type mapping because `data`'s size is not
  dependent on the code. -->
Unless otherwise specified, the `data` field should be interpreted as zero:
- `unspecified`: `data` is vendor-defined.
- `bad_checksum`: `data` should hold the expected checksum.
- `bad_packet_size`: `data` is the offending packet length.
- `bad_message_size`: `data` is the offending message length.

If an explicit response is not defined for the command definitions in the
following sections, the Error Message is the expected response with "No Error".
The Error Message can occur as the response for any command that fails
processing.


### Firmware Version

This command gets the target firmware the version.

`message FirmwareVersion.Request`
| Type        | Name         | Description |
|-------------|--------------|-------------|
| `AreaIndex` | `area_index` | Area index. |

`message FirmwareVersion.Response`
| Type   | Name               | Description                                    |
|--------|--------------------|------------------------------------------------|
| `[32]` | `firmware_version` | Firmware Version Number; usually encoded as ASCII. |

<!-- TODO: We cannot express this is an open enum. -->
`enum FirmwareVersion.AreaIndex`
| Value  | Name        | Description          |
|--------|-------------|----------------------|
| `0x00` | `all`       | The entire firmware. |
| `0x01` | `riot_core` | The RIoT Core.       |


### Device Capabilities

Device Capabilities provides information on device functionality.  The message
and packet maximums are negotiated based on the shared capabilities.  Some
additional considerations apply when determining appropriate maximum sizes:


1.  Per the MCTP specification, the packet payload size must be no less than 64
    bytes.

2.  Per section 3.5, the message payload size must be no more than 4096 bytes.

3.  The total packet size supported by the device will include the encapsulation
    defined in section 3.2, excluding the first byte of destination address.
    For example, a maximum packet payload of 247 bytes will result in 256 bytes
    being transmitted for every packet:
    *   1 byte destination address.
    *   3 bytes SMBus header.
    *   4 bytes MCTP header.
    *   247 bytes payload.
    *   1 byte CRC-8.

4.  Some commands do not support being separated across messages.  Devices must
    support a maximum message size that is compatible with the expected payloads
    for supported commands.

5.  Master devices, such as PA-RoTs, should support the maximum message size of
    4096 bytes to ensure full compatibility with any slave device.

<!-- TODO: convert these to a proper markdown table. As it is, it's a bit too
    complex to do mechnaically. -->


`message DeviceCapabilities.Request`
| Type       | Name              | Description                   |
|------------|-------------------|-------------------------------|
| `b16`      | `max_message_len` | Maximum message payload size. |
| `b16`      | `max_packet_len`  | Maximum packet payload size.  |
| `Features` | `features`        | RoT features.                 |

`message DeviceCapabilities.Response`
| Type       | Name              | Description                                 |
|------------|-------------------|---------------------------------------------|
| `b16`      | `max_message_len` | Maximum message payload size.               |
| `b16`      | `max_packet_len`  | Maximum packet payload size.                |
| `Features` | `features`        | RoT features.                               |
| `b8`       | `message_timeout` | Maximum message timeout in units of 10ms.   |
| `b8`       | `crypto_timeout`  | Maximum cryptographic operation timeout in units of 100ms. |

`message DeviceCapabilities.Features`
| Type      | Name                 | Description                               |
|-----------|----------------------|-------------------------------------------|
| `b1`      | `has_hashing`        | Whether hashing and KDF is available.     |
| `b1`      | `has_auth`           | Whether authentication is available.      |
| `b1`      | `has_aead`           | Whether AEAD-based confidentiality is available. |
| `0b0`     | `_`                  | Reserved.                                 |
| `BusRole` | `bus_role`           | The RoT's bus roles.                      |
| `RotType` | `rot_type`           | Cerberus RoT types the device implements. |
| `b5`      | `_`                  | Reserved.                                 |
| `b1`      | `has_fw_protection`  | Whether this RoT supports firmware protection. |
| `b1`      | `has_policy_support` | Whether this RoT supports policies.       |
| `b1`      | `has_pfms`           | Whether this RoT supports PFMs.           |
| `b1`      | `rsa2048`            | Whether 2048-bit RSA is supported.        |
| `b1`      | `rsa3072`            | Whether 3072-bit RSA is supported.        |
| `b1`      | `rsa4096`            | Whether 4096-bit RSA is supported.        |
| `b1`      | `ecc160`             | Whether 160-bit ECC is supported.         |
| `b1`      | `ecc256`             | Whether 256-bit ECC is supported.         |
| `0b0`     | `_`                  | Reserved.                                 |
| `b1`      | `has_ecdsa`          | Whether this RoT supports ECDSA keys.     |
| `b1`      | `has_rsa`            | Whether this RoT supports RSA keys.       |
| `b1`      | `aes128`             | Whether 128-bit AES is supported.         |
| `b1`      | `aes256`             | Whether 256-bit AES is supported.         |
| `b1`      | `aes384`             | Whether 384-bit AES is supported.         |
| `b4`      | `_`                  | Reserved.                                 |
| `b1`      | `has_ecc`            | Whether ECC is supported.                 |


### Device Id

`message DeviceId.Request`
| Type | Name | Description |
|------|------|-------------|

`message DeviceId.Response`
| Type  | Name                  | Description               |
|-------|-----------------------|---------------------------|
| `b16` | `vendor_id`           | PCIe Vendor ID.           |
| `b16` | `device_id`           | PCIe Device ID.           |
| `b16` | `subsystem_vendor_id` | PCIe Subsystem Vendor ID. |
| `b16` | `subsystem_id`        | PCIe Subsystem ID.        |


### Device Information

This command gets information about the target device.

`message DeviceInfo.Request`
| Type    | Name    | Description        |
|---------|---------|--------------------|
| `Index` | `index` | Information index. |

`message DeviceInfo.Response`
| Type  | Name   | Description               |
|-------|--------|---------------------------|
| `...` | `data` | Requested identifier blob |

<!-- TODO: We cannot express this is an open enum. -->
`enum DeviceInfo.Index`
| Value  | Name  | Description             |
|--------|-------|-------------------------|
| `0x00` | `uci` | Unique Chip Identigier. |


### Export CSR

Exports the Device Identification Certificate Self Signed Certificate Signing
Request.  The initial Certificate is self-signed, until CA signed and imported.
Once the CA signed version of the certificate has been imported to the device,
the self-signed certificate is replaced.

`message ExportCSR.Request`
| Type | Name   | Description             |
|------|--------|-------------------------|
| `b8` | `slot` | Certificate chain slot. |

`message ExportCSR.Response`
| Type  | Name  | Description                 |
|-------|-------|-----------------------------|
| `...` | `csr` | Certificate Signing Request |


### Import Certificate

Imports the signed Device Id certificate chain into the device.
Each import command sends a single certificate in the chain to the device, and
there is no requirement on the order in which these certificates must be
imported.  Upon verification of a complete chain, the device is sealed, and no
further imports can occur without changes to firmware.  A response message is
returned before certificate chain validation has been performed with the new
certificate and only indicates if the certificate was accepted by the device.
The complete state of certificate provisioning can be queried with the Get
Certificate State request.

Upon receiving a certificate, the device will start to authenticate the stored
certificate chain.  When sending multiple certificates, it may be necessary to
ensure the previous authentication step has completed before sending a new
certificate.  The authentication status can be checked with the Get Certificate
State command.

`message ImportCert.Request`
| Type    | Name   | Description                       |
|---------|--------|-----------------------------------|
| `Type`  | `type` | Certificate type.                 |
| `[b16]` | `cert` | DER-encoded certificate contents. |

`enum ImportCert.Type`
| Value  | Name           | Description                  |
|--------|----------------|------------------------------|
| `0x00` | `device_id`    | Owner-signed device ID cert. |
| `0x01` | `root`         | Owner root CA cert.          |
| `0x02` | `intermediate` | Owner intermediate CA cert.  |


### Get Certificate State

Determine the state of the certificate chain for signed certificates that have
been sent to the device.  The request for this command contains no additional
payload.

`message GetCertState.Request`
| Type | Name | Description |
|------|------|-------------|

`message GetCertState.Response`
| Type    | Name    | Description                                    |
|---------|---------|------------------------------------------------|
| `State` | `state` | The state of the chain.                        |
| `[3]`   | `error` | Error details, if chain validation has failed. |

`enum GetCertState.State`
| Value  | Name                 | Description                                 |
|--------|----------------------|---------------------------------------------|
| `0x00` | `provisioned`        | A valid chain has been provisioned.         |
| `0x01` | `not_provisioned`    | A valid chain has not yet been provisioned. |
| `0x02` | `validation_pending` | The stored chain is being validated.        |


### GET DIGESTS

This command is derived from the similar USB Type C-Authentication Message.  The
Protocol Version byte is not included, the Message Type is present in the
Command byte of the MCTP message.  See: Table 4 MCTP Message Format.  The
response is different, since certificate handling deviates from USB Type C, and
the reserved bytes are repurposed.

This command can be sent at any time.  If authentication was previously
established, this command will renegotiate/override the previous authentication
and session establishment.  The byte 2, reserved USB Type C – authentication
specification, is used to describe the key exchange algorithm.  This is relevant
when the requester and responder support multiple key exchange algorithms.

Slots that do not contain a valid certificate chain will generate a response
with 0 digests.  Payload byte 2 will indicate that no digests are returned.


`message GetDigests.Requests`
| Type              | Name                | Description                        |
|-------------------|---------------------|------------------------------------|
| `b8`              | `slot`              | Slot Number (0 to 7) of the target Certificate Chain to read. |
| `KeyExchangeAlgo` | `key_exchange_algo` | Key Exchange Algorithm.            |


`message GetDigests.Response`
| Type       | Name           | Description                                    |
|------------|----------------|------------------------------------------------|
| `0x01`     | `capabilities` | Capabilities field.                            |
| `[32][b8]` | `digests`      | SHA256 digests of the certificates in the chain, starting from the root. |

`enum GetDigests.KeyExchangeAlgo`
| Value  | Name   | Description                                         |
|--------|--------|-----------------------------------------------------|
| `0x00` | `none` | No key exchange is desired.                         |
| `0x01` | `ecdh` | Elliptic-curve Diffe-Hellman exchange is requested. |


### GET CERTIFICATE

This command retrieves the public attestation certificate chain for the AC-RoT.
It follows closely the USB Type C Authentication Specification.

If the device does not have a certificate for the requested slot or index, the
certificate contents in the response will be empty.

`message GetCert.Request`
| Type  | Name         | Description                                           |
|-------|--------------|-------------------------------------------------------|
| `b8`  | `slot`       | Slot Number (0 to 7) of the target Certificate Chain to read. |
| `b8`  | `cert_index` | Certificate to read, zero-indexed from the root Certificate. |
| `b16` | `offset`     | Offset: offset in bytes from start of the Certificate to read. |
| `b16` | `len`        | Length: number of bytes to read.                      |

`message GetCert.Response`
| Type  | Name         | Description                                           |
|-------|--------------|-------------------------------------------------------|
| `b8`  | `slot`       | Slot Number of the target Certificate Chain returned. |
| `b8`  | `cert_index` | Certificate index of the returned certificate         |
| `...` | `cert_data`  | Requested contents of target Certificate Chain.       |


### CHALLENGE

The PA-RoT will send this command providing the first nonce in the key exchange.

`message Challenge.Request`
| Type   | Name    | Description                                               |
|--------|---------|-----------------------------------------------------------|
| `b8`   | `slot`  | Slot Number (0 to 7) of the target Certificate Chain to read. |
| `0x00` | `_`     | Reserved.                                                 |
| `[32]` | `nonce` | Random nonce.                                             |

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

The firmware digests are measurements of the security descriptors and the
firmware of the target components.  This firmware measurement data does not
include Cerberus PCD, CFM or PFM.  These measurements are returned in PMR1.  The
numbers are retrieved in the 6.44 Get Configuration Ids.  The USB-C Context Hash
is not included in the CHALLENGE.  This is replaced with contextual measurements
for the device.  Note: The attestation Certificate derivation will include the
measurement of firmware and security descriptors.  PMR0 is anticipated to be
the least changing PMR as it contains the measurement of security descriptors
and device initial boot loader.


### Key Exchange

Key Exchange is used to establish an encrypted channel with a device.  The
session establishment flow is detailed in Section 5 Authentication.

Upon receiving this message with Key Type 0, both sides can establish an
encrypted session.  If Key Type 1, the paired key is also compared, and paired
functionality is unlocked if verified to match the expected key.  The Key Type
1 can only be performed under an encrypted session, as the session key is
required for the HMAC.  When initially calculating a pairing key (K<sub>P</sub>)
for device binding, the HMAC specified in the Key Type 0 request for the session
will be used with the KDF.

When closing an established session (Key Type 2), the response will be
transmitted in plain text if the session was successfully terminated.

The HMAC type specified in this message applies to all HMAC operations for the
established session, including any subsequent pairing messages.  Since session
keys (`K_S` and `K_M`) are 256-bit keys, they will always be generated using
SHA-256 regardless of the type of HMAC used for key exchange messages.

`message KeyExchange.Request`
| Type             | Name       | Description |
|------------------|------------|-------------|
| `KeyType`        | `key_type` | Key type.   |
| `Data(key_type)` | `key_data` | Key data.   |

`enum KeyExchange.Request.Data(KeyExchange.KeyType)`
| Type                             | Name          |
|----------------------------------|---------------|
| `KeyExchange.SessionKey.Request` | `session_key` |
| `KeyExchange.PairingKey.Request` | `pairing_key` |
| `KeyExchange.Destroy.Request`    | `destroy`     |

`message KeyExchange.SessionKey.Request`
| Type       | Name          | Description                        |
|------------|---------------|------------------------------------|
| `HashType` | `hmac_digest` | HMAC hash algorithm.               |
| `...`      | `pk_req`      | ASN.1 DER-encoded ECDH public key. |

`message KeyExchange.PairingKey.Request`
| Type  | Name               | Description                                |
|-------|--------------------|--------------------------------------------|
| `b16` | `key_len`          | Length in bytes of the pairing key.        |
| `...` | `pairing_key_hmac` | HMAC of the pairing key: `HMAC(K_M, K_P)`. |

`message KeyExchange.Destroy.Request`
| Type  | Name           | Description                                |
|-------|----------------|--------------------------------------------|
| `...` | `session_hmac` | HMAC of the session key: `HMAC(K_M, K_S)`. |

`message KeyExchange.Response`
| Type             | Name       | Description |
|------------------|------------|-------------|
| `KeyType`        | `key_type` | Key type.   |
| `Data(key_type)` | `key_data` | Key data.   |

`enum KeyExchange.Response.Data(KeyExchange.KeyType)`
| Type                              | Name          |
|-----------------------------------|---------------|
| `KeyExchange.SessionKey.Response` | `session_key` |
| `b0`                              | `pairing_key` |
| `b0`                              | `destroy`     |

`message KeyExchange.SessionKey.Response`
| Type    | Name         | Description                                         |
|---------|--------------|-----------------------------------------------------|
| `0x00`  | `_`          | Reserved.                                           |
| `[b16]` | `pk_resp`    | ASN.1 DER-encoded ECDH public key.                  |
| `[b16]` | `signature`  | Signature over the session keys: `SIGN(pk_req || pk_resp)`. |
| `[b16]` | `alias_hmac` | HMAC of the Alias certificate: `HMAC(K_M, alias_cert)`. |

`enum KeyExchange.KeyType`
| Value  | Name          | Description                                  |
|--------|---------------|----------------------------------------------|
| `0x00` | `session_key` | Establishing an initial session key.         |
| `0x01` | `pairing_key` | Requesting a switch to a paired-key session. |
| `0x02` | `destroy`     | Ends a session.                              |

`enum KeyExcahange.HashType`
| Value  | Name     | Description |
|--------|----------|-------------|
| `0x00` | `sha256` | SHA-256.    |
| `0x01` | `sha384` | SHA-384.    |
| `0x02` | `sha512` | SHA-512.    |


### Session Sync

Check the state of an encrypted session. The message must always be encrypted.

`message SessionSync.Request`
| Type  | Name    | Description    |
|-------|---------|----------------|
| `[4]` | `nonce` | Random number. |

`message SessionSync.Response`
| Type  | Name   | Description                           |
|-------|--------|---------------------------------------|
| `...` | `hmac` | HMAC of the nonce: `HMAC(K_M, nonce)` |


### Get Log Info

Get the internal log information for the RoT.

`message GetLogInfo.Request`
| Type | Name | Description |
|------|------|-------------|

`message GetLogInfo.Response`
| Type  | Name                | Description                             |
|-------|---------------------|-----------------------------------------|
| `b32` | `debug_bytes`       | Debug Log (0x01) length in bytes.       |
| `b32` | `attestation_bytes` | Attestation Log (0x02) length in bytes. |
| `b32` | `tamper_bytes`      | Tamper Log (0x03) length in bytes.      |


### Get Log

Get the internal logs for the RoT.  There are 3 types of logs available:  The
Debug Log, which contains Cerberus application information and machine state.
The Attestation measurement log, this log format is like the TCG log, and the
Tamper log.  It is not possible to clear or reset the tamper counter.

`message GetLog.Request`
| Type      | Name       | Description                        |
|-----------|------------|------------------------------------|
| `LogType` | `log_type` | Log type.                          |
| `b32`     | `offset`   | Offset in bytes from start of log. |

`message GetLog.Response`
| Type  | Name       | Description          |
|-------|------------|----------------------|
| `...` | `contents` | Contents of the log. |

`enum GetLog.LogType`
| Value  | Name              | Description      |
|--------|-------------------|------------------|
| `0x01` | `debug_log`       | Debug Log.       |
| `0x02` | `attestation_log` | Attestation Log. |
| `0x03` | `tamper_log`      | Tamper Log.      |

Length determined by end of log, or packet size based on device capabilities see
section: 6.7 Device Capabilities.  If response spans multiple MCTP messages, end
of response will be determined by an MCTP message which has a payload less than
maximum payload supported by both devices.  To guarantee a response will never
fall exactly on the max payload boundary, the responder must send back an extra
packet with zero payload.

#### Attestation Log Format

The attestation log will report individual measurements for all components of
each PMR.  Each measurement will be a single entry in the log.  The entire log
consists of each entry sequentially concatenated.  The structure of the entry
closely resembles TCG events.  See TCG PC Client Platform Firmware Profile
Specification.

Log Entry Header:

`message GetLog.Header`
| Type  | Name            | Description                                       |
|-------|-----------------|---------------------------------------------------|
| `0xc` | `start_marker`  | Log entry start marker.                           |
| `0xb` | `header_format` | Header format for this specification.             |
| `b16` | `len`           | Total length of the entry, including this header. |
| `b32` | `id`            | Unique entry identifier.                          |

`message GetLog.AttestationEntry`
| Type     | Name               | Description                                  |
|----------|--------------------|----------------------------------------------|
| `Header` | `log_entry_header` | Log entry header.                            |
| `b32`    | `tcg_event`        | TCG event type.                              |
| `b8`     | `measurement_idx`  | Measurement index within a single PMR.       |
| `b8`     | `pmr_index`        | Index of the PMR for the measurement.        |
| `0x0000` | `_`                | Reserved.                                    |
| `b8`     | `digest_num`       | Number of digests.                           |
| `0x0000` | `_`                | Reserved.                                    |
| `0x000b` | `digest_type`      | Digest algorithm ID. (Fixed to SHA-256).     |
| `[32]`   | `digest`           | SHA-256 digest used to extend the measurement. |
| `b32`    | `measurement_len`  | Measurement length.                          |
| `[32]`   | `measurement`      | Measurement.                                 |

#### Debug Log Format

The debug log reported by the device has no specified format, as this can vary
between different devices and it is not necessary for attestation.  It is
expected that diagnostic utilities for the device will be able to understand the
exposed log information.  A recommended entry format is provided here.

Suggested Debug Entry format:

`message GetLog.DebugEntry`
| Type     | Name               | Description                         |
|----------|--------------------|-------------------------------------|
| `Header` | `log_entry_header` | Log entry header.                   |
| `b16`    | `format_version`   | Format version for the debug entry. |
| `b8`     | `severity`         | Severity of the entry.              |
| `b8`     | `source`           | ID of the source of the message.    |
| `b8`     | `entry_type`       | ID of the entry type.               |
| `[4]`    | `param1`           | Message-specific argument.          |
| `[4]`    | `param2`           | Message-specific argument.          |


### Clear Debug/Attestation Log

Clears the log in the RoT. Note that the tamper log cannot be cleared.

`message ClearLog.Request`
| Type             | Name       | Description                                  |
|------------------|------------|----------------------------------------------|
| `GetLog.LogType` | `log_type` | Log type to clear; cannot be the tamper log. |

Note: in clearing the attestation log, it is automatically recreated using
current measurements.


### Get Attestation Data

Get the raw data used to generate measurements reported in the attestation log.
Not all measurements will have raw data available, as this is mostly intended
for measurements generated over small amounts of data or text.  Requests for
entries that don’t provide the measured data will generate a normal response
with an empty payload.  Requests for invalid entries will generate an error
response.

While this command is intended to support small pieces of that data that should
fit into a single MCTP message, an offset parameter is included in the request
to support scenarios where the required data is too large.


`message GetAttestationData.Request`
| Type  | Name          | Description                          |
|-------|---------------|--------------------------------------|
| `b8`  | `pmr_index`   | Platform Measurement Register index. |
| `b8`  | `entry_index` | Entry index within the PMR.          |
| `b32` | `offset`      | Byte offset within the entry.        |

`message GetAttestationData.Response`
| Type  | Name          | Description        |
|-------|---------------|--------------------|
| `...` | `measurement` | The measured data. |


### Get Host State

Retrieve the reset state of the host processor being protected by Cerberus.

`message GetHostState.Request`
| Type | Name      | Description       |
|------|-----------|-------------------|
| `b8` | `port_id` | Port ID to query. |

`message GetHostState.Response`
| Type         | Name          | Description       |
|--------------|---------------|-------------------|
| `ResetState` | `reset_state` | Host reset state. |

`enum GetHostState.ResetState`
| Value  | Name      | Description                                |
|--------|-----------|--------------------------------------------|
| `0x00` | `running` | Host is running.                           |
| `0x01` | `reset`   | Host is in reset.                          |
| `0x02` | `unknown` | Host is neither running nor held in reset. |


### Get PFM Id

Retrieves PFM identifiers.

`message PFM.GetId.Request`
| Type                  | Name         | Description             |
|-----------------------|--------------|-------------------------|
| `b8`                  | `port_id`    | Port the PFM describes. |
| `Manifest.Region`     | `pfm_region` | PFM region to query.    |
| `Manifest.Identifier` | `id`         | Identifier returned.    |
<!-- There isn't a way to specify a default value for `id`. -->

`message PFM.GetId.Response`
| Type  | Name    | Description                        |
|-------|---------|------------------------------------|
| `b8`  | `valid` | Whether the PFM is valid (0 or 1). |
| `...` | `id`    | The requested ID value.            |

`enum Manifest.Region`
| Value  | Name      | Description                      |
|--------|-----------|----------------------------------|
| `0x00` | `active`  | The active PFM.                  |
| `0x01` | `pending` | The pending PFM if there is one. |

`enum Manifest.Identifier`
| Value  | Name          | Description                        |
|--------|---------------|------------------------------------|
| `0x00` | `version_id`  | The version in the PFM.            |
| `0x01` | `platform_id` | The Platform ID string in the PFM. |


### Get PFM Supported Firmware

`message PFM.GetSupportedFW.Request`
| Type              | Name         | Description                |
|-------------------|--------------|----------------------------|
| `b8`              | `port_id`    | Port ID the PFM describes. |
| `Manifest.Region` | `pfm_region` | PFM region to query.       |
| `u32`             | `offset`     | Offset in bytes.           |

`message PFM.GetSupportedFW.Response`
| Type  | Name       | Description                        |
|-------|------------|------------------------------------|
| `b8`  | `valid`    | Whether the PFM is valid (0 or 1). |
| `[4]` | `id`       | The PFM's version.                 |
| `...` | `versions` | The supported FW versions.         |

If response spans multiple MCTP messages, end of response will be determined by
an MCTP packet which has payload less than maximum payload supported by both
devices.  To guarantee a response will never fall exactly on the max payload
boundary, the responder should send back an extra packet with zero payload.


### Prepare PFM

Provisions RoT for incoming PFM.

`message PFM.Prepare.Request`
| Type  | Name         | Description                        |
|-------|--------------|------------------------------------|
| `b8`  | `port_id`    | Port ID the PFM describes.         |
| `u32` | `total_size` | Total size of the update in bytes. |


#### Update PFM

The flash descriptor structure describes the regions of flash for the device.

`message PFM.Update.Request`
| Type  | Name      | Description                |
|-------|-----------|----------------------------|
| `b8`  | `port_id` | Port ID the PFM describes. |
| `...` | `payload` | The next chunk of data.    |

The PFM payload includes a signature and monotonic forward only Id.  The PFM
signature is verified upon receipt of all PFM payloads.  PFMs are activated upon
the activation command.  Note if a system is rebooted after receiving a PFM, the
PFM is atomically activated.  To activate before reboot, issue the Activate PFM
command.

## Activate PFM

Upon valid PFM update, the update command seals the PFM committal method.  If
committing immediately, flash reads and writes should be suspended when this
command is issued.  The RoT will master the SPI bus and verify the newly updated
PFM.  This command can only follow a valid PFM update.

`message PFM.Activate.Request`
| Type                  | Name         | Description                           |
|-----------------------|--------------|---------------------------------------|
| `b8`                  | `port_id`    | Port ID the PFM describes.            |
| `Manifest.Activation` | `activation` | Activation strategy for the manifest. |

`enum Manifest.Activation`
| Value  | Name        | Description           |
|--------|-------------|-----------------------|
| `0x00` | `reboot`    | Activate on reboot.   |
| `0x01` | `immediate` | Activate immediately. |

If reboot only has been issued, the option for "Immediately" committing the PFM
is not available until a new PFM is updated.


### Get CFM Id

Retrieves the Component Firmware Manifest Id

`message CFM.GetId.Request`
| Type                  | Name         | Description          |
|-----------------------|--------------|----------------------|
| `Manifest.Region`     | `pfm_region` | CFM region to query. |
| `Manifest.Identifier` | `id`         | Identifier returned. |

`message CFM.GetId.Response`
| Type  | Name    | Description                        |
|-------|---------|------------------------------------|
| `b8`  | `valid` | Whether the CFM is valid (0 or 1). |
| `...` | `id`    | The requested ID value.            |


### Prepare CFM

Provisions RoT for incoming Component Firmware Manifest.

`message CFM.Prepare.Request`
| Type  | Name         | Description                        |
|-------|--------------|------------------------------------|
| `u32` | `total_size` | Total size of the update in bytes. |


### Update CFM

The flash descriptor structure describes the regions of flash for the device.

`message CFM.Update.Request`
| Type  | Name      | Description             |
|-------|-----------|-------------------------|
| `...` | `payload` | The next chunk of data. |

The CFM payload includes CFM signature and monotonic forward only Id.  CFM
signature is verified upon receipt of all CFM payloads.  CFMs are activated
upon the activation command.  Note if a system is rebooted after receiving a
CFM, the pending CFM is verified and atomically activated.  To activate before
reboot, issue the Activate CFM command.


### Activate CFM

Upon valid CFM update, the update command seals the CFM committal method.  The
RoT will master I2C and attest Components in the Platform Configuration Data
against the CFM.

`message CFM.Activate.Request`
| Type                  | Name         | Description                           |
|-----------------------|--------------|---------------------------------------|
| `Manifest.Activation` | `activation` | Activation strategy for the manifest. |


### Get CFM Component IDs

`message CFM.GetSupportedFW.Request`
| Type              | Name         | Description          |
|-------------------|--------------|----------------------|
| `Manifest.Region` | `pfm_region` | CFM region to query. |
| `u32`             | `offset`     | Offset in bytes.     |

`message CFM.GetSupportedFW.Response`
| Type  | Name       | Description                        |
|-------|------------|------------------------------------|
| `b8`  | `valid`    | Whether the CFM is valid (0 or 1). |
| `[4]` | `id`       | The CFM's version.                 |
| `...` | `versions` | The supported component IDs.       |

If response spans multiple MCTP messages, end of response will be determined by
an MCTP packet which has payload less than maximum payload supported by both
devices.  To guarantee a response will never fall exactly on the max payload
boundary, the responder should send back an extra packet with zero payload.


### Get PCD Id

Retrieves the PCD Id.

`message PCD.GetId.Request`
| Type                  | Name | Description          |
|-----------------------|------|----------------------|
| `Manifest.Identifier` | `id` | Identifier returned. |

`message PCD.GetId.Response`
| Type  | Name    | Description                        |
|-------|---------|------------------------------------|
| `b8`  | `valid` | Whether the PCD is valid (0 or 1). |
| `...` | `id`    | The requested ID value.            |


### Prepare PCD

Provisions RoT for incoming Platform Configuration Data.

`message PCD.Prepare.Request`
| Type  | Name         | Description                        |
|-------|--------------|------------------------------------|
| `u32` | `total_size` | Total size of the update in bytes. |

### Update PCD

The flash descriptor structure describes the regions of flash for the device.

`message PCD.Update.Request`
| Type  | Name      | Description             |
|-------|-----------|-------------------------|
| `...` | `payload` | The next chunk of data. |


The PCD payload includes PCD signature and monotonic forward only Id.  PCD
signature is verified upon receipt of all PCD payloads.  PCD is activated upon
the activation command.  Note if a system is rebooted after receiving a PCD.


### Activate PCD

Upon valid PCD update, the activate command seals the PCD committal.

`message PCD.Activate.Request`
| Type | Name | Description |
|------|------|-------------|


### Platform Configuration

The following table describes the Platform Configuration Data Structure.

`message PCD.Structure`
| Type         | Name         | Description                             |
|--------------|--------------|-----------------------------------------|
| `[4]`        | `version_id` | The PCD version ID.                     |
| `u16`        | `len`        | The length of the PCD.                  |
| `Policy[b8]` | `policies`   | The policies in the PCD.                |
| `...`        | `signature`  | Signature over the contents of the PCD. |

`message PCD.Policy`
| Type            | Name                | Description |
|-----------------|---------------------|-------------|
| `b32`           | `device_id`         |             |
| `b8`            | `channel`           |             |
| `b8`            | `address`           |             |
| `b1`            | `threshold_active`  |             |
| `b1`            | `policy_active`     |             |
| `b1`            | `auto_recovery`     |             |
| `b1`            | `debug_enabled`     |             |
| `b1`            | `power_control`     |             |
| `0b000`         | `_`                 |             |
| `b8`            | `power_control_idx` |             |
| `FailureAction` | `failure_action`    |             |

`enum PCD.FailureAction`
| Value  | Name                 | Description |
|--------|----------------------|-------------|
| `0x00` | `platform_defined`   |             |
| `0x01` | `report_only`        |             |
| `0x02` | `automatic_recovery` |             |
| `0x03` | `power_control`      |             |

The Power Control Index informs the PA-RoT of the index assigned to power
sequence the Component.  This informs the PA-RoT which control register needs
to be asserted in the platform power sequencer.


### Prepare Firmware Update

Provisions RoT for incoming firmware update.

`message Firmware.Prepare.Request`
| Type  | Name         | Description                        |
|-------|--------------|------------------------------------|
| `u32` | `total_size` | Total size of the update in bytes. |


### Update Firmware

The flash descriptor structure describes the regions of flash for the device.

`message Firmware.Update.Request`
| Type  | Name      | Description             |
|-------|-----------|-------------------------|
| `...` | `payload` | The next chunk of data. |


### Update Status

The Update Status reports the update payload status.  The update status will be
status for the last operation that was requested.  This status will remain the
same until another operation is performed or Cerberus is reset.

`message UpdateStatus.Request`
| Type         | Name          | Description                                   |
|--------------|---------------|-----------------------------------------------|
| `UpdateType` | `update_type` | The update type to query.                     |
| `b8`         | `port_id`     | The port ID associated with this update, if any. |

`message UpdateStatus.Response`
| Type  | Name     | Description                                               |
|-------|----------|-----------------------------------------------------------|
| `[4]` | `status` | The status of the update; see the Firmware Update specification for details. |

`enum UpdateStatus.UpdateType`
| Value  | Name                | Description                   |
|--------|---------------------|-------------------------------|
| `0x00` | `firmware`          | An RoT firmware update.       |
| `0x01` | `pfm`               | A PFM update.                 |
| `0x02` | `cfm`               | A CFM update.                 |
| `0x03` | `pcd`               | A PCD update.                 |
| `0x04` | `host_firmware`     | The host's firmware update.   |
| `0x05` | `recovery_firmware` | RoT recovery image update.    |
| `0x06` | `reset_config`      | A reset configuration update. |


### Extended Update Status

The Extended Update Status reports the update payload status along with the
remaining number of update bytes expected.  The update status will be status for
the last operation that was requested.  This status will remain the same until
another operation is performed or Cerberus is reset.

`message ExtendedUpdateStatus.Request`
| Type                      | Name          | Description                      |
|---------------------------|---------------|----------------------------------|
| `UpdateStatus.UpdateType` | `update_type` | The update type to query.        |
| `b8`                      | `port_id`     | The port ID associated with this update, if any. |

`message ExtendedUpdateStatus.Response`
| Type  | Name              | Description                                      |
|-------|-------------------|--------------------------------------------------|
| `[4]` | `status`          | The status of the update; see the Firmware Update specification for details. |
| `u32` | `bytes_remaining` | Expected update bytes remaining.                 |


### Activate Firmware Update

Alerts Cerberus that sending of update bytes is complete, and that verification
of update should start.  This command has no payload, the ERROR response zero is
expected.

`message Firmware.Activate.Request`
| Type | Name | Description |
|------|------|-------------|


### Reset Configuration

Resets configuration parameters back to the default state.  Depending on the
request parameters, different amounts of types of configuration can be erased,
and each type of configuration may require different levels of authorization to
complete.

If authorization is required for the operation to complete, the response will
contain a device-specific, one-time use, authorization token that must be signed
with the PFM key to unlock the operation.  The authorization token has the
following behavior:

1.  A request for the same operation without providing the signed authorization
    token will generate a new token that invalidates any old token.  This is
    true even if the old token has not been used yet.

2.  After an authorization token has been used to unlock an operation, it can
    never be used again.  A new token must be requested.  This is true even if
    the requested operation was not able to complete successfully.

3.  A failure to authorize the request when providing a signed token does not
    invalidate the current authorization token in the device.

If authorization is not required, or the request is sent with a signed token, a
standard error response will be returned indicating the status.

`message ResetConfig.Request`
| Type      | Name          | Description                                      |
|-----------|---------------|--------------------------------------------------|
| `ResetOp` | `operation`   | Type of reset operation to request.              |
| `...`     | `authz_token` | Device-specific authorization token, signed with a PFM key. |

`enum ResetConfig.ResetOp`
| Value  | Name              | Description                                     |
|--------|-------------------|-------------------------------------------------|
| `0x00` | `erase_manifests` | Revert the device to unprotected bypass state by erasing all PFMs and CFMs. |
| `0x01` | `factory_reset`   | Erase all configuration data except for imported certificates. |

`message ResetConfig.Response`
| Type  | Name          | Description                          |
|-------|---------------|--------------------------------------|
| `...` | `authz_token` | Device-specific authorization token. |


### Get Configuration Ids

This command retrieves PFM Ids, CFM Ids, PCD Id, and signed digest of request
nonce and response ids.

<!-- Currently cannot express `platform_id`s; it's not quite specified how to
  separate it into separate IDs; two `...`s in a row is actually forbidden. -->
`message GetConfigIDs.Request`
| Type             | Name           | Description                  |
|------------------|----------------|------------------------------|
| `[32]`           | `nonce`        | Random nonce.                |
| `b8`             | `pfm_count`    | Number of PFM IDs.           |
| `b8`             | `cfm_count`    | Number of PFM IDs.           |
| `[4][pfm_count]` | `pfm_ids`      | The PFM Version IDs.         |
| `[4][cfm_count]` | `cfm_ids`      | The CFM Version IDs.         |
| `[4]`            | `pcd_id`       | The PCD Version ID.          |
| `...`            | `platform_ids` | The Platform IDs.            |
| `...`            | `signature`    | `SIGN(request || response)`. |


### Recover Firmware

Start the firmware recovery process for the device.  Not all devices will
support all types of recovery.  The implementation is device specific.

`message RecoverFirmware.Request`
| Type | Name             | Description                                 |
|------|------------------|---------------------------------------------|
| `b8` | `port_id`        | The port to begin recovery for.             |
| `b8` | `enter_recovery` | Whether to enter recovery mode, or exit it. |

### Prepare Recovery Image

Provisions RoT for incoming Recovery Image for Port.

`message RecoveryImage.Prepare.Request`
| Type  | Name         | Description                        |
|-------|--------------|------------------------------------|
| `b8`  | `port_id`    | Port ID of the device.             |
| `u32` | `total_size` | Total size of the update in bytes. |


### Update Recovery Image

The flash descriptor structure describes the regions of flash for the device.

`message RecoveryImage.Update.Request`
| Type  | Name      | Description             |
|-------|-----------|-------------------------|
| `b8`  | `port_id` | Port ID of the device.  |
| `...` | `payload` | The next chunk of data. |


### Activate Recovery Image

Signals recovery image has been completely sent and verification of the image
should start.  Once the image has been verified, it can be used for host
firmware recovery.

`message RecoveryImage.Activate.Request`
| Type | Name      | Description            |
|------|-----------|------------------------|
| `b8` | `port_id` | Port ID of the device. |


### Get Recovery Image Id

Retrieves the recovery image identifiers.

`message RecoveryImage.GetId.Request`
| Type                  | Name      | Description            |
|-----------------------|-----------|------------------------|
| `b8`                  | `port_id` | Port ID of the device. |
| `Manifest.Identifier` | `id`      | Identifier returned.   |
<!-- There isn't a way to specify a default value for `id`. -->

`message RecoveryImage.GetId.Response`
| Type  | Name | Description             |
|-------|------|-------------------------|
| `...` | `id` | The requested ID value. |


### Get Platform Measurement Register

Returns the Cerberus Platform Measurement Register (PMR), which is a digest of
the Cerberus Firmware, PFM, CFM and PCD.  This information contained in PMR0 is
Cerberus firmware.  PMR1-2 are reserved for PFM/CFM/PCD and other configuration.
PMR3-4 are reserved for external usages.

Attestation requires that at least PMR0 be maintained, since that is the
measurement reported in the challenge response message.  At a minimum, PMR0
should contain any security configuration of the device and all firmware
components.  This includes any firmware actively running as well as firmware
that was run to boot the device.  For ease of attestation, PMR0 is intended to
only contain static information.  If variable data will also be measured, these
measurements should be exposed through PMR1-2.

`message GetPMR.Request`
| Type   | Name        | Description                        |
|--------|-------------|------------------------------------|
| `b8`   | `pmr_index` | The index of the PMR to read from. |
| `[32]` | `nonce`     | A random nonce.                    |

`message GetPMR.Response`
| Type   | Name    | Description     |
|--------|---------|-----------------|
| `[32]` | `nonce` | A random nonce. |
[`[b8]`| `pmr_value` | The contents of the PMR. |
|`...` | `signature` | `SIGN(request || response)`. |

PMR1-4 are cleared on component reset.  PMR0 is cleared and re-built on Cerberus
reset.


### Update Platform Measurement Register

External updates to PMR3-4 are permitted.  Attempts to update PMR0-2 will result
error.  Only SHA2 is supported for measurement extension.  SHA1 and SHA3 are
not applicable.  Note:  The measurement can only be updated over an
authenticated and secured channel.

`message UpdatePMR.Request`
| Type  | Name        | Description                     |
|-------|-------------|---------------------------------|
| `b8`  | `pmr_index` | The index of the PMR to extend. |
| `...` | `data`      | The data to extend by.          |


### Reset Counter

Provides Cerberus and Component Reset Counter since power-on.

`message ResetCounter.Request`
| Type      | Name      | Description                |
|-----------|-----------|----------------------------|
| `Counter` | `counter` | The counter type.          |
| `b8`      | `port_id` | The port ID of the device. |

`message ResetCounter.Response`
| Type  | Name          | Description                    |
|-------|---------------|--------------------------------|
| `b16` | `reset_count` | The number of resets recorded. |

`enum ResetCounter.Counter`
| Value  | Name              | Description                                     |
|--------|-------------------|-------------------------------------------------|
| `0x00` | `local_device`    | A local device.                                 |
| `0x01` | `external_device` | A protected external device, not including external AC-RoTs challenged by the device. |
<!-- NOTE: we cannot specify an open enum yet. -->


### Message Unseal

This command starts unsealing an attestation message.  The ciphertext is limited
to what can fit in a single message along with the other pieces necessary for
unsealing.

`message Unseal.Request`
| Type                    | Name          | Description                        |
|-------------------------|---------------|------------------------------------|
| `SeedType`              | `seed_type`   | The seed for the decryption key.   |
| `HashType`              | `hash_type`   | The hash to use for the HMAC.      |
| `b3`                    | `_`           | Reserved.                          |
| `SeedParams(seed_type)` | `seed_params` | Additional parameters for the seed.|
| `[b16]`                 | `seed`        | Seed data.                         |
| `[b16]`                 | `ciphertext`  | Encrypted, sealed messaged.        |
| `[64]`                  | `pmr0_policy` | Sealing value for PMR0. Unused bytes should be zeroed. |
| `[64]`                  | `pmr1_policy` | Sealing value for PMR1. Unused bytes should be zeroed. |
| `[64]`                  | `pmr2_policy` | Sealing value for PMR2. Unused bytes should be zeroed. |
| `[64]`                  | `pmr3_policy` | Sealing value for PMR3. Unused bytes should be zeroed. |
| `[64]`                  | `pmr4_policy` | Sealing value for PMR4. Unused bytes should be zeroed. |

`enum Unseal.SeedType`
| Value  | Name   | Description                                    |
|--------|--------|------------------------------------------------|
| `0b00` | `rsa`  | Seed is encrypted with an RSA public key.      |
| `0b01` | `ecdh` | Seed is an ECDH public key, ASN.1 DER-encoded. |

`enum Unseal.SeedData(Unseal.SeedKind)`
| Type              | Name   |
|-------------------|--------|
| `SeedParams.RSA`  | `rsa`  |
| `SeedParams.ECDH` | `ecdh` |

`message Unseal.SeedParams.RSA`
| Type            | Name             | Description                    |
|-----------------|------------------|--------------------------------|
| `PaddingScheme` | `padding_scheme` | The RSA padding scheme to use. |
| `b7`            | `_`              | Reserved.                      |

`enum Unseal.SeedParams.PaddingScheme`
| Value   | Name          | Description        |
|---------|---------------|--------------------|
| `0b000` | `pkcs1`       | PKCS#1 v1.5        |
| `0b001` | `oeap_sha1`   | OEAP with SHA-1.   |
| `0b010` | `oeap_sha256` | OEAP with SHA-256. |

`message Unseal.SeedParams.ECDH`
| Type | Name        | Description                                             |
|------|-------------|---------------------------------------------------------|
| `b1` | `hash_seed` | Whether the actual seed is a SHA-256 hash of the raw seed value. |
| `b7` | `_`         | Reserved.                                               |


### Message Unseal Result

This command retrieves the current status of an unsealing process.

`message UnsealResult.Request`
| Type | Name | Description |
|------|------|-------------|

`message UnsealResult.Response`
| Type    | Name               | Description                  |
|---------|--------------------|------------------------------|
| `b32`   | `unsealing_status` | Unsealing status.            |
| `[b16]` | `unsealed_key`     | The unsealed encryption key. |


# Platform Active RoT (PA-RoT)

The PA-RoT is responsible for challenging the AC-RoT’s and collecting their
firmware measurements.  The PA-RoT retains a private manifest of active
components that includes addresses, buses, firmware versions, digests and
firmware topologies.

The manifest informs the PA-RoT on all the Active Components in the system.  It
provides their I2C addresses, and information on how to verify their
measurements against a known or expected state.  Polices configured in the
Platform RoT determine what action it should take should the measurements fail
verification.

In the Cerberus designed motherboard, the PA-RoT orchestrates power-on.  Only
Active Components listed in the challenge manifest, that pass verification will
be released from power-on reset.

## Platform Firmware Manifest (PFM) and Component Firmware Manifest

The PA-RoT contains a Platform Firmware Manifest (PFM) that describes the
firmware permitted on the Platform.  The Component Firmware Manifest (CFM)
describes the firmware permitted for components in the Platform.  The Platform
Configuration Data (PCD), specific to each SKU describes the number of Component
types in the platform and their respective locations.

Note: The PFM and CFM are different from the boot key manifest described in the
Processor Secure Boot Requirements specification.  The PFM and CFM describe
firmware permitted to run in the system across Platform and Active Components.
The CFM is a complement to the PFM, generated by the PA-RoT for the measurement
comparison of components in the system.  This complement is as the Reported
Firmware Manifest (RFM), which is like the TCG log.  The PFM and RFM are stored
encrypted in the PA-RoT.  The symmetric encryption key for the PA-RoT is
hardware generated and unique to each microcontroller.  The symmetric key in the
PA-Rot is not exportable or firmware readable; and only accessible to the crypto
engine for encryption/decryption.  The AES Galois/Counter Mode (GCM) encryption
a unique auditable tag to any changes to the manifest at both an application
level and persistent storage level.

The following table lists the attributes stored in the PFM for each Active
component:

| Attribute             | Description                                 |
|-----------------------|---------------------------------------------|
| Description           | Device Part or Description                  |
| Device Type           | Underlying Device Type of AC-RoT            |
| Remediation Policy    | Remediation actions for integrity failure.  |
| Firmware Version      | List of firmware versions                   |
| Flash Areas/Offsets   | List of offset and digests, used and unused |
| Measurement           | Firmware Measurements                       |
| Measurement Algorithm | Algorithm used to calculate measurement.    |
| Public Key            | Public keys in the key manifest.            |
| Digest Algorithm      | Algorithm used to calculate.                |
| Signature             | Firmware signature(s)                       |

The PA-RoT actively takes measurements of flash from platform firmware, the PFM
provides metadata that instructs the RoT on measurement and signature
verification.  The PA-RoT stores the measurements in the RFM.  The PA-Rot then
challenges the AC-RoTs for their measurements using the Platform Configuration
Data.  it compares measurements from the AC-RoT’s to the CFM, while recording
measurements in the RFM.

The measurements of the Platform firmware and Component firmware are compared to
the PFM and CFM.  Should a mismatch occur, the PA-RoT would raise an event log
and invoke the policy action defined for the Platform and/or Component.  A
variety of actions can be automated for a PFM/CFM challenge failure.  Actions
are defined in the CFM and PCD files.

Note:  The PA-RoT and AC-RoT enforce secure boot and only permit the download of
digitally signed and unrevoked firmware.  A PFM or CFM mismatch can only occur
when firmware integrity is brought into question.


## RoT External Communication interface

The PA-RoT connects to the platform through, either SPI, QSPI depending on the
motherboard.  Although the PA-RoT physically connects to the SPI bus, the
microprocessor appears transparent to the host as it presents only a flash
interface.  The management interface into the PA-RoT and AC-RoTs is an  I2C bus
channeled through the Baseboard Management Controller (BMC).  The BMC can reach
all AC-RoTs in the platform.  The BMC bridges the PA-RoT to the Rack Manager,
which in-turn bridges the rack to the Datacenter management network.  The
interface into the PA-RoT is as follows:

> TODO: figure 12

The Datacenter Management (DCM) software can communicate with the PA-RoT
Out-Of-Band (OOB) through the Rack Manager.  The Rack Manager allows tunneling
through to the Baseboard Management Controller, which connects to the PA-RoT
over I2C.  This channel is assumed insecure, which is why all communicates are
authenticated and encrypted.  The Datacenter Management Software can collect
the RFM measurements and other challenge data over this secure channel.  Secure
updates are also possible over this channel.


## Host Interface

The host can communicate with the PA-RoT and AC-RoTs through the BMC host
interface.  Similar to the OOB path, the BMC bridges the host-side LPC/eSPI
interface to the I2C interface on the RoT.  The host through BMC is an unsecure
channel, and therefore requires authentication and confidentiality.

## Out Of Band (OOB) Interface

The OOB interface is essential for reporting potential firmware compromises
during power-on.  Should firmware corruption occur during power-on, the OOB
channel can communicate with the DCM software while the CPU is held in reset.
If the recovery policy determines the system should remain powered off, it’s
still possible for the DCM software to interrogate the PA-RoT for detailed
status and make a determination on the remediation.

The OOB communication to Cerberus requires TLS and Certificate Authentication.


# Legacy Interface

The legacy interface is defined for backward combability with devices that do
not support MCTP.  These devices must provide a register set with specific
offsets for Device Capabilities, Receiving Alias Certificate, accepting a Nonce,
and providing an offset for Signed Firmware Measurements.  The payload
structures will closely match that of the MCTP protocol version.  Legacy
interfaces to no support session based authentication but permit signed
measurements.


## Protocol Format

The legacy protocol leverages the SMBus Write/Read Word and Block commands.
The interface is register based using similar read and write subroutines of I2C
devices.  The data transmit and receive requirements are 32 bytes or greater.
Large payloads can be truncated and retrieved recursively spanning multiple
block read or write commands.

The block read SMBUS command is specified in the SMBUS specification.  Slave
address write and command code bytes are transmitted by the master, then a
repeated start and finally a slave address read.  The master keeps clocking as
the slaves responds with the selected data.  The command code byte can be
considered register space.


### PEC Handling

An SMBus legacy protocol implementation may leverage the 8bit SMBus Packet Error
Check (PEC) for transactional data integrity.  The PEC is calculated by both the
transmitter and receiver of each packet using the 8-bit cyclic redundancy check
(CRC-8) of both read or write bus transaction.  The PEC accumulates all bytes
sent or received after the start condition.

An Active RoT that receives an invalid PEC can optionally NACK the byte that
carried the incorrect PEC value or drop the data for the transaction and any
further transactions (read or write) until the next valid read or write Start
transaction is received.


### Message Splitting

The protocol supports Write Block and Read Block commands.  Standard SMBus
transactions are limited to 32 bytes of data.  It is expected that some Active
Component RoTs with intrinsic Cerberus capabilities may have limited I2C message
buffer designed around the SMBus protocol that limit them to 32 bytes.  To
overcome hardware limitations in message lengths, the Capabilities register
includes a buffer size for determining the maximum packet size for messages.
This allows the Platform’s Active RoT to send messages larger than 32 bytes.
If the Active Component RoT only permits 32 bytes of data, the Platform’s Active
RoT can segment the Read or Write Blocks into multiple packets totaling the
entire message.  Each segment includes decrementing packet number that
sequentially identifies the part of the overall message.  To stay within the
protocol length each message segment must be no longer than 255 bytes.


### Payload Format

The payload portions of the SMBus Write and Read blocks will encapsulate the
protocol defined in this specification.  The SMBus START and STOP framing and
ACK/NACK bit conditions are omitted from this portion of the specification for
simplification.  To review the specifics of START and STOP packet framing and
ACK/NACK conditions refer to the SMBus specification.

The data blocks of the Write and Read commands will encapsulate the message
payload.  The encapsulated payload includes a uint16 register offset and data
section.


### Register Format

The SMBUS command byte indexes the register, while additional writes offsets
index inside the register space.  The offset and respective response is
encapsulated into the data portions of I2C Write and Read Block commands.  The
PA-RoT is always the I2C master, therefore Write and Read commands are described
from the perspective of the I2C master.

Certain registers may contain partial or temporary data while the register is
being written across multiple commands.  The completion or sealing of register
writes can be performed by writing the seal register to the zero offset.

The following diagram depicts register read access flow for a large register
space:

> TODO: Figure 14

The following diagram depicts register write access flow for a large register
space, with required seal (update complete bit):

> TODO: Figure 15


### Legacy Active Component RoT Commands

The following table describes the commands accepted by the Active Component RoT.
All commands are master initiated.  The command number is not representative of
a contiguous memory space, but an index to the respective register

| Register Name                   | Command | Length | R/W | Description                                         |
|---------------------------------|---------|--------|-----|-----------------------------------------------------|
| Status                          | 0x30    | 2      | R   | Command Status                                      |
| Firmware Version                | 0x32    | 16     | R/W | Retrieve firmware version information               |
| Device Id                       | 0x33    | 8      | R   | Retrieves Device Id                                 |
| Capabilities                    | 0x34    | 9      | R   | Retrieves Device Capabilities                       |
| Certificate Digest              | 0x3c    | 32     | R   | SHA256 of Device Id Certificate                     |
| Certificate                     | 0x3d    | 4096   | R/W | Certificate from the AC-Rot                         |
| Challenge                       | 0x3e    | 32     | W   | Nonce written by RoT                                |
| Platform Configuration Register | 0x03    | 0x5e   | R   | Reads firmware measurement, calculated with S Nonce |


### Legacy Command Format

The following section describes the register format for AC-RoT that do not
implement SMBUS and comply with the legacy measurement exchange protocol.

#### Status

The SMBUS read command reads detailed information on error status.  The status
register is issued between writing the challenge nonce and reading the
Measurement.  The delay time for deriving the Measurement must comply with the
Capabilities command.

`message SMBUS.Status`
| Type   | Name         | Description          |
|--------|--------------|----------------------|
| `Code` | `code`       | The status code.     |
| `b8`   | `error_data` | Optional error data. |

`enum SMBUS.Status.Code`
| Value  | Name          | Description |
|--------|---------------|-------------|
| `0x00` | `complete`    |             |
| `0x01` | `in_progress` |             |
| `0x02` | `error`       |             |

<!-- NOTE: all of the table references below are broken and ened to be replaced
     with proper anchor links. -->

#### Firmware Version

The SMBUS write command payload sets the index.  The subsequent SMBUS read
command reads the response.  For register payload description see response:
Table 11 Firmware Version Response

#### Device Id

The SMBUS read command reads the response.  For register payload
description see response: Table 1 Field Definitions.

#### Device Capabilities

The SMBUS read command reads the response.  For register payload description see
response: Table 13 Device Capabilities Response 

#### Certificate Digest

The SMBUS read command reads the response.  For register payload description
see response: Table 24 `GET DIGEST` Response

The PA-Rot will use the digest to determine if it has the certificate already
cached.  Unlike MCTP, only the Alias and Device Id cert is supported.
Therefore, it must be CA signed by a mutually trusted CA, as the CA Public Cert
is not present

#### Certificate

The SMBUS write command writes the offset into the register space.  For register
payload description see response:  Table 26 `GET CERTIFICATE` Response

Unlike MCTP, only the Alias and Device Id certificates are supported. Therefore,
it must be CA signed by mutually trusted CA, as the CA Public Cert is not
present in the reduced challenge

The SMBUS write command writes a nonce for measurement freshness.

`message SMBUS.Certificate`
| Type   | Name    | Description   |
|--------|---------|---------------|
| `[32]` | `nonce` | Random nonce. |

#### Measurement

The SMBUS read command that reads the signed measurement with the nonce from the
hallenge above.  The PA-RoT must poll the Status register for completion after
issuing the Challenge and before reading the Measurement.

`message SMBUS.Measurement`
| Type   | Name        | Description                 |
|--------|-------------|-----------------------------|
| `[b8]` | `hash`      | `Hash(nonce || Hash(PMR0))` |
| `...`  | `signature` | Signature over `hash`.      |


# References
1.  DICE Architecture
    <https://trustedcomputinggroup.org/work-groups/dice-architectures>
2.  RIoT
    <https://www.microsoft.com/en-us/research/publication/riot-a-foundation-for-trust-in-the-internet-of-things>
3.  DICE and RIoT Keys and Certificates
    <https://www.microsoft.com/en-us/research/publication/device-identity-dice-riot-keys-certificates>
4.  USB Type C Authentication Specification
    <http://www.usb.org/developers/docs>
5.  PCIe Device Security Enhancements specification
    <https://www.intel.com/content/www/us/en/io/pci-express/pcie-device-security-enhancements-spec.html>
6.  NIST Special Publication 800-108 - Recommendation for Key Derivation Using Pseudorandom Functions.
    <http://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-108.pdf>
7.  TCG PC Client Platform Firmware Profile Specification
    <https://trustedcomputinggroup.org/resource/pc-client-specific-platform-firmware-profile-specification>
