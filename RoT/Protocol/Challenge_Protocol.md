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
following table shows an MCTP encapsulated message; note that offsets are given
in bits, rather than bytes.

| Payload     | Description                            |
|-------------|----------------------------------------|
| 7:0         | I2C destination address.               |
| 15:8        | Command code; must be `0x0f`.          |
| 23:16       | Number of bytes in the packet payload. |
| 31:24       | I2C source address.                    |
| 35:32       | Reserved (should be zero).             |
| 39:36       | MCTP Header version.                   |
| 47:40       | Destination EID.                       |
| 55:48       | Source EID.                            |
| 56:56       | Start of message (SOM) flag.           |
| 57:57       | End of message (EOM) flag.             |
| 59:58       | Sequence number.                       |
| 60:60       | Tag owner.                             |
| 63:61       | Message tag.                           |
| 64+N:64     | Packet payload.                        |
| 64+N+8:64+N | PEC                                    |

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
| Payload  | Description                              |
|----------|------------------------------------------|
| 0:0      | Integrity check flag.                    |
| 7:1      | MCTP message type (always `0x7e`).       |
| 23:8     | MCTP PCI Vendor ID (always `0x1414`).    |
| 24:24    | Request type.                            |
| 25:25    | Reserved (should be zero).               |
| 26:26    | Set if encrypted (see below).            |
| 31:27    | Reserved.                                |
| 39:32    | Cerberus command byte.                   |
| Variable | The command payload.                     |

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

| Payload   | Description                                                        |
|-----------|--------------------------------------------------------------------|
| 0:0       | Integrity check flag.                                              |
| 7:1       | MCTP message type (always `0x7e`).                                 |
| 23:8      | MCTP PCI Vendor ID (always `0x1414`).                              |
| 24:24     | Request type.                                                      |
| 25:25     | Reserved (should be zero).                                         |
| 26:26     | Set if encrypted (always, in this case).                           |
| 31:27     | Reserved.                                                          |
| Variable  | Ciphertext; corresponds to the command byte and its payload above. |
| N+15:N    | AES-GCM tag.                                                       |
| N+31:N+15 | AES-GCM initialization vector.                                     |

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


### Error Message

The error command is returned for command responses when the command was not
completed as proposed, it also acts as a generic status for commands without
response whereby "No Error" code would indicate success.  The Message Tag,
Sequence Number and Command match the response to the corresponding request.
The Message Body is returned as follows:

| Payload | Description |
|---------|-------------|
| 1       | Error code  |
| 2:5     | Error data  |

"Error code" is one of the following:


    Table 9 Error Codes


| Error Code             | Value | Description                              | Data           |
|------------------------|-------|------------------------------------------|----------------|
| No Error               | 0x00  | Success                                  | 0x00           |
| Invalid Request        | 0x01  | Invalidated data in the request          | 0x00           |
| Busy                   | 0x03  | Device is busy processing other commands | 0x00           |
| Unspecified            | 0x04  | Unspecified error occurred               | Vendor defined |
| Invalid Checksum       | 0xf0  | Invalid checksum                         | Checksum       |
| Out of Order Message   | 0xf1  | EOM before SOM                           | 0x00           |
| Authentication         | 0xf2  | Authentication not established           | 0x00           |
| Out of Sequence Window | 0xf3  | Message received out of Sequence Window  | 0x00           |
| Invalid Packet Length  | 0xf4  | Packet received with unexpected size     | Packet Length  |
| Message Overflow       | 0xf5  | Message exceeded maximum length          | Message Length |

If an explicit response is not defined for the command definitions in the
following sections, the Error Message is the expected response with "No Error".
The Error Message can occur as the response for any command that fails
processing.

### Firmware Version

This command gets the target firmware the version.

#### Request

| Payload | Description |
|---------|-------------|
| 1       | Area Index  |

Area indexes are as follows:
*   0x00: Entire Firmware
*   0x01: RIoT Core
*   Additional indexes are vendor-defined.

#### Resposne

| Payload | Description                     |
|---------|---------------------------------|
| 1:32    | Firmware Version Number (ASCII) |


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


#### Request

<table>
    <tr>
        <td> <strong>Payload</strong> </td>
        <td><strong>Description</strong> </td>
    </tr>
    <tr>
        <td>1:2 </td>
        <td>Maximum Message Payload Size </td>
    </tr>
    <tr>
        <td>3:4 </td>
        <td>Maximum Packet Payload Size </td>
    </tr>
    <tr>
        <td>5 </td>
        <td>Mode:
            <br> [7:6]
            <br> 00 = AC-RoT
            <br> 01 = PA-RoT
            <br> 10 = External
            <br> 11 = Reserved
            <br> [5:4] Master/Slave
            <br> 00 = Unknown
            <br> 01 = Master
            <br> 10 = Slave
            <br> 11 = both master and slave
            <br> [3] Reserved
            <br> [2:0] Security
            <br> 000 = None
            <br> 001 = Hash/KDF
            <br> 010 = Authentication [Certificate Auth]
            <br> 100 = Confidentiality [AES] </td>
    </tr>
    <tr>
        <td>6 </td>
        <td>[7] PFM support
            <br> [6] Policy Support
            <br> [5] Firmware Protection
            <br> [4-0] Reserved </td>
    </tr>
    <tr>
        <td>7 </td>
        <td>PK Key Strength:
            <br> [7] RSA
            <br> [6] ECDSA
            <br> [5:3] ECC
            <br> 000: None
            <br> 001: 160bit
            <br> 010: 256bit
            <br> 100: Reserved
            <br> [2:0] RSA:
            <br> 000: None
            <br> 001: RSA 2048
            <br> 010: RSA 3072
            <br> 100: RSA 4096 </td>
    </tr>
    <tr>
        <td>8 </td>
        <td>Encryption Key Strength:
            <br> [7] ECC
            <br> [6:3] Reserved
            <br> [2:0] AES:
            <br> 000: None
            <br> 001: 128 bit
            <br> 010: 256 bit
            <br> 100: 384 bit </td>
    </tr>
