## Nexus Channel Core - Resource Type Specification

Version: 0.9.0

### Purpose

This document outlines the subset of CBOR functionality that is required for
Nexus Channel Core devices to understand data payloads representing valid
resource models.

This document also outlines the structure and format of resource type
definitions.

From a developer's perspective, this document provides the information required
to implement (or select) a CBOR serializer or deserializer, as well as
design a Nexus Channel Core resource type/model that will be understood by
other devices.

### CBOR Restrictions

Nexus Channel Core resources *may* consist of the following [CBOR data types](https://tools.ietf.org/html/rfc7049#section-2.1):

* Unsigned Integer (CBOR Major Type 0)
* Negative Integer (CBOR Major Type 1)
* Byte String (CBOR Major Type 2)
* UTF-8 Text String (CBOR Major Type 3)
* Array of data items (CBOR Major Type 4)
* Map of pairs of data items (CBOR Major Type 5)
* Boolean True/False ([Major Type 7](https://tools.ietf.org/html/rfc7049#section-2.3), values 20-21)

Note that an array in Nexus Channel Core must contain only the following types,
and may not contain other nested arrays:

* Unsigned Integer
* Negative Integer
* Byte String
* UTF-8 Text String
* Boolean

This implies that a valid Nexus Channel Core CBOR parsing layer must be
capable of handling any of the above CBOR data types, and that no resource type
may specify a property that cannot be represented by one of the above CBOR data
types.

Floating-point numbers (IEEE 754) are not supported.

The `Content-Type` of any valid Nexus Channel Core resource when represented
as CBOR is always `application/vnd.ocf+cbor` (value 10000).

### Defining a New Resource: Resource Type (`rt` and `rte`)

All resources must have a "Resource Type" (`rt`) property, which follows the
format specified in [OCF Core Specification V2.1.1

Regardless of the URI that a resource instance is hosted on, the resource type
precises defines what properties that resource must expose, and how those
properties may be interacted with (RETRIEVE/GET and/or UPDATE/POST).

Additionally, once a resource is accepted to the [Nexus Channel Core resource
model registry](resource_type_registry.md)
it will be assigned an arbitrary "Resource Type Enumerator" ('rte') value,
an unsigned integer value that is unique to that resource. This is meant to
reduce the need to store static string "Resource Types" in devices with limited
memory.

Devices hosting a resource that has an assigned 'rte' *must* expose it as part
of the resource's properties on a "GET" request. Devices *may* also expose the
'rt' string value if available.

A resource must have exactly one Resource Type ('rt').

### Defining a New Resource: Properties

The properties of a resource define values that are read from or written
to the resource. Each property has a name, and may be 'required' by every
implementation of the resource, or may be 'optional'.

For example, the Nexus Channel Core [Battery resource](resource_types/core/101-battery/redoc_wrapper.md)
only *requires* the 'charge' (cg) percentage 0-100% to be implemented, but
defines a number of other optional properties to expose if available.

The OpenAPI specification document must include the following for each
property in a resource:

* Description (Text description of what this property represents)
* Type (integer, boolean, string, array)
* example (an 'example' valid value for this property)
* Format (integer width [e.g. `uint16`] or string type [e.g. `binary`])
* maxLength (for strings only)

For array properties, the following properties must additionally be defined:

* uniqueItems (true if every array item must be unique, false otherwise)
* minItems (minimum number of items in the array)

Any property *may* include the `readOnly` or `writeOnly` attribute if desired.

Other [OpenAPI](https://swagger.io/docs/specification/data-models/data-types/)
properties may be used to clarify the specification.

Typically, resource properties are defined as objects in the OpenAPI spec file
within the 'definitions' section, and referenced elsewhere in the specification
using the [`$ref`](https://swagger.io/docs/specification/using-ref/) tag.

### Defining a New Resource: Total Size

It is *recommended* that all Nexus Channel Core resources do not return more
than 100 bytes (in CBOR representation) when responding to a `GET` or `POST`
request, for maximum compatibility on networks with smaller packet transmission
size.

100 bytes allows room for lower-layer protocol overhead (headers, check fields),
the CoAP header, and addressing information (source, destination) while still
remaining below 127 bytes total, the maximum payload size of 802.15.4 networks.

To remain below this recommended limit, specifying short property names to aid
implementers (e.g. 'lv' instead of 'level') may be helpful, as long as the entire
unabbreviated property name is clearly indicated in the resource type specification.

### Defining a New Resource: Path

The path indicate thes recommended URI for the resource (e.g. '/my/resource/path')
and whether the resource supports GET (retrieve values), POST (update values),
or both. A given resource has exactly one path.

It is recommended to avoid URIs beginning with `nx/`, as these will overlap
with Nexus Channel security-specific resources.

However, as a device may host multiple instances of a specific resource type,
and there may be conflicting recommended resource URIs, there is no guarantee
that the recommended URI will be used on all implementing devices.

For both GET and POST requests, the possible responses (CoAP status code and
content) must be specified. For example, a simple read-only resource might
always return a status code '200' while including all property values in
the response body.

If the resource can return an error code (e.g. 405, 401, 400) the OpenAPI
spec file must indicate an description of what is required to succeed after
seeing this error.

The required payload must be specified for POST requests.

Note that a device might host multiple instances of a specific resource type,
so there is *no guarantee* that the recommended URI will be used for every
instance of a given resource in field devices.

Instead, devices rely on [resource discovery](#resource-discovery) to determine
what resources another device is hosting, and what URIs to use to access those
resource instances.

### Defining a New Resource: Resource Interface (`if`)

OCF specifies an interface property (`if`) which is required for every compliant
resource. This interface allows for different 'views' of the same resource, so
that a GET request might return different results if a different interface
string was provided in a CoAP query parameter.

However, Nexus Channel Core encourages use of a limited subset of
interfaces, and does not require implementations to support this query parameter
feature.

In other words, it is acceptable to define one 'interface' that is always used by
a resource type, and implement the resource such that no 'interface selection' by
query parameters is supported.

Nexus Channel Core requires that the `if` (interface) property is present
in the YAML *resource model*, and must specify at least one of the two interfaces below:

* `oic.if.rw` - Indicates that this resource is 'read-write', some data may be modified by another device
* `oic.if.r` - Indicates that this resource is 'read-only', no data may be modified by another device

If a resource type uses one of the above two interface types,
implementing devices may exclude the `if` property from the on-device instance
of the resource.

If a resource type does *not* use one of the above two interface types,
implementing devices must always include the `if` property from the on-device
instance of the resource.

Other OCF interface types, like `oic.if.baseline`, *may* be specified, but there
is no guarantee that other Nexus Channel Core devices will understand or honor
that interface, and one of the above 'primary' interface types must also
be defined.

### Resource Discovery

**Note**: This section describes specified functionality that is not yet
available in the reference implementation, but is committed for a future release.

A device may discover resources on another device by making a CoAP GET request
to the special URI `nx/res`, which must elicit a response with an array, one
element for each resource hosted by the device. Each element in the array
is a map containing the following keys:

* `rtr` - Integer value representing a 'resource type'
* `href` - Text string indicating the URI where this resource is accessed

Each resource entry in the array *may* also contain:

* `if` - if the resource supports more than one interface, or if the resource
uses an interface other than `oic.if.r` or `oic.if.rw`.
* `rt` - if the device is able to store the full `rt` instead of only the `rtr`.

No other properties/keys may be present.

An example response in human-readable CBOR diagnostic format is:

```
[
  {"href": "/batt1", "rtr": 763},
  {"href": "/batt2", "rtr": 763},
  {"href": "/tamper", "rtr": 628},
  {"href": "/grinder", "rtr": 902, "rt": "custom.com.my.custom.grinder", "if": ["oic.if.rw", "oic.if.baseline"]}
]
```

The above response indicates to the requester that the device is hosting
four total resources. There are two resources of rtr type 763 at URIs `batt1`
and `batt2` (likely a system with multiple batteries being monitored
separately), a resource of rtr type 628 at URI `tamper`, and a resource of
type `custom.com.my.custom.grinder` (rtr 902) at URI `grinder`, which also
supports OIC `interface` queries (an optional OCF feature not required by
Nexus Channel Core devices).

A typical use of the `nx/res` discovery resource is:

1. New device enters the local network
2. New device determines what other nodes are present (using lower layer protocol)
3. New device sends CoAP `GET` to `nx/res` at each other node (multicast or unicast)
4. Each other node sends a reply, indicating their resources to the new device
5. New device determines whether to further interact with any other device resource based on application logic.

The `nx/res` resource is functionally similar to to the OCF standard `oic/res`
discovery resource, except providing a minimal subset of information.

Nexus Channel Core devices are not required to implement the `oic/res` resource.

### Reference: Nexus Channel Core Resource Type Registry

A complete registry of all Nexus Channel Core resource types (and a mapping to
the associated `rtr` value and specification) can be found [here](resource_type_registry.md).

The 'Resource Type Registry' is a list of validated, publicly available
resource type definitions to promote interoperability between Nexus Channel
Core devices.

Anyone may contribute new resource types to this repository, as long as they meet
the requirements listed in this document, and are not 'functional duplicates' of
existing resources (in which case, proposing updates to the existing resource
may be more appropriate).

Contributing a resource type makes it openly, publicly available for other
parties to use. Please see the [Contributor License Agreement](https://github.com/angaza/nexus-embedded/blob/master/CONTRIBUTING.md) for more details.

### Example: Compliant CBOR Payload

Below is an example of the CBOR payload received in response to a CoAP
"GET" request to a compliant Nexus Channel Core resource.

This 99-byte response is *larger* than a typical resource might provide,
but is used to illustrate a variety of supported data types and format.

```
0xbf, 0x62, 0x72, 0x74, 0x9f, 0x78, 0x18, 0x61, 0x6e, 0x67, 0x61, 0x7a,
0x61, 0x2e, 0x63, 0x6f, 0x6d, 0x2e, 0x6e, 0x65, 0x78, 0x75, 0x73, 0x2e,
0x6c, 0x69, 0x6e, 0x6b, 0x2e, 0x68, 0x73, 0xff, 0x63, 0x72, 0x74, 0x65,
0x19, 0x02, 0x0a, 0x62, 0x69, 0x66, 0x9f, 0x69, 0x6f, 0x69, 0x63, 0x2e,
0x69, 0x66, 0x2e, 0x72, 0x77, 0xff, 0x62, 0x63, 0x44, 0x50, 0x40, 0xe2,
0x01, 0x00, 0x40, 0xe2, 0x01, 0x00, 0x8d, 0xd0, 0x80, 0xd0, 0x8e, 0x18,
0x38, 0xc4, 0x62, 0x72, 0x44, 0x40, 0x62, 0x74, 0x49, 0x00, 0x62, 0x74,
0x54, 0x19, 0x01, 0x2c, 0x62, 0x73, 0x4c, 0x81, 0x00, 0x62, 0x73, 0x43,
0x81, 0x00, 0xff
```

The above list of bytes can be pasted into [cbor.me](http://cbor.me/), which
will display the human-readable representation. Below, we will also step
through the meaning of each portion of this payload.

First, the payload begins with `0xbf`, opening an indefinite map (a list of
key/value pairs). Every OCF resource consists of a map of key/value pairs (one
for each 'property' that the resource contains).

#### Common Property: `rt`

`627274` indicates the first key, a property named `rt`. This is the
'resource type' of this resource.  

`9F7818616E67617A612E636F6D2E6E657875732E6C696E6B2E6873FF` is the value
associated with this key. It is an array containing one element, the
text string "angaza.com.nexus.link.hs". (If present, the `rt` field is
contained within an array to remain compatible with the OCF specifications).

All resources of rt "angaza.com.nexus.link.hs" must provide a
[defined set of properties](https://github.com/angaza/nexus-embedded/blob/master/ocf_resource_models/NexusChannelLinkHandshakeResURI.swagger.yaml),
which we should expect to see as we continue decoding this payload. This
particular resource is used to enable Nexus Channel linking features (on
devices which support Nexus Channel secure links, e.g. devices that have this
resource).

#### Common Property: `rtr`

`63727465` is the key for property `rtr`. This is the 'resource type registry'
value of this resource. It is an integer assigned to the specific `rt` for
this resource. In most cases, devices will only expose the `rtr` property (and
not the `rt` string property), for a given resource.

`19020A` is the value. This decodes to integer value '522'.

#### Common Property: `if`

`626966` is the key for property `if`. This is the OCF-mandated array of
supported interfaces for this resource.

`9F696F69632E69662E7277FF` is the value, which is an array containing the
a single element - the text string `oic.if.rw`. This indicates that the
resource has properties that are writeable.

#### Property `cD`

`626344` is the key for property `cD`, the 'challenge data'
used during link handshakes.

`5040E2010040E201008DD080D08E1838C4` is the value. `50` begins
a 16-byte long bytestring, and the remainder bytes are the bytestring itself.

(as a string: "@\xE2\x01\x00@\xE2\x01\x00\x8D\xD0\x80\xD0\x8E\x188\xC4")

#### Property `rD`

`627244` is the key for property `rD`, the 'response data'
used during link handshakes.

`40` is the value, indicating an empty bytestring.

#### Property `tI`

`627449` is the key for property `tI`, the time since the handshake started.

`00` is the value, an integer 0.

#### Property `tT`

`627454` is the key for property `tT`, the fixed timeout value for handshakes.

`19012C` is the value, an integer 300.

#### Property `sL`

`62734C` is the key for property `sL`, the supported Nexus Channel link security modes.

`8100` is an array containing one value, 0. Note that in CBOR, short arrays (up
to 23 elements long) carry very little overhead (`81` indicates 'an array
with 1 data item', and the only remaining information is the single value in
this array).

#### Property `sC`

`727343` is the key for property `sC`, the supported Nexus Channel link handshake modes.

`8100` is an array containing one value, 0.

The following `FF` terminates the indefinite map that was opened with the very
first byte, `BF`, and ends the payload.
