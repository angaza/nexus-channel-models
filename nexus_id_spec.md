# Nexus Channel Core - Nexus ID Specification

Version: 0.9.0

## Purpose and Background

The Nexus ID is a globally-unique identifier that is *required* to be stored
by all devices that implement Nexus Channel Core. It solves many challenges
facing manufacturers of interoperable devices that may need to be managed by
external software platforms.

## Structure

The Nexus ID is a 6-byte structure that consists of two parts:

```
 <-2 bytes-> <----4 bytes---->
+-----------+-----------------+
| Authority |    Device ID    |
| ID        |                 |
+-----------+-----------------+
```

* 2-byte 'Authority ID' indicating what party assigned the Device ID
* 4-byte 'Device ID' unique for all IDs assigned from a given authority

Authority IDs are assigned by an administrator of Nexus IDs (currently only Angaza,
but in the future this could be an industry-wide organization like GOGLA). Authority
IDs function to define unique address spaces. For example, the special Authority ID
`0x0000` is reserved for PAYG IDs, that is, IDs that are meant to be used for devices
registered on PAYG software platforms.

To a manufacturer of a PAYG device, it is possible, then, to request a single Nexus ID
in this Authority ID space for use as an internal serial number, a PAYG ID, and an ID
usable for inter-device communication with Nexus Channel Core!

Because Device IDs within a given Authority ID are administered by a central authority
and globally unique, as long as participating manufacturers guarantee that there are no
collisions within their own assigned Device ID space, then there cannot be any identity
collisions between devices using Nexus IDs.

## External vs. Internal Identifiers

Many device management software platforms require a device-specific identifier
to reference value-added services related to that device. For example, Angaza 
uses a device's identifier to reference its related PAYG loan account.

As described in the previous section, the Device ID part of the Nexus ID *can*
be used as an external identifier for these purposes, if a manufacturer wishes to
reduce the number of serialization schemes in use. However, it is not required that
the Nexus ID is exposed externally. It must just be assigned and stored internally
in the device firmware to avoid addressing collisions between devices on a Nexus
Channel Core network.

## Link Layer: Assumptions and Flexibility

Nexus Channel Core operates on the application layer and is designed to work with
a wide range of link layers. The structure of the Nexus ID is intentionally
designed to make this link-layer flexibility possible.

Nexus Channel Core makes only a few assumptions about the Nexus ID and the link
layer:

* Nexus Channel Core specifies the destination Nexus ID for each request to the
link layer to send a message
* Nexus Channel Core requires the link layer to provide the source and
destination Nexus ID of a received message
* Link layer should have the capability to send a message to every device in
the network (broadcast) and to receive a reply (Nexus Channel Core does not do any
bridging; all nodes in the network must be addressable by the link layer)

Nexus Channel Core doesn't care how the above are met. This makes it easy for highly-
constrained networks to meet the requirements.

### Transmitting Nexus IDs in Highly-Constrained Networks

For networks containing highly bandwidth-constrained buses, the link layer may
abbreviate or elide the source and destination Nexus IDs. Therefore, the Nexus IDs
may not be visible on the wire. This is permissible as long as they can be
reconstructed in full by the receiver's link layer before passing the message upward
to the Nexus Channel Core layer.

In this scenario, is assumed that the lower-level links have a way to map Nexus Channel
Core "addresses" (6-byte Nexus IDs) to lower-level link node addresses if not
sending the full 6-byte source and destination addresses directly on the wire. The 
details of such an address bridging scheme is not described in this specification.

For networks containing highly memory-constrained devices, middleware may hard-code
a constant (known at factory production time) in the link layer as the Nexus ID source
address of the device (for outbound messages) and allocate a static 6-byte field
for the source address (for inbound messages).

### Protocol Bridging

Because the Nexus ID is globally unique, it can be bridged to other protocols, again,
as long as the Nexus ID can be fully reconstructed. For example, for devices that are exposed
to the Internet, the Nexus ID can be expanded to a valid IPV6 address. The size of the Nexus
ID as 48 bits was selected to allow conversion of the Nexus ID into a valid IPV6 address using
EUI-64 expansion (same as MAC address expansion, described in [RFC 2373](https://tools.ietf.org/html/rfc2373#page-19)).

This can be accomplished by running an EUI-64 conversion on the Nexus ID to get the lower 64 bits
of an IPV6 address, and then getting the (fixed) upper 64-bits by registering with an IPV6
administrator, such as ARIN.

## Special Authority ID Values

As specified in the last section, Nexus Channel Core requires that the link layer
is able to broadcast a message, or send it to every other device on the local
network. This is indicated by a special Authority ID in the Nexus ID that is reserved
to make this requirement and other common manufacturing use cases easy to support.

The reserved Authority IDs are:

* **0xFFFF**: reserved for development/testing (anyone can use or make IDs in this
space in a pre-production/testing context)
* **0x0000-0x00FF**: reserved for core 'GOGLA-related' use cases currently, including a
globally-unique PAYG ID scheme (`0x0000`)
* **0xFE80**: reserved for dynamically-assigned Nexus IDs (e.g. Nexus IDs that aren't
globally-unique, e.g. if the link layer has a scheme where it dynamically assigns
identities to devices that have no factory assigned ID or outside world mapping to those
device IDs)
* **0xFFxx (except 0xFFFF)**: reserved to indicate that this message should be broadcast
to all devices in the local network. As such, it will only be used as a destination
address. The lower byte is variable 'xx' so that in the future it is possible to add scope
context to the broadcast. However, it is expected that in most cases, the link layer will
not support broadcast scopes and ignore the lower byte (the API also includes a 'broadcast'
flag so the destination address can be completely ignored by the link layer in that case
as well). This broadcast may elicit zero or one responses from connected devices, implying
that the broadcast contains content that is understood by one specific device. A link layer
*may* support broadcasting with responses from all recipients, which can speed up resource
discovery, but this is not required.

### IPV6 Semantics

The special Authority ID values 0xFE80 and 0xFFxx were chosen to correspond to published
IPV6 standards. `0xFE80` is the [link-local unicast address](https://tools.ietf.org/html/rfc4291#section-2.4).
`0xFFxx` includes the "variable-scope" multicast addresses that are currently
[registered with the IANA](https://www.iana.org/assignments/ipv6-multicast-addresses/ipv6-multicast-addresses.xhtml#variable).

Examples:

* `{0x0000, 0x12345678} // globally-unique PAYG ID; globally-unique Nexus ID`
* `{0xFE80, 0x00000010} // dynamically-assigned by link layer, valid only in the local system and not globally-unique`

## Reference Implementation

The [nexus-embedded](https://github.com/angaza/nexus-embedded/tree/master/nexus)
implementation meets the above specifications, and assumes a compliant
underlying link layer is present.