</table>


#### Response

<table>
    <tr>
        <td><strong>Payload</strong> </td>
        <td><strong>Description</strong> </td>
    </tr>
    <tr>
        <td>1:2 </td>
        <td>Maximum Message Payload Size </td>
    </tr>
    <tr>
        <td>3:4 </td>
        <td>Maximum Packet Payload Size </td>
    </tr>
    <tr>
        <td>5 </td>
        <td>Mode:
            <br> [7:6]
            <br> 00 = AC-RoT
            <br> 01 = PA-RoT
            <br> 10 = External
            <br> 11 = Reserved
            <br> [5:4] Master/Slave
            <br> 00 = Unknown
            <br> 01 = Master
            <br> 10 = Slave
            <br> 11 = both master and slave
            <br> [3] Reserved
            <br> [2:0] Security
            <br> 000 = None
            <br> 001 = Hash/KDF
            <br> 010 = Authentication [Certificate Auth]
            <br> 100 = Confidentiality [AES] </td>
    </tr>
    <tr>
        <td>6 </td>
        <td>[7] PFM support
            <br> [6] Policy Support
            <br> [5] Firmware Protection
            <br> [4-0] Reserved </td>
    </tr>
    <tr>
        <td>7 </td>
        <td>PK Key Strength:
            <br> [7] RSA
            <br> [6] ECDSA
            <br> [5:3] ECC
            <br> 000: None
            <br> 001: 160bit
            <br> 010: 256bit
            <br> 100: Reserved
            <br> [2:0] RSA:
            <br> 000: None
            <br> 001: RSA 2048
            <br> 010: RSA 3072
            <br> 100: RSA 4096 </td>
    </tr>
    <tr>
        <td>8 </td>
        <td>Encryption Key Strength:
            <br> [7] ECC
            <br> [6:3] Reserved
            <br> [2:0] AES:
            <br> 000: None
            <br> 001: 128 bit
            <br> 010: 256 bit
            <br> 100: 384 bit </td>
    </tr>
    <tr>
        <td>9 </td>
        <td>Maximum Message timeout: multiple of 10ms </td>
    </tr>
    <tr>
        <td>10 </td>
        <td>Maximum Cryptographic Message timeout: multiple of 100ms </td>
    </tr>
</table>


### Device Id

Eight bytes response.

#### Request

Empty body.

### Response

| Payload | Description              |
|---------|--------------------------|
| 1:2     | Vendor ID; LSB           |
| 3:4     | Device ID; LSB           |
| 5:6     | Subsystem Vendor ID; LSB |
| 7:8     | Subsystem ID; LSB        |


### Device Information

This command gets information about the target device.

#### Request

| Payload | Description       |
|---------|-------------------|
| 1       | Information Index |


Information indexes are as follows:
*   0x00: Unique Chip Identifier.
*   Additional indexes are vendor-defined.

#### Response

| Payload | Description               |
|---------|---------------------------|
| 1:N     | Requested identifier blob |


### Export CSR

Exports the Device Identification Certificate Self Signed Certificate Signing
Request.  The initial Certificate is self-signed, until CA signed and imported.
Once the CA signed version of the certificate has been imported to the device,
the self-signed certificate is replaced.

#### Request

| Payload | Description        |
|---------|--------------------|
| 1       | Index: Default = 0 |

#### Response

| Payload | Description                 |
|---------|-----------------------------|
| 1:N     | Certificate Signing Request |


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

#### Request

| Payload | Description        |
|---------|--------------------|
| 1       | Type               |
| 2:3     | Certificate Length |
| 4:N     | Certificate        |


Certificate types are as follows:
*   0x00: Device Id Certificate
*   0x01: Root CA Certificate
*   0x02: Intermediate CA Certificate


### Get Certificate State

Determine the state of the certificate chain for signed certificates that have
been sent to the device.  The request for this command contains no additional
payload.

#### Request

Empty body.

#### Response

| Payload | Description                                    |
|---------|------------------------------------------------|
| 1       | State                                          |
| 2:4     | Error details, if chain validation has failed. |

Possible states:
*   0x00: A valid chain has been provisioned.
*   0x01: A valid chain has not yet been provisioned.
*   0x02: The stored chain is being validated.

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

#### Request

| Payload | Description                                                   |
|---------|---------------------------------------------------------------|
| 1       | Slot Number (0 to 7) of the target Certificate Chain to read. |
| 2       | Key Exchange Algorithm                                        |

Potential key exchange algorithms are:
*   0x00: None
*   0x01: ECDH

#### Response

| Payload | Description                                                              |
|---------|--------------------------------------------------------------------------|
| 1       | Capabilities Field (always 0x01).                                        |
| 2       | Number of digests returned.                                              |
| 3:N     | SHA256 digests of the certificates in the chain, starting from the root. |


### GET CERTIFICATE

This command retrieves the public attestation certificate chain for the AC-RoT.
It follows closely the USB Type C Authentication Specification.

If the device does not have a certificate for the requested slot or index, the
certificate contents in the response will be empty.

#### Request

| Payload | Description                                                    |
|---------|----------------------------------------------------------------|
| 1       | Slot Number (0 to 7) of the target Certificate Chain to read.  |
| 2       | Certificate to read, zero-indexed from the root Certificate.   |
| 3:4     | Offset: offset in bytes from start of the Certificate to read. |
| 5:6     | Length: number of bytes to read.                               |

