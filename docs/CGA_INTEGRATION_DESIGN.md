# Conformal Geometric Algebra (CGA) Integration Design

**Document Version:** 1.0
**Date:** 2025-11-30
**Status:** Research & Design Proposal
**Author:** TiaCAD Architecture Team

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [CGA Mathematical Foundations](#2-cga-mathematical-foundations)
3. [Mapping CGA to TiaCAD Types](#3-mapping-cga-to-tiacad-types)
4. [Architecture Design](#4-architecture-design)
5. [DSL Extensions](#5-dsl-extensions)
6. [Unified Constraint Solving](#6-unified-constraint-solving)
7. [Implementation Roadmap](#7-implementation-roadmap)
8. [Code Examples](#8-code-examples)
9. [Integration Points](#9-integration-points)
10. [References & Resources](#10-references--resources)

---

## 1. Executive Summary

### 1.1 What is Conformal Geometric Algebra?

Conformal Geometric Algebra (CGA) is a mathematical framework that extends traditional geometric algebra (Clifford algebra) to provide a unified representation for all geometric primitives and operations in Euclidean space. For 3D geometry, CGA uses a 5-dimensional algebra GA(4,1) where:

- **Points, lines, planes, circles, and spheres** have natural multivector representations
- **Transformations** (rotations, translations, reflections) are unified as "motors" (versors)
- **Geometric relationships** (intersection, projection, distance, tangency) reduce to algebraic operations
- **Constraints** become linear or bilinear equations in a uniform framework

### 1.2 Why CGA for TiaCAD?

TiaCAD currently uses coordinate-based geometry with CadQuery as the backend. While this works well for simple models, it has limitations:

| Current Approach | CGA Approach |
|------------------|--------------|
| Separate handling for points, lines, circles | Unified multivector representation |
| Special-case code for each constraint type | All constraints are algebraic equations |
| Quaternions OR matrices for rotations | Motors unify all rigid transforms |
| Coordinate-based intersection algorithms | Algebraic intersection via wedge/meet |
| Case-by-case distance formulas | Inner product gives distance directly |

### 1.3 Strategic Fit

CGA aligns perfectly with TiaCAD's evolution:

```
Phase 3 (v3.0) ────────────────────────────────────────────────────────────
  Current: SpatialRef + Frame for position/orientation
  CGA Enhancement: SpatialRef backed by CGA null vectors
                   Frame backed by CGA motors

Phase 4 (v3.1-3.2) ────────────────────────────────────────────────────────
  Current: DAG + Explicit Constraints (planned)
  CGA Enhancement: Constraint equations become uniform CGA algebra
                   Solver works on multivector equations

Phase 5+ ──────────────────────────────────────────────────────────────────
  Current: Constraint Solver (planned)
  CGA Enhancement: Linear algebra on multivector coefficients
                   Unified solver for all geometric constraints
```

### 1.4 Key Benefits

1. **Unified Representation**: Points, lines, planes, circles, spheres all stored as multivectors
2. **Algebraic Constraints**: Every constraint becomes `p ∧ X = 0` or `p · X = 0`
3. **Elegant Transforms**: Motors (spinors) replace quaternion + translation pairs
4. **Symbolic Clarity**: Pure algebra, no coordinate special cases
5. **Future-Proof**: CGA extends to robotics, physics simulation, graphics

---

## 2. CGA Mathematical Foundations

### 2.1 The Conformal Space GA(4,1)

CGA extends 3D Euclidean space R³ to a 5D space with signature (4,1):

```
Basis vectors: e₁, e₂, e₃, e₊, e₋

Where:
  e₁² = e₂² = e₃² = e₊² = +1  (positive signature)
  e₋² = -1                     (negative signature, Minkowski-like)

Derived null vectors:
  e₀ = ½(e₋ - e₊)   # Origin point
  e∞ = e₋ + e₊       # Point at infinity

Properties:
  e₀² = e∞² = 0     # Both are null vectors
  e₀ · e∞ = -1      # Their inner product is -1
```

### 2.2 Embedding Euclidean Points

A 3D Euclidean point **x** = (x, y, z) is embedded into CGA as a null vector:

```
X = x + ½|x|²e∞ + e₀

Where:
  x = x·e₁ + y·e₂ + z·e₃  (3D vector part)
  |x|² = x² + y² + z²

Properties:
  X² = 0  (X is a null vector)
  X · e∞ = -1  (normalized)
```

**Python (clifford library):**
```python
from clifford.g3c import *

def up(xyz):
    """Embed 3D point into CGA (up projection)"""
    x, y, z = xyz
    return eo + x*e1 + y*e2 + z*e3 + 0.5*(x*x + y*y + z*z)*einf

def down(X):
    """Extract 3D point from CGA (down projection)"""
    # Normalize so X · e∞ = -1
    X = X / (-X | einf)
    return (X | e1, X | e2, X | e3)
```

### 2.3 Geometric Objects as Blades

All geometric objects in CGA are represented as blades (wedge products):

| Object | Grade | Construction | OPNS Representation |
|--------|-------|--------------|---------------------|
| **Point** | 1 | `X = up(x)` | Null vector |
| **Point Pair** | 2 | `X₁ ∧ X₂` | Two points |
| **Line** | 3 | `X₁ ∧ X₂ ∧ e∞` | Through 2 points + infinity |
| **Circle** | 3 | `X₁ ∧ X₂ ∧ X₃` | Through 3 points |
| **Plane** | 4 | `X₁ ∧ X₂ ∧ X₃ ∧ e∞` | Through 3 points + infinity |
| **Sphere** | 4 | `X₁ ∧ X₂ ∧ X₃ ∧ X₄` | Through 4 points |

**Alternative IPNS (dual) representations:**

```python
# Sphere (IPNS): center C, radius r
S = C - 0.5 * r**2 * einf

# Plane (IPNS): normal n, distance d from origin
P = n + d * einf

# Line (IPNS): direction d, moment m
L = d + m * einf

# Circle: intersection of sphere and plane
C = S ∧ P
```

### 2.4 Inner Product: Distance and Incidence

The inner product (·) encodes geometric relationships:

```python
# Distance between two points (squared)
d² = -2 * (X₁ · X₂)

# Point on sphere/plane/circle test
X · S = 0  # X lies on sphere S
X · P = 0  # X lies in plane P
X · C = 0  # X lies on circle C

# Perpendicularity
L₁ · L₂ = 0  # Lines are perpendicular

# Tangency
L · S = 0  # Line tangent to sphere
```

### 2.5 Wedge Product: Construction and Intersection

The wedge product (∧) constructs higher-dimensional objects:

```python
# Line through two points
L = X₁ ∧ X₂ ∧ einf

# Circle through three points
C = X₁ ∧ X₂ ∧ X₃

# Plane through three points
P = X₁ ∧ X₂ ∧ X₃ ∧ einf
```

The **meet** operation (regressive product) computes intersections:

```python
# Intersection of two planes
L = P₁ ∨ P₂  # Line of intersection

# Intersection of plane and sphere
C = P ∨ S    # Circle of intersection

# Intersection of two spheres
C = S₁ ∨ S₂  # Circle of intersection

# Intersection of line and plane
X = L ∨ P    # Point of intersection
```

### 2.6 Motors: Unified Rigid Transforms

Motors (also called versors or spinors) represent rigid transformations in SE(3):

```python
# Rotation by angle θ around axis n (through origin)
R = exp(-θ/2 * n)  # n is a bivector (e.g., e12 for XY plane)
# Apply: X' = R X R̃  (sandwich product, R̃ is reverse)

# Translation by vector t
T = exp(-t/2 * einf)  # t = tx*e1 + ty*e2 + tz*e3
T = 1 + 0.5 * t ∧ einf  # Equivalent form
# Apply: X' = T X T̃

# Motor (combined rotation + translation)
M = T * R  # Translation after rotation
# Apply: X' = M X M̃

# Motor interpolation (sclerp)
M_t = M₁ * (M₁̃ * M₂)^t  # t ∈ [0, 1]
```

**Key property:** Motors work on ALL geometric objects, not just points:

```python
# Transform a line
L' = M L M̃

# Transform a circle
C' = M C M̃

# Transform a sphere
S' = M S M̃

# Everything uses the same formula!
```

---

## 3. Mapping CGA to TiaCAD Types

### 3.1 Type Correspondence

TiaCAD's current types map naturally to CGA representations:

| TiaCAD Type | Current Implementation | CGA Backing |
|-------------|------------------------|-------------|
| `SpatialRef` (point) | `np.ndarray[3]` position | Null vector (grade-1) |
| `SpatialRef` (face) | position + orientation | Null vector + bivector normal |
| `SpatialRef` (edge) | position + direction + tangent | Null vector + bivector direction |
| `Frame` | origin + 3 axis vectors | Motor from reference frame |
| Point | `(x, y, z)` tuple | `up(x, y, z)` null vector |
| Line | 2 points | Grade-3 blade `X₁ ∧ X₂ ∧ e∞` |
| Plane | point + normal | Grade-4 blade (IPNS: `n + d·e∞`) |
| Circle | 3 points or center+radius+normal | Grade-3 blade `X₁ ∧ X₂ ∧ X₃` |
| Sphere | center + radius | Grade-4 blade (IPNS: `C - ½r²e∞`) |
| Transform | translate/rotate sequences | Motor (spinor) |

### 3.2 Proposed CGA-Backed Types

```python
# tiacad_core/cga/types.py

from clifford.g3c import *
import numpy as np
from dataclasses import dataclass
from typing import Optional, Literal

@dataclass
class CGAPoint:
    """CGA representation of a 3D point."""
    mv: MultiVector  # Null vector representation

    @classmethod
    def from_coords(cls, x: float, y: float, z: float) -> 'CGAPoint':
        return cls(up(e1*x + e2*y + e3*z))

    def to_coords(self) -> tuple[float, float, float]:
        vec = down(self.mv)
        return (float(vec|e1), float(vec|e2), float(vec|e3))

    def distance_to(self, other: 'CGAPoint') -> float:
        return np.sqrt(-2 * float(self.mv | other.mv))


@dataclass
class CGALine:
    """CGA representation of a 3D line."""
    mv: MultiVector  # Grade-3 blade

    @classmethod
    def from_two_points(cls, p1: CGAPoint, p2: CGAPoint) -> 'CGALine':
        return cls((p1.mv ^ p2.mv ^ einf).normal())

    @classmethod
    def from_point_direction(cls, point: CGAPoint, direction: tuple) -> 'CGALine':
        dx, dy, dz = direction
        p2 = CGAPoint.from_coords(
            point.to_coords()[0] + dx,
            point.to_coords()[1] + dy,
            point.to_coords()[2] + dz
        )
        return cls.from_two_points(point, p2)


@dataclass
class CGAPlane:
    """CGA representation of a 3D plane."""
    mv: MultiVector  # Grade-4 blade (OPNS) or grade-1 (IPNS)

    @classmethod
    def from_point_normal(cls, point: CGAPoint, normal: tuple) -> 'CGAPlane':
        nx, ny, nz = normal
        n = e1*nx + e2*ny + e3*nz
        n = n / np.sqrt(float(n|n))  # Normalize
        d = float(point.mv | n)
        return cls(n + d * einf)

    @classmethod
    def from_three_points(cls, p1: CGAPoint, p2: CGAPoint, p3: CGAPoint) -> 'CGAPlane':
        return cls((p1.mv ^ p2.mv ^ p3.mv ^ einf).normal())


@dataclass
class CGACircle:
    """CGA representation of a 3D circle."""
    mv: MultiVector  # Grade-3 blade

    @classmethod
    def from_three_points(cls, p1: CGAPoint, p2: CGAPoint, p3: CGAPoint) -> 'CGACircle':
        return cls((p1.mv ^ p2.mv ^ p3.mv).normal())

    @classmethod
    def from_center_radius_normal(cls, center: CGAPoint, radius: float,
                                   normal: tuple) -> 'CGACircle':
        # Circle is intersection of sphere and plane
        S = CGASphere.from_center_radius(center, radius)
        P = CGAPlane.from_point_normal(center, normal)
        return cls((S.mv ^ P.mv).normal())


@dataclass
class CGASphere:
    """CGA representation of a 3D sphere."""
    mv: MultiVector  # Grade-4 blade (OPNS) or grade-1 (IPNS)

    @classmethod
    def from_center_radius(cls, center: CGAPoint, radius: float) -> 'CGASphere':
        # IPNS representation
        return cls(center.mv - 0.5 * radius**2 * einf)

    @classmethod
    def from_four_points(cls, p1: CGAPoint, p2: CGAPoint,
                         p3: CGAPoint, p4: CGAPoint) -> 'CGASphere':
        return cls((p1.mv ^ p2.mv ^ p3.mv ^ p4.mv).normal())


@dataclass
class CGAMotor:
    """CGA representation of a rigid transformation (SE(3))."""
    mv: MultiVector  # Even-grade multivector (spinor)

    @classmethod
    def identity(cls) -> 'CGAMotor':
        return cls(1 + 0*e1)  # Scalar 1

    @classmethod
    def translation(cls, tx: float, ty: float, tz: float) -> 'CGAMotor':
        t = e1*tx + e2*ty + e3*tz
        return cls(1 + 0.5 * (t ^ einf))

    @classmethod
    def rotation(cls, axis: tuple, angle_rad: float) -> 'CGAMotor':
        ax, ay, az = axis
        norm = np.sqrt(ax**2 + ay**2 + az**2)
        ax, ay, az = ax/norm, ay/norm, az/norm

        # Bivector for rotation plane (dual of axis)
        B = ax*e23 + ay*e31 + az*e12

        c, s = np.cos(angle_rad/2), np.sin(angle_rad/2)
        return cls(c - s*B)

    @classmethod
    def from_axis_angle_point(cls, axis: tuple, angle_rad: float,
                              point: tuple) -> 'CGAMotor':
        """Rotation around an axis through a point (not origin)."""
        R = cls.rotation(axis, angle_rad)
        T1 = cls.translation(-point[0], -point[1], -point[2])
        T2 = cls.translation(point[0], point[1], point[2])
        return T2 * R * T1  # Translate to origin, rotate, translate back

    def apply(self, obj: MultiVector) -> MultiVector:
        """Apply motor to any CGA object (sandwich product)."""
        return self.mv * obj * ~self.mv  # M X M̃

    def __mul__(self, other: 'CGAMotor') -> 'CGAMotor':
        """Compose two motors."""
        return CGAMotor(self.mv * other.mv)

    def inverse(self) -> 'CGAMotor':
        """Inverse motor (reverse)."""
        return CGAMotor(~self.mv)

    def interpolate(self, other: 'CGAMotor', t: float) -> 'CGAMotor':
        """Smooth interpolation between motors (sclerp)."""
        delta = ~self.mv * other.mv
        # Extract rotor and translator components, interpolate each
        # (Full implementation requires logarithm/exponential)
        return CGAMotor(self.mv * (delta ** t))
```

### 3.3 SpatialRef Integration

The existing `SpatialRef` can be backed by CGA:

```python
# tiacad_core/geometry/spatial_references.py (enhanced)

@dataclass
class SpatialRef:
    position: np.ndarray           # (3,) always present
    orientation: Optional[np.ndarray]  # (3,) optional normal/direction
    tangent: Optional[np.ndarray]      # (3,) optional tangent (for edges)
    ref_type: Literal['point', 'face', 'edge', 'axis']

    # NEW: CGA backing (lazily computed)
    _cga_point: Optional[CGAPoint] = None
    _cga_motor: Optional[CGAMotor] = None  # For frame transforms

    @property
    def cga_point(self) -> CGAPoint:
        if self._cga_point is None:
            self._cga_point = CGAPoint.from_coords(*self.position)
        return self._cga_point

    def frame(self) -> 'Frame':
        """Generate local coordinate system."""
        # Existing implementation...

    def cga_frame_motor(self) -> CGAMotor:
        """Get motor that transforms from world to local frame."""
        if self._cga_motor is None:
            frame = self.frame()
            # Build motor from frame alignment
            self._cga_motor = CGAMotor.from_frame(frame)
        return self._cga_motor

    def offset(self, delta, in_local_frame=True) -> 'SpatialRef':
        """Offset in local frame using CGA motor."""
        if in_local_frame:
            motor = self.cga_frame_motor()
            delta_point = CGAPoint.from_coords(*delta)
            world_delta = motor.apply(delta_point.mv)
            new_pos = self.position + np.array(down(world_delta))
        else:
            new_pos = self.position + np.array(delta)

        return SpatialRef(
            position=new_pos,
            orientation=self.orientation,
            tangent=self.tangent,
            ref_type=self.ref_type
        )
```

---

## 4. Architecture Design

### 4.1 Module Structure

```
tiacad_core/
├── cga/                          # NEW: CGA Engine
│   ├── __init__.py
│   ├── algebra.py                # Core CGA operations (wedge, meet, inner)
│   ├── types.py                  # CGAPoint, CGALine, CGAPlane, etc.
│   ├── motor.py                  # Motor (SE(3)) operations
│   ├── constraints.py            # CGA-based constraint equations
│   └── solver.py                 # Constraint solver using CGA algebra
│
├── geometry/
│   ├── base.py                   # GeometryBackend (unchanged)
│   ├── cadquery_backend.py       # CadQuery implementation (unchanged)
│   ├── spatial_references.py     # ENHANCED: CGA backing for SpatialRef
│   └── cga_backend.py            # NEW: Pure CGA geometry operations
│
├── parser/
│   ├── tiacad_parser.py          # ENHANCED: CGA type support
│   ├── constraint_builder.py     # NEW: Parse constraints to CGA equations
│   └── ...
│
└── solver/                       # NEW: Constraint solving
    ├── __init__.py
    ├── equation_system.py        # Build system of CGA equations
    ├── gauss_newton.py           # Numeric solver for CGA multivectors
    └── symbolic_solver.py        # Symbolic CGA equation solving
```

### 4.2 Dependency Graph

```
                          ┌─────────────────┐
                          │  clifford lib   │ (external)
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │  cga/algebra.py │
                          └────────┬────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
     ┌────────▼────────┐  ┌────────▼────────┐  ┌────────▼────────┐
     │  cga/types.py   │  │  cga/motor.py   │  │cga/constraints.py
     └────────┬────────┘  └────────┬────────┘  └────────┬────────┘
              │                    │                    │
              └────────────────────┼────────────────────┘
                                   │
                   ┌───────────────┼───────────────┐
                   │               │               │
          ┌────────▼────────┐      │     ┌────────▼────────┐
          │ spatial_refs.py │      │     │ cga/solver.py   │
          └────────┬────────┘      │     └────────┬────────┘
                   │               │               │
          ┌────────▼────────┐      │     ┌────────▼────────┐
          │ spatial_resolver│      │     │constraint_builder│
          └────────┬────────┘      │     └────────┬────────┘
                   │               │               │
                   └───────────────┼───────────────┘
                                   │
                          ┌────────▼────────┐
                          │  tiacad_parser  │
                          └─────────────────┘
```

### 4.3 Integration with CadQuery Backend

CGA and CadQuery serve different purposes:

- **CadQuery**: Creates actual 3D solid geometry (BREP), exports to STL/STEP
- **CGA**: Handles geometric relationships, constraints, transformations

They work together:

```python
class CadQueryBackend(GeometryBackend):

    def create_loft(self, profiles: List[CGACircle], ruled: bool) -> Workplane:
        """Loft between CGA circles."""
        # Convert CGA circles to CadQuery sketch profiles
        cq_profiles = []
        for circle in profiles:
            center, radius, normal = circle.extract_params()
            # Create CadQuery sketch at position
            wp = cq.Workplane().center(center[0], center[1])
            wp = wp.workplane(offset=center[2])
            wp = wp.circle(radius)
            cq_profiles.append(wp)

        # Use CadQuery loft
        return cq.Workplane().loft(cq_profiles, ruled=ruled)

    def apply_motor(self, geom: Workplane, motor: CGAMotor) -> Workplane:
        """Apply CGA motor transformation to CadQuery geometry."""
        # Extract rotation axis, angle, and translation from motor
        axis, angle, translation = motor.decompose()

        # Apply as CadQuery operations
        result = geom
        if angle != 0:
            result = result.rotate(
                axisStartPoint=(0, 0, 0),
                axisEndPoint=axis,
                angleDegrees=np.degrees(angle)
            )
        if np.any(translation != 0):
            result = result.translate(tuple(translation))

        return result
```

---

## 5. DSL Extensions

### 5.1 CGA-Aware YAML Syntax

```yaml
# tiacad v4.0+ with CGA support

metadata:
  name: CGA-Enhanced Model
  cga_enabled: true  # Enable CGA engine

parameters:
  base_w: 100
  hole_r: 10

# Parts (unchanged, but internally CGA-backed)
parts:
  base:
    primitive: box
    parameters:
      width: ${base_w}
      height: 60
      depth: 10

# Explicit CGA geometric objects (NEW)
geometry:
  points:
    p1: [0, 0, 0]
    p2: [${base_w}, 0, 0]
    p3: [${base_w}, 60, 0]

  lines:
    edge1:
      through: [p1, p2]  # Line through two points

    axis:
      point: p1
      direction: [0, 0, 1]  # Z-axis through p1

  planes:
    ground:
      point: p1
      normal: [0, 0, 1]

    side:
      through: [p1, p2, [0, 0, 10]]  # Three points define plane

  circles:
    hole_profile:
      center: [50, 30, 10]
      radius: ${hole_r}
      normal: [0, 0, 1]

    fitted:
      through: [p1, p2, p3]  # Circle through 3 points

  spheres:
    ball:
      center: [50, 30, 50]
      radius: 20

# Motors for transformations (NEW)
motors:
  rotate_45:
    type: rotation
    axis: [0, 0, 1]
    angle: 45deg
    origin: base.center

  move_up:
    type: translation
    vector: [0, 0, 20]

  combined:
    compose: [rotate_45, move_up]  # Motor composition

  sweep_motor:
    interpolate:
      from: identity
      to: rotate_45
      steps: 10  # For sweep operations

# Constraints (CGA-powered)
constraints:
  # Point on plane
  p3_on_ground:
    type: incidence
    point: p3
    object: ground
    # Internally: p3 · ground = 0

  # Perpendicular lines
  edges_perpendicular:
    type: perpendicular
    objects: [edge1, axis]
    # Internally: edge1 · axis = 0

  # Tangent line to sphere
  edge_tangent:
    type: tangent
    line: edge1
    object: ball
    # Internally: edge1 · ball = 0

  # Fixed distance
  point_distance:
    type: distance
    from: p1
    to: p2
    value: ${base_w}
    # Internally: -2(p1 · p2) = base_w²

  # Circle in plane
  circle_coplanar:
    type: coplanar
    object: hole_profile
    plane: ground
    # Internally: hole_profile ∧ ground = 0

# Operations using motors
operations:
  rotated_part:
    type: transform
    input: base
    motor: rotate_45  # Apply named motor

  lofted_shape:
    type: loft
    circles: [hole_profile, fitted]  # CGA circles
    ruled: false
    motor_path: sweep_motor  # Optional motor interpolation

  projected:
    type: project
    input: p3
    onto: ground
    # CGA projection: p3 - (p3 · ground) * ground / ground²
```

### 5.2 Inline CGA Operators

For advanced users, expose CGA algebra directly:

```yaml
# Advanced CGA expressions (v4.1+)
geometry:
  # Wedge product for construction
  line_from_points:
    expr: "p1 ^ p2 ^ einf"  # CGA expression

  # Meet for intersection
  intersection:
    expr: "plane1 & plane2"  # & is meet operator

  # Projection
  projected_point:
    expr: "project(p1, onto=plane1)"

constraints:
  # Direct CGA equation
  custom:
    cga_expr: "(p1 ^ line1) == 0"  # Point on line
```

---

## 6. Unified Constraint Solving

### 6.1 Constraints as CGA Equations

Every geometric constraint reduces to a multivector equation:

| Constraint | YAML | CGA Equation |
|------------|------|--------------|
| Point on line | `incidence: {point: p, object: L}` | `p ∧ L = 0` |
| Point on plane | `incidence: {point: p, object: P}` | `p · P = 0` |
| Point on sphere | `incidence: {point: p, object: S}` | `p · S = 0` |
| Point on circle | `incidence: {point: p, object: C}` | `p · C = 0` |
| Perpendicular lines | `perpendicular: {objects: [L1, L2]}` | `L1 · L2 = 0` |
| Parallel lines | `parallel: {objects: [L1, L2]}` | `L1 ∧ L2 ∧ e∞ = 0` |
| Tangent line-sphere | `tangent: {line: L, object: S}` | `L · S = 0` |
| Distance | `distance: {from: p1, to: p2, value: d}` | `-2(p1 · p2) = d²` |
| Coplanar circle | `coplanar: {object: C, plane: P}` | `C · P = 0` |
| Concentric spheres | `concentric: {objects: [S1, S2]}` | `S1 ∧ S2 = 0` (dual) |
| Flush faces | `flush: {faces: [F1, F2]}` | `F1 · F2 = ±1` |
| Coaxial | `coaxial: {axes: [A1, A2]}` | `A1 ∧ A2 = 0` |

### 6.2 Solver Architecture

```python
# tiacad_core/solver/equation_system.py

from dataclasses import dataclass
from typing import List, Dict
from clifford.g3c import *
import numpy as np

@dataclass
class CGAVariable:
    """A geometric variable to be solved."""
    name: str
    initial_value: MultiVector
    dof: int  # Degrees of freedom (3 for point, etc.)

@dataclass
class CGAConstraint:
    """A constraint as a CGA equation."""
    name: str
    equation: callable  # Function(variables) -> MultiVector
    target: float = 0.0  # Usually 0 for incidence


class CGAConstraintSystem:
    """System of CGA constraint equations."""

    def __init__(self):
        self.variables: Dict[str, CGAVariable] = {}
        self.constraints: List[CGAConstraint] = []

    def add_variable(self, name: str, initial: MultiVector, dof: int = 3):
        self.variables[name] = CGAVariable(name, initial, dof)

    def add_point_on_line(self, point_name: str, line_name: str):
        """Add constraint: point lies on line."""
        def equation(vars):
            p = vars[point_name]
            L = vars[line_name]
            return p ^ L  # Should equal 0

        self.constraints.append(CGAConstraint(
            f"{point_name}_on_{line_name}",
            equation
        ))

    def add_point_on_plane(self, point_name: str, plane_name: str):
        """Add constraint: point lies in plane."""
        def equation(vars):
            p = vars[point_name]
            P = vars[plane_name]
            return p | P  # Inner product, should equal 0

        self.constraints.append(CGAConstraint(
            f"{point_name}_on_{plane_name}",
            equation
        ))

    def add_distance(self, p1_name: str, p2_name: str, distance: float):
        """Add constraint: fixed distance between points."""
        def equation(vars):
            p1 = vars[p1_name]
            p2 = vars[p2_name]
            return -2 * (p1 | p2) - distance**2  # Should equal 0

        self.constraints.append(CGAConstraint(
            f"distance_{p1_name}_{p2_name}",
            equation
        ))

    def add_perpendicular(self, L1_name: str, L2_name: str):
        """Add constraint: lines are perpendicular."""
        def equation(vars):
            L1 = vars[L1_name]
            L2 = vars[L2_name]
            return L1 | L2  # Should equal 0

        self.constraints.append(CGAConstraint(
            f"perp_{L1_name}_{L2_name}",
            equation
        ))

    def residual(self, var_values: Dict[str, MultiVector]) -> np.ndarray:
        """Compute residual vector for all constraints."""
        residuals = []
        for constraint in self.constraints:
            result = constraint.equation(var_values)
            # Extract scalar components as residual
            residuals.append(self._mv_to_scalar(result))
        return np.array(residuals)

    def jacobian(self, var_values: Dict[str, MultiVector],
                 epsilon: float = 1e-8) -> np.ndarray:
        """Compute Jacobian numerically."""
        n_constraints = len(self.constraints)
        n_dofs = sum(v.dof for v in self.variables.values())

        J = np.zeros((n_constraints, n_dofs))

        dof_idx = 0
        for var_name, var in self.variables.items():
            for i in range(var.dof):
                # Perturb this DOF
                perturbed = var_values.copy()
                perturbed[var_name] = self._perturb_mv(
                    var_values[var_name], i, epsilon
                )

                r_plus = self.residual(perturbed)
                r_minus = self.residual(var_values)

                J[:, dof_idx] = (r_plus - r_minus) / epsilon
                dof_idx += 1

        return J

    def solve(self, max_iter: int = 100, tol: float = 1e-10) -> Dict[str, MultiVector]:
        """Solve using Gauss-Newton iteration."""
        var_values = {name: var.initial_value
                      for name, var in self.variables.items()}

        for iteration in range(max_iter):
            r = self.residual(var_values)

            if np.linalg.norm(r) < tol:
                print(f"Converged in {iteration} iterations")
                return var_values

            J = self.jacobian(var_values)

            # Gauss-Newton step: delta = -(J^T J)^(-1) J^T r
            delta = -np.linalg.lstsq(J, r, rcond=None)[0]

            # Apply delta to variables
            dof_idx = 0
            for var_name, var in self.variables.items():
                var_delta = delta[dof_idx:dof_idx + var.dof]
                var_values[var_name] = self._apply_delta(
                    var_values[var_name], var_delta
                )
                dof_idx += var.dof

        raise RuntimeError(f"Failed to converge after {max_iter} iterations")
```

### 6.3 Example: Solving a Constrained Assembly

```python
# Example: Four points forming a square with constraints

system = CGAConstraintSystem()

# Variables: 4 points with initial guesses
system.add_variable("p1", up([0, 0, 0]))
system.add_variable("p2", up([10, 0, 0]))
system.add_variable("p3", up([10, 10, 0]))
system.add_variable("p4", up([0, 10, 0]))

# Constraints:
# - All points in XY plane
xy_plane = e1^e2^e3^einf  # XY plane through origin
for pname in ["p1", "p2", "p3", "p4"]:
    system.add_point_on_plane(pname, "xy_plane")

# - Equal side lengths (50mm)
system.add_distance("p1", "p2", 50)
system.add_distance("p2", "p3", 50)
system.add_distance("p3", "p4", 50)
system.add_distance("p4", "p1", 50)

# - Right angles (perpendicular edges)
system.add_perpendicular("L_12", "L_23")  # Edges as lines
system.add_perpendicular("L_23", "L_34")
system.add_perpendicular("L_34", "L_41")

# Solve
solution = system.solve()

# Extract coordinates
for name in ["p1", "p2", "p3", "p4"]:
    print(f"{name}: {down(solution[name])}")
```

---

## 7. Implementation Roadmap

### 7.1 Phase Overview

```
Phase CGA-1: Foundation (4 weeks)
├── Week 1-2: Core CGA types and algebra
└── Week 3-4: Integration with SpatialRef

Phase CGA-2: Constraint Engine (6 weeks)
├── Week 5-6: Constraint equation framework
├── Week 7-8: Gauss-Newton solver
└── Week 9-10: YAML constraint syntax

Phase CGA-3: Advanced Features (4 weeks)
├── Week 11-12: Motor interpolation, sweeps
└── Week 13-14: Loft enhancements, optimization

Phase CGA-4: Polish & Docs (2 weeks)
├── Week 15: Examples, testing
└── Week 16: Documentation, release
```

### 7.2 Phase CGA-1: Foundation

**Goal:** Establish CGA types and integrate with existing SpatialRef

**Week 1-2: Core CGA Types**
- [ ] Add `clifford` dependency to pyproject.toml
- [ ] Create `tiacad_core/cga/__init__.py`
- [ ] Implement `cga/algebra.py` (wrapper around clifford)
- [ ] Implement `cga/types.py`:
  - `CGAPoint`, `CGALine`, `CGAPlane`
  - `CGACircle`, `CGASphere`
  - `CGAMotor`
- [ ] Write 50+ unit tests for CGA types
- [ ] Verify algebraic identities (p² = 0, distance formula, etc.)

**Week 3-4: SpatialRef Integration**
- [ ] Add `_cga_point` property to `SpatialRef`
- [ ] Add `cga_frame_motor()` method to `SpatialRef`
- [ ] Update `Frame` to support CGA motor backing
- [ ] Update `offset()` to optionally use CGA
- [ ] Write 30+ integration tests
- [ ] Performance benchmarks (CGA vs current)

**Deliverables:**
- ✅ CGA types working with clifford backend
- ✅ SpatialRef enhanced with CGA backing
- ✅ 80+ new tests, all passing
- ✅ No performance regression

### 7.3 Phase CGA-2: Constraint Engine

**Goal:** Enable constraint specification and solving

**Week 5-6: Constraint Framework**
- [ ] Create `cga/constraints.py`:
  - Constraint base class
  - Incidence, distance, perpendicular, parallel
  - Tangent, coplanar, concentric
- [ ] Create `solver/equation_system.py`
- [ ] Implement residual computation
- [ ] Write 40+ constraint tests

**Week 7-8: Solver Implementation**
- [ ] Implement Jacobian computation (numerical)
- [ ] Implement Gauss-Newton iteration
- [ ] Add convergence detection
- [ ] Add under/over-constrained detection
- [ ] Write 30+ solver tests

**Week 9-10: YAML Integration**
- [ ] Create `parser/constraint_builder.py`
- [ ] Update JSON schema for `constraints:` section
- [ ] Update `TiaCADParser` to parse constraints
- [ ] Integrate solver into build pipeline
- [ ] Write 25+ integration tests

**Deliverables:**
- ✅ Constraint equations as CGA algebra
- ✅ Working Gauss-Newton solver
- ✅ YAML `constraints:` section parsed
- ✅ 95+ new tests, all passing

### 7.4 Phase CGA-3: Advanced Features

**Goal:** Motors for transforms, enhanced loft/sweep

**Week 11-12: Motor Operations**
- [ ] Implement motor decomposition (axis, angle, translation)
- [ ] Implement motor interpolation (sclerp)
- [ ] Update `TransformTracker` to use motors
- [ ] Add `motors:` section to YAML
- [ ] Create sweep-by-motor operation
- [ ] Write 35+ tests

**Week 13-14: Enhanced Geometry**
- [ ] Loft between CGA circles
- [ ] Sweep along motor path
- [ ] Project point/line onto plane (CGA-based)
- [ ] Intersection operations (meet)
- [ ] Write 30+ tests

**Deliverables:**
- ✅ Motor-based transforms working
- ✅ Enhanced loft/sweep operations
- ✅ 65+ new tests, all passing

### 7.5 Phase CGA-4: Polish

**Goal:** Documentation, examples, release preparation

**Week 15: Testing & Examples**
- [ ] Create 10+ example YAML files using CGA features
- [ ] Full integration test suite
- [ ] Performance optimization
- [ ] Edge case handling

**Week 16: Documentation**
- [ ] Update YAML_REFERENCE.md with CGA sections
- [ ] Create CGA_TUTORIAL.md
- [ ] Update ARCHITECTURE docs
- [ ] Release notes for v4.0

**Deliverables:**
- ✅ 10+ production examples
- ✅ Complete documentation
- ✅ TiaCAD v4.0 release ready

### 7.6 Success Metrics

| Metric | Target |
|--------|--------|
| New tests | 250+ |
| Test coverage | >95% |
| Constraint types | 10+ |
| Example files | 10+ |
| Performance overhead | <10% |
| Solver convergence | 99%+ on valid systems |

---

## 8. Code Examples

### 8.1 Simple Constraint Example

```yaml
# example: bracket_mount.yaml
# A bracket constrained to sit flush on a base

parameters:
  base_size: [100, 60, 10]
  bracket_size: [30, 40, 5]
  mount_offset: [20, 15, 0]  # Position on base surface

parts:
  base:
    primitive: box
    parameters:
      width: ${base_size[0]}
      height: ${base_size[1]}
      depth: ${base_size[2]}

  bracket:
    primitive: box
    parameters:
      width: ${bracket_size[0]}
      height: ${bracket_size[1]}
      depth: ${bracket_size[2]}

# CGA-based constraints
constraints:
  # Bracket bottom face sits on base top face
  flush_mount:
    type: flush
    face_a: bracket.face_bottom
    face_b: base.face_top
    gap: 0

  # Bracket position on base surface
  bracket_position:
    type: offset_in_plane
    point: bracket.center
    plane: base.face_top
    offset: ${mount_offset}

# No explicit transforms needed - solver computes positions!
operations:
  final:
    type: union
    inputs: [base, bracket]
```

### 8.2 Motor-Based Animation

```yaml
# example: rotating_assembly.yaml
# Parts that rotate around a shared axis

parameters:
  rotation_angle: 45  # degrees
  arm_length: 80

parts:
  hub:
    primitive: cylinder
    parameters:
      radius: 15
      height: 20

  arm:
    primitive: box
    parameters:
      width: ${arm_length}
      height: 10
      depth: 5

motors:
  arm_rotation:
    type: rotation
    axis: [0, 0, 1]
    angle: ${rotation_angle}deg
    origin: hub.center

  # For animation/sweep, interpolate the motor
  arm_sweep:
    interpolate:
      from: identity
      to:
        type: rotation
        axis: [0, 0, 1]
        angle: 360deg
        origin: hub.center
      steps: 36  # 10° increments

operations:
  positioned_arm:
    type: transform
    input: arm
    transforms:
      - translate:
          to:
            from: hub.center
            offset: [30, 0, 0]  # Offset from hub

  rotated_arm:
    type: motor_transform
    input: positioned_arm
    motor: arm_rotation

  swept_volume:
    type: sweep_by_motor
    input: arm
    motor: arm_sweep
    union_result: true  # Union all positions
```

### 8.3 Circle Loft with CGA

```yaml
# example: transition_duct.yaml
# Loft from rectangle to circle using CGA circles

parameters:
  length: 100
  inlet_size: [40, 40]
  outlet_radius: 15

geometry:
  circles:
    # Inlet: approximate rectangle with 4-arc superellipse
    inlet_circle:
      type: superellipse
      center: [0, 0, 0]
      a: ${inlet_size[0] / 2}
      b: ${inlet_size[1] / 2}
      n: 4  # Squareness parameter
      normal: [0, 0, 1]

    # Outlet: true circle
    outlet_circle:
      center: [0, 0, ${length}]
      radius: ${outlet_radius}
      normal: [0, 0, 1]

    # Intermediate for smoother transition
    mid_circle:
      center: [0, 0, ${length / 2}]
      radius: ${(inlet_size[0]/2 + outlet_radius) / 2}
      normal: [0, 0, 1]

operations:
  duct:
    type: loft
    profiles: [inlet_circle, mid_circle, outlet_circle]
    ruled: false
    # CGA ensures smooth interpolation between circles
```

### 8.4 Fully Constrained Assembly

```yaml
# example: four_bar_linkage.yaml
# A planar four-bar mechanism with geometric constraints

parameters:
  crank_length: 30
  coupler_length: 60
  rocker_length: 40
  ground_length: 50
  crank_angle: 45  # Input angle in degrees

geometry:
  points:
    # Ground pivots (fixed)
    O2: [0, 0, 0]           # Crank pivot (ground)
    O4: [${ground_length}, 0, 0]  # Rocker pivot (ground)

    # Moving pivots (solved)
    A:
      initial: [${crank_length}, 0, 0]  # Will be solved
    B:
      initial: [${crank_length}, ${coupler_length}, 0]  # Will be solved

constraints:
  # Crank angle constraint (input)
  crank_angle:
    type: angle
    from: O2
    to: A
    reference: [1, 0, 0]  # X-axis
    value: ${crank_angle}deg

  # Link length constraints
  crank_len:
    type: distance
    from: O2
    to: A
    value: ${crank_length}

  coupler_len:
    type: distance
    from: A
    to: B
    value: ${coupler_length}

  rocker_len:
    type: distance
    from: B
    to: O4
    value: ${rocker_length}

  # All points in XY plane
  planar:
    type: coplanar
    points: [O2, O4, A, B]
    plane:
      point: [0, 0, 0]
      normal: [0, 0, 1]

parts:
  crank:
    primitive: box
    parameters:
      width: ${crank_length}
      height: 5
      depth: 3
    # Position/rotation computed from solved A

  coupler:
    primitive: box
    parameters:
      width: ${coupler_length}
      height: 5
      depth: 3
    # Position/rotation computed from solved A, B

  rocker:
    primitive: box
    parameters:
      width: ${rocker_length}
      height: 5
      depth: 3
    # Position/rotation computed from solved B, O4

operations:
  link_crank:
    type: transform
    input: crank
    motor:
      from_points: [O2, A]  # Motor aligns link to these solved points

  link_coupler:
    type: transform
    input: coupler
    motor:
      from_points: [A, B]

  link_rocker:
    type: transform
    input: rocker
    motor:
      from_points: [O4, B]

  assembly:
    type: union
    inputs: [link_crank, link_coupler, link_rocker]
```

---

## 9. Integration Points

### 9.1 Morphogen Integration

CGA provides perfect interop with Morphogen's field-based geometry:

```python
# Morphogen agent positions → CGA points
agent_positions = morphogen.get_agent_positions()  # List of (x,y,z)
cga_points = [CGAPoint.from_coords(*p) for p in agent_positions]

# CGA circle fitting through agent cluster
if len(cga_points) >= 3:
    fitted_circle = CGACircle.from_three_points(
        cga_points[0], cga_points[1], cga_points[2]
    )

# TiaCAD can then loft/sweep using this circle
tiacad_model.add_circle("fitted", fitted_circle)
tiacad_model.add_loft(["base_circle", "fitted"])
```

### 9.2 Philbrick Integration

CGA motors enable smooth analog-style interpolation:

```python
# Philbrick computes continuous motor path
def philbrick_motor_curve(t: float) -> CGAMotor:
    """Analog integrator for motor interpolation."""
    # Smooth S-curve from motor_start to motor_end
    s = philbrick.integrate(t, damping=0.8)
    return motor_start.interpolate(motor_end, s)

# TiaCAD sweep along motor path
sweep_points = []
for t in np.linspace(0, 1, 100):
    motor = philbrick_motor_curve(t)
    point = motor.apply(base_point.mv)
    sweep_points.append(down(point))

tiacad_model.add_sweep_path(sweep_points)
```

### 9.3 Backend Abstraction

CGA operations can be a new `CGABackend`:

```python
class CGABackend(GeometryBackend):
    """Pure CGA geometry backend (for constraint solving, not BREP)."""

    def __init__(self):
        self.objects: Dict[str, MultiVector] = {}

    def create_point(self, x, y, z) -> str:
        name = self._generate_name("point")
        self.objects[name] = up(e1*x + e2*y + e3*z)
        return name

    def create_line(self, p1_name: str, p2_name: str) -> str:
        p1 = self.objects[p1_name]
        p2 = self.objects[p2_name]
        name = self._generate_name("line")
        self.objects[name] = (p1 ^ p2 ^ einf).normal()
        return name

    def intersect(self, obj1_name: str, obj2_name: str) -> str:
        """CGA meet operation."""
        a = self.objects[obj1_name]
        b = self.objects[obj2_name]
        name = self._generate_name("intersection")
        self.objects[name] = (a.dual() ^ b.dual()).dual()  # Meet
        return name

    def distance(self, p1_name: str, p2_name: str) -> float:
        p1 = self.objects[p1_name]
        p2 = self.objects[p2_name]
        return np.sqrt(-2 * float(p1 | p2))
```

---

## 10. References & Resources

### 10.1 Academic References

1. **Dorst, Fontijne, Mann** - *Geometric Algebra for Computer Science* (2007)
   - Comprehensive textbook, Chapter 13 covers CGA

2. **Perwass** - *Geometric Algebra with Applications in Engineering* (2009)
   - Applications focus, good for implementation

3. **Hadfield** - *Applications of Geometric Algebra in Mathematical Engineering* (2019)
   - Hugo Hadfield's PhD thesis, clifford library author
   - URL: https://hh409.user.srcf.net/thesis/HugoHadfieldThesis.pdf

4. **Zamora-Esquivel** - *Geometric Constraint Solving with Conformal Geometric Algebra*
   - Springer, 2011
   - URL: https://link.springer.com/chapter/10.1007/978-0-85729-811-9_11

### 10.2 Software Libraries

1. **clifford** (Python)
   - Production-ready CGA library
   - Documentation: https://clifford.readthedocs.io/en/latest/
   - GitHub: https://github.com/pygae/clifford
   - Install: `pip install clifford`

2. **ganja.js** (JavaScript)
   - Visualization for CGA
   - URL: https://enkimute.github.io/ganja.js/

3. **pyganja** (Python)
   - Python bindings for ganja.js visualization
   - GitHub: https://github.com/pygae/pyganja

### 10.3 Online Resources

1. **Clifford CGA Tutorial**
   - https://clifford.readthedocs.io/en/latest/tutorials/cga/index.html

2. **bivector.net**
   - Community resources for geometric algebra
   - URL: https://bivector.net/

3. **Grokipedia CGA Article**
   - https://grokipedia.com/page/Conformal_geometric_algebra

4. **Haskell CGA Tutorial** (mathematical focus)
   - https://crypto.stanford.edu/~blynn/haskell/cga.html

### 10.4 TiaCAD Internal References

- `docs/ARCHITECTURE_DECISION_V3.md` - Current architecture decisions
- `docs/TIACAD_EVOLUTION_ROADMAP.md` - Evolution planning
- `tiacad_core/geometry/spatial_references.py` - Current SpatialRef implementation
- `tiacad_core/parser/loft_builder.py` - Current loft implementation

---

## Appendix A: CGA Cheat Sheet

### A.1 Embedding and Projection

```python
from clifford.g3c import *

# Embed 3D point into CGA
X = up(e1*x + e2*y + e3*z)

# Project CGA point to 3D
x, y, z = down(X)
```

### A.2 Object Construction

```python
# Line through two points
L = (X1 ^ X2 ^ einf).normal()

# Plane through point with normal
P = n + d*einf  # n = normal, d = distance

# Circle through three points
C = (X1 ^ X2 ^ X3).normal()

# Sphere with center and radius
S = C - 0.5*r**2*einf  # C = center embedding
```

### A.3 Geometric Tests

```python
# Distance between points
d = sqrt(-2 * (X1 | X2))

# Point on object test
X | S == 0  # X on sphere S
X | P == 0  # X on plane P
X ^ L == 0  # X on line L

# Perpendicularity
L1 | L2 == 0  # Lines perpendicular
```

### A.4 Intersections (Meet)

```python
# Meet via dual
def meet(A, B):
    return (A.dual() ^ B.dual()).dual()

# Or use regressive product if available
result = A & B
```

### A.5 Motors

```python
# Translation
T = 1 + 0.5 * (t ^ einf)  # t = translation vector

# Rotation (through origin)
R = cos(theta/2) - sin(theta/2) * B  # B = rotation bivector

# Combined motor
M = T * R

# Apply to object
X_new = M * X * ~M  # Sandwich product
```

---

## Appendix B: Constraint Equation Reference

| Constraint | CGA Equation | Notes |
|------------|--------------|-------|
| Point on line | `X ∧ L = 0` | OPNS |
| Point on plane | `X · P = 0` | IPNS |
| Point on sphere | `X · S = 0` | IPNS |
| Point on circle | `X · C = 0` | IPNS |
| Distance(X,Y) = d | `-2(X·Y) = d²` | Squared distance |
| Lines perpendicular | `L₁ · L₂ = 0` | Inner product |
| Lines parallel | `L₁ ∧ L₂ ∧ e∞ = 0` | Share point at infinity |
| Planes parallel | `P₁ ∧ P₂ = 0` | Dual |
| Tangent line-sphere | `L · S = 0` | |
| Tangent plane-sphere | `P · S = 0` | |
| Circle in plane | `C · P = 0` | Coplanar |
| Concentric spheres | Centers equal | Same center point |
| Coaxial cylinders | Axes parallel & same line | |

---

*Document generated for TiaCAD CGA integration planning*
*Version 1.0 - 2025-11-30*
