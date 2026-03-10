---
sidebar_position: 7
---

# Cases

## Overview

Cases add a pretty basic and minimal 3D aspect to the generation process.
In this phase, we take different outlines (defined in the previous section, even the "private" ones), extrude and position them in space, and combine them into one 3D-printable object.
That's it.
Declarations might look something like the following:

```yaml
cases:
    case_name:
        - what: outline # default option
          name: <outline ref>
          extrude: num # default = 1
          shift: [x, y, z] # default = [0, 0, 0]
          rotate: [ax, ay, az] # default = [0, 0, 0]
          operation: add | subtract | intersect # default = add
        - what: case
          name: <case_ref>
          # extrude makes no sense here...
          shift: # same as above
          rotate: # same as above
          operation: # same as above
        - ...
    ...
```

:::note
Individual case parts can be both arrays or objects, just like with outline parts previously.
Use whichever is more convenient.
:::

When the `what` is `outline`, `name` specifies which outline to import onto the xy plane, while `extrude` specifies how much it should be extruded along the z axis.
When the `what` is `case`, `name` specifies which previously defined case to use.
After having established our base 3D object, it is (relatively!) `rotate`d, `shift`ed, and combined with what we have so far according to `operation`.
If we only want to use an object as a building block for further objects, we can employ the same "start with an underscore" trick we learned at the outlines section to make it "private".

Individual case parts can again be listed as an object instead of an array, if that's more comfortable for inheritance/reuse (just like for outlines).
And speaking of outline similarities, the `[+, -, ~]` plus name shorthand is available again.
First it will try to look up cases, and then outlines by the name given.
Stacking is omitted as it makes no sense here.

## Examples

<details><summary>Simple Extrusion</summary>
<p>

Takes a previously defined outline and extrudes it into a 3D object. This is the most basic case operation.

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
  board:
    - what: rectangle
      where: true
      size: [u-1, u-1]
      bound: true
      fillet: 2
cases:
  bottom:
    - name: board
      extrude: 1
```

</p>
</details>

<details><summary>Case with Boolean Operations</summary>
<p>

Cases support boolean operations (add, subtract, intersect) as well as 3D transformations (shift, rotate). Here we create a plate with switch cutouts by subtracting smaller rectangles from the main board outline.

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
  board:
    - what: rectangle
      where: true
      size: [u-1, u-1]
      bound: true
      fillet: 2
  _switch_cutouts:
    - what: rectangle
      where: true
      size: [14, 14]
cases:
  plate:
    - name: board
      extrude: 1.5
    - name: _switch_cutouts
      extrude: 1.5
      operation: subtract
```

</p>
</details>

<details><summary>Combining Cases</summary>
<p>

Previously defined cases can be referenced and combined. The `[+, -, ~]` shorthand operators work the same as for outlines: `+` for union, `-` for subtraction, and `~` for intersection. You can also use 3D shift and rotation to position parts in space.

```yaml
points:
  zones:
    matrix:
outlines:
  _square:
    - what: rectangle
      where: true
      size: [8, 8]
  _circle:
    - what: circle
      where: true
      radius: 3
cases:
  _cube:
    - name: _square
      extrude: 8
  _cylinder:
    - name: _circle
      extrude: 8
  _hollowed_cube:
    target:
      name: _cube
      what: case
    tool:
      name: _cylinder
      what: case
      operation: subtract
  _rotated_cylinder:
    - name: _circle
      extrude: 8
      shift: [0, 4, 4]
      rotate: [90, 0, 0]
  combination:
    - "_hollowed_cube"
    - "~_rotated_cylinder"
```

</p>
</details>