#### Resposne

| Payload | Description                                           |
|---------|-------------------------------------------------------|
| 1       | Slot Number of the target Certificate Chain returned. |
| 2       | Certificate index of the returned certificate         |
| 3:N     | Requested contents of target Certificate Chain.       |


### CHALLENGE

The PA-RoT will send this command providing the first nonce in the key exchange.

#### Request

| Payload | Description                                                   |
|---------|---------------------------------------------------------------|
| 1       | Slot Number (0 to 7) of the target Certificate Chain to read. |
| 2       | Reserved                                                      |
| 3:35    | Random nonce                                                  |

#### Resposne

| Payload | Description                                                        |
|---------|--------------------------------------------------------------------|
| 1       | Slot Number of the Certificate Chain used.                         |
| 2       | Certificate slot mask                                              |
| 3       | MinProtocolVersion supported by device                             |
| 4       | MaxProtocolVersion supported by device                             |
| 5:6     | Reserved                                                           |
| 7:38    | Random nonce                                                       |
| 39      | Number of components used to generate the PMR0 measurement         |
| 40      | Length of PMR0 (L)                                                 |
| 41:40+L | Value of PMR0 (Aggregated Firmware Digest)                         |
| 41+L:N  | Signature over concatenated request and response message payloads. |

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

#### Request

| Payload | Description                                          |
|---------|------------------------------------------------------|
| 1       | Key Type                                             |
| 2:N     | Key data.  Format is defined by the type of request. |

Possible key types:
*   0x00: Session key.
*   0x01: Paired-Key HMAC.
*   0x02: Destroy Session.

##### Session Key Data

| Payload | Description                                |
|---------|--------------------------------------------|
| 1       | HMAC Algorithm                             |
| 2:N     | ASN.1 DER encoded ECC public key (`PKreq`) |

Possible HMAC Hash Algorithms:
*   0x00: SHA-256
*   0x01: SHA-384
*   0x02: SHA-512

The HMAC type specified in this message applies to all HMAC operations for the
established session, including any subsequent pairing messages.  Since session
keys (`K_S` and `K_M`) are 256-bit keys, they will always be generated using
SHA-256 regardless of the type of HMAC used for key exchange messages.

##### Paired-Key HMAC Data

| Payload | Description                               |
|---------|-------------------------------------------|
| 1:2     | Length in bytes of the pairing key        |
| 3:N     | HMAC of the pairing key: `HMAC(K_M, K_P)` |

##### Destroy Session Data

| Payload | Description                           |
|---------|---------------------------------------|
| 1:N     | HMAC of session key: `HMAC(K_M, K_S)` |

#### Response

| Payload | Description                                               |
|---------|-----------------------------------------------------------|
| 1       | Key Type from request                                     |
| 2:N     | Response data.  Format is defined by the type of request. |

##### Session Key Data 

| Payload | Description                                           |
|---------|-------------------------------------------------------|
| 1       | Reserved.  Set to 0.                                  |
| 2:3     | Key Length                                            |
| 4:N     | DER-encoded ECC public key (`PKresp`)                 |
| N+1:N+2 | Signature Length                                      |
| N+3:M   | Signature over session keys: `SIGN(PKreq || PKresp)`  |
| M+1:M+2 | HMAC Length                                           |
| M+3:H   | HMAC of the Alias certificate: `HMAC(KM, alias_cert)` |

##### Paired-Key HMAC Data

Empty body.

##### Destory Session Data

Empty body.


### Session Sync

Check the state of an encrypted session.
The message must always be encrypted.

#### Request.

| Payload | Description             |
|---------|-------------------------|
| 1:4     | Random number (`RNreq`) |

#### Response.

| Payload | Description                           |
|---------|---------------------------------------|
| 1:N     | HMAC of the nonce:  `HMAC(KM, RNreq)` |


### Get Log Info

Get the internal log information for the RoT.

#### Request

Empty body.

#### Response

| Payload | Description                            |
|---------|----------------------------------------|
| 1:4     | Debug Log (0x01) Length in bytes       |
| 5:8     | Attestation Log (0x02) Length in bytes |
| 9:12    | Tamper Log (0x03) Length in bytes      |


### Get Log

Get the internal logs for the RoT.  There are 3 types of logs available:  The
Debug Log, which contains Cerberus application information and machine state.
The Attestation measurement log, this log format is like the TCG log, and the
Tamper log.  It is not possible to clear or reset the tamper counter.

#### Request

| Payload | Description |
|---------|-------------|
| 1       | Log Type    |
| 2:5     | Offset      |

Possible log types:
*   0x01: Debug Log.
*   0x02: Attestation Log.
*   0x03: Tamper Log.

#### Response 

| Payload | Description          |
|---------|----------------------|
| 1:N     | Contents of the log. |


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

| Offset | Description                                                                             |
|--------|-----------------------------------------------------------------------------------------|
| 1      | Log entry start marker:  [7:4]: 0x0c [3:0]: Header format, 0x0b per this specification. |
| 2:3    | Total length of the entry, including the header                                         |
| 4:7    | Unique entry identifier                                                                 |

Attestation Entry format:

| Offset | Description                                    |
|--------|------------------------------------------------|
| 1:7    | Log Entry Header                               |
| 8:11   | TCG Event Type                                 |
| 12     | Measurement index within a single PMR.         |
| 13     | Index of the PMR for the measurement.          |
| 14:15  | Reserved, set to 0.                            |
| 16     | Number of digests                              |
| 17:19  | Reserved, set to 0.                            |
| 20:21  | Digest algorithm Id (`0x0b`, SHA-256)          |
| 22:53  | SHA-256 digest used to extend the measurement. |
| 54:57  | Measurement length                             |
| 58:89  | Measurement                                    |

