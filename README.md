# WASI Type Annotations

This document describes the `annotated` type, an anticipated feature of
Interface Types, and its use in WASI. Here's a quick example of what
`annotated` could be used for, in a simple [Logo]-like API:

```
/// Move the turtle forward `how_far` meters.
forward: function(
  turtle: turtle,
  how_far: annotated<f32, "wasi.dev", "unit:m">,
)

/// Turn the turtle left by `how_much`, an angle measured as a fraction of
/// a full circle, where `0.25` means turn left and `0.75` means (effectively)
/// turn right.
turn_left: function(
  turtle: turtle,
  how_much: annotated<f32, "wasi.dev", "unit:τ">,
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
`annotated<u32, "wasi.dev", "unit:m/s">`, with `m/s` indicating that the
value in meters per second. A file size parameter may have type
`annotated<u64, "wasi.dev", "unit:B">` indicating a size in bytes. An
email address parameter may have type
`annotated<string, "wasi.dev", "input:email">`. Or a non-WASI type might look
like `annotated<string, "example.com", "foo">`.

One use for these annotations is to guide user interfaces to components.
For example, in a GUI, a component containing a `input:phone` may pop up a
specialized phone-number entry widget, or a time:UTC` may pop up its date
picker. Or a command-line interface might allow users to type "4 GiB" when
providing a value for a "unit:B" parameter. However, note that the annotations
are independent of the actual interface.

Annotated types *do not perform validation*. For example, `input:email`
does not ensure that string values contain valid email addresses, or conform
to any grammar at all. WASI programs are encouraged to trap on encountering
inputs that are not valid according to their annotations rather than
employing explicit error handling. In many cases, User interfaces that use
annotations to provide specialized input methods will often have already
validated the inputs, providing user-friendly feedback before it gets passed
in, but this is not guaranteed.

### Names defined under the `wasi.dev` DNS name

