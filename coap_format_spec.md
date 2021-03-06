# Nexus Channel Core - CoAP Specification

Version: 0.9.1

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

The following option type must be present in all requests:

* Uri-Path (Option Number 11). This option will be present one time for each
portion of the `uri-path`. For example, a resource located at URI `batt` would
have one `uri-path` option in the GET/POST request, but a resource located at
URI `batt/secondary` would have two `uri-path` options (one for `batt`, one
for `secondary`).
Typical Nexus Channel Core resource URIs follow a pattern of xx(x)/yy(y)/(zzz),
with xx(x) indicating the category of resource,  yy(y) indicating the primary
resource type, and (zzz) optionally further defining the resource instance.

The following option type *should* be present in all responses:

* Content-Format (Option Number 12) with a value of 10000, representing
`application/vnd.ocf+cbor` (all Nexus Channel Core compliant resources
must be instances of OCF compliant resource models).

The following option type *may* be present in requests:

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

No IP networking or internet access is assumed or implied for any devices.
Instead, Nexus Channel Core CoAP messages are exchanged over lower-level links
including but not limited to:

* OpenPAYGOLink
* Bluetooth Low Energy (BLE 5.0)
* I2C
* UART
* LIN-Based Buses
* 802.15.4 networks
* 2.4G Networks

The COAP header and CBOR payload contain no address information, however,
every Nexus Channel Core device has a "Nexus ID". This Nexus ID is a 6-byte
identifier which is globally unique and that supports multicast and limited-scope
(such as link-local) address assignment. Read more [here](nexus_id_spec.md).

## Reference Implementation

The [nexus-embedded](https://github.com/angaza/nexus-embedded/tree/master/nexus)
implementation meets the above specifications, and assumes a compliant
underlying link layer is present.

### Appendix: Example Compliant CoAP Headers

1. `GET` to `/batt` resource URI:

10 bytes, as below:

`51 01 00 04 8a b4 62 61 74 74`

* `51 01 00 04` - Base CoAP header, with message ID 4 and token length 1.
* `8a` - Token value
* `b4` - Option delta 'b' (11, `uri-path`), length 4
* `62 61 74 74` - the value for `uri-path`, which is 'batt' in ASCII

2. `GET` to `/batt/main` resource URI (demonstrating longer URI paths):

15 bytes, as below:

`51 01 00 01 c1 b4 62 61  74 74 04 6d 61 69 6e`

* `51 01 00 01` - Base CoAP header, with message ID 1 and token length 1.
* `c1` - Token value
* `b4` - Option delta '0xb' (11, `uri-path`), length 4
* `62 61 74 74` - the value for `uri-path`, which is 'batt' in ASCII
* `04` - Option delta '0x0` (0, still `uri-path`), length 4
* `6d 61 69 6e` - the value for `uri-path, which is `main` in ASCII
