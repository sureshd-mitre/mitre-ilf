# OCSF Protobuf Encoding

This directory contains a sample Protobuf encoding for OCSF.

Although encoding in general is _informative_ in OCSF, enabling the
interoperation between systems in a single organization and indeed
between organizations indicates that a **canonical** protobuf encoding
has significant utility.  This directory contains that canonical
encoding.

## Semantic Type Considerations

### Type Mappings

Types are mapped as follows:

| OCSF Type    | Protobuf Type               |
| ------------ | --------------------------- |
| `boolean_t`  | `bool`                      |
| `float_t`    | `double`                    |
| `integer_t`  | `int32`                     |
| `long_t`     | `int64`                     |
| `string_t`   | `string`                    |
| `json_t`     | `google.protobuf.Value`     |
| `datetime_t` | `google.protobuf.Timestamp` |

Types not mentioned above are mapped to the protobuf type of their
underlying type.

This mapping is straightforward, with the following appendices:

`json_t` is represented by the well-known protobuf type that models a
JSON value.  Note that JSON values include both primitives (e.g.,
strings and numbers) as well as objects and arrays.

`datetime_t` type is mapped to the well-known
[Timestamp](https://protobuf.dev/reference/protobuf/google.protobuf/#timestamp)
protobuf object to capture the intended semantics.

Note that the Protobuf JSON encoding for the well-known `Timestamp`
matches the OCSF JSON representation.

### Optionality

Most attributes in OCSF are not required; for non-array primitive
types, the corresponding protobuf descriptor will mark them explicity
as `optional`.  This has implications for most SDKs generated by
`protoc`.  For example, in Go, the field of the struct becomes of
pointer type, adding an extra indirection with the benefit that the
presence or absence of the field can be consistently determined.

Required attributes in OCSF are not marked as optional.  In most
cases, an SDK will still permit distinguishing between present and
absent values because the zero value is not a valid value anyway.  For
example, `class_id` is required and non-zero; if the application finds
it to have a zero value, the distinction between missing and invalid
is not significant because missing _is_ invalid for required types.

(Note that OCSF's JSON encoding does not distinguish between `null`
and missing values, since null is not a type in OCSF.)

### Structured Types

Structured types in OCSF include objects, events (or classes), and
arrays.

### Object and Event Types

Objects and Events are mapped to Protobuf `message` objects in the
straightforward way, preserving the OCSF name of the field (e.g.,
`status_id` instead of the more conventional Protobuf `statusId`).
This is so that the JSON encoding (with the option **Use proto field
name instead of lowerCamelCase name** enabled) matches the OCSF JSON
encoding.

### The Un-extended Object

The one exception to how objects are mapped, is the _unextended_
`object`.  This is used in a few places in the schema (such as the
`unmapped` attribute) to represent a schema-free JSON object.  For the
Protobuf encoding, `object` is mapped to the well-known
[Struct](https://protobuf.dev/reference/protobuf/google.protobuf/#struct)
type.

This has the desirable property that the native protobuf JSON encoding
knows how to serialize and deserialize these types into plain JSON.

Note that primitive JSON values such as string and number, and JSON
arrays, are **NOT** valid values for an `object`.

#### Field IDs

The Protobuf wire encoding requires consistent field ids for message
objects.  To facilitate interoperation and schema evolution, a
separate control file (the canonical version of which is included in
this repo in `control.json`) is used "remember" the field ids assigned
to fields in messages.

### Enumerated Values

Enumerated **integer** values are built using protobuf's `enum`
declaration.  Because OCSF often (not always) refines the set of
enumerands on a per-event basis, the protobuf encoding supports that
by making the enumeration types sub-names of the message types.
Furthermore, because Protobuf uses "C++" scoping rules, in which
enumerated values are in the same scope as the enum type itself
(rather than children of the type), we mangle the enumerand names as
well.

For example:
```
message FileActivity {
    enum ActionId {
        ACTION_ID_UNKNOWN = 0;  // The action was unknown. The <code>disposition_id</code>
                                // attribute may still be set to a non-unknown value, for
                                // example 'Count', 'Uncorrected', 'Isolated',
                                // 'Quarantined' or 'Exonerated'.
        ACTION_ID_ALLOWED = 1;  // The activity was allowed. The
                                // <code>disposition_id</code> attribute should be set to
                                // a value that conforms to this action, for example
                                // 'Allowed', 'Approved', 'Delayed', 'No Action', 'Count'
                                // etc.
        ACTION_ID_DENIED  = 2;  // The attempted activity was denied. The
                                // <code>disposition_id</code> attribute should be set to
                                // a value that conforms to this action, for example
                                // 'Blocked', 'Rejected', 'Quarantined', 'Isolated',
                                // 'Dropped', 'Access Revoked, etc.
        ACTION_ID_OTHER   = 99; // The action was not mapped. See the <code>action</code>
                                // attribute, which contains a data source specific value.
    }

    enum ActivityId {
        ACTIVITY_ID_UNKNOWN        = 0;
        ACTIVITY_ID_CREATE         = 1;  // A request to create a new file on a file
                                         // system.
        ACTIVITY_ID_READ           = 2;  // A request to read data from a file on a file
                                         // system.
        ... etc ...
    }
    ...
    ActionId action_id = 3;
    ActivityId activity_id = 4;
    ...
}
```

The protobuf encoding preserves the numerical value of the enumerand.
This is mostly for consistency with OCSF, but also helps with JSON
Encoding interoperability (see below).

However, not every enumeration in OCSF includes a 0 value, which is
required in proto3, so in cases where the 0 value is missing, an
`UNKNOWN` enumerand with the 0 value is inserted.

No special treatment is given for enumerated **string** values in
OCSF, they are just treated as strings by the protobuf encoding.

## Protobuf JSON Encoding

This encoding concerns itself primarily with the protobuf descriptor
files (`.proto`) and wire representation.

Protobuf defines a JSON encoding as well, but that JSON encoding can
be tricky to line up (see notes on 64-bit integers for example).
Thus, we do not specifically claim interoperability between
JSON-encoded protobof and the OCSF JSON encoding, since that is
achievable only using non-standard protobuf mods .  However, despite
that, this encoding strives to preserve compability where possible
(_e.g._, in field names)

In addition, by default the protobuf JSON encoding uses strings to
encode enumerations and converts to `lowerCamelCase` identifiers,
both of which are inconsistent with OCSF JSON encoding.

To obtain the closest representation of OCSF JSON, disable these
features.  See the [Protobuf programming
guide](https://protobuf.dev/programming-guides/proto3/#json) section
on JSON mapping for notes, specifically **Emit enum values as integers
instead of strings** and **Use proto field name instead of
lowerCamelCase name**.

### SDK specific notes

Protobuf's JSON encoding represents `int64` and `uint64` as strings by
default (per the [I-JSON
recommendation](https://datatracker.ietf.org/doc/html/rfc7493#section-2.2)).
Some SDKs have workarounds or configurations that support unwrapping,
which are described here to facilitated use cases where
interoperability between Protobuf JSON and OCSF JSON is important.

#### Go

In the protojson encoder for Go,
`google.golang.org/protobuf/encoding/protojson`, this can be
accomplished using non-default marshal options like so:

```
import "google.golang.org/protobuf/encoding/protojson"

func EncodeToJSON(item *proto.FileActivity) ([]byte, error) {

	// Note that the default behavior is to use strings for enums,
	// and to force lowerCamelCase, neither of which is what we
	// want for OCSF

	opt := protojson.MarshalOptions{
		UseEnumNumbers: true, // otherwise it will use strings
		UseProtoNames:  true, // otherwise it will try to force lowerCamelCase
	}
	return opt.Marshal(&item)
}
```

#### C++

In **C++**, a configuration option supports bare-JSON 64-bit integers.
See [this
commit](https://github.com/protocolbuffers/protobuf/commit/330e10d53fe1c12757f1cdd7293d0881eac4d01e).

#### Java

In **Java**, the key functionality for ProtoJSON encoding is in the utils
package.  This package could be forked without forking the core
Protobuf library itself, and [this
line](https://github.com/protocolbuffers/protobuf/blob/8434c12d160fcf2f6adc572f4e94947fb57c82c3/java/util/src/main/java/com/google/protobuf/util/JsonFormat.java#L1149)
changed appropriately.