#### Debug Log Format

The debug log reported by the device has no specified format, as this can vary
between different devices and it is not necessary for attestation.  It is
expected that diagnostic utilities for the device will be able to understand the
exposed log information.  A recommended entry format is provided here.

Suggested Debug Entry format:

| Offset | Description                      |
|--------|----------------------------------|
| 1:7    | Log Entry Header                 |
| 8:9    | Format version of the entry.     |
| 10     | Severity of the entry.           |
| 11     | ID of the source of the message. |
| 12     | ID for the entry message type.   |
| 13:16  | Message specific argument.       |
| 17:20  | Message specific argument.       |

### Clear Debug/Attestation Log

Clear the log in the RoT. Note that the tamper log cannot be cleared.

#### Request

| Payload | Description |
|---------|-------------|
| 1       | Log Type    |


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

#### Request

| Payload | Description                   |
|---------|-------------------------------|
| 1       | Platform Measurement Register |
| 2       | Entry Index                   |
| 3:6     | Offset                        |

#### Response

| Payload | Description        |
|---------|--------------------|
| 1:N     | The measured data. |


### Get Host State

Retrieve the reset state of the host processor being protected by Cerberus.


#### Request

| Payload | Description |
|---------|-------------|
| 1       | Port Id     |

#### Response
| Payload | Description      |
|---------|------------------|
| 1       | Host Reset State |

Possible states:
*   0x00 - Host is running.
*   0x01 - Host is held in reset.
*   0x02 - Host is not running, nor held in reset.

### Get PFM Id

Retrieves PFM identifiers.

#### Request

| Payload      | Description                                                  |
|--------------|--------------------------------------------------------------|
| 1            | Port Id                                                      |
| 2            | PFM Region:  0x00 = Active, 0x01 = Pending                   |
| 3 (optional) | Identifier:  0x00 = Version Id (default), 0x01 = Platform Id |

#### Response

| Payload | Description              |
|---------|--------------------------|
| 1       | PFM Valid (0x00 or 0x01) |
| 2:5     | PFM Version Id           |

| Payload | Description                              |
|---------|------------------------------------------|
| 1       | PFM Valid (0 or 1)                       |
| 2:N     | PFM Platform Id as null-terminated ASCII |


### Get PFM Supported Firmware 

#### Request

| Payload | Description                               |
|---------|-------------------------------------------|
| 1       | Port Id                                   |
| 2       | PFM Region: 0x00 = Active, 0x01 = Pending |
| 3:6     | Offset                                    |

#### Response

| Payload | Description               |
|---------|---------------------------|
| 1       | PFM Valid (0x00 or 0x01)  |
| 2:5     | PFM Version Id            |
| 6:N     | PFM supported FW versions |

If response spans multiple MCTP messages, end of response will be determined by
an MCTP packet which has payload less than maximum payload supported by both
devices.  To guarantee a response will never fall exactly on the max payload
boundary, the responder should send back an extra packet with zero payload.


### Prepare PFM

Provisions RoT for incoming PFM.

#### Request

| Payload | Description |
|---------|-------------|
| 1       | Port Id     |
| 2:5     | Total size  |

#### Update PFM

The flash descriptor structure describes the regions of flash for the device.

#### Request

| Payload | Description |
|---------|-------------|
| 1       | Port Id     |
| 2:N     | PFM Payload |

PFM payload includes PFM signature and monotonic forward only Id.  PFM signature
is verified upon receipt of all PFM payloads.  PFMs are activated upon the
activation command.  Note if a system is rebooted after receiving a PFM, the
PFM is atomically activated.  To activate before reboot, issue the Activate PFM
command.


## Activate PFM

Upon valid PFM update, the update command seals the PFM committal method.  If
committing immediately, flash reads and writes should be suspended when this
command is issued.  The RoT will master the SPI bus and verify the newly updated
PFM.  This command can only follow a valid PFM update.

#### Request

| Payload | Description                                         |
|---------|-----------------------------------------------------|
| 1       | Port Id                                             |
| 2       | Activation:  0x00 = Reboot only, 0x01 = Immediately |

If reboot only has been issued, the option for "Immediately" committing the PFM
is not available until a new PFM is updated.


### Get CFM Id

Retrieves the Component Firmware Manifest Id

#### Request

| Payload      | Description                                                  |
|--------------|--------------------------------------------------------------|
| 1            | Port Id                                                      |
| 2            | CFM Region:  0x00 = Active, 0x01 = Pending                   |
| 3 (optional) | Identifier:  0x00 = Version Id (default), 0x01 = Platform Id |

#### Response

| Payload | Description              |
|---------|--------------------------|
| 1       | CFM Valid (0x00 or 0x01) |
| 2:5     | CFM Version Id           |

| Payload | Description                              |
|---------|------------------------------------------|
| 1       | CFM Valid (0 or 1)                       |
| 2:N     | CFM Platform Id as null-terminated ASCII |


### Prepare CFM

Provisions RoT for incoming Component Firmware Manifest.

#### Request

| Payload | Description |
|---------|-------------|
| 1:5     | Total size  |


### Update CFM

The flash descriptor structure describes the regions of flash for the device.

#### Request

| Payload | Description |
|---------|-------------|
| 1:N     | CFM Payload |

The CFM payload includes CFM signature and monotonic forward only Id.  CFM
signature is verified upon receipt of all CFM payloads.  CFMs are activated
upon the activation command.  Note if a system is rebooted after receiving a
CFM, the pending CFM is verified and atomically activated.  To activate before
reboot, issue the Activate CFM command.


