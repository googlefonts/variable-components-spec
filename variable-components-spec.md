# Variable Components

A proposal for an add-on to OpenType 1.8 by Black[Foundry]

*(Work in progress!)*

### Authors:
* Just van Rossum ([@justvanrossum](https://github.com/justvanrossum))
* Jérémie Hornus ([@JeremieHornus](https://github.com/JeremieHornus))

## Table of Contents

- [What are Variable Components?](#what-are-variable-components)
  - [Use cases](#use-cases)
- [Proposal overview](#proposal-overview)
  - [Extend the OpenType format](#extend-the-OpenType-format)
- [Format overview](#format-overview)
  - [The Component data structure](#the-component-data-structure)
    - [Transformation](#transformation)
- [How to process VarC data?](#how-to-process-varc-data)
- [Format details](#format-details)
  - [Notes on precision](#notes-on-precision)
  - [Field formats](#field-formats)
- [New axis flag for the vartable](#new-axis-flag-for-the-fvar-table)
- [Notes on non-linear interpolation](#notes-on-non-linear-interpolation)

# What are Variable Components?

TrueType has had the capability to form composite glyphs since its inception. A
composite glyph references other glyphs as “components”. This is a simple way to
save data storage, for glyphs that can be composed of other glyphs, such as
diacritics. Components can be arbitrarily positioned in the composite glyph, and
can be scaled and rotated and skewed if needed.

“Variable Components”, as described in this document, add further parameters to
customize the appearance of components in the composite glyph.

With OpenType 1.8, “variations” were added to the format, allowing for live
interpolation between (say) regular and bold. These variations are global to the
font: they are controlled by the user for the font as a whole. Such a font is no
longer static, and the end user can navigate a _design space_ along dimensions
(axes) defined by the font. The chosen variation is defined by the _design space
location_, as a set of coordinates in the _design space_.

“Variable Components” add the possibility to place a _variable glyph_ in a
composite glyph, specifying the interpolation settings (the _design space
location_) for that single occurrence. A single glyph may define its own _design
space_, for composites to use as they see fit.

## Use cases

Some design tools already implement a methodology like Variable Components (for
example “Smart Components” in Glyphs.app). Users find it often a more efficient
way to design certain glyphs. Upon export as TTF, such components have to be
converted to traditional outlines.

However, a lot of space could be saved if the Variable Component information
would be stored in the font, instead of traditional outlines: a component
reference will typically take up less space than a discrete outline.

This is especially true for CJK fonts, which tend to contain very large numbers
of glyphs, many of which can be composed of variations of more basic glyphs.

More generally, fonts often contain glyphs that can be seen as variations of
regular glyphs, for example superior and inferior numerals and small caps. It
can be beneficial to represent these variations as local instances using
Variable Components.

# Proposal overview

The proposal presented here was informed by the following priorities:

- Only _add_ to the OpenType 1.8 format, if possible
- Avoid changing existing OpenType 1.8 tables as much as possible
- Use _existing data structures_ whenever possible
- Build on TrueType-flavored data
- Use the existing mechanisms available in OpenType 1.8 as much as possible:
  - Use `glyf` table composites/components
  - Use `gvar` for variations
- Make it easy for existing implementations to adapt
- Store the Variable Component data in a compact form, to maximize space saving

The first thing we need to pin down is how to do “local design spaces”. How does
a glyph define its own designspace, to be used by composites?

Here are some of the insights that led to the current design, in order:

1. The composite is in full control of the design space location of the
component
1. The _global_ design space location can affect the composite glyph, but it
_does not need to affect_ the design space location of the base glyph
_directly_.
1. A “base glyph” is just a regular `glyf`-based glyph, using `gvar` for
variations, but it needs to be able to use axes that are not user-controllable.
1. We can use `fvar` axes, but we need to be able to flag an axis as “This axis
is for variable component use only, do not expose it to the user, ever, at all”.
This is perhaps more strict than the definition of the existing “hidden” axis
flag, and we need to establish whether a new axis flag may be needed.
1. The total number of axes that can be used by a font (as specified in the
`fvar` table) does not have an unreasonable limit per se (65536), but it is not
without cost: in some places – for example in `gvar` variation tuples or
VarStore regions – there is a value specified for _every single_ axis in the
font, even if that axis does not participate in a certain variation. This is
especially relevant for `gvar`, as there can be many tuple variations (many
glyphs × many variations per glyph), so adding even a single axis to a font can
have a significant impact on the file size. So: let’s not use more axes than
strictly necessary.
1. A Variable Component axis is not exposed to the user, and there is no need
for “user coordinates”: the composite will only ever use “normalized
coordinates” to specify a design space location. Also: we don't consider
`avar`-like functionality necessary here.
1. A Variable Component axis is internally always referenced by its axis index.
The “axis tag” is completely irrelevant. (Axis tags are only used for user
interaction, and are not referenced anywhere in the font outside of the `fvar`
table.)
1. The previous points lead to the conclusion that a single axis can be (re)used
for _different purposes_ by _different base glyphs_. The axis _identity_ is no
longer attached to a function that is meaningful for the end user, or any
specific meaning at all. For example, base glyph X may use axis #2 to implement
“stretching”, but base glyph Y may use the same axis #2 to implement “bending”.
The _meaning_ of an axis is completely local to the base glyph. Each component
specifies the local design space location for its base glyph.
1. Concluding, the _local design space_ for a base glyph does not need any more
information than what we already have: it is completely defined by its
variations’ locations in the `gvar` table.

## Extending the OpenType format

To make Variable Components work, the only thing that is missing from OpenType
1.8 is the capability to store some additional information for each component of
composite glyphs. The core of this proposal is to add a single new table, called
`VarC`, that will provide a space for all new information.

A Variable Component reference needs the following information:

- The base glyph ID. This specifies which glyph we are referencing.
- A transformation (offset, scale, rotation, etc.)
- A design space location
- Variations for the transformation and the design space location, so the
composite itself can become a variable glyph (whether as a “normal” glyph, or
referenced as a Variable Component by another glyph)

We use the composites/components mechanism from the `glyf` table, so some of
these values are already taken care of: the base glyph ID and the offset. 

Components in the `glyf` table can optionally specify a scale value, or x/y
scale values, or a 2×2 transformation matrix, but we chose not to use these for
several reasons:

- Scale factors (and matrix values) are Fixed2Dot14, meaning they are limited to
the range -2.0..+2.0, which is a problem for some use cases.
- `gvar` only supports interpolation of the component offset values, not of the
scale values or the matrix.
- To interpolate 2×2 transformation matrices in a useful way is non-obvious and
non-trivial, even when decomposing into scale, rotations and skew values.

Summarizing:

- For the base glyph ID, the component offset and its variations, we rely on
`glyf` + `gvar` Additional transformation values (scale, rotation, etc.) and
its variations will be stored in `VarC`
- The component design space location and its variations will also be stored in
`VarC`

Base glyphs are totally ordinary `glyf` + `gvar` glyphs, but can also be
composites themselves, using Variable Components, so we fully embrace the
recursive nature of TrueType components.

# Format overview

High level structure of the `VarC` table:

| name | ... |
|-|-|
| Version | version field, initially `0x00010000` |
| numGlyphs | the number of glyphs |
| GlyphData[numGlyphs] | array of glyph data |
| VarStore | variation data |

`GlyphData` is an array of offsets to `Glyph` subtables, indexed by `GlyphID`.

`numGlyphs` must be less than or equal to the `numGlyphs` field in the `maxp`
table.

A `Glyph` subtable is an array of variable length `Component` data.

The `VarC` table depends on the `glyf` table: to parse a `VarC` `Glyph`, one
needs to know the number of components from the `glyf` table. That number is
not duplicated in `VarC`.

## The Component data structure

The `Component` data structure stores the additional transformation fields,
the designspace location for the components, and indices into the `VarStore`
for each value that needs variations.

### Transformation

The transformation data consists of individual optional fields, which can be
used to construct a transformation matrix.

Transformation fields:

| name | default value |
|-|-|
| Rotation | 0 |
| ScaleX | 1 |
| ScaleY | 1 |
| SkewX | 0 |
| SkewY | 0 |
| TCenterX | 0 |
| TCenterY | 0 |

The `TCenterX` and `TCenterY` values represent the “center of transformation”.
This is separate from the component offset as stored in the `glyf` table.

Details of how to build a transformation matrix, as pseudo-Python/fontTools
code, where `(X, Y)` is the component offset from the `glyf` table:

    # Using fontTools.misc.transform.Transform
    t = Transform()  # Identity
    t = t.translate(X + TCenterX, Y + TCenterY)
    t = t.rotate(Rotation)
    t = t.scale(ScaleX, ScaleY)
    t = t.skew(SkewX, SkewY)
    t = t.translate(-TCenterX, -TCenterX)

The transformation fields are stored as individual fields, and are interpolated
as individual fields. If the client needs a transformation matrix, then this
matrix needs to be constructed *after* interpolation.

Rationale for using a transformation center, using Rotation as an example:

- Rotation by default happens around the origin of the component
- For some cases this may be good enough, as the base glyph can be designed
this way
- However, in other cases components may need to determine the rotation center
locally, depending on how the component is used. Imagine a base glyph that
represents a horizontal bar. In one glyph, this bar should be rotated using the
left side as the center, and in another, it should be rotated using the right
side as the center.
- This all wouldn’t make a difference if it wasn’t for interpolation: it’s
really about how the component moves when transitioning from one composite
master to another. _(This should be illustrated visually)_

The design space location for components is stored as an array of axis indices
and a matching array of axis values.

The VarStore subtable is used to store variation deltas. It uses 16 bit integer
values, but we use these for various flavors of 16 bit fixed values, too.

# How to process VarC data?

When preparing a glyph outline for the rasterizer, the following logic needs to
be applied:

Inputs:
- glyph ID
- designspace location

Output:
- outline ready to be sent to the rasterizer

Steps:
- If the glyph is a composite and has an entry in the `VarC` table:
  - for each component:
    - Using the input designspace location, interpolate the transformation
    fields and the component's designspace location
    - Retrieve the outline using this algorithm recursively, but using the
    component's designspace location and glyph ID as inputs instead.
    - Transform the outline according to the transformation
- Else:
  - Proceed as usual, but apply the entire algorithm recursively, allowing for
  nested Variable Components

To clarify: Variable Components completely determine the designspace location
for the base glyph. Any axis not specified by a Variable Component has to be
interpreted as set to its *default*, regardless of the global designspace
location. In other words, Variable Components do not implicitly pass the global
designspace location down to the base glyphs. (It _can't_ pass down local
designspace coordinates, as local designspace may reuse axis IDs for different
purposes. Axis X may do something completely different for glyph A than for
glyph B.

_This may be opened for discussion: it can be useful to pass down the global
designspace coordinates down to base glyphs (unless overridden), but then we
need to distinguish between global `fvar` axes and local (anonymous) `fvar`
axes, due to the reusable nature of local axes in this design. To allow this, we
need a new `fvar` axis flag in addition to the "hidden" flag. Please discuss
here: https://github.com/BlackFoundryCom/variable-components-spec/issues/1_

# Format details

| type | name | value |
|-|-|-|
| Version | Version | 0x00010000 |
| uint16 | numGlyphs |
| LOffset | GlyphData[numGlyphs] |
| LOffset | VarStore |

GlyphData: this is an array of offsets to glyph data subtables. It is indexed by
glyph ID. If an offset is zero, then there is no `Glyph` data for this glyph.
The numGlyphs field less than or equal to the total number of glyphs in the
font.

VarStore: existing data structure to store all variation data, as used by GDEF,
HVAR, VVAR, MVAR, etc.

Glyph: the data for a single glyph contains the component data for all
components. The number of components is derived from the `glyf` table.

Component:

| type | name | notes |
|-|-|-|
| uint16 | flags | see below |
| uint8 or uint16 | numAxes | This is a uint16 if bit 3 of `flags` is set, else a uint8 |
| uint8 or uint16 | axisIndices[numAxes] | This is a uint16 if bit 3 of `flags` is set, else a uint8<br/>The most significant bit of each axisIndex tells whether this axis has a VarIdx in the VarIdxs array below. Bits 0..6 (uint8) or 0..14 (uint16) form the axis index. |
| Coord16 | axisValues[numAxes] | The axis value for each axis |
| Angle16 | Rotation | Optional, only present if it 5 of `flags` is set |
| Scale16 | ScaleX | Optional, only present if it 6 of `flags` is set |
| Scale16 | ScaleY | Optional, only present if it 7 of `flags` is set |
| Angle16 | SkewX | Optional, only present if it 8 of `flags` is set |
| Angle16 | SkewY | Optional, only present if it 9 of `flags` is set |
| Int16 | TCenterX | Optional, only present if it 10 of `flags` is set |
| int16 |  TCenterY | Optional, only present if it 11 of `flags` is set |
| uint8 | entryFormat | See below |
| VarIdx | VarIdxs[varIdxCount] | see below |

- Each `VarIdx` value represents an index into the `VarStore`, which contains all
  variation data.
- `varIdxCount` is determined by the sum of:
  - The number of axes that have a `VarIdx`
  - The number of transformation fields, if bit 4 of `flags` is set
- `VarIdx` entries are 1, 2, 3 or 4 bytes long. This is determined by the
`entryFormat` field, see below.

Component flags:

| bit number | meaning |
|-|-|
| 0..2 | Number of integer bits for ScaleX and ScaleY, mask: 0x07 |
| 3 | axis indices are shorts (clear = bytes, set = shorts) |
| 4 | Transformation fields have VarIdx |
| 5 | have Rotation |
| 6 | have ScaleX |
| 7 | have ScaleY |
| 8 | have SkewX |
| 9 | have SkewY |
| 10 | have TCenterX |
| 11 | have TCenterY |
| 12 | If ScaleY is missing: take value from ScaleX (to be discussed here: https://github.com/BlackFoundryCom/variable-components-spec/issues/2) |
| 13 | (reserved, set to 0) |
| 14 | (reserved, set to 0) |
| 15 | (reserved, set to 0) |

## Notes on precision

We chose to store all relevant fields as 16-bit values for maximum compactness,
and compatibility with the VarStore format. The downside of this is that we need
to choose the range of the fields carefully, as the range of _delta values_ may
exceed the range of master values by a factor related to the number of axes
involved.

In one case (component scale factors) we chose to use a three-bit field to
specify the number of integer bits to be used for scale factors. This gives us
a flexible range that can be made to fit the required range.

In other cases (`Angle16` and `Coord16`) we simply chose a larger range than
required for the master values, so there is some wiggle room for delta values
that are outside of the master value range.

Delta values are stored in the `VarStore` subtable, and they use the same
formats as their corresponding master values.

We observe that https://github.com/googlefonts/colr-gradients-spec/ adds 32-bit
value support to the VarStore format. `VarC` table could benefit from that as
well, at the expense of compactness.

## Field formats

**`Angle16`**: this is an int16 value used to represent an angle. To scale an
angle in degrees to this format, multiply the angle by `0x8000 / (4 * 360)`.
This gives us an effective range of -1440 degrees to +1440 degrees. Master
values are expected to be between -360 and +360 degrees. The extra headroom
is to allow for delta values that are outside of the master range.

**`Scale16`**: this is an int16 used as a 16 bit Fixed number, where the number
of integer bits is specified by bits 0..2 of the `flags` field. This allows us
to use 16 bits precision for a flexible range of scale values, depending on what
the component needs. It avoids having a small maximum (as with Fixed2Dot14,
which goes from -2 to +2) while sticking to 16 bits precision. The number of
precision bits is 16 - number-of-integer-bits.

**`Coord16`**: this is in int16 value used to represent a coordinate in a
designspace location. This is defined as a Fixed4Dot12. Master values are
expected to be between -1.0 and +1.0, but delta values may be outside that
range.

**`VarIdx`** array: this is a compactly stored array with `VarIdx` values, which
reference items in the VarStore. A VarIdx value is normally 32 bit, using 16
bits for the `outer` index and 16 bits for the `inner` index. The array items
are 1, 2, 3 or 4 bytes long, and are formatted as specified by the `entryFormat`
field. This is identical to the `entryFormat` field of the
[`DeltaSetIndexMap` subtable](https://docs.microsoft.com/en-us/typography/opentype/spec/hvar#table-formats)
from the `HVAR` table.

# New axis flag for the fvar table

The axes used to implement local designspaces for components should never be
exposed to users, and should me marked as such with a new axis flag:

| Mask | Name | Description |
|-|-|-|
| 0x0002 | INTERNAL_AXIS | The axis is only used internally, and should not be exposed in user interfaces. Used to implement local designspaces for Variable Components. |

This is a backwards-compatible change, and therefore the `fvar` table version
does not need to be updated.

# Notes on non-linear interpolation

Because the (local) axes for VariableComponents are controlled by global (user)
axes, this proposal contains the possibility to do non-linear interpolation,
without the need for duplicate fvar axis tags (\*). However, it is currently
limited to variable components. It is possible to change the proposal in a small
way that could apply the same control over designspace location and
transformation on the outlines of the glyph, if it is not a composite.

\*) By giving multiple fvar axes the same axis tags, many implementations allow
multiple axes to be controlled from a single value.
