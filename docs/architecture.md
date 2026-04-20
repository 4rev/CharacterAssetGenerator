# Architecture

## Overview

The Modular Character Asset Generator is a **C++ application with a Python ML sidecar**, structured as a UE5 Editor plugin. The pipeline is fully local — no cloud services, no data leaving the machine.

---

## High-Level Pipeline

```
┌─────────────────────────────────────────────────────┐
│                  UE5 EDITOR PLUGIN                   │
│                                                     │
│  ┌──────────┐   ┌─────────────┐   ┌──────────────┐ │
│  │  Image   │→  │ Segmentation│→  │  Per-Part    │ │
│  │  Input   │   │   (SAM2)    │   │  2D → 3D     │ │
│  └──────────┘   └─────────────┘   └──────┬───────┘ │
│                                          │          │
│  ┌──────────────────────────────────────┘          │
│  ↓                                                  │
│  ┌──────────┐   ┌─────────────┐   ┌──────────────┐ │
│  │  Mesh    │→  │  Assembly   │→  │    Export    │ │
│  │ Cleanup  │   │  & Rigging  │   │  (FBX/glTF)  │ │
│  └──────────┘   └─────────────┘   └──────────────┘ │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │         Self-Improvement Engine             │   │
│  │   Tier 1 (heuristics) → Tier 2 (local ML)  │   │
│  │           → Tier 3 (API, optional)          │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
         │ opt-in only                    ↑
         ↓ anonymised corrections         │ model updates
┌─────────────────────────────────────────────────────┐
│              GLOBAL SERVER (lightweight)             │
│         aggregation → retraining → redistribution   │
└─────────────────────────────────────────────────────┘
```

---

## Module Breakdown

### 1. Image Input Layer (`src/core/image/`)

Handles loading and preprocessing of concept art.

- Formats: PNG, JPG, BMP, TIFF, WebP
- Multi-view support: front / side / back tagged as a view group
- Preprocessing: normalize to [0,1] float, aspect-preserving letterbox resize to 512×512 default
- Style fingerprinting via CLIP ONNX — detects: `realistic`, `anime`, `dark_fantasy`, `cartoon`, `stylized`
- Output: `ImageSession` struct with per-view normalized buffers and style fingerprint

---

### 2. Python Sidecar IPC (`src/core/ipc/`)

C++ launches a Python 3.11 subprocess and communicates via local socket.

```
C++ (orchestrator)          Python sidecar
      │                           │
      │── JSON command ──────────>│
      │                    [ML inference]
      │<── JSON result ───────────│
```

Protocol:
```json
// Command
{ "cmd": "reconstruct", "part": "helmet", "image_b64": "...", "settings": { "density": "high" } }

// Result
{ "status": "ok", "mesh_path": "/tmp/helmet_001.glb", "confidence": 0.87 }
```

Features:
- Heartbeat every 2 seconds; auto-restart on loss
- Named pipe (Windows) or Unix domain socket (Linux)
- All ML inference isolated to Python — C++ never loads PyTorch directly

---

### 3. Segmentation Engine (`src/ml/segmentation/`)

Identifies and isolates individual clothing/equipment regions from the concept art.

**Models used:**
- SAM2 (Segment Anything v2) — ONNX export, runs via ONNX Runtime C++ API
- Component classifier — small ONNX model (MobileNetV3-based) classifying each mask

**Component categories:**
```cpp
enum class ComponentType {
    Body_Base,
    Hair, Beard, Eyebrows,
    Shirt, Pants, Dress, Robe, Underwear,
    Helmet, ChestArmor, Pauldron, Gauntlets, Greaves, Boots,
    Cape, Belt, Backpack, Necklace,
    WeaponPrimary, WeaponSecondary, Shield, Quiver,
    Unknown
};
```

**Output per part:**
- Clean masked image with transparent background
- Padding applied for 3D reconstruction input
- Confidence score from classifier

---

### 4. 2D → 3D Reconstruction (`python_sidecar/reconstruction/`)

Runs in Python sidecar. Two backends, automatically selected per component type:

| Backend | Best For | Speed (CPU) | Speed (GPU) |
|---|---|---|---|
| TripoSR | Hard surface: armor, weapons, helmets | ~60s | ~15s |
| SF3D (Stability AI) | Soft surface: clothing, hair, cloth | ~45s | ~12s |

Fallback chain: primary backend failure → retry with secondary → return error.

Output: `.glb` file per component.

---

### 5. Mesh Processing (`src/mesh/`)

Three sub-stages applied to each generated `.glb`:

**5a. Cleanup (`src/mesh/cleanup/`)**
- Remove degenerate/zero-area faces
- Fill holes (watertight guarantee)
- Fix non-manifold edges
- Smooth normals
- Libraries: OpenMesh, CGAL, libigl

**5b. UV Unwrapping (`src/mesh/uv/`)**
- xatlas per-component UV generation
- Component-specific padding: 4px armor, 2px clothing
- Atlas packing efficiency target: ≥ 75%

**5c. Texture Projection (`src/mesh/texture/`)**
- Project original concept art onto reconstructed mesh
- Multi-angle blending when side/back views provided
- Inpainting for occluded/backface regions
- Output resolution: 1024² (LOD0), 512² (LOD1), 256² (LOD2)

---

### 6. Assembly & Rigging (`src/mesh/assembly/`)

**Base Body Templates:**
- Stored as read-only FBX assets: male, female, creature variants
- Master skeleton compatible with UE5 Mannequin retargeting
- Standard sockets: `head`, `hand_l`, `hand_r`, `spine_01/02/03`, `foot_l`, `foot_r`

