# TiaCAD v3.0 Auto-References Guide

**Version:** 3.0.0
**Last Updated:** 2025-11-07
**Status:** Complete

---

## Overview

**Auto-references** are one of the most powerful features in TiaCAD v3.0. Every part you create automatically provides a set of spatial references without requiring explicit definition. This eliminates boilerplate and makes your YAML files cleaner and more maintainable.

### What are Auto-References?

Auto-references are automatically generated spatial references that provide access to:
- **Points**: Centers, origins, and key locations
- **Faces**: Top, bottom, left, right, front, back faces
- **Axes**: Principal axes through the part center

### Key Benefits

✅ **No Manual Definition** - References are available immediately
✅ **Consistent Naming** - Same pattern across all parts
✅ **Full Orientation** - Includes position + normal/tangent vectors
✅ **Local Frame Offsets** - Intuitive positioning relative to faces
✅ **Cleaner YAML** - Less boilerplate, more readability

---

## Universal Auto-References

Every part (box, cylinder, sphere, cone) automatically provides these 11 references:

### Point References

| Reference | Description | Example Use Case |
|-----------|-------------|------------------|
| `{part}.center` | Bounding box center | Centering objects, alignment |
| `{part}.origin` | Part origin (0,0,0 in part's local space) | Reference to creation point |

### Face References

| Reference | Description | Direction | Example Use Case |
|-----------|-------------|-----------|------------------|
| `{part}.face_top` | Top face | +Z | Stacking parts vertically |
| `{part}.face_bottom` | Bottom face | -Z | Positioning on surfaces |
| `{part}.face_right` | Right face | +X | Side attachments |
| `{part}.face_left` | Left face | -X | Side attachments |
| `{part}.face_front` | Front face | +Y | Front mounting |
| `{part}.face_back` | Back face | -Y | Back mounting |

### Axis References

| Reference | Description | Direction Vector | Example Use Case |
|-----------|-------------|------------------|------------------|
| `{part}.axis_x` | X-axis through center | [1, 0, 0] | Horizontal rotation |
| `{part}.axis_y` | Y-axis through center | [0, 1, 0] | Horizontal rotation |
| `{part}.axis_z` | Z-axis through center | [0, 0, 1] | Vertical rotation |

---

## Basic Usage Examples

### Example 1: Simple Vertical Stack

Stack parts on top of each other without manual calculations:

```yaml
parts:
  base:
    primitive: box
    parameters: {width: 100, height: 10, depth: 100}

  pillar:
    primitive: cylinder
    parameters: {radius: 5, height: 50}
    translate:
      to: base.face_top  # Auto-reference!

  cap:
    primitive: box
    parameters: {width: 15, height: 5, depth: 15}
    translate:
      to: pillar.face_top  # Auto-reference!
```

**Result:** Three parts perfectly stacked without any manual coordinate calculations!

### Example 2: Side-by-Side Placement

Position parts next to each other:

```yaml
parts:
  left_block:
    primitive: box
    parameters: {width: 30, height: 20, depth: 20}
    translate:
      to: [-20, 0, 0]

  right_block:
    primitive: box
    parameters: {width: 30, height: 20, depth: 20}
    translate:
      to: left_block.face_right  # Position at right face
```

### Example 3: Rotation Around Axes

Rotate parts around their principal axes:

```yaml
operations:
  rotated_gear:
    type: transform
    input: gear
    transforms:
      - rotate:
          angle: 45
          around: gear.axis_z  # Auto-generated axis reference
```

---

## Advanced Usage

### Local Frame Offsets

Offsets follow the reference's local coordinate system, making positioning intuitive:

```yaml
parts:
  platform:
    primitive: box
    parameters: {width: 100, height: 10, depth: 100}

  elevated_post:
    primitive: cylinder
    parameters: {radius: 5, height: 30}
    translate:
      to:
        from: platform.face_top
        offset: [0, 0, 5]  # 5 units ABOVE the face (along its normal)

  corner_post:
    primitive: cylinder
    parameters: {radius: 3, height: 30}
    translate:
      to:
        from: platform.face_top
        offset: [40, 40, 0]  # 40 units in X and Y, on the face surface
```

**Local Frame Orientation:**
- **Face references**: Normal = Z-axis, tangents = X/Y axes
- **X offset**: Moves along face tangent (right/left)
- **Y offset**: Moves along face tangent (forward/back)
- **Z offset**: Moves perpendicular to face (away from face)

### Combining Multiple References

Use auto-references throughout your design:

```yaml
parameters:
  base_size: 100
  post_height: 40
  cap_size: 20

parts:
  base:
    primitive: box
    parameters:
      width: '${base_size}'
      height: 10
      depth: '${base_size}'

  # Four corner posts using base.face_top + offsets
  post_ne:
    primitive: cylinder
    parameters: {radius: 3, height: '${post_height}'}
    translate:
      to:
        from: base.face_top
        offset: [40, 40, 0]

  post_nw:
    primitive: cylinder
    parameters: {radius: 3, height: '${post_height}'}
    translate:
      to:
        from: base.face_top
        offset: [-40, 40, 0]

  post_se:
    primitive: cylinder
    parameters: {radius: 3, height: '${post_height}'}
    translate:
      to:
        from: base.face_top
        offset: [40, -40, 0]

  post_sw:
    primitive: cylinder
    parameters: {radius: 3, height: '${post_height}'}
    translate:
      to:
        from: base.face_top
        offset: [-40, -40, 0]

  # Top platform stacked on one post
  top_platform:
    primitive: box
    parameters:
      width: '${base_size}'
      height: 5
      depth: '${base_size}'
    translate:
      to: post_ne.face_top
```

### Assembly with Auto-References

Build complex assemblies efficiently:

```yaml
parts:
  # Mounting base
  mount_base:
    primitive: box
    parameters: {width: 150, height: 20, depth: 100}

  # Vertical bracket mounted to back face
  bracket:
    primitive: box
    parameters: {width: 150, height: 80, depth: 10}
    translate:
      to: mount_base.face_back

  # Support arm attached to bracket side
  support_arm:
    primitive: box
    parameters: {width: 10, height: 80, depth: 60}
    translate:
      to:
        from: bracket.face_right
        offset: [-5, 0, 30]  # Slight overlap + outward offset

  # Mounting plate at end of arm
  mount_plate:
    primitive: box
    parameters: {width: 40, height: 40, depth: 5}
    translate:
      to: support_arm.face_front
```

---

## Common Patterns

### Pattern 1: Vertical Stacking

**Problem:** Stack parts vertically with precise alignment

**Solution:** Use `face_top` references

```yaml
parts:
  layer1:
    primitive: box
    parameters: {width: 100, height: 10, depth: 100}

  layer2:
    primitive: box
    parameters: {width: 90, height: 10, depth: 90}
    translate:
      to: layer1.face_top

  layer3:
    primitive: box
    parameters: {width: 80, height: 10, depth: 80}
    translate:
      to: layer2.face_top
```

### Pattern 2: Centered Mounting

**Problem:** Mount a part centered on another's face

**Solution:** Use face reference (centers automatically)

```yaml
parts:
  base_plate:
    primitive: box
    parameters: {width: 100, height: 5, depth: 100}

  mounted_cylinder:
    primitive: cylinder
    parameters: {radius: 15, height: 30}
    translate:
      to: base_plate.face_top  # Automatically centered!
```

### Pattern 3: Offset Mounting

**Problem:** Mount a part offset from center

**Solution:** Use face reference with local offset

```yaml
parts:
  panel:
    primitive: box
    parameters: {width: 200, height: 100, depth: 5}

  knob:
    primitive: cylinder
    parameters: {radius: 8, height: 15}
    translate:
      to:
        from: panel.face_front
        offset: [60, 30, 0]  # Offset in panel's local frame
```

### Pattern 4: Edge Alignment

**Problem:** Align parts at edges, not centers

**Solution:** Use calculated offsets based on dimensions

```yaml
parameters:
  base_width: 100
  pillar_offset: '${base_width / 2 - 5}'  # 5 = pillar radius

parts:
  base:
    primitive: box
    parameters: {width: '${base_width}', height: 10, depth: 100}

  corner_pillar:
    primitive: cylinder
    parameters: {radius: 5, height: 50}
    translate:
      to:
        from: base.face_top
        offset: ['${pillar_offset}', '${pillar_offset}', 0]
```

### Pattern 5: Rotating Around Part Axes

**Problem:** Rotate a part around its own axis

**Solution:** Use auto-generated axis references

```yaml
operations:
  rotated_part:
    type: transform
    input: bracket
    transforms:
      - rotate:
          angle: 30
          around: bracket.axis_z  # Rotates around part's vertical axis
```

---

## Primitive-Specific Behaviors

### Box

All 6 faces are distinct and well-defined:
- `face_top`, `face_bottom`: Perpendicular to Z-axis
- `face_left`, `face_right`: Perpendicular to X-axis
- `face_front`, `face_back`: Perpendicular to Y-axis

```yaml
parts:
  box:
    primitive: box
    parameters: {width: 100, height: 50, depth: 30}

  # All face references available
  mounted_on_top:
    translate:
      to: box.face_top

  mounted_on_side:
    translate:
      to: box.face_right
```

### Cylinder

Top and bottom faces are circular, sides are continuous:
- `face_top`, `face_bottom`: Circular end caps
- Side faces (`left`, `right`, `front`, `back`): Points on cylindrical surface

```yaml
parts:
  shaft:
    primitive: cylinder
    parameters: {radius: 10, height: 100}

  # Top/bottom faces are useful
  gear:
    translate:
      to: shaft.face_top

  # Side faces point to surface locations
  mounting_bracket:
    translate:
      to: shaft.face_right  # Right side of cylinder
```

### Sphere

All face references point to surface locations along cardinal directions:

```yaml
parts:
  ball:
    primitive: sphere
    parameters: {radius: 25}

  # Face references are surface points
  connector:
    translate:
      to: ball.face_top  # Point on top of sphere
```

### Cone

Base and apex faces differ in size:
- `face_bottom`: Circular base (large radius)
- `face_top`: Circular apex (small radius or point)

```yaml
parts:
  cone:
    primitive: cone
    parameters: {radius: 30, height: 60}

  # Base and apex have different characteristics
  base_plate:
    translate:
      to: cone.face_bottom

  tip_cap:
    translate:
      to: cone.face_top
```

---

## Best Practices

### 1. Prefer Auto-References Over Manual Coordinates

**❌ Don't do this:**
```yaml
references:
  base_top:
    type: face
    part: base
    selector: ">Z"
    at: center

parts:
  pillar:
    translate:
      to: base_top
```

**✅ Do this instead:**
```yaml
parts:
  pillar:
    translate:
      to: base.face_top  # Cleaner, more direct
```

### 2. Use Local Frame Offsets for Intuitive Positioning

**❌ Don't calculate world coordinates:**
```yaml
# If base is at [0, 0, 0] and has height 10...
pillar:
  translate:
    to: [25, 25, 10]  # Hard-coded, brittle
```

**✅ Use local frame offsets:**
```yaml
pillar:
  translate:
    to:
      from: base.face_top
      offset: [25, 25, 0]  # Relative to face, robust
```

### 3. Name Parts Descriptively

Auto-references use part names, so choose clear names:

**❌ Generic names:**
```yaml
parts:
  part1:
    primitive: box
  part2:
    translate:
      to: part1.face_top  # What is part1?
```

**✅ Descriptive names:**
```yaml
parts:
  base_platform:
    primitive: box
  support_pillar:
    translate:
      to: base_platform.face_top  # Clear intent!
```

### 4. Combine with Parameters for Flexibility

```yaml
parameters:
  platform_height: 10
  pillar_height: 50
  spacing: 20

parts:
  platform:
    primitive: box
    parameters:
      width: 100
      height: '${platform_height}'
      depth: 100

  pillar:
    primitive: cylinder
    parameters:
      radius: 5
      height: '${pillar_height}'
    translate:
      to:
        from: platform.face_top
        offset: ['${spacing}', 0, 0]
```

### 5. Use Named References for Complex Queries

When auto-references aren't sufficient, define custom references:

```yaml
references:
  # Complex selector that auto-references can't provide
  mounting_face:
    type: face
    part: bracket
    selector: ">Z and <X"  # Top-left face
    at: center

parts:
  mounted_part:
    translate:
      to: mounting_face
```

---

## Troubleshooting

### Issue: "Reference not found"

**Error:**
```
Error: Reference 'base.face_top' not found
```

**Solutions:**
1. Check part name spelling
2. Ensure the part is defined before use
3. Verify the face name (must be: `face_top`, `face_bottom`, `face_left`, `face_right`, `face_front`, `face_back`)

### Issue: Unexpected positioning

**Problem:** Part appears in wrong location

**Debugging steps:**
1. Check if you're using `from:` for offsets (required)
2. Verify offset values (positive/negative directions)
3. Remember: offsets are in local frame, not world coordinates
4. Test without offsets first to verify base reference

**Example:**
```yaml
# If this positions incorrectly:
translate:
  to:
    from: base.face_top
    offset: [10, 0, 5]

# Try without offset first:
translate:
  to: base.face_top

# Then add offset incrementally
```

### Issue: Wrong face selected

**Problem:** `face_top` doesn't point where expected

**Cause:** Face directions are based on world axes (±X, ±Y, ±Z), not part orientation after transforms

**Solution:** For transformed parts, define custom references:

```yaml
parts:
  rotated_base:
    primitive: box
    parameters: {width: 100, height: 10, depth: 100}
    transforms:
      - rotate: {angle: 45, axis: Z, origin: [0, 0, 0]}

# After rotation, face_top still refers to +Z face
# If you need the "new top", define a custom reference:
references:
  actual_top:
    type: face
    part: rotated_base
    selector: ">Z"  # Selects the face now pointing up
```

---

## When NOT to Use Auto-References

Auto-references are powerful, but not always appropriate:

### 1. Complex Face Selection

When you need specific faces with complex selectors:

```yaml
# Auto-reference can't express this
references:
  specific_edge_face:
    type: face
    part: bracket
    selector: ">Z and <X and |Y"  # Complex boolean selector
```

### 2. Partial Face References

When you need specific points on a face (not the center):

```yaml
references:
  corner_point:
    type: face
    part: plate
    selector: ">Z"
    at: min  # Minimum corner, not center
```

### 3. After Complex Transforms

When parts have been heavily transformed, canonical directions may not match expectations. Use custom references for clarity.

---

## Examples in Repository

See these example files for auto-references in action:

- `examples/auto_references_box_stack.yaml` - Simple vertical stacking
- `examples/auto_references_cylinder_assembly.yaml` - Cylinder-specific usage
- `examples/auto_references_with_offsets.yaml` - Local frame offset examples
- `examples/auto_references_rotation.yaml` - Rotating around auto-generated axes
- `examples/week5_assembly.yaml` - Complex assembly with auto-references

---

## Summary

Auto-references in TiaCAD v3.0 provide:

✅ **Automatic availability** - No explicit definition needed
✅ **Consistent naming** - Same pattern for all parts
✅ **11 references per part** - Points, faces, and axes
✅ **Local frame offsets** - Intuitive positioning
✅ **Cleaner YAML** - Less boilerplate, more clarity

Master auto-references to build complex assemblies with minimal effort!

---

**Version:** 3.0.0
**Last Updated:** 2025-11-07
**See Also:**
- [YAML_REFERENCE.md](../YAML_REFERENCE.md) - Complete YAML syntax reference
- [MIGRATION_GUIDE_V3.md](MIGRATION_GUIDE_V3.md) - Migrating from v0.3.0
- [V3_IMPLEMENTATION_STATUS.md](V3_IMPLEMENTATION_STATUS.md) - Implementation details