### Activate CFM

Upon valid CFM update, the update command seals the CFM committal method.  The
RoT will master I2C and attest Components in the Platform Configuration Data
against the CFM.

#### Request

| Payload | Description                                         |
|---------|-----------------------------------------------------|
| 1       | Activation:  0x00 = Reboot only, 0x01 = Immediately |


### Get CFM Component IDs

#### Request

| Payload | Description                                |
|---------|--------------------------------------------|
| 1       | CFM Region: 0x00 = Active,  0x01 = Pending |
| 2:5     | Offset                                     |


#### Response

| Payload | Description                 |
|---------|-----------------------------|
| 1       | CFM Valid (0x00 or 0x01)    |
| 2:5     | CFM Version Id              |
| 6:N     | CFM supported component IDs |

If response spans multiple MCTP messages, end of response will be determined by
an MCTP packet which has payload less than maximum payload supported by both
devices.  To guarantee a response will never fall exactly on the max payload
boundary, the responder should send back an extra packet with zero payload.


### Get PCD Id

Retrieves the PCD Id.

#### Request

| Payload      | Description                                                 |
|--------------|-------------------------------------------------------------|
| 1 (optional) | Identifier: 0x00 = Version Id (default), 0x01 = Platform Id |

#### Response

| Payload | Description              |
|---------|--------------------------|
| 1       | PCD Valid (0x00 or 0x01) |
| 2:5     | PCD Version Id           |

| Payload | Description                              |
|---------|------------------------------------------|
| 1       | PCD Valid (0x00 or 0x01)                 |
| 2:N     | PCD Platform Id as null-terminated ASCII |


### Prepare PCD

Provisions RoT for incoming Platform Configuration Data.

#### Request

| Payload | Description |
|---------|-------------|
| 1:4     | Total size  |


### Update PCD

The flash descriptor structure describes the regions of flash for the device.

#### Request

| Payload | Description |
|---------|-------------|
| 1:N     | PCD Payload |


The PCD payload includes PCD signature and monotonic forward only Id.  PCD
signature is verified upon receipt of all PCD payloads.  PCD is activated upon
the activation command.  Note if a system is rebooted after receiving a PCD.


### Activate PCD

Upon valid PCD update, the activate command seals the PCD committal.

#### Request

Empty body.


### Platform Configuration

The following table describes the Platform Configuration Data Structure.

<!-- TODO: convert this to a proper table -->
<table>
    <tr>
        <td><strong>Payload</strong> </td>
        <td><strong>Description</strong> </td>
    </tr>
    <tr>
        <td>1:3 </td>
        <td>Platform Configuration Data Id </td>
    </tr>
    <tr>
        <td>4:5 </td>
        <td>Length </td>
    </tr>
    <tr>
        <td>6 </td>
        <td>Policy Count </td>
    </tr>
    <tr>
        <td>7:N </td>
        <td>Each AC-RoT has 1 entry.
            The Configuration Data determines the feature
            enablement and attestation
            <br>
            <table>
                <tr>
                    <td>Byte </td>
                    <td>Description </td>
                </tr>
                <tr>
                    <td>1 </td>
                    <td>Device Id </td>
                </tr>
                <tr>
                    <td>4 </td>
                    <td>Channel </td>
                </tr>
                <tr>
                    <td>5 </td>
                    <td>Slave Address </td>
                </tr>
                <tr>
                    <td>6 </td>
                    <td>[7:5] Threshold Count
                        <br> [4] Power Control
                        <br> 0 = Disabled
                        <br> 1 = Enabled
                        <br> [3] Debug Enabled
                        <br> 0 = Disabled
                        <br> 1 = Enabled
                        <br> [2] Auto Recovery
                        <br> 0 = Disabled
                        <br> 1 = Enabled
                        <br> [1] Policy Active
                        <br> 0 = Disabled
                        <br> 1 = Enabled
                        <br> [0] Threshold Active
                        <br> 0 = Disabled
                        <br> 1 = Enabled </td>
                </tr>
                <tr>
                    <td>7 </td>
                    <td>Power Ctrl Index </td>
                </tr>
                <tr>
                    <td>8 </td>
                    <td>Failure Action </td>
                </tr>
            </table>
        </td>
    </tr>
    <tr>
        <td>N:N </td>
        <td>Signature of payload </td>
    </tr>
</table>

The Power Control Index informs the PA-RoT of the index assigned to power
sequence the Component.  This informs the PA-RoT which control register needs
to be asserted in the platform power sequencer.

Possible failure actions:
*   0x00: Platform defined.
*   0x01: Report only.
*   0x02: Automatic recovery.
*   0x03: Power control.


### Prepare Firmware Update

Provisions RoT for incoming firmware update.

#### Request

| Payload | Description |
|---------|-------------|
| 1:4     | Total size  |


### Update Firmware

The flash descriptor structure describes the regions of flash for the device.

#### Request

| Payload | Description                                |
|---------|--------------------------------------------|
| 1:N     | Firmware Update payload, header signature. |


### Update Status

The Update Status reports the update payload status.  The update status will be
status for the last operation that was requested.  This status will remain the
same until another operation is performed or Cerberus is reset.

#### Request

| Payload | Description |
|---------|-------------|
| 1       | Update Type |
| 2       | Port Id     |

Possible update type:
*   0x00: Firmware
*   0x01: PFM
*   0x02: CFM
*   0x03: PCD
*   0x04: Host Firmware
*   0x05: Recovery Firmware
*   0x06: Reset Configuration

#### Response

| Payload | Description                                                    |
|---------|----------------------------------------------------------------|
| 1:4     | Update Status.  See firmware update specification for details. |


### Extended Update Status

