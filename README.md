# WASI Type Annotations

This document describes *Type Annotations*, an anticipated feature of
Interface Types, and their use in WASI. Here's a quick example of what
Type Annotations could be used for, in a simple [Logo]-like API:

```
/// Move the turtle forward `how_far` meters.
forward: function(
  turtle: turtle,
  how_far: annotated<f32, "wasi.dev", "units:m">,
)

/// Turn the turtle left by `how_much`, an angle measured in fractions of
/// a full circle.
turn_left: function(
  turtle: turtle,
  how_much: annotated<f32, "wasi.dev", "units:circ">,
)

/// Set the pen color of the turtle to `color`, which is record containing
/// red, green, and blue fields.
pen_color: function(
  turtle: turtle,
  color: annotated<rgb, "wasi.dev", "input:color">,
)
```

## What does `annotated` mean?

Interface types is expected to define a type `annotated<T, dns-name, type-name>`.
The parameters mean:

 - `T`: the type that `annotated<T, ...>` is semantically equivalent to in wasm.

 - `dns-name`: A string containing a fully-qualified DNS name. This isn't
    resolved or interpreted here, and just serves as a namespace qualifier to
    discourage collisions. Developers defining new annotations should use the
    name of a DNS name they are associated with. Official WASI types use the
    dns-name "wasi.dev".

 - `type-name`: A string annotation. This string is also uninterpreted, but it
   should ideally describe the purpose of the type.

For example, a parameter interpreted as a velocity value may have type
`annotated<u32, "wasi.dev", "units:m/s">`, with `m/s` indicating that the
value in meters per second. A file size parameter may have type
`annotated<u64, "wasi.dev", "units:By">` indicating a size in bytes. An
email address parameter may have type
`annotated<string, "wasi.dev", "input:email">`. Or a non-WASI type might look
like `annotated<string, "example.com", "foo">`.