| Interface Type | `type-name`    | Meaning                    |
| -------------- | -------------------- | -------------------------- |
| `string`       | `input:search`       | [Search]                   |
| `record`       | `input:color`        | [Color] represented as a record containing `red`, `green`, and `blue` numeric values |
| `string`       | `address:email`      | [E-mail Address]           |
| `string`       | `address:phone`      | [Phone number]             |
| `u8`           | `time:month`         | [Month] index, 1-based     |
| `u8`           | `time:week`          | [Week] index, 1-based      |
| `u8`           | `time:weekday`       | Weekday index, 1-based     |
| `u128`         | `time:UTC`           | UTC time as nanoseconds since 1970-01-01T00:00:00Z minus leap seconds |
| `u128`         | `time:TAI`           | TAI time as nanoseconds since 1970-01-01T00:00:00Z |
| `f*`           | `math:angle:radians` | Mathematical angle measure in radians |
| `f*`           | `math:probability`   | Mathematical probability   |
| `record`       | `math:complex`       | A complex number with `real` and `imag` fields |
| `string`       | `media:extension`    | Filename extension string, excluding the dot |
| `string`       | `media:media-type`   | A [Media Type] (aka MIME type) name |
| `*<u8>`        | `media:` + `<media type>`   | [Media Type] (aka MIME type) data |
| `f*`           | `unit:` + `<unit>`   | A quantity, described [below](#unit) |
| `u*`           | `currency:` + `<currency>` | A quantity of currency, described below [below](#currency) |
| `string`       | `language:language-tag`    | [IETF BCP 47 language tag]   |
| `bool`         | `schema:Boolean:` + `<property-name>`  | A [schema.org Boolean property name] |
| `n*`           | `schema:Number:` + `<property-name>`   | A [schema.org Number property name] |
| `u128`         | `schema:DateTime:UTC:` + `<property-name>` | A [schema.org DateTime property name], represented as UTC time nanoseconds since 1970-01-01T00:00:00Z minus leap seconds |
| `string`       | `schema:Text:` + `<property-name>`     | A [schema.org Text property name] |
| `record`       | `schema:Time:` + `<property-name>`     | A [schema.org Time property name], represented as a record containing the time of day with `seconds`, `minutes`, and `hours` |
| `record`       | `schema:Date:` + `<property-name>`     | A [schema.org Date property name], represented as a record containing a `year` number, a `month` index, and a `day` index |
| `string`       | `schema:DataType:` + `<property-name>` | A [schema.org DataType property name] |
| `record`       | `schema:Thing:` + `<thing-name>`       | A [schema.org Thing name], represented as a record with a subset of the fields in the schema |

Abbreviations:
 - `f*` refers to any floating-point type
 - `u*` refers to any unsigned integer type
 - `i*` refers to any signed or unsigned integer type
 - `n*` refers to any number type - signed or unsigned integer, or floating-point
 - `*<u8>` refers to any list or stream of `u8`

#### `<unit>`

The `<unit>` grammar is modeled after the syntax used in Wikipedia pages to
describe units and derived units, for example on the [SI derived unit] page.
For example, a newton metre second per kilogram is expressed as `N⋅m⋅s/kg`.

It's built on the [SI base units] (except with `g` instead of `kg` so that we
don't need to special-case it in the grammar) and the
[SI derived units with special names]. It also includes a few additional units
to address specific use cases, but in general it avoids customary units, to
promote interchange.

##### `<unit>` Grammar:

```ebnf
unit = unit , "⋅" , factor
     | unit , "/" , factor
     | factor
     ;

factor = base , [ exponent ] ;
base = literal
     | "(" , unit , ")"
     ;

exponent = [ "-" ] , exp_magnitude ;
exp_magnitude = exp_digit excluding zero , { exp_digit } ;
exp_digit excluding zero = "¹" | "²" | "³" | "⁴" | "⁵" | "⁶" | "⁷" | "⁸" | "⁹" ;
exp_digit = "⁰" | exp_digit excluding zero

literal = metric_literal | binary_literal | other_literal ;
metric_literal = metric_prefix , metric_symbol ;
binary_literal = binary_prefix , binary_symbol ;

; <https://en.wikipedia.org/wiki/Metric_prefix>
metric_prefix = big_metric_prefix | small_metric_prefix ;
big_metric_prefix = "da" | "h" | "k" | "M" | "G" | "T" | "P" | "E" | "Z" | "Y" ;
small_metric_prefix = "d" | "c" | "m" | "μ" | "n" | "p" | "f" | "a" | "z" | "y" ;

; <https://en.wikipedia.org/wiki/Binary_prefix>
binary_prefix = "Ki" | "Mi" | "Gi" | "Ti" | "Pi" | "Ei" | "Zi" | "Yi" ;

metric_symbol = simple_unit | named_unit ;

; <https://en.wikipedia.org/wiki/SI_base_unit>, but with g instead of kg to
; simplify the grammar.
simple_unit = "s" | "m" | "g" | "A" | "K" | "mol" | "cd" ;

; <https://en.wikipedia.org/wiki/SI_derived_unit#Derived_units_with_special_names>
named_unit = "Hz" | "rad" | "sr" | "N" | "Pa" | "J" | "W" | "C" | "V" | "F" | "Ω"
           | "S" | "Wb" | "T" | "H" | "°C" | "lm" | "lx" | "Bq" | "Gy" | "Sv" | "kat" ;

; <https://en.wikipedia.org/wiki/Byte>
binary_symbol = "B" ;

; <https://en.wikipedia.org/wiki/Turn_(angle)>, for representing dyadic ratios of
; plane angles without rounding.
other_literal = "τ" ;
```

#### `<currency>`

The `<currency>` must be a standard active alphabetic code from [ISO 4217].
Other letter sequences are reserved for future use. The integer value
represents a quantity of currency denominated in the [minor units] of the
specified currency.

#### Examples of wasi.dev types

The `unit:` annotations allow the use of units, which includes:
 - SI units, including base units such as `km`, `s`, `mg`, etc., as well as
   derived units such as `m/s`, `m/s²`, etc.

 - Byte sizes, using `B` to represent bytes, optionally with IEC binary
   prefixes to represent multiples of larger sizes, for example `MiB`, `GiB`,
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

An example of a function which defines a product offer:
```
/// Define an offer named `name`, with a price of `price` denominated in US
/// cents (hundreths of US dollars), with availability as described in
/// `availability`.
offer: function(
    name: annotated<string, "wasi.dev", "schema:Text:name">,
    price: annotated<annotated<u64, "wasi.dev", "currency:USD">, "wasi.dev", "schema:Text:price">,
    availability: annotated<string, "wasi.dev", "schema:Text:availability">,
)
```

[E-mail Address]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#e-mail_address_field
[Search]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#search_field
[Phone number]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#phone_number_field
[Month]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#month
[Time]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#time
[Week]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#week
[Color]: https://developer.mozilla.org/en-US/docs/Learn/Forms/HTML5_input_types#color_picker_control
[Media Type]: https://www.iana.org/assignments/media-types/media-types.xhtml
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
[SI base units]: https://en.wikipedia.org/wiki/SI_base_unit
[SI derived unit]: https://en.wikipedia.org/wiki/SI_derived_unit
[SI derived units with special names]: https://en.wikipedia.org/wiki/SI_derived_unit#Derived_units_with_special_names
[Byte]: https://en.wikipedia.org/wiki/Byte
[Turn]: https://en.wikipedia.org/wiki/Turn_(angle)
[ISO 4217]: https://en.wikipedia.org/wiki/ISO_4217
[minor units]: https://en.wikipedia.org/wiki/ISO_4217#Minor_units_of_currency
