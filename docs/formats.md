---
sidebar_position: 9
---

# File formats

## Input

### Config

Ergogen accepts its configuration in the following formats:

- **YAML** (`.yaml` or `.yml`): The most common and recommended format for human-written configs.
- **JSON** (`.json`): A valid alternative if you prefer JSON. Conversion from YAML is trivial and Ergogen auto-detects the format.
- **JavaScript** (`.js`): A `.js` file that, when evaluated, produces the config object. This is useful when you need procedural features like loops, conditionals, or parametric functions to generate your config programmatically.

Since the config is ultimately just a data structure, you can use any language or tool to generate the JSON or YAML input if the declarative approach isn't flexible enough.

### Bundles {#bundles}

Bundles are `.zip` or `.ekb` (Ergogen Keyboard Bundle) archives that package a config file together with additional resources:

- **Custom footprints**: `.js` files placed in a `footprints/` directory within the bundle. These are automatically loaded and made available under the `what` key in the PCB config, using the file's basename as the identifier.
- **Custom templates**: `.js` files placed in a `templates/` directory within the bundle. These provide custom PCB templates that can be referenced via the `template` key in the PCB config.

This allows designers to extend Ergogen's built-in footprint and template sets without modifying the Ergogen codebase.

## Output

Ergogen generates the following output files:

### Points
- **`points.yaml`**: The canonical, fully resolved point definitions in YAML format.
- **`demo.svg`**: A 2D SVG visualization of all key positions, useful for quickly verifying your layout.

### Outlines
- **`.svg`**: Each named outline is exported as an SVG file for preview and verification.
- **`.dxf`**: Each named outline is also exported in DXF format, suitable for laser cutting or CNC machining.

### Cases
- **`.jscad`**: Each named case is exported as an OpenJSCAD script. These can be opened in [OpenJSCAD](https://openjscad.xyz/) to preview and export as STL for 3D printing.

### PCBs
- **`.kicad_pcb`**: Each named PCB is exported as a KiCAD PCB file. The PCB is **un-routed**, meaning all components are placed and nets are assigned, but traces have not been drawn. Open the file in KiCAD to complete routing manually or with an auto-router.