One use for these annotations is to guide user interfaces to components.
For example, in a GUI, a component containing a `input:phone` may pop up a
specialized phone-number entry widget, or a time:UTC` may pop up its date
picker. However, note that the annotations are independent of the actual
interface.

Annotated types *do not perform validation*. For example, `input:email`
does not ensure that string values contain valid email addresses, or conform
to any grammar at all. WASI programs are encouraged to trap on encountering
inputs that are not valid according to their annotations rather than
employing explicit error handling. In many cases, User interfaces that use
annotations to provide specialized input methods will often have already
validated the inputs, providing user-friendly feedback before it gets passed
in, but this is not guaranteed.

### Names defined under the `wasi.dev` DNS name

| Type     | Name                 | Description                |
| -------- | -------------------- | -------------------------- |
| `string` | `input:search`       | [Search]                   |
| `record` | `input:color`        | [Color] represented as a record containing `red`, `green`, and `blue` numeric values |
| `string` | `address:email`      | [E-mail Address]           |
| `string` | `address:phone`      | [Phone number]             |
| `u8`     | `time:month`         | [Month] index, 1-based     |
| `u8`     | `time:week`          | [Week] index, 1-based      |
| `u8`     | `time:weekday`       | Weekday index, 1-based     |
| `u128`   | `time:UTC` | UTC time as nanoseconds since 1970-01-01T00:00:00Z minus leap seconds |
| `u128`   | `time:TAI` | TAI time as nanoseconds since 1970-01-01T00:00:00Z |
| `i*`     | `math:angle:turns` | Mathematical angle measure in turns divided by the number of representable values |
| `f*`     | `math:angle:radians` | Mathematical angle measure in radians |
| `f*`     | `math:probability`   | Mathematical probability   |
| `record` | `math:complex` | A complex number with `real` and `imag` fields |
| `string` | `media:media-type`   | A [Media Type] (aka MIME type) name |
| `*<u8>`  | `media:` + `<media type>` | [Media Type] (aka MIME type) data |
| `string` | `media:` + `<extension>` | Filename extension string, excluding the dot |
| `f*`     | `units:` + `<c/s ucum>` | A [UCUM] type specified in case-sensitive ("c/s") format, excluding [Customary units] and [Legacy units] |
| `u*`     | `currency:` + `<sign>` + `/` + `<denominator>` | A quantity of currency denominated according to `<sign>`, for example `$` or a USV in [U+20A0–U+20CF], divided by `<denominator>`.
| `string` | `locale:language` | [IETF BCP 47 language tag]   |
| `bool`   | `schema:Boolean:` + `<property-name>` | A [schema.org Boolean property name] |
| `n*`     | `schema:Number:` + `<property-name>` | A [schema.org Number property name] |
| `u128`   | `schema:DateTime:UTC:` + `<property-name>` | A [schema.org DateTime property name], represented as UTC time nanoseconds since 1970-01-01T00:00:00Z minus leap seconds |
| `string` | `schema:Text:` + `<property-name>` | A [schema.org Text property name] |
| `record` | `schema:Time:` + `<property-name>` | A [schema.org Time property name], represented as a record containing the time of day with `seconds`, `minutes`, and `hours` |
| `record` | `schema:Date:` + `<property-name>` | A [schema.org Date property name], represented as a record containing a `year` number, a `month` index, and a `day` index |
| `string` | `schema:DataType:` + `<property-name>` | A [schema.org DataType property name] |
| `record` | `schema:Thing:` + `<thing-name>` | A [schema.org Thing name], represented as a record with a subset of the fields in the schema |

Abbreviations:
 - `f*` refers to any floating-point type
 - `u*` refers to any unsigned integer type
 - `i*` refers to any signed or unsigned integer type
 - `n*` refers to any number type - signed or unsigned integer, or floating-point
 - `*<u8>` refers to any list or stream of `u8`

## Examples of wasi.dev types

The `units:` annotations allow the use of [UCUM] units, which includes:
 - SI units, including base units such as `m`, `s`, `kg`, etc., as well as
   prefixed and derived units such as `cm`, `ns`, `ug`, `m/s`, `m/s2`
   (meters per second per second), etc.

 - Byte sizes, using `By` to represent bytes, optionally with IEC binary
   prefixes to represent multiples of larger sizes, for example `MiBy`, `GiBy`,
   etc.

The `schema:` annotations allow the use of [schema.org] property names, which
include many commonly used fields, including:

 - [`arrivalTime`], [`checkinTime`], [`endTime`]
 - [`longitude`], [`latitude`],
 - [`reviewRating`], [`reviewBody`]

and many more, as well as records containing property fields, including:

 - Creative works: [`CreativeWork`], [`Book`], [`Movie`], [`MusicRecording`], [`Recipe`], [`TVSeries`]
 - Embedded non-text objects: [`AudioObject`], [`ImageObject`], [`VideoObject`]
 - [`Event`]
 - [`Organization`]
 - [`Person`]
 - [`Place`], [`LocalBusiness`], [`Restaurant`]
 - [`Product`], [`Offer`], [`AggregateOffer`]
 - [`Review`], [`AggregateRating`]

and many more.

[E-mail Address]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#e-mail_address_field
[Search]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#search_field
[Phone number]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#phone_number_field
[Month]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#month
[Time]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#time
[Week]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#week
[Color]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#color_picker_control
[Media Type]: https://www.iana.org/assignments/media-types/media-types.xhtml
[UCUM]: https://ucum.org/ucum.html
[U+20A0–U+20CF]: https://www.unicode.org/charts/PDF/U20A0.pdf
[IETF BCP 47 language tag]: https://www.rfc-editor.org/info/bcp47
[Logo]: https://en.wikipedia.org/wiki/Logo_(programming_language)
[schema.org Boolean property name]: https://schema.org/Boolean
[schema.org Number property name]: https://schema.org/Number
[schema.org DateTime property name]: https://schema.org/DateTime
[schema.org Text property name]: https://schema.org/Text
[schema.org Time property name]: https://schema.org/Time
[schema.org Date property name]: https://schema.org/Date
[schema.org DataType property name]: https://schema.org/DataType
[schema.org Thing name]: https://schema.org/Thing
[Customary units]: https://ucum.org/ucum.html#section-Customary-Unit-Atoms
[Legacy units]: https://ucum.org/ucum.html#section-Other-Legacy-Units
[schema.org]: https://schema.org
[`arrivalTime`]: https://schema.org/arrivalTime
[`checkinTime`]: https://schema.org/checkinTime
[`endTime`]: https://schema.org/endTime
[`longitude`]: https://schema.org/longitude
[`latitude`]: https://schema.org/latitude
[`CreativeWork`]: https://schema.org/CreativeWork
[`Book`]: https://schema.org/Book
[`Movie`]: https://schema.org/Movie
[`MusicRecording`]: https://schema.org/MusicRecording
[`Recipe`]: https://schema.org/Recipe
[`TVSeries`]: https://schema.org/TVSeries
[`AudioObject`]: https://schema.org/AudioObject
[`ImageObject`]: https://schema.org/ImageObject
[`VideoObject`]: https://schema.org/VideoObject
[`Event`]: https://schema.org/Event
[`Organization`]: https://schema.org/Organization
[`Person`]: https://schema.org/Person
[`Place`]: https://schema.org/Place
[`LocalBusiness`]: https://schema.org/LocalBusiness
[`Restaurant`]: https://schema.org/Restaurant
[`Product`]: https://schema.org/Product
[`Offer`]: https://schema.org/Offer
[`AggregateOffer`]: https://schema.org/AggregateOffer
[`Review`]: https://schema.org/Review
[`AggregateRating`]: https://schema.org/AggregateRating
[`reviewRating`]: https://schema.org/reviewRating
[`reviewBody`]: https://schema.org/reviewBody