**Mesh Fitting:**
- Generated clothing wrapped to base body surface
- Offset by component type (learned from SQLite corrections)
- No interpenetration guarantee

**Skinning:**
- Bounded biharmonic weights via libigl
- Max 8 bone influences per vertex (UE5 hardware limit)
- Problem zone post-corrections applied from DB (shoulders, ankles, fingers)

**Output:**
- Separate Skeletal Mesh asset per component
- All share master skeleton reference
- Material slots named consistently for Blueprint swapping

---

### 7. Self-Improvement Engine (`src/ml/feedback/`)

Three-tier quality assessment:

```
Every generated mesh
        ↓
[Tier 1] Local heuristics (C++, free, always runs)
    - watertight check
    - UV overlap detection
    - poly count validation
    - normal consistency
    - texture stretch metric
    - silhouette match score
        ↓ fails or uncertain
[Tier 2] Local ONNX classifier (free, runs locally)
    - trained on accumulated SQLite correction history
    - output: good / needs_fix / bad + confidence
        ↓ bad or confidence < 0.6
[Tier 3] AI API call (costs money, rarely triggered)
    - vision model analyzes rendered mesh
    - returns JSON fix instructions
    - C++ fix executor applies corrections
    - budget cap enforced
```

**Correction DB Schema (SQLite):**
```sql
CREATE TABLE corrections (
    id INTEGER PRIMARY KEY,
    component_type TEXT NOT NULL,
    style_fingerprint_id INTEGER,
    settings_before TEXT,        -- JSON
    settings_after TEXT,         -- JSON
    quality_before REAL,
    quality_after REAL,
    source TEXT,                 -- 'tier1' | 'tier2' | 'tier3'
    created_at INTEGER
);
```

**Learning cycle:**
- Corrections accumulate in SQLite during normal use
- Tier 2 classifier retrained weekly from correction history
- Pre-emptive settings applied on future runs when confidence ≥ 0.8

---

### 8. Global Network (`src/server/`)

Opt-in, disabled by default.

**What is uploaded (anonymised):**
```json
{
  "style": "dark_fantasy",
  "component": "chest_armor",
  "quality_before": 0.51,
  "quality_after": 0.84,
  "settings_applied": { "remesh_density": 0.8, "uv_padding": 4 },
  "anon_id": "a3f7b2..."   // rotating weekly hash, not tied to identity
}
```

**What is NEVER uploaded:**
- Images
- Meshes
- API keys
- File paths
- Machine identifiers
- User identity of any kind

**Global update cycle:**
- Daily: correction records aggregated on server
- Weekly: Tier 2 classifier retrained on all user data → ONNX exported → pushed to CDN
- Monthly: style fingerprint library published → downloaded by all clients

**Plugin update sync:**
- Silent background check on Editor launch
- SHA256 integrity verification before applying
- Hot-loaded without Editor restart

---

## Data Flow Diagram

```
[User image]
    │
    ▼
[ImageSession] ──────────────────────────────┐
    │                                         │
    ▼                                         ▼
[SAM2 segmentation]                    [Style fingerprint]
    │                                         │
    ▼                                         ▼
[Per-part masked images]            [SQLite lookup]
    │                                         │
    ▼                                         ▼
[TripoSR / SF3D via sidecar]     [Pre-emptive settings]
    │                                         │
    └──────────────┬──────────────────────────┘
                   ▼
          [Raw .glb per part]
                   │
                   ▼
         [Mesh cleanup + UV + Texture]
                   │
                   ▼
         [Fit to base body + Skinning]
                   │
                   ▼
         [Tier 1 heuristics] ──fail──▶ [Tier 2 classifier]
                   │                          │
                   │ pass              confident fail
                   │                          │
                   │                          ▼
                   │                  [Tier 3 API call]
                   │                          │
                   └──────────────────────────┘
                                   │
                                   ▼
                          [FBX/glTF export]
                                   │
                                   ▼
                     [UE5 Content Browser import]
```

---

## Technology Stack

| Layer | Technology | License |
|---|---|---|
| Build | CMake 3.26+, vcpkg | MIT |
| C++ standard | C++20 | — |
| Image loading | OpenCV, stb_image | Apache 2 / MIT |
| ONNX inference | ONNX Runtime C++ | MIT |
| Mesh processing | OpenMesh, libigl, CGAL | LGPL / MPL2 / GPL* |
| UV unwrapping | xatlas | MIT |
| 3D reconstruction | TripoSR, SF3D | MIT / Stability AI |
| Segmentation | SAM2 | Apache 2 |
| Serialisation | nlohmann/json | MIT |
| Networking | Boost.Asio, libcurl | BSL / MIT |
| Database | SQLite3 | Public domain |
| Cryptography | libsodium | ISC |
| Export | Assimp, Autodesk FBX SDK | BSD / Autodesk |
| UE5 integration | Unreal Engine 5 | Epic EULA |
| ML training | PyTorch | BSD |
| Server | Rust (axum) | MIT |

*CGAL: verify license for your use case. Optional — libigl covers most operations.*

---

## Performance Targets

| Operation | CPU Target | GPU Target |
|---|---|---|
| Image load + preprocess | < 200ms | — |
| SAM2 segmentation | < 8s | < 1.5s |
| TripoSR reconstruction (per part) | < 60s | < 15s |
| Mesh cleanup + UV (per part) | < 10s | — |
| Texture projection (per part) | < 5s | — |
| Mesh fitting + skinning (per part) | < 20s | — |
| Tier 1 heuristics | < 500ms | — |
| Full character (6 parts, GPU) | ~3–4 min | — |
