# TiaCAD Testing Guide

**Version:** 1.0 (v3.1)
**Created:** 2025-11-11
**Status:** Active Documentation

---

## Table of Contents

1. [Overview](#overview)
2. [Running Tests](#running-tests)
3. [Test Categories](#test-categories)
4. [Testing Utilities](#testing-utilities)
5. [Writing New Tests](#writing-new-tests)
6. [CI/CD Integration](#cicd-integration)

---

## Overview

TiaCAD has a comprehensive test suite with **950+ tests** covering:

- **Unit tests**: Fast, isolated tests for individual functions/classes
- **Integration tests**: Multi-component tests for end-to-end workflows
- **Correctness tests**: Verify geometric correctness (attachment, rotation, dimensions)
- **Parser tests**: YAML parsing and geometry building
- **Validation tests**: Rule-based validation of models

### Test Statistics (v3.1)

| Category | Count | Coverage |
|----------|-------|----------|
| Parser Tests | 518 | 95% |
| Correctness Tests (NEW) | 60+ | 90% |
| Integration Tests | 20+ | 85% |
| Unit Tests | 350+ | 92% |
| **Total** | **950+** | **90%** |

---

## Running Tests

### Run All Tests

```bash
pytest
```

### Run Tests by Category (Using Markers)

```bash
# Attachment correctness tests only
pytest -m attachment

# Rotation correctness tests only
pytest -m rotation

# Dimensional accuracy tests only
pytest -m dimensions

# All correctness tests
pytest -m "attachment or rotation or dimensions"

# Integration tests only
pytest -m integration

# Parser tests only
pytest -m parser
```

### Run Tests by Directory

```bash
# All correctness tests
pytest tiacad_core/tests/test_correctness/

# All parser tests
pytest tiacad_core/tests/test_parser/

# All testing utility tests
pytest tiacad_core/tests/test_testing/
```

### Run Specific Test File

```bash
pytest tiacad_core/tests/test_correctness/test_attachment_correctness.py
```

### Run Specific Test Function

```bash
pytest tiacad_core/tests/test_correctness/test_rotation_correctness.py::TestBasicRotations::test_box_rotate_90deg_around_z_axis
```

### Run with Verbose Output

```bash
pytest -v
```

### Run with Coverage Report

```bash
# Generate coverage report
pytest --cov=tiacad_core --cov-report=html

# Open HTML coverage report
open htmlcov/index.html
```

### Run Only Failed Tests from Last Run

```bash
pytest --lf  # --last-failed
```

### Run Tests in Parallel (Faster)

```bash
pytest -n auto  # requires pytest-xdist
```

---

## Test Categories

### Pytest Markers

TiaCAD uses pytest markers to organize tests by category. See `pytest.ini` for full list.

#### Core Categories

- `unit`: Fast, isolated unit tests
- `integration`: Multi-component integration tests
- `slow`: Tests taking > 5 seconds

#### Correctness Categories (v3.1 - NEW)

- `attachment`: Verify parts attach at correct locations with proper contact
- `rotation`: Verify parts orient correctly according to specifications
- `dimensions`: Verify measurements match specifications (size, volume, surface area)
- `visual`: Visual regression tests (future: v3.2)

#### Feature Categories

- `parser`: YAML parser and builder tests
- `spatial`: Spatial reference resolution tests
- `backend`: Geometry backend tests
- `validation`: Validation rule tests

### Examples

```bash
# Run fast tests only (excludes slow tests)
pytest -m "not slow"

# Run all correctness tests except visual
pytest -m "(attachment or rotation or dimensions) and not visual"

# Run parser and integration tests
pytest -m "parser or integration"
```

---

## Testing Utilities

TiaCAD v3.1 introduces testing utilities to simplify correctness verification.

### 1. Measurement Utilities

Module: `tiacad_core/testing/measurements.py`

#### `measure_distance(part1, part2, ref1="center", ref2="center")`

Measure distance between two parts at specified reference points.

**Examples:**

```python
from tiacad_core.testing.measurements import measure_distance

# Distance between centers
dist = measure_distance(box1, box2)

# Distance from box top to cylinder bottom
dist = measure_distance(
    box, cylinder,
    ref1="face_top",
    ref2="face_bottom"
)

# Verify zero-distance attachment
assert dist < 0.001, "Parts should be touching"
```

#### `get_bounding_box_dimensions(part)`

Extract bounding box dimensions from a part.

**Returns:** Dict with keys: `width`, `height`, `depth`, `center`, `min`, `max`

**Example:**

```python
from tiacad_core.testing.measurements import get_bounding_box_dimensions

dims = get_bounding_box_dimensions(box)

assert abs(dims['width'] - 50.0) < 0.1
assert abs(dims['height'] - 30.0) < 0.1
assert abs(dims['depth'] - 20.0) < 0.1
```

### 2. Orientation Utilities

Module: `tiacad_core/testing/orientation.py`

#### `get_orientation_angles(part, reference="center")`

Extract rotation angles (roll, pitch, yaw) from part orientation.

**Returns:** Dict with keys: `roll`, `pitch`, `yaw`, `has_orientation`

**Example:**

```python
from tiacad_core.testing.orientation import get_orientation_angles

angles = get_orientation_angles(box, reference="face_top")

# Verify yaw angle is 45°
assert abs(angles['yaw'] - 45.0) < 0.1
```

#### `get_normal_vector(part, face_ref="face_top")`

Get the normal vector of a specific face.

**Returns:** Numpy array [x, y, z] (normalized to unit length)

**Example:**

```python
from tiacad_core.testing.orientation import get_normal_vector

normal = get_normal_vector(box, "face_top")

# Verify top face points up (+Z direction)
assert normal[2] > 0.9
assert np.linalg.norm(normal) == pytest.approx(1.0)
```

#### `parts_aligned(part1, part2, axis='z', ref1="center", ref2="center", tolerance=0.1)`

Check if two parts are aligned along a specific axis.

**Example:**

```python
from tiacad_core.testing.orientation import parts_aligned

# Verify boxes are aligned along Z axis (vertically stacked)
assert parts_aligned(bottom_box, top_box, axis='z', tolerance=0.01)
```

### 3. Dimension Utilities

Module: `tiacad_core/testing/dimensions.py`

#### `get_dimensions(part)`

Extract dimensions, volume, and surface area from a part.

**Returns:** Dict with keys: `width`, `height`, `depth`, `volume`, `surface_area`

**Example:**

```python
from tiacad_core.testing.dimensions import get_dimensions

dims = get_dimensions(box)

assert abs(dims['width'] - 50.0) < 0.01
assert abs(dims['volume'] - 30000) < 30  # 50*30*20
```

#### `get_volume(part)`

Get part volume in cubic units.

**Example:**

```python
from tiacad_core.testing.dimensions import get_volume

volume = get_volume(box)
expected = 10 * 20 * 30  # 6000

# Verify within 1% accuracy
assert abs(volume - expected) < expected * 0.01
```

#### `get_surface_area(part)`

Get part surface area in square units.

**Example:**

```python
from tiacad_core.testing.dimensions import get_surface_area

area = get_surface_area(box)
expected = 2 * (10*20 + 10*30 + 20*30)  # Box surface area

# Verify within 1% accuracy
assert abs(area - expected) < expected * 0.01
```

---

## Writing New Tests

### Test Structure

```python
"""
Brief description of what this test module covers.

Author: Your Name
Version: 1.0
"""

import pytest
from tiacad_core.testing.measurements import measure_distance
from tiacad_core.testing.orientation import get_normal_vector
from tiacad_core.testing.dimensions import get_volume
from tiacad_core.part import Part
from tiacad_core.geometry import CadQueryBackend
import cadquery as cq


class TestFeatureName:
    """Test description"""

    def setup_method(self):
        """Setup test fixtures"""
        self.backend = CadQueryBackend()

    def test_something_specific(self):
        """Test that something specific works correctly"""
        # Arrange
        part = Part(
            name="test_part",
            geometry=cq.Workplane("XY").box(10, 10, 10),
            backend=self.backend
        )

        # Act
        result = measure_distance(part, other_part)

        # Assert
        assert result == expected_value


# Add pytest marker
pytestmark = pytest.mark.your_category
```

### Best Practices

1. **Use descriptive test names**: `test_cylinder_on_box_top_zero_distance` not `test_attachment1`

2. **One assertion per test** (when possible): Makes failures easier to diagnose

3. **Use testing utilities**: Don't re-implement measurement logic

4. **Add docstrings**: Explain what the test verifies

5. **Use appropriate tolerances**:
   - Distances: `< 0.01` units
   - Angles: `< 0.1` degrees
   - Volumes: `< 1%` of expected value

6. **Add pytest markers**: Categorize tests for easy filtering

### Example: Complete Attachment Test

```python
@pytest.mark.attachment
def test_cylinder_on_box_top_zero_distance(self):
    """Test cylinder attached to box top face (distance = 0)"""
    # Create a box centered at origin
    box = Part(
        name="box",
        geometry=cq.Workplane("XY").box(20, 20, 10),
        backend=self.backend
    )

    # Create cylinder on top of box
    # Box is 10 units tall, so top face is at z=5
    # Cylinder with height 10 centered at z=10 (base at z=5)
    cylinder = Part(
        name="cylinder",
        geometry=cq.Workplane("XY").workplane(offset=10).cylinder(10, 5),
        backend=self.backend
    )

    # Distance between box top face and cylinder bottom face should be ~0
    dist = measure_distance(
        box, cylinder,
        ref1="face_top",
        ref2="face_bottom"
    )

    assert dist < 0.01, f"Expected ~0 distance, got {dist}"
```

---

## CI/CD Integration

### GitHub Actions Workflow

TiaCAD tests run automatically on every push and pull request.

**Workflow file:** `.github/workflows/test.yml`

```yaml
name: TiaCAD Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
      - name: Run tests
        run: pytest --cov=tiacad_core --cov-report=xml
      - name: Upload coverage
        uses: codecov/codecov-action@v2
        with:
          file: ./coverage.xml
```

### Running Tests Locally Before Push

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=tiacad_core

# Run only fast tests
pytest -m "not slow"

# Run correctness tests
pytest -m "attachment or rotation or dimensions"
```

### Pre-commit Hooks (Recommended)

Create `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: local
    hooks:
      - id: pytest-check
        name: pytest-check
        entry: pytest -m "not slow"
        language: system
        pass_filenames: false
        always_run: true
```

Install hooks:

```bash
pip install pre-commit
pre-commit install
```

---

## Troubleshooting

### Common Issues

**Issue: Tests fail with import errors**

Solution: Ensure you're in the project root and TiaCAD is installed:

```bash
pip install -e .
# or
export PYTHONPATH=/path/to/tiacad:$PYTHONPATH
```

**Issue: Tests take too long**

Solution: Run fast tests only or use parallel execution:

```bash
pytest -m "not slow"
# or
pytest -n auto  # requires pytest-xdist
```

**Issue: Coverage report not generating**

Solution: Install pytest-cov:

```bash
pip install pytest-cov
pytest --cov=tiacad_core
```

**Issue: Marker warnings**

Solution: Ensure markers are registered in `pytest.ini`. All TiaCAD markers are pre-registered.

---

## Additional Resources

- [Testing Confidence Plan](./TESTING_CONFIDENCE_PLAN.md) - Strategic testing plan
- [Testing Roadmap](./TESTING_ROADMAP.md) - Implementation roadmap
- [Testing Quick Reference](./TESTING_QUICK_REFERENCE.md) - Quick command reference
- [pytest documentation](https://docs.pytest.org/)

---

## Appendix: Quick Reference

### Most Common Commands

```bash
# Run all tests
pytest

# Run correctness tests only
pytest -m "attachment or rotation or dimensions"

# Run specific test file
pytest tiacad_core/tests/test_correctness/test_rotation_correctness.py

# Run with coverage
pytest --cov=tiacad_core --cov-report=html

# Run only failed tests from last run
pytest --lf

# Show slowest 10 tests
pytest --durations=10
```

### Test Markers Quick Reference

| Marker | Description | Example |
|--------|-------------|---------|
| `attachment` | Attachment correctness | `pytest -m attachment` |
| `rotation` | Rotation correctness | `pytest -m rotation` |
| `dimensions` | Dimensional accuracy | `pytest -m dimensions` |
| `integration` | Integration tests | `pytest -m integration` |
| `parser` | Parser tests | `pytest -m parser` |
| `slow` | Slow tests (>5s) | `pytest -m "not slow"` |

### Testing Utility Import Quick Reference

```python
# Measurements
from tiacad_core.testing.measurements import (
    measure_distance,
    get_bounding_box_dimensions,
    get_distance_between_points,
)

# Orientation
from tiacad_core.testing.orientation import (
    get_orientation_angles,
    get_normal_vector,
    parts_aligned,
)

# Dimensions
from tiacad_core.testing.dimensions import (
    get_dimensions,
    get_volume,
    get_surface_area,
)
```

---

**End of Document**
