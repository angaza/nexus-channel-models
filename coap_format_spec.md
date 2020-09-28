# Nexus Channel Core - CoAP Specification

Version: 0.9.0

## Purpose and Background

Nexus Channel Core is based on the [OCF Specification](https://openconnectivity.org/specs/OCF_Core_Specification_v2.1.1.pdf),
which subsequently relies on the [CoAP Standard](https://tools.ietf.org/html/rfc7252) when
sending messages between devices.

Nexus Channel Core highlights a subset of functionality from the above
specifications applicable to constrained devices, while still producing
IETF standards-compliant CoAP messages.

A Nexus Channel Core device may implement *more* of the CoAP standard than
listed in this specification (for example, confirmable messages). However,
when communicating to other Nexus Channel Core devices, only the subset of
functionality outlined in this document is guaranteed to be available.

Some helpful terminology from the CoAP specification:

* 'server' - a device 'hosting' a resource, which allows other devices to
read or modify the resource properties
* 'client' - a device 'accessing' a resource (to read or modify it) by sending
a `GET` or `POST` request to another device (a 'server') hosting the
specified resource

In Nexus Channel Core, devices often are both clients and servers. For example,
an SHS might 'serve' a battery resource to expose its own battery state, but
it might also act as a client towards other devices acting as servers (for example,
to adjust their demand-response load shedding parameters).

'Server' and 'client' are in no way related to device functionality, capability,
or power flow direction.

## CoAP Header - Base

The following parameters *must* be present in the CoAP header:

* CoAP (Ver)sion number is '1' (01 binary).
* (T)ype field is '1' (Non-confirmable) for both requests and responses.
* "Token Length Field" (TKL) is 1.
* "Code" may be any valid CoAP code (value depends on OCF resource model)
* "Message ID" is unique over a period of 300 seconds (no two messages
sent by a device over any 300 second window should have the same message ID)

The use of non-confirmable messages implies that devices using Nexus Channel
Core do not need to expect or handle "CON" or "ACK" CoAP message types.

In other words, most device exchanges in Nexus Channel Core look similar
to the following (example of a request to get battery charge percentage):

```
Client              Server
   |                  |
   |   NON [0x7a11]   |
   |   GET /batt      |
   |   (Token 0x74)   |
   +----------------->|
   |                  |
   |   NON [0x23bc]   |
   |   2.05 Content   |
   |   (Token 0x74)   |
   |    {'cp': 58}    |
   |<-----------------+
   |                  |
```

(Modified from [RFC 7252 Figure 6](https://tools.ietf.org/html/rfc7252))

Devices do, however, need to handle retries at the application layer, as there
is no required underlying retry logic specified by Nexus Channel Core
(Nexus Channel Core does not require constrained devices to implement the
CoAP `CON` message retry behavior).

## CoAP Header - Token

A 1-byte token must be present, and the value should be unique for any
currently pending message exchange between any two devices. That is, device
A and device B must not have two message exchanges (e.g. exchange 1 to endpoint
'/x/y' and exchange 2 to endpoint '/c/d') active at the same time with the
same token value.

## CoAP Header - Options

The following option type must be present:

* Uri-Path (Option Number 11). Typical Nexus Channel Core resource
URIs follow a pattern of xx(x)/yy(y)/(zzz), with xx(x) indicating the category
of resource,  yy(y) indicating the primary resource type, and (zzz) optionally
urther defining the resource. In this way, the URI may *roughly* mirror the
OCF resource definition 'type' (rt). However, this URI scheme is suggested and
not required.

The following option type *should* be present when sending content:

* Content-Format (Option Number 12) with a value of 10000, representing
`application/vnd.ocf+cbor` (all Nexus Channel Core compliant resources
must be instances of OCF compliant resource models).

The following option type *may* be present:

* Uri-Query (Option Number 15) representing an OCF query string, e.g.
`if=oic.if.baseline`. Although hosted resources do have valid OCF interface
types, this query parameter is unlikely to be supported on most Nexus Channel
Core devices.

## CoAP Message Types

Must support nonconfirmable ("NON") message requests and replies. No other
types are required (no CON, ACK, or RST).

## Message ID and Token Use

Implementations *should* use the message ID to deduplicate messages in the
presence of retries (two messages with the same message ID from a single
source do not both need to be processed). However, in implementations where
only a single request/response message exchange is active at any given time,
message ID may be ignored.

Implementations *should* use the token value to match a specific request with
a response (particularly in implementations where a client might make multiple
distinct requests to the same endpoint). However, in implementations where
only a single request/response message exchange is active at any given time,
the token value may be ignored.

## Message Size

The CoAP header plus any encapsulated payload *should* be <= 120 bytes in
length, to allow drop-in compatibility with link layers that expect the
transmitted packet to fit within 128 bytes after adding header and footer data).

## Link Layer: Device Addressing

No IP networking or internet access or assumed or implied for any devices.
Instead, Nexus Channel Core CoAP messages are exchanged over lower-level links
including but not limited to:

* Bluetooth Low Energy (BLE 5.0)
* I2C
* UART
* LIN-Based Buses
* 802.15.4 networks
* 2.4G Networks

The COAP header and CBOR payload contain no address information, however,
every Nexus Channel Core device has a "Nexus ID". This Nexus ID is a 6-byte
identifier which is globally unique. It consists of two parts:

* 4-byte 'device ID' unique for all IDs assigned from a given 'authority'
* 2-byte 'authority ID' indicating what party assigned the device ID

Every time a Nexus Channel Core message is sent or received between nodes,
the source and destination Nexus ID addresses are included. This is done by
middleware code that sits between the resource models and the link layer,
which is aware of the devices "Nexus ID", and adds this information before
passing the CoAP packet to the link layer. In many cases, this may be as
simple as hard-coding a constant (known at factory production time) in the
link layer as the 'Nexus ID source address' of the device, and letting
the application code 'fill in' a static 6-byte field with the destination
address.

The Nexus ID addresses might not be visible on the wire, as the link-layer
may abbreviate or elide them as long as they can be reconstructed in full by
the receiver's link layer before passing the message upward to the Nexus
Channel Core layer.

It is assumed that the lower-level links have a way to 'map' Nexus Channel Core
"Addresses" (6-byte Nexus IDs) to lower-level link node addresses if not
sending the full 6-byte source and destination addresses directly on
the wire. The details of this address bridging is not described in this
specification.

## Link Layer: Multicasting

Nexus Channel Core assumes that multicasting is possible, that is, that any
device may send a 'broadcast' that is received by all other devices on the
local network.

This broadcast may elicit zero or one responses from connected devices (implying
that this broadcast contains content that is understood by one specific
device).

A link layer *may* support broadcasting with responses from all recipients,
which can speed up resource discovery, but is not strictly required.

## Reference Implementation

The [nexus-embedded](https://github.com/angaza/nexus-embedded/tree/master/nexus)
implementation meets the above specifications, and assumes a compliant
underlying link layer is present.
