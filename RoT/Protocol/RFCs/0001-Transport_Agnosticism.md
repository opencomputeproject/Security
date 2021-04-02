* Name: Transport_Agnosticism
* Date: 2021-04-02
* Pull Request: [#16](https://github.com/opencomputeproject/Security/pull/16)

# Objective

Cerberus currently mandates that its messages be exchanged over an MCTP bus,
using a Microsoft-specific PCI vendor ID. This is a limitation for deployments
that might want to speak Cerberus over a completely different bus, such as
SPI, I3C, exotic buses like NVME, or over plain old TCP or UDP for the
purposes of conformance testing.

This RFC describes a "transport agnostic" model for Cerberus with the following
goals:
- The existing MCTP binding of the protocol works with minimal (or no) changes.
- Cerberus can be used over an arbitrary transport layer without directly
  implementing special support for it in a generic Cerberus library.
- References to MCTP in the Cerberus Challenge Protocol specification will be
  moved to an appendix, describing it as a possible option for transporting
  Cerberus.

# Proposal

"Transport agnostic" Cerberus specifies an *admissible transport* as any protocol
or bus that satisfies a set of properties. These properties reflect the
properties of MCTP already used by Cerberus.

1. An admissible transport is a mechanism for sending a *message* (a dynamically-\
   sized buffer of bytes) from one (not necessarily addressable) endpoint to
   another. In other words, the admissible transport is responsible for removing
   frames from packets and assembling them in sequence.

2. Each endpoint has a known *maximum message length*, measured in bytes. The
   transport is responsible for negotiating this parameter for each device a
   Cerberus device intends to speak to. This parameter must be made available
   to the Cerberus protocol but Cerberus itself does not negotiate it. This
   parameter must be at least 64 bytes.

3. Each message is either the request or response half of a Cerberus command, as
   defined in the Challenge Protocol. To uniquely identify the type of an
   incoming message, the binding of Cerberus to the admissible transport must
   specify a *transport-specific header* that contains at a minimum the following
   parameters. How they are encoded is irrelevant to Cerberus, beyond that they
   be computable without receiving the entire incoming message:
   - The command type byte (as specified in the Challenge Protocol).
   - Whether this message is the request or response half of the command.
   - Whether the payload is authenticated, i.e. whether the payload carried
     wether the payload carried a MAC or was encrypted using a shared secret.
   - The length of the incoming message, in bytes.
   - Addressing information that determines which device sent the message
     (the actual contents of this information is irrelevant to Cerberus).

4. Messages are "addressed", in two respects:
   a. A server can take a request and reply to the original sender with a
      response, and a client can match up requests and their responses.
   b. Cerberus can use addressing information of unspecified format to
      construct requests to a specific device (this opaque addressing
      information could be present in a PCD, for example).

By construction, MCTP almost fulfills the above requirements: it provides framing
of a message of arbitrary size, and the PCIe Vendor command provides a location
for the transport-specific header. Note that Cerberus itself performs the
size negotiation at the moment, so that would be moved out of the Challenge
Protocol and into the transport layer.

The above list is everything Cerberus needs: Cerberus can be spoken over any
admissible transport with no change to the actual bytes present in the
encoded messages.

# Specification Changelist

This RFC seeks the following changes to the Challenge Protocol spec:

- Chapter 3 (Protocol and Hierarchy) should be replaced with the list of
  requirements given above. We do not recommend using the above text
  exactly; instead, the new Chapter 3 should carefully elaborate each point,
  to make it possible for third parties to evaluate whether their chosen
  transport layer is admissible.

- The current contents of Chapter 3, which describes Cerberus-over-MCTP, should
  be moved into an appendix, provided as an example admissible transport layer
  for I2C connections. The same should be done for Section 8.1, which describes
  a legacy protocol built on top of SMBus.

- The first few sections of Chapter 6 should be modified to not reference MTCP,
  and instead refer to the abstract Cerberus header (request bit, type, length)
  derived from the transport-specific header.

- The MCTP-specific errors in Section 6.5 should be removed. Failures such
  during message assembly should be handled at the transport layer; the
  transport should define an error-reporting mechanism for surfacing such errors
  to a client. The nature of this mechanism is not relevant to Cerberus.

- Mentions of MCTP in Section 6.7 should be removed. The maximum message/packet
  size fields should be removed, since the transport layer should negotiate
  these while establishing a session, and pass the maximum message size field
  along to Cerberus.

- The same should be done for Section 6.13.

- The same should also be done for Section 6.18. Since the response makes
  reference to payload sizes, we recommend that, instead of replying with
  multiple messages of maximum size, we add a "continue" bit to the Get Log
  response. When this bit is set, the client must send another request, with
  the offset field increased by the length of the Get Log log contents field.

- The same applies to Sections 6.20, 6.23, and 6.31.

# Implementation Guidance

Manticore currently implements the above proposal via the following procedure:
1. Receive a vtable from the library user that provides a function for blocking
   until a request arrives from somewhere.

2. When that function returns, Manticore's machinery calls a different function
   in the vtable for parsing the parts of the transport-specific header it needs
   to select a message parser and handler (request bit and type byte).

3. Manticore calls into the appropriate parser, which calls a third function in
   the vtable that provides a `read(2)` interface over the message payload to
   parse the message.

4. Pass the parsed message into the appropriate request handler, which constructs
   a response.

5. Manticore then calls a fourth function in the vtable, passing along the type
   of the response, that instructs the transport to prepare for a response. Note
   that the original request contained addressing information that the vtable
   tracks and re-uses for addressing the reply.

6. The transport encodes a response header; Manticore's serializer calls into a
   `write(2)` interface to encode the response. The transport may packetize
   the response as its internal buffer fills up.

7. Manticore calls the final, sixth function to flush the response, sending the
   final packet.

As the author understands it, this is somewhat close to how the Microsoft
implementation handles messages, although it calls the MCTP library directly
rather than virtually.

Manticore's vtable interface can he found at
https://github.com/lowRISC/manticore/blob/e7a532/src/net.rs#L126.

# Future Work

Transport security. This proposal does not cover removing the USB-C-based
transport security from the Cerberus protocol, but we hope to give those
details a similar treatment so that Cerberus can be spoken over transports
which provide their own security such as TLS, QUIC, etc.
