---
sidebar_position: 6
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Outlines

## Overview

Once the raw points are available, we often want to turn them into solid, continuous outlines.
We do this by selecting an arbitrary subset of points and placing shapes there to form a part, and then use boolean operations (i.e., addition, subtraction, or intersection) to combine parts into a final outline to export.
We'll get back to how an individual part looks soon &ndash; but first, we need to get familiar with binding and filtering.

<br />











## Binding

While the points are enough to place properly positioned and rotated shapes (most commonly, rectangles, representing the keys of the board), these usually won't combine into a contiguous shape since there won't be any overlap.
So the first part of outline generation is thinking about "binding", where we can make the individual switch holes reach out towards (or, _bind_ to) each other.
Think of this as a kind of "neighbor declaration", telling Ergogen which directions to grow towards (and by how much) to reach the next-door point.

Of course, overlap could be achieved by placing larger shapes at each of the points, causing them to overlap by default, but since everything is placed by its center point, these larger shapes would result in larger outside margins as well.
With bind, we can declare the selective directions in which to grow the shapes placed, so that their final combination can become contiguous, yet with as little (or as much) margin as we might want.

### Explicit

The fully customizable way to add binding to points is through the key-level attribute `bind`:

```yaml
bind: num | [num_x, num_y] | [num_t, num_r, num_b, num_l] # defer to autobind by default
```

To recap, key-level declaration means that `bind` should be specified in the `points` section, benefiting from the same extension process every key-level attribute does.
Valid values follow CSS standards, so `num` applies to all directions, `num_x` horizontally, `num_y` vertically, and the `t`/`r`/`b`/`l` versions to top/right/bottom/left, respectively.

:::tip
Don't recall seeing `bind` in the [Keys](./points.md#keys) section, where supposedly all key-level attributes were listed?
That's because those were only the ones with meaning to the layout system.
Apart from those, *anything* can be declared as a key-level attribute, and some might gain meaning in later stages, like `bind` did just now.
:::

### Automatic