The Extended Update Status reports the update payload status along with the
remaining number of update bytes expected.  The update status will be status for
the last operation that was requested.  This status will remain the same until
another operation is performed or Cerberus is reset.

#### Request

| Payload | Description |
|---------|-------------|
| 1       | Update Type |
| 2       | Port Id     |

Possible update type:
*   0x00: Firmware
*   0x01: PFM
*   0x02: CFM
*   0x03: PCD
*   0x04: Host Firmware
*   0x05: Recovery Firmware
*   0x06: Reset Configuration

#### Response

| Payload | Description                                                    |
|---------|----------------------------------------------------------------|
| 1:4     | Update Status.  See firmware update specification for details. |
| 5:8     | Expected update bytes remaining.                               |


### Activate Firmware Update

Alerts Cerberus that sending of update bytes is complete, and that verification
of update should start.  This command has no payload, the ERROR response zero is
expected.

#### Request

Empty body.


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

#### Request

| Payload | Description                                                          |
|---------|----------------------------------------------------------------------|
| 1       | Type of reset operation to request.                                  |
| 2:N     | (Optional) Device-specific authorization token, signed with PFM key. |

Possible reset types:
*   0x00: Revert the device into the unprotected (bypass) state by erasing all
    PFMs and CFMs.
*   0x01: Perform a factory reset by removing all configuration.  This does not
    include signed device certificates.

#### Response

| Payload | Description                         |
|---------|-------------------------------------|
| 1:N     | Device-specific authorization token |


### Get Configuration Ids

This command retrieves PFM Ids, CFM Ids, PCD Id, and signed digest of request
nonce and response ids.

#### Request

| Payload | Description  |
|---------|--------------|
| 1:32    | Random Nonce |

#### Response

