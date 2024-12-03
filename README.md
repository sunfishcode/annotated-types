# WASI Type Annotations

This document describes the `annotated` type, a feature that I may propose to
be added to Wasm Components in the future. Here's a quick example of what
`annotated` could be used for, in a simple [Logo]-like API:

```
resource turtle {
    /// Move the turtle forward `how_far` centimeters.
    forward: function(
        how_far: annotated<f32, "unit:cm">,
    );

    /// Turn the turtle left by `how_much`, an angle measured as a fraction of
    /// a full circle. For example, `0.25` means turn to face to the left, `0.5`
    /// means turn to face backward, and `-0.25` (or `0.75`) means turn to the
    /// right.
    turn_left: function(
        how_much: annotated<f32, "math:angle:τ">,
    );

    /// Set the pen color of the turtle to `color`, which is record containing
    /// red, green, and blue fields.
    pen_color: function(
        color: annotated<rgb, "input:color">,
    );
}
```

If this turns out to be useful, one could also imagine adding dedicated syntax,
which could perhaps look like this:
```
  how_far: f32 @ unit:cm,
```
Or maybe something completely different. Who knows!

## What does `annotated` mean?

In the component model, a type `annotated<T, type-name>` would be semantically
equivalent to type `T`. It would have `type-name` as an associated uninterpreted
string that describes additional interpretation or intent.

For example, a parameter that conveys a velocity quantity may have type
`annotated<u32, "unit:m/s">`, with `m/s` indicating that the
value in meters per second. An email address parameter may have type
`annotated<string, "input:email">`.

One use for these annotations is to guide user interfaces. For example, in a
GUI, a component requiring a `input:phone` parameter might pop up a specialized
phone-number entry widget, or a time:UTC` may pop up its date picker. Or a
command-line interface might allow users to type "4 GiB" when providing a value
for a "unit:B" parameter.

Annotated types *do not perform validation*. For example, `input:email`
does not ensure that string values contain valid email addresses.

### Names:

| `type-name`          | Interface Type  | Meaning                    |
| -------------------- | --------------- | -------------------------- |
| `input:search`       | `string`        | [Search]                   |
| `input:color`        | `record`        | [Color] represented as a record containing `red`, `green`, and `blue` numeric values |
| `address:email`      | `string`        | [E-mail Address]           |
| `address:phone`      | `string`        | [Phone number]             |
| `time:month`         | `u8`            | [Month] index, 1-based     |
| `time:week`          | `u8`            | [Week] index, 1-based      |
| `time:weekday`       | `u8`            | Weekday index, 1-based     |
| `time:UTC`           | `i128`          | UTC time as nanoseconds since 1970-01-01T00:00:00Z minus leap seconds |
| `time:TAI`           | `i128`          | TAI time as nanoseconds since 1970-01-01T00:00:00Z |
| `math:angle:τ`       | `f*`            | Mathematical angle measure in [turns] |
| `math:angle:radians` | `f*`            | Mathematical angle measure in radians |
| `math:probability`   | `f*`            | Mathematical probability; a value in `[0.0,1.0]` |
| `math:complex`       | `record { f*, f* }` | A complex number with `real` and `imag` fields |
| `media:extension`    | `string`        | Filename extension string, excluding the dot |
| `media:media-type`   | `string`        | A [Media Type] (aka MIME type) name |
| `media:` + `<media type>`  | `*<u8>`   | [Media Type] (aka MIME type) data |
| `unit:` + `<unit>`   | `n*`            | A quantity, described [below](#unit) |
| `currency:` + `<currency>` | `u*`      | A quantity of currency, described below [below](#currency) |
| `language:language-tag`    | `string`  | [IETF BCP 47 language tag]   |

Abbreviations:
 - `f*` refers to any floating-point type
 - `u*` refers to any unsigned integer type
 - `s*` refers to any signed integer type
 - `i*` refers to any signed or unsigned integer type
 - `n*` refers to any number type - signed or unsigned integer, or floating-point
 - `*<u8>` refers to any list or stream of `u8`

#### `<unit>`

The `<unit>` grammar is modeled after the syntax used in the [SI Brochure]
and Wikipedia pages to describe units and derived units, for example on the
[SI derived unit] page. For example, a newton metre second per kilogram is
expressed as `N⋅m⋅s/kg`.

It's built on the [SI base units] (except with `g` instead of `kg` so that we
don't need to special-case it in the grammar) and the
[SI derived units with special names]. It also includes a few additional units
to address specific use cases, but in general it avoids customary units, to
promote interchange.

TODO: The [SI Brochure] recommends against having two `/` signs not isolated
from each other with parens.

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
```

#### `<currency>`

The `<currency>` must be a standard active alphabetic code from [ISO 4217].
Other letter sequences are reserved for future use. The integer value
represents a quantity of currency denominated in the [minor units] of the
specified currency.

#### Examples

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
    name: annotated<string, "schema:Text:name">,
    price: annotated<annotated<u64, "currency:USD">, "schema:Text:price">,
    availability: annotated<string, "schema:Text:availability">,
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
[SI Brochure]: https://www.bipm.org/en/publications/si-brochure/
[Byte]: https://en.wikipedia.org/wiki/Byte
[Turn]: https://en.wikipedia.org/wiki/Turn_(angle)
[ISO 4217]: https://en.wikipedia.org/wiki/ISO_4217
[minor units]: https://en.wikipedia.org/wiki/ISO_4217#Minor_units_of_currency
[turns]: https://en.wikipedia.org/wiki/Turn_(angle)
