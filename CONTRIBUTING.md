# Contributing

Thank you for your interest in contributing. This guide covers everything you need to get started.

---

## Table of Contents

1. [Running the Test Suite](#running-the-test-suite)
2. [Adding a New Fix Type](#adding-a-new-fix-type)
3. [Adding a New Component Category](#adding-a-new-component-category)
4. [Retraining the Local Classifier](#retraining-the-local-classifier)
5. [Code Style](#code-style)
6. [Submitting a Pull Request](#submitting-a-pull-request)
7. [Issue Templates](#issue-templates)

---

## Running the Test Suite

### Prerequisites
- CMake build must succeed (see README.md)
- Python sidecar dependencies installed (`pip install -r python_sidecar/requirements.txt`)
- Test assets present in `tests/test_assets/` (downloaded via `cmake --build build --target fetch_test_assets`)

### Run all tests
```bash
cmake --build build --config Release
ctest --test-dir build --output-on-failure
```

### Run a specific module
```bash
ctest --test-dir build -R "mesh_cleanup" --output-on-failure
ctest --test-dir build -R "ipc" --output-on-failure
ctest --test-dir build -R "sqlite" --output-on-failure
```

### Run with verbose output
```bash
ctest --test-dir build --verbose
```

### Test categories

| Category | What it covers |
|---|---|
| `unit_core` | Image loading, IPC protocol, SQLite CRUD |
| `unit_mesh` | Watertight check, UV overlap, decimation, skinning |
| `unit_ml` | ONNX model load, classifier inference, heuristic scores |
| `unit_export` | FBX/glTF write + re-read round-trip |
| `integration` | Full pipeline: image → mesh → UE5 export |

### Adding a new test

Tests live in `tests/unit/[module]/`. Each test file follows this pattern:

```cpp
// tests/unit/mesh/test_uv_overlap.cpp
#include "catch2/catch.hpp"
#include "mesh/uv.h"

TEST_CASE("UV islands do not overlap after xatlas", "[mesh][uv]") {
    auto mesh = loadTestMesh("tests/test_assets/helmet_001.glb");
    auto uvResult = unwrapUV(mesh, ComponentType::Helmet);
    REQUIRE(uvResult.hasOverlaps() == false);
}
```

---

## Adding a New Fix Type

The fix library lives in `src/ml/feedback/fix_library.cpp`. Adding a fix involves three steps:

### Step 1 — Implement the fix function

```cpp
// In src/ml/feedback/fix_library.cpp
MeshResult applyMyNewFix(const Mesh& mesh, const FixParams& params) {
    // Your fix logic here using OpenMesh / libigl
    // Must not modify the input mesh — return a new Mesh
    return modifiedMesh;
}
```

### Step 2 — Register the fix by name

```cpp
// In the FixLibrary constructor
registerFix("my_new_fix", applyMyNewFix);
```

### Step 3 — Add a test

```cpp
// In tests/unit/ml/test_fix_library.cpp
TEST_CASE("my_new_fix improves target metric", "[fix][ml]") {
    auto mesh = loadTestMesh("tests/test_assets/problematic_mesh.glb");
    auto result = fixLibrary.apply("my_new_fix", mesh, {});
    
    // Fix must improve, not worsen, the metric it targets
    REQUIRE(result.qualityScore() >= mesh.qualityScore());
    // Fix must not break other passing heuristics
    REQUIRE(heuristics.checkWatertight(result) == true);
}
```

### Fix requirements checklist

- [ ] Returns a new `Mesh` — does not mutate input
- [ ] Idempotent (applying twice = applying once)
- [ ] Does not break any currently-passing heuristic
- [ ] Has a unit test covering the happy path
- [ ] Has a unit test for edge case (e.g. already-fixed mesh)
- [ ] Documented in `docs/fix_library.md`

---

## Adding a New Component Category

Component categories are defined in `src/core/component_type.h`.

### Step 1 — Add to the enum

```cpp
// src/core/component_type.h
enum class ComponentType {
    // ... existing types ...
    MyNewComponent,   // ← add here
    Unknown
};
```

### Step 2 — Add string mapping

```cpp
// src/core/component_type.cpp
{ "my_new_component", ComponentType::MyNewComponent },
```

### Step 3 — Add socket mapping

```cpp
// src/mesh/assembly/socket_map.cpp
{ ComponentType::MyNewComponent, "spine_02" },  // attach to this socket
```

### Step 4 — Add default pipeline settings

```cpp
// src/core/pipeline_settings.cpp
{ ComponentType::MyNewComponent, PipelineSettings{
    .reconstructionBackend = Backend::TripoSR,
    .targetPolyCountLOD0   = 3000,
    .uvPaddingPx           = 3,
    .bodyOffsetMm          = 5.0f
}},
```

### Step 5 — Add classifier training label

Add the new label to `training/labels.json` and provide at least 20 labelled example images in `training/data/my_new_component/`.

### Step 6 — Add tests

- Unit test: classifier correctly identifies the new component type
- Unit test: socket attachment produces correct transform

---

## Retraining the Local Classifier

The Tier 2 quality classifier is retrained from your local correction history. This runs automatically weekly, but you can trigger it manually.

### Trigger retraining manually

```bash
cd training
python retrain.py --db-path ~/.config/CharacterAssetGenerator/corrections.db --output classifier_new.onnx
```

### What it does

1. Reads all correction records from your local SQLite DB
2. Extracts features: mesh metrics + rendered thumbnail embeddings
3. Trains a small MobileNetV3-based binary classifier
4. Validates on a held-out 20% split
5. Exports to ONNX if validation accuracy ≥ previous model

### Minimum data requirements

- At least 50 correction records for training to begin
- Below 50 records, the script exits cleanly with a message — it will not produce an undertrained model

### Deploying the retrained model

```bash
# Copy to plugin's model directory
cp classifier_new.onnx ~/.config/CharacterAssetGenerator/models/classifier.onnx
# The plugin hot-loads the new model on next use — no restart required
```

### Contributing a retrained model to the global pool

If you have enabled global sharing, your correction data already feeds the global retraining pipeline automatically. You do not need to upload model weights manually.

---

## Code Style

- **C++ standard:** C++20
- **Formatting:** clang-format with the config in `.clang-format` at repo root — run `clang-format -i src/**/*.cpp src/**/*.h` before committing
- **Naming:** `snake_case` for functions and variables, `PascalCase` for types and classes
- **Error handling:** use typed `Result<T, Error>` returns — no raw exceptions in library code
- **Comments:** explain *why*, not *what* — the code explains what

---

## Submitting a Pull Request

1. Fork the repository
2. Create a branch: `git checkout -b feature/my-new-fix-type`
3. Make your changes
4. Run the full test suite: `ctest --test-dir build --output-on-failure`
5. Run clang-format: `clang-format -i src/**/*.cpp src/**/*.h`
6. Commit with a descriptive message: `feat: add cloth_simulation fix type for loose clothing`
7. Open a PR against `main` — fill in the PR template

### PR checklist

- [ ] Tests pass
- [ ] New code has tests
- [ ] clang-format applied
- [ ] Acceptance criteria for the relevant task still pass (see `docs/acceptance_criteria.md`)
- [ ] `PRIVACY.md` still accurate (if you touched any upload/network code)

---

## Issue Templates

Use the templates in `.github/ISSUE_TEMPLATE/`:

- `bug_report.md` — for crashes, incorrect output, or broken features
- `feature_request.md` — for new features or pipeline improvements
- `new_fix_type.md` — specifically for proposing a new entry to the fix library

For privacy concerns, use the `privacy` label on any issue.