| Payload                 | Description                 |
|-------------------------|-----------------------------|
| 1:32                    | Random Nonce                |
| 33                      | Number of PFM Ids (P)       |
| 34                      | Number of CFM Ids (C)       |
| 35:(P*4 + C*4 + 4) (V') | PFM, CFM, PCD Version IDs   |
| V'+1:M                  | PFM, CFM, PCD Platfomrm IDs |
| M+1:SIGN                | `SIGN(request || response)` |


### Recover Firmware

Start the firmware recovery process for the device.  Not all devices will
support all types of recovery.  The implementation is device specific.

#### Request

| Payload | Description                                                        |
|---------|--------------------------------------------------------------------|
| 1       | Port Id                                                            |
| 2       | Firmware image to use: 0x00 = Exit Recovery, 0x01 = Enter Recovery |


### Prepare Recovery Image

Provisions RoT for incoming Recovery Image for Port.

#### Requests

| Payload | Description |
|---------|-------------|
| 1       | Port Id     |
| 2:5     | Total size  |


### Update Recovery Image

The flash descriptor structure describes the regions of flash for the device.

#### Request

| Payload | Description            |
|---------|------------------------|
| 1       | Port Id                |
| 2:N     | Recovery Image Payload |


### Activate Recovery Image

Signals recovery image has been completely sent and verification of the image
should start.  Once the image has been verified, it can be used for host
firmware recovery.

### Request

| Payload | Description |
|---------|-------------|
| 1       | Port Id     |


### Get Recovery Image Id

Retrieves the recovery image identifiers.

#### Request

| Payload      | Description                                                 |
|--------------|-------------------------------------------------------------|
| 1            | Port Id                                                     |
| 2 (optional) | Identifier: 0x00 = Version Id (default), 0x01 = Platform Id |

#### Response

| Payload | Description                                         |
|---------|-----------------------------------------------------|
| 1:N     | Recovery Image Platform Id as null-terminated ASCII |


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

#### Request

| Payload | Description                 |
|---------|-----------------------------|
| 1       | Platform Measurement Number |
| 2:33    | Random Nonce                |

#### Response

| Payload | Description                 |
|---------|-----------------------------|
| 1:32    | Random Nonce                |
| 33      | Measurement length (L)      |
| 34:33+L | Platform Measurement Value  |
| 34+L:N  | `SIGN(request ++ response)` |

PMR1-4 are cleared on component reset.  PMR0 is cleared and re-built on Cerberus
reset.


### Update Platform Measurement Register

External updates to PMR3-4 are permitted.  Attempts to update PMR0-2 will result
error.  Only SHA2 is supported for measurement extension.  SHA1 and SHA3 are
not applicable.  Note:  The measurement can only be updated over an
authenticated and secured channel.

#### Reqyest

| Payload | Description                 |
|---------|-----------------------------|
| 1       | Platform Measurement Number |
| 2:N     | Measurement Extension       |


### Reset Counter

Provides Cerberus and Component Reset Counter since power-on.

#### Request

| Payload | Description        |
|---------|--------------------|
| 1       | Reset Counter Type |
| 2       | Port Id            |

Possible reset counter types:
*   0x00: Local Device
*   0x01: Protected External Devices (if applicable).
    These does not include external AC-RoTs that are challenged by the device.
*   Other values are vendor-defined.

#### Response

| Payload | Description |
|---------|-------------|
| 1:2     | Reset Count |


### Message Unseal 

This command starts unsealing an attestation message.  The ciphertext is limited
to what can fit in a single message along with the other pieces necessary for
unsealing.

#### Request

<!-- TODO: make this a proper table. -->
<table>
    <tr>
        <td><strong>Payload</strong> </td>
        <td><strong>Description</strong> </td>
    </tr>
    <tr>
        <td>1 </td>
        <td>[7:5] Reserved
            <br> [4:2] HMAC Type:
            <br> 000 – SHA256
            <br> [1:0] Seed Type:
            <br> 00 – RSA: Seed is encrypted with an RSA public key
            <br> 01 – ECDH: Seed is an ECC public key, ASN.1/DER encoded </td>
    </tr>
    <tr>
        <td>2 </td>
        <td>Additional Seed Parameters
            <br> RSA:
            <br> [7:3] Reserved
            <br> [2:0] Padding Scheme:
            <br> 000 – PKCS#1 v1.5
            <br> 001 – OAEP using SHA1
            <br> 010 – OAEP using SHA256
            <br> ECDH:
            <br> [7:1] Reserved
            <br> [0]: Seed Processing:
            <br> 0 – No additional processing. Raw ECDH output is the seed.
            <br> 1 – Seed is a SHA256 hash of the ECDH output. </td>
    </tr>
    <tr>
        <td>3:4 </td>
        <td>Seed Length (S) </td>
    </tr>
    <tr>
        <td>5:4+S (S') </td>
        <td>Seed </td>
    </tr>
    <tr>
        <td>S'+1:S'+2 </td>
        <td>Cipher Text Length (C) </td>
    </tr>
    <tr>
        <td>S'+3:S'+2+C (C') </td>
        <td>Cipher Text </td>
    </tr>
    <tr>
        <td>C'+1:C'+2 </td>
        <td>HMAC Length (H) </td>
    </tr>
    <tr>
        <td>C'+3:C'+2+H (H') </td>
        <td>HMAC </td>
    </tr>
    <tr>
        <td>H'+1:H'+64 (P0') </td>
        <td>PMR0 Sealing, 0's to ignore. Unused bytes are first and must be set to 0. </td>
    </tr>
    <tr>
        <td>P0'+1:P0'+64 (P1') </td>
        <td>PMR1 Sealing, 0's to ignore. Unused bytes are first and must be set to 0. </td>
    </tr>
    <tr>
        <td>P1'+1:P1'+64 (P2') </td>
        <td>PMR2 Sealing, 0's to ignore. Unused bytes are first and must be set to 0. </td>
    </tr>
    <tr>
        <td>P2'+1:P2'+64 (P3') </td>
        <td>PMR3 Sealing, 0's to ignore. Unused bytes are first and must be set to 0. </td>
    </tr>
    <tr>
        <td>P3'+1:P3'+64 </td>
        <td>PMR4 Sealing, 0's to ignore. Unused bytes are first and must be set to 0. </td>
    </tr>
</table>


### Message Unseal Result

This command retrieves the current status of an unsealing process.

#### Request

Empty body.

#### Response

| Payload | Description      |
|---------|------------------|
| 1:4     | Unsealing status |

| Payload | Description           |
|---------|-----------------------|
| 1:4     | Unsealing status      |
| 5:6     | Encryption Key Length |
| 7:N     | Encryption Key        |

The Seal/Unseal flow is described in the Cerberus Attestation Integration
specification.

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


    Table 98 PFM Attributes


<table> <tr> <td><strong>Attribute</strong> </td>
<td><strong>Description</strong> </td> </tr> <tr> <td>Description </td>
<td>Device Part or Description </td> </tr> <tr> <td>Device Type </td>
<td>Underlying Device Type of AC-RoT </td> </tr> <tr> <td>Remediation Policy
</td> <td>Policy(s) defining default remediation actions for integrity failure.
</td> </tr> <tr> <td>Firmware Version </td> <td>List of firmware versions </td>
</tr> <tr> <td>Flash Areas/Offsets </td> <td>List of offset and digests, used
and unused </td> </tr> <tr> <td>Measurement </td> <td>Firmware Measurements
</td> </tr> <tr> <td>Measurement Algorithm </td> <td>Algorithm used to calculate
measurement.  </td> </tr> <tr> <td>Public Key </td> <td>Public keys in the key
manifest </td> </tr> <tr> <td>Digest Algorithm </td> <td>Algorithm used to
calculate </td> </tr> <tr> <td>Signature </td> <td>Firmware signature(s) </td>
</tr> </table>


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


    Figure 12 External Communication Interface


    

<p id="gdcalert11" ><span style="color: red; font-weight: bold">>>>>>
gd2md-html alert: inline image link here (to images/image11.png).  Store image on
your image server and adjust path/filename/extension if necessary.
</span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert12">Next
alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image11.png "image_tooltip")


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


    Figure 14 Register Read Flow



<p id="gdcalert12" ><span style="color: red; font-weight: bold">>>>>>
gd2md-html alert: inline image link here (to images/image12.png).  Store image on
your image server and adjust path/filename/extension if necessary.
</span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert13">Next
alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image12.png "image_tooltip")


The following diagram depicts register write access flow for a large register
space, with required seal (update complete bit):


    Figure 15 Register Write Flow



<p id="gdcalert13" ><span style="color: red; font-weight: bold">>>>>>
gd2md-html alert: inline image link here (to images/image13.png).  Store image on
your image server and adjust path/filename/extension if necessary.
</span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert14">Next
alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image13.png "image_tooltip")

### Legacy Active Component RoT Commands

The following table describes the commands accepted by the Active Component RoT.
All commands are master initiated.  The command number is not representative of
a contiguous memory space, but an index to the respective register

Table 99 Commands


<table> <tr> <td><strong>Register Name</strong> </td>
<td><strong>Command</strong> </td> <td><strong>Length</strong> </td>
<td><strong>R/W</strong> </td> <td><strong>Description</strong> </td> </tr> <tr>
<td>Status </td> <td>0x30 </td> <td>2 </td> <td>R </td> <td>Command Status </td>
</tr> <tr> <td>Firmware Version </td> <td>0x32 </td> <td>16 </td> <td>R/W </td>
<td>Retrieve firmware version information </td> </tr> <tr> <td>Device Id </td>
<td>0x33 </td> <td>8 </td> <td>R </td> <td>Retrieves Device Id </td> </tr> <tr>
<td>Capabilities </td> <td>0x34 </td> <td>9 </td> <td>R </td> <td>Retrieves
Device Capabilities </td> </tr> <tr> <td>Certificate Digest </td> <td>3C </td>
<td>32 </td> <td>R </td> <td>SHA256 of Device Id Certificate </td> </tr> <tr>
<td>Certificate </td> <td>3D </td> <td>4096 </td> <td>R/W </td> <td>Certificate
from the AC-Rot </td> </tr> <tr> <td>Challenge </td> <td>3E </td> <td>32 </td>
<td>W </td> <td>Nonce written by RoT </td> </tr> <tr> <td>Platform Configuration
Register </td> <td>0x03 </td> <td>0x5e </td> <td>R </td> <td>Reads firmware
measurement, calculated with S Nonce </td> </tr> </table>

### Legacy Command Format

The following section describes the register format for AC-RoT that do not
implement SMBUS and comply with the legacy measurement exchange protocol.



        1. Status

The SMBUS read command reads detailed information on error status.  The status
register is issued between writing the challenge nonce and reading the
Measurement.  The delay time for deriving the Measurement must comply with the
Capabilities command.


    Table 100 Status Register


<table> <tr> <td><strong>Payload</strong> </td> <td><strong>Description</strong>
</td> </tr> <tr> <td>1 </td> <td>Status: <p>

    00 = Complete <p>

    01 In Progress <p>

    02 Error </td> </tr> <tr> <td>2 </td> <td>Error Data or Zero </td> </tr>
    </table>




        2. Firmware Version

The SMBUS write command payload sets the index.  The subsequent SMBUS read
command reads the response.  For register payload description see response:
Table 11 Firmware Version Response



        3. Device Id

    The SMBUS read command reads the response.  For register payload
    description see response:  Table 1 Field Definitions 

        4. Device Capabilities

    The SMBUS read command reads the response.  For register payload
    description see response: 


Table 13 Device Capabilities Response 

        5. Certificate Digest

The SMBUS read command reads the response.  For register payload description
see response: Table 24 GET DIGEST Response

The PA-Rot will use the digest to determine if it has the certificate already
cached.  Unlike MCTP, only the Alias and Device Id cert is supported.
Therefore, it must be CA signed by a mutually trusted CA, as the CA Public Cert
is not present



        6. Certificate

The SMBUS write command writes the offset into the register space.  For register
payload description see response:  Table 26 GET CERTIFICATE Response


#### Unlike MCTP, only the Alias and Device Id cert is supported.  Therefore,
it must be CA signed by mutually trusted CA, as the CA Public Cert is not
present in the reduced challenge

The SMBUS write command writes a nonce for measurement freshness.


    Table 101 Challenge Register


<table> <tr> <td><strong>Payload</strong> </td> <td><strong>Description</strong>
</td> </tr> <tr> <td>1:32 </td> <td>Random 32 byte nonce chosen by PA-RoT </td>
</tr> </table>




        7. Measurement

The SMBUS read command that reads the signed measurement with the nonce from the
hallenge above.  The PA-RoT must poll the Status register for completion after
issuing the Challenge and before reading the Measurement.


    Table 102 Measurement Register


<table> <tr> <td><strong>Payload</strong> </td> <td><strong>Description</strong>
</td> </tr> <tr> <td>1 </td> <td>Length (L) of following hash digest.  </td>
</tr> <tr> <td>2:33 </td> <td>H(Challenge Nonce || H(Firmware Measurement/PMR0))
</td> </tr> <tr> <td>34:N </td> <td>Signature of HASH [2:33] </td> </tr>
</table>

# References
    1. DICE Architecture
    [https://trustedcomputinggroup.org/work-groups/dice-architectures](https://trustedcomputinggroup.org/work-groups/dice-architectures)
    2. RIoT
    [https://www.microsoft.com/en-us/research/publication/riot-a-foundation-for-trust-in-the-internet-of-things](https://www.microsoft.com/en-us/research/publication/riot-a-foundation-for-trust-in-the-internet-of-things)
    3. DICE and RIoT Keys and Certificates
    [https://www.microsoft.com/en-us/research/publication/device-identity-dice-riot-keys-certificates](https://www.microsoft.com/en-us/research/publication/device-identity-dice-riot-keys-certificates)
    4. USB Type C Authentication Specification
    [http://www.usb.org/developers/docs/](http://www.usb.org/developers/docs/)
    5. PCIe Device Security Enhancements specification
    [https://www.intel.com/content/www/us/en/io/pci-express/pcie-device-security-enhancements-spec.html](https://na01.safelinks.protection.outlook.com/?url=https%3A%2F%2Fwww.intel.com%2Fcontent%2Fwww%2Fus%2Fen%2Fio%2Fpci-express%2Fpcie-device-security-enhancements-spec.html&data=02%7C01%7Cbryankel%40microsoft.com%7C6b6c323d9f5a430b6e2308d5c00880fc%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C636626065116355800&sdata=Kebb47PfKoWc8jO1KHCDCxMriLH5gHncp3lCqyT6WAo%3D&reserved=0)
    6. **NIST Special Publication 800-108 ** Recommendation for Key Derivation
    Using Pseudorandom Functions.
    [http://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-108.pdf](http://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-108.pdf)
    7. TCG PC Client Platform Firmware Profile Specification** **
    [https://trustedcomputinggroup.org/resource/pc-client-specific-platform-firmware-profile-specification](https://trustedcomputinggroup.org/resource/pc-client-specific-platform-firmware-profile-specification)
