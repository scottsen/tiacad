# Awesome Guitar Hanger v3.1

A professional, 3D-printable wall-mounted guitar hanger demonstrating TiaCAD v3.0's most powerful features.

![Build Status](https://img.shields.io/badge/build-passing-brightgreen)
![TiaCAD Version](https://img.shields.io/badge/TiaCAD-v3.1-blue)
![3D Printable](https://img.shields.io/badge/3D%20print-ready-orange)

## 🎸 What It Is

A structurally-sound guitar hanger with:
- **Mounting plate** with countersunk screw holes
- **Support beam** with reinforcement brackets
- **Dual cradle arms** with grip texture and 12° upward tilt
- **Complete assembly** - exports as single printable part!

## ✨ TiaCAD Features Demonstrated

### 1. **Auto-References** (v3.0's Killer Feature!)
```yaml
references:
  beam_attach_point:
    type: point
    from: mounting_plate.face_front  # 🎯 Auto-reference!
    offset: [0, 0, 0]
```

All spatial references are derived from auto-references like:
- `mounting_plate.face_front`
- `mounting_plate.center`
- `support_beam.face_top`

No manual coordinate calculation needed!

### 2. **Boolean Unions** (Complete Assembly)
```yaml
structure_assembled:
  type: boolean
  operation: union
  inputs:
    - mounting_plate
    - beam_positioned
    - left_arm_positioned
    - right_arm_positioned
    - left_bracket_positioned
    - right_bracket_positioned
```

**Result**: Single unified part, ready to print!

### 3. **Structural Reinforcements**
Unlike simpler examples, this hanger includes:
- Reinforcement brackets connecting beam to plate
- Proper thickness (15-20mm) for strength
- Grip texture on arms
- Countersunk screw holes

### 4. **Named Spatial References**
```yaml
references:
  left_arm_attach:
    type: point
    from: beam_front_center
    offset: ['${-arm_spacing / 2}', 0, 0]
```

Clean, maintainable positioning with meaningful names.

### 5. **Parametric Design**
21 parameters make it fully customizable:
```yaml
parameters:
  plate_width: 120
  arm_spacing: 75
  arm_tilt_angle: 12
  screw_diameter: 5.0
  # ... and 17 more!
```

## 📐 Specifications

| Component | Dimensions | Details |
|-----------|-----------|---------|
| **Mounting Plate** | 120×90×15mm | With countersunk holes |
| **Support Beam** | 90×12×20mm | Extra thick for rigidity |
| **Cradle Arms** | 40×80×12mm | 75mm spacing, 12° tilt |
| **Brackets** | 30×25×12mm | Structural reinforcement |
| **Screw Holes** | 5mm dia | For #10 wood screws |
| **Countersink** | 10mm dia × 3mm | Flush screw heads |

## 🛠️ Build It

```bash
# Validate the design
tiacad validate examples/awesome_guitar_hanger.yaml

# Build 3MF (multi-material, color support)
tiacad build examples/awesome_guitar_hanger.yaml -o awesome_guitar_hanger.3mf

# Build STL (universal compatibility)
tiacad build examples/awesome_guitar_hanger.yaml -o awesome_guitar_hanger.stl

# Build with stats
tiacad build examples/awesome_guitar_hanger.yaml --stats
```

## 🎨 Design Highlights

### Compared to Original Examples

| Feature | simple_guitar_hanger | guitar_hanger_with_holes | **awesome_guitar_hanger** |
|---------|---------------------|-------------------------|--------------------------|
| **Parts Connected** | ❌ No | ❌ No | ✅ Yes (union) |
| **Auto-references** | ❌ No | ❌ No | ✅ Yes |
| **Screw Holes** | ❌ No | ✅ Yes | ✅ Yes (countersunk) |
| **Reinforcements** | ❌ No | ❌ No | ✅ Yes (brackets) |
| **Grip Texture** | ❌ No | ❌ No | ✅ Yes |
| **3D Printable** | ⚠️ Assembly | ⚠️ Assembly | ✅ Single part |
| **Structural** | ⚠️ Thin | ⚠️ Thin | ✅ Strong |

## 📊 Build Statistics

- **Parts**: 25 (7 primitives + 18 operations)
- **Parameters**: 21 (fully configurable)
- **Operations**: 18 (transforms, unions, differences)
- **Build Time**: ~0.7s
- **Output Size**: 114KB (STL), 173KB (3MF)

## 🔧 Customization Guide

### Change Guitar Size
```yaml
parameters:
  arm_spacing: 75  # Distance between arms (adjust for neck width)
  arm_width: 40    # Contact area width
  arm_length: 80   # How far arms extend
```

### Adjust Wall Mounting
```yaml
parameters:
  screw_diameter: 5.0      # Match your screw size
  screw_spacing: 70        # Horizontal distance between screws
  screw_offset_from_top: 20  # Vertical position
```

### Modify Strength
```yaml
parameters:
  plate_thickness: 15    # Back plate thickness
  beam_thickness: 20     # Support beam thickness
  bracket_height: 25     # Reinforcement bracket size
```

## 🏗️ Assembly Structure

```
awesome_guitar_hanger
└─ structure_with_grips (UNION)
   ├─ structure_assembled (UNION)
   │  ├─ mounting_plate
   │  ├─ beam_positioned
   │  ├─ left_arm_positioned
   │  ├─ right_arm_positioned
   │  ├─ left_bracket_positioned
   │  └─ right_bracket_positioned
   ├─ left_grip_1 (grip texture)
   ├─ left_grip_2
   ├─ right_grip_1
   └─ right_grip_2
└─ SUBTRACT screw holes
   ├─ left_screw_complete
   └─ right_screw_complete
```

## 🎓 Learning Path

This example is perfect for learning:

1. **Auto-references** - See how they simplify positioning
2. **Boolean unions** - Learn to create complete assemblies
3. **Spatial reference chains** - Build complex derived references
4. **Real-world design** - Practical structural considerations
5. **Parametric CAD** - Make designs configurable

## 📝 Code Walkthrough

### Step 1: Define Primitives
```yaml
parts:
  mounting_plate: # 120×90×15mm back plate
  support_beam:   # 90×12×20mm horizontal beam
  cradle_arm:     # 40×80×12mm arm template
  # ... more parts
```

### Step 2: Create Spatial References (Using Auto-References!)
```yaml
references:
  beam_attach_point:
    from: mounting_plate.face_front  # 🎯 Auto-reference!
  left_arm_attach:
    from: beam_front_center
    offset: ['${-arm_spacing / 2}', 0, 0]
```

### Step 3: Position Parts
```yaml
operations:
  beam_positioned:
    type: transform
    input: support_beam
    transforms:
      - translate: beam_attach_point  # Clean named reference!
```

### Step 4: Unite Into Assembly
```yaml
structure_assembled:
  type: boolean
  operation: union
  inputs: [mounting_plate, beam_positioned, ...]  # Single part!
```

### Step 5: Subtract Holes
```yaml
awesome_guitar_hanger:
  type: boolean
  operation: difference
  base: structure_with_grips
  subtract: [left_screw_complete, right_screw_complete]
```

## 🖨️ 3D Printing Tips

- **Material**: PLA or PETG recommended
- **Infill**: 30-50% for strength
- **Layer Height**: 0.2mm standard
- **Supports**: None needed if printed flat
- **Orientation**: Plate against build surface
- **Print Time**: ~6-8 hours (depends on size)

## 🚀 Future Enhancements

Want to improve it further? Try adding:
- Fillets on edges (when TiaCAD supports it)
- Felt padding inserts (separate parts)
- Multiple arm spacings (parameters)
- Wall anchor guides
- Logo or text engraving

## 📚 Related Examples

- `simple_guitar_hanger.yaml` - Basic transforms
- `guitar_hanger_with_holes.yaml` - Boolean difference intro
- `guitar_hanger_named_points.yaml` - Named references (no auto-refs)
- `v3_bracket_mount.yaml` - Another complete assembly example

## 🙏 Credits

Created to demonstrate TiaCAD v3.1 features:
- Auto-references for spatial positioning
- Boolean unions for complete assemblies
- Real-world structural design patterns
- Production-ready 3D printing workflows

---

**Built with**: [TiaCAD](https://github.com/tiainnovation/tiacad) v3.1
**Status**: ✅ Tested, validated, and ready to print!