To spare us the `bind` declaration whenever possible, Ergogen offers an `autobind` key-level attribute as well.
Its value is a single number (`10` by default), and the relevant directions are calculated automatically (by looking at intra- and inter-column bounding boxes).
Basically, if we want bound shapes, we only need to say so (by setting `bound: true`, see [below](#common-attributes)) in most cases &ndash; or specify a larger `autobind` value once if `10` wasn't enough to bridge the gaps.
And if autobinding fails for a more complex shape, we can always fall back to explicit `bind` declarations.

### Examples

<details><summary>Explicit bind</summary>
<p>

With explicit `bind`, you specify exactly how much each key's rectangle should extend in each direction (top, right, bottom, left), following CSS conventions. This lets shapes reach towards their neighbors without increasing the outside margin. Compare the `raw` outline (no binding, gaps between keys) with the `bound` outline (binding fills the gaps).

```yaml
points:
  zones:
    matrix:
      columns:
        pinky:
          rows:
            bottom.bind: [0, 5, 0, 0]
            home.bind: [0, 5, 0, 0]
            top.bind: [0, 5, 0, 0]
        ring:
          rows:
            bottom.bind: [0, 5, 0, 5]
            home.bind: [0, 5, 0, 5]
            top.bind: [0, 5, 0, 5]
        middle:
          rows:
            bottom.bind: [0, 0, 0, 5]
            home.bind: [0, 0, 0, 5]
            top.bind: [0, 0, 0, 5]
      rows:
        bottom:
        home:
        top:
outlines:
  raw:
    - what: rectangle
      where: true
      size: [u-1, u-1]
      bound: false
  bound:
    - what: rectangle
      where: true
      size: [u-1, u-1]
      bound: true
```

**`raw` outline** (no binding &mdash; gaps between keys):

![Explicit bind raw](./assets/outlines_explicit_bind_raw.svg)

**`bound` outline** (binding fills the gaps):

![Explicit bind bound](./assets/outlines_explicit_bind_bound.svg)

</p>
</details>




<details><summary>Autobind</summary>
<p>

When `autobind` is set (default is `10`), Ergogen automatically extends each key's rectangle toward its closest neighbors, up to the specified amount. This eliminates most manual `bind` declarations for standard layouts. Simply set `bound: true` on the shape.

```yaml
points:
  key:
    autobind: 10
  zones:
    matrix:
      columns:
        pinky:
        ring:
        middle:
        index:
      rows:
        bottom:
        home:
        top:
outlines:
  board:
    - what: rectangle
      where: true
      size: [u-1, u-1]
      bound: true
```

![Autobind board](./assets/outlines_autobind_board.svg)

</p>
</details>













## Filtering

Filtering is how Ergogen decides which points to use when placing the shape we're currently placing.
After all, the points section might contain lots of zones, multiple *kinds* of points, helpers for mounting holes or one-off PCB footprints, etc.
So being able to easily select a subset of these points can come in handy.

### Basics

First up, let's see what a filter means depending on what datatype we use when declaring it:

- **`undefined`**: if left empty, a filter produces the default `[0, 0, 0°]` origin point.

- **boolean**: if the filter is `true`, all points are used; if it's `false`, no points are used.

- **string**: represents a single/simple filter &ndash; the workings of which we'll discuss in a second.

- **object**, or **array that contains an object** somewhere: will be parsed as an [anchor](./points.md#anchors), returning the single resulting point.

  :::note
  Although there can be valid anchor declarations that are neither objects, nor arrays containing an object at any depth, these are not supported where filters are expected because Ergogen would have no way to decide what it's looking at.
  Remember, however, that every anchor **can** be represented in full object form &ndash; any other representation is just a shorthand for convenience.
  :::

- **array containing no objects** at any depth: complex filter, see [Advanced usage](#advanced-usage).

So the undefined and boolean cases are easy, objects just redirect to anchors, and arrays are more advanced.
What about strings, then?

At their simplest, strings just compare the given value against the name of each key and check for straight equality.
Since names are unique, this makes it easy to single out a point, but nothing more. How do we get "real" subsets?

Enter the `tags` key-level attribute.
It can be either an array (containing string tags, or "labels" that should apply to the given point), or an object (in which case the keys from its key/value pairs count).
Arrays are probably more readable, while objects might be more easily extendable via inheritance or preprocessing.
Use whichever form makes sense.

:::tip
`tags` is yet another key-level attribute that gains meaning during outlining only, like `bind` did above.
:::

By default, string filters consider not only the name of each key but their tags, too.
And combining their basic exact matching behavior with a non-unique field leads to easy subset selection. Yay!

But wait, there's more!
If the string is surrounded by `/`s (slashes), it's interpreted as a regex, and exact matching changes to pattern matching.
So we might not even *need* tags for, say, differentiating zones because we know that key names by default are formatted as `zone_column_row` so we can just say something like `/^matrix_.*/` to filter any key whose name starts with the substring `matrix_`.
The usual regex flags are also supported if specified after the trailing slash, so feel free to use case-insensitive, multiline, or even unicode expressions should the need arise.

Finally, if it would be easier to select what we **don't** want instead of what we **do** want, filters support negation if prefixed by a `-` (minus).
So while saying `matrix_pinky_home` select only that one key, `-matrix_pinky_home` selects everything *except* that key.
This also works with both tags and regexes, of course, so `-alpha` selects everything that isn't tagged with `alpha` (assuming the existence of an alpha tag), and `-/pinky/` selects keys where the "pinky" substring *isn't* found anywhere within the name or any of its tags.



### Advanced usage

Every single filter actually consists of three components:

1. **which** key-level attributes to check against,
2. **how** to check against them, and
3. **what** value to check against them.

So far, we've only used the third component, as the **which** part was always the default `name` and `tags`, while the **how** part was interpreted as the special "similarity" operator, handling both exact matches and regexes.
But what if we want to check against some other key-level attribute; or check in a different way?

Enter full form filters.
In the background, writing `something` gets translated as `meta.name,meta.tags ~ something`, where `meta` is each key's metadata containing all key-level attributes (see [Keys](./points.md#keys)) and `~` is the similarity operator.
So if we want to check against something else (say, we declared our own `foobar` field among the other key-level attributes), then we can simply say `meta.foobar ~ something`.
As for operators, only similarity (`~`) is implemented for now, but others (such as mathematical relations) will be added in the future.

For even more advanced usage, we can combine simple filters with AND/OR logical relations into complex filters using arrays.
Odd levels of array nesting represent OR, while even levels represent AND.
So, for example, writing `[something, other]` would mean that all points are returned where either `something` **OR** `other` matches the name/tags, while `[[something, other]]` would only return points where both `something` **AND** `other` matches (note the double arrays in the latter case).

### Examples

<details><summary>Tags</summary>
<p>

Tags allow easy subset selection. By assigning tags to keys in the points section, you can filter outlines to include only the tagged keys.

```yaml
points:
  zones:
    matrix:
      key:
        tags:
          key: true
      columns:
        pinky:
          key.tags.pinky: true
        ring:
        middle:
        index:
          key.tags.index: true
      rows:
        bottom:
        home:
        top:
outlines:
  all_keys:
    - what: rectangle
      where: key
      size: 14
  only_pinky:
    - what: rectangle
      where: pinky
      size: 14
```

**`all_keys`** (every key tagged with `key`):

![Tags all keys](./assets/outlines_tags_all_keys.svg)

**`only_pinky`** (only keys tagged with `pinky`):

![Tags only pinky](./assets/outlines_tags_only_pinky.svg)

</p>
</details>

<details><summary>Regexes</summary>
<p>

Strings surrounded by `/` are treated as regular expressions. This is useful for matching keys by name patterns without needing tags.

```yaml
points:
  zones:
    matrix:
      columns:
        pinky:
        ring:
        middle:
        index:
      rows:
        bottom:
        home:
        top:
outlines:
  only_pinky:
    - what: rectangle
      where: /pinky/
      size: 14
  home_row:
    - what: rectangle
      where: /.*_home$/
      size: 14
```

**`only_pinky`**:

![Regex only pinky](./assets/outlines_regex_only_pinky.svg)

**`home_row`**:

![Regex home row](./assets/outlines_regex_home_row.svg)

</p>
</details>




<details><summary>Negation</summary>
<p>

Prefixing a filter with `-` negates it, selecting everything that does **not** match. This works with both plain strings and regexes.

```yaml
points:
  zones:
    matrix:
      columns:
        pinky:
        ring:
        middle:
        index:
      rows:
        bottom:
        home:
        top:
outlines:
  everything_except_pinky:
    - what: rectangle
      where: -/pinky/
      size: 14
```

![Negation](./assets/outlines_negation_everything_except_pinky.svg)

</p>
</details>




<details><summary>Full filters</summary>
<p>

Full-form filters let you check against any key-level attribute. The syntax is `meta.attribute ~ value`, where `~` is the similarity operator. You can check custom attributes defined in the points section.

```yaml
points:
  zones:
    matrix:
      columns:
        pinky:
          key.side: left
        ring:
          key.side: left
        middle:
          key.side: right
        index:
          key.side: right
      rows:
        bottom:
        home:
        top:
outlines:
  left_side:
    - what: rectangle
      where: "meta.side ~ left"
      size: 14
```

![Full filter left side](./assets/outlines_full_filter_left_side.svg)

</p>
</details>




<details><summary>Combination</summary>
<p>

Filters can be combined using arrays. Odd levels of nesting represent OR, even levels represent AND. For example, `[a, b]` matches keys where `a` OR `b` matches, while `[[a, b]]` matches keys where both `a` AND `b` match.

```yaml
points:
  zones:
    matrix:
      key.tags.key: true
      columns:
        pinky:
          key.tags.pinky: true
        ring:
        middle:
        index:
          key.tags.index: true
      rows:
        bottom:
        home:
        top:
outlines:
  pinky_or_index:
    - what: rectangle
      where:
        - /pinky/
        - /index/
      size: 14
```

![Combination filter](./assets/outlines_combination_pinky_or_index.svg)

</p>
</details>

<br />







## Parts

With this, we can finally move on to the outlines themselves.
The relevant section in the config will look something like this:

<Tabs>
<TabItem value="array" label="Array notation" default>

```yaml
outlines:
  <outline_name>:
    - <part>
    - <part>
    - ...
  ...
```

</TabItem>
<TabItem value="object" label="Object notation">

```yaml
outlines:
  <outline_name>:
    part1: <part>
    part2: <part>
    ...
  ...
```

</TabItem>
</Tabs>

:::note
Listing parts within an outline can be an object as well as an array (see "Object notation" tab).
Objects might be beneficial if part names are important for config readability (or when YAML or built-in inheritance is used), while arrays are a bit more terse.
Use whichever form makes more sense.
:::

Operations are performed in order, and the resulting shape is exported as an output.
Additionally, it is going to be available for further outline declarations to use (through the `outline` type, see below) under the name specified (`<outline_name>`, in this case).

Now let's see how those `<part>`s are made.



### Common attributes

Each part has the following common attributes:

- **`what`**: declares *what* shape we want to place &ndash; see [Shapes](#shapes).

- **`where`**: declares *where* we want to place those shapes &ndash; this is where we can use the previously discussed filters.

- **`operation`**: indicates how we want the current part to combine with the cumulative result of previous parts.
Options include:

  - **`add`**: produces an union &ndash; this is the default operation.
  - **`subtract`**: subtracts this part from the in-progress result.
  - **`intersect`**: computes the intersection of this part and the in-progress result.
  - **`stack`**: just draws the current part "on top of" the in-progress result (possibly crossing lines instead of calculating unions).

    :::tip
    `stack` can be used as a computationally "cheaper" `subtract` in some cases, but it's mostly for being able to visualize individual parts in the context of other parts and getting a sense of what happens (i.e., debugging).
    :::

- **`bound`**: boolean value, representing whether we want to activate binding on the shapes or not.
If `false`, the shapes are placed as-is.
If `true`, the corresponding binding rectangles are added to each relevant side of each shape and the results union'ed.

- **`asym`**: the field is a companion to the `where` filter and represents how filtering should treat mirrored points.
  The same values are available that we've discussed in the [Mirroring](./points.md#mirroring) section &ndash; the canonical choices are `source`/`clone`/`both`.

  - The default `source` only returns the points matched by the filter.

  - `clone` returns only the mirrored versions of the points that would be matched by the filter.

  - `both` returns both the regular matches and their mirror images.

    :::caution
    If the filter translates to an anchor, this check is ***strict*** &ndash; meaning that Ergogen will error out if the mirror image doesn't exist.
    On the other hand, the mirror check is permissive for regular filters, including them if they exist and ignoring the cases where they don't.
    :::

- **`adjust`**: a relative anchor by which to adjust the position of each shape &ndash; similarly to the key-level `adjust` attribute.

  :::tip
  This field makes it possible to place shapes not only **at** certain filtered points, but also **below** or **next to** those points.
  :::

- **`scale`**: an optional multiplier by which to scale the resulting shape.
  The default is `1` for no scaling.

- **`expand`**: a number in mm's by which to expand (or shrink, if the number is negative) the current outline.
  Differs from `scale`ing because it draws and external (or internal) "outline" for the starting shape, thereby usually changing the shape itself, too, not just its size.
  For more info, see the relevant [Maker.JS docs](https://maker.js.org/docs/advanced-drawing/#Expanding%20paths).

- **`joints`**: a companion to `expand`, specifying which type of treatment to apply to the joints during expansion/shrinking.

  - `0` or `round` means the corners will be rounded (thereby having **zero** joints);
  - `1` or `pointy` means the corners will stay (thereby still having **one** joint); and
  - `2` or `beveled` means the corners will get beveled (thereby having **two** joints).

- **`fillet`**: this number (if greater than the default zero) triggers a filleting operation on the (almost-)completed part and rounds its corners with the given radius.
  If the radius is larger than either of the corner's neighboring line segments, that corner is skipped.

  :::tip
  Once a corner is filleted, it won't be filleted again, so it's safe to apply a `fillet` with increasingly smaller radii to catch every sharp corner if desired.
  :::





### Shapes

Shapes can have their own, shape-specific attributes on top of the ones already discussed above.
Additionally, each shape can introduce shape-specific units to the evaluation context to further avoid repetition.

:::tip
Say we'd want to express that a rectangle of size `10` should be adjusted half of its width to the right.
We could write `adjust.shift: [5, 0]`, of course, but then if the size changes, the shift needs to change as well.
Instead, we could write `adjust.shift: [.5 sx, 0]`, referencing the size's x value (i.e., its width).
:::

With this, let's see a list of what actual shapes we can place, what extra attributes they have, and what extra units they introduce:

- **`rectangle`**: A basic rectangle primitive.

  - **`size`**: Either a number or an array in the form `[num_x, num_y]`, representing the width/height of the rectangle(s) to place. If it's a single number `num`, it's interpreted as `[num, num]` (i.e., a square). Mandatory. Introduces `sx` and `sy` as units for width and height, respectively.

  - **`bevel`**: Optional beveling for the rectangles, default is `0`.
  
  - **`corner`**: Optional corner radius for the rectangles, default is `0`.

    :::caution
    `size` represents the final size of the resulting rectangle, so any `bevel` or `corner` values are subtracted from it appropriately to make room for the bevels/radii.
    This can lead to an error if the size is too small (or the `bevel`/`corner` values are too large).
    :::

    :::tip
    Corners and bevels can be used simultaneously.
    Corner radii are applied after beveling, leading to rounded bevels.
    :::

- **`circle`**: A basic circle primitive.

  - **`radius`**: The radius of the circle to place. Mandatory. Introduces `r` as a unit.

- **`poly`**: A custom polygon.

  - **`points`**: Mandatory array of anchors, representing the points of the polygon.
    Each item of the array is a regular anchor &ndash; the only difference is that if its `ref` is unspecified, the polygon's previous point will be assumed (to simulate a continuous chain).
    For the first point, `[0, 0, 0°]` is assumed to be the starting point by default (as the polygon will be placed using a `[0, 0]` origin anyway).
  
- **`outline`**: Allows reuse of an already existing outline as a primitive for further outlines.

  - **`name`**: The name to identify the outline to place. Mandatory.

  - **`origin`**: An optional anchor to specify which point in the existing outline to consider as the origin (i.e., the location of the outline by which it's placed at the requested points during outlining).

    :::tip
    `origin` is functionally identical to the globally available `adjust`, only it applies before placing each outline at the target points while `adjust` applies afterwards.
    Both options are available for flexibility, feel free to use either (or both in conjunction, if appropriate).
    :::










### Syntactic sugar

At this point, we're done with actual outline functionality, but there are some extra shorthands and conveniences worth mentioning.

The first kind are `string shorthands`, where a part within an outline is given by a single string instead of a whole object.
This is a streamlined way to refer to already existing outlines and combine them further.
The format of this string should start with a symbol from `[+, -, ~, ^]`, followed by a name, and is equivalent to adding/subtracting/intersecting/stacking an outline of that name, respectively.
More specifically, `~something` is equivalent to:

```yaml
what: outline
where: undefined # meaning [0, 0, 0°], so just placing the outline where it is
name: something
operation: intersect
```

If the symbol prefix is missing, addition is assumed &ndash; so simply naming outlines as parts works, too.

Another minor shorthand is declaring the `expand` and `joints` fields all at once using just the `expand` field, and specifying its value as the number for the expansion, followed by either `)`, `>`, or `]` (representing `round`, `pointy`, and `beveled` joints, respectively).
So an `expand` value of `3]` would translate to:

```yaml
expand: 3
joints: beveled
```

Finally, "private" outlines: if we only want to use an outline as a building block for further outlines, we can start its name with an underscore (e.g., `_my_name`) to prevent it from being actually exported.
(By convention, a starting underscore is kind of like a "private" marker.)










### Examples

<details><summary>Shapes</summary>
<p>

Ergogen supports several shape primitives: rectangles (with optional corner rounding and beveling), circles, and polygons. These are combined to form complex outlines.

```yaml
points:
  zones:
    matrix:
      columns:
        one:
        two:
      rows:
        bottom:
        top:
outlines:
  rectangles:
    - what: rectangle
      where: true
      size: [14, 14]
  circles:
    - what: circle
      where: true
      radius: 7
  rounded_rect:
    - what: rectangle
      where: true
      size: [u-1, u-1]
      bound: true
      corner: 3
  beveled_rect:
    - what: rectangle
      where: true
      size: [u-1, u-1]
      bound: true
      bevel: 2
  filleted:
    - what: rectangle
      where: true
      size: [u-1, u-1]
      bound: true
      fillet: 2
```

**`rectangles`**:

![Rectangles](./assets/outlines_shapes_rectangles.svg)

**`circles`**:

![Circles](./assets/outlines_shapes_circles.svg)

**`rounded_rect`**:

![Rounded rect](./assets/outlines_shapes_rounded_rect.svg)

**`beveled_rect`**:

![Beveled rect](./assets/outlines_shapes_beveled_rect.svg)

**`filleted`**:

![Filleted](./assets/outlines_shapes_filleted.svg)

</p>
</details>

<details><summary>Boolean operations</summary>
<p>

Parts within an outline are combined using boolean operations: `add` (union), `subtract` (cut out), `intersect` (keep overlap), and `stack` (layer without boolean). The shorthand string syntax `+name`, `-name`, `~name`, `^name` is also available.

```yaml
points:
  key:
    padding: cy
    bind: 0.1
  zones:
    matrix:
      columns:
        one:
        two:
      rows:
        bottom:
        top:
outlines:
  _base:
    - what: rectangle
      where: true
      size: cy
      bound: true
  _holes:
    - what: circle
      where: true
      radius: 5
  _fillet:
    - name: _base
      fillet: 2
  _scaled:
    - name: _fillet
      scale: 0.5
  board_with_holes:
    - "_base"
    - "-_holes"
  intersected:
    - "_base"
    - "~_scaled"
  expanded:
    - name: _base
      expand: 1
```

**`board_with_holes`** (base minus circles):

![Board with holes](./assets/outlines_boolean_board_with_holes.svg)

**`intersected`** (base intersected with scaled version):

![Intersected](./assets/outlines_boolean_intersected.svg)

**`expanded`** (base expanded by 1mm):

![Expanded](./assets/outlines_boolean_expanded.svg)

</p>
</details>

<details><summary>Asymmetry</summary>
<p>

The `asym` key controls how shapes are placed at mirrored points: `source` places only at original (non-mirrored) points, `clone` places only at mirrored points, and `both` (default) places at all points.

```yaml
points:
  zones:
    matrix:
      columns:
        pinky:
        ring:
        middle:
        index:
      rows:
        bottom:
        home:
        top:
  mirror:
    ref: matrix_index_home
    distance: 2 u
outlines:
  source_only:
    - what: rectangle
      where: true
      size: 14
      asym: source
  clone_only:
    - what: rectangle
      where: true
      size: 14
      asym: clone
  both_sides:
    - what: rectangle
      where: true
      size: 14
      asym: both
```

**`source_only`** (original side only):

![Source only](./assets/outlines_asymmetry_source_only.svg)

**`clone_only`** (mirrored side only):

![Clone only](./assets/outlines_asymmetry_clone_only.svg)

**`both_sides`** (both sides):

![Both sides](./assets/outlines_asymmetry_both_sides.svg)

</p>
</details>

<details><summary>Adjustments</summary>
<p>

The `adjust` key applies an anchor-like transformation after the shape has been placed at the target point. This is useful for offsetting shapes relative to key positions, for example, to place screw holes between keys.

```yaml
points:
  zones:
    matrix:
      columns:
        pinky:
        ring:
        middle:
        index:
      rows:
        bottom:
        home:
        top:
outlines:
  switches:
    - what: rectangle
      where: true
      size: [14, 14]
  adjusted_circles:
    - what: circle
      where: true
      radius: 3
      adjust:
        shift: [0, -7]
```

**`switches`** (rectangles at key positions):

![Switches](./assets/outlines_adjustments_switches.svg)

**`adjusted_circles`** (circles shifted down by 7mm):

![Adjusted circles](./assets/outlines_adjustments_adjusted_circles.svg)

</p>
</details>
