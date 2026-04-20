# Architecture

## Design Philosophy

**Do well what we can do well today. Grow slowly as AI models improve.**

This project does not try to build a self-improving global ML system. It builds a **solid local pipeline** for concept-art → rigged modular hard-surface assets, structured so that future model upgrades (SAM3, next-gen 2D→3D reconstruction, Claude Opus 5/6/7) drop in as single-file adapter swaps — not rewrites.

Everything that is not reliably good today is **out of V1 scope**.

---

## V1 Scope (what we actually ship)

### In scope
- **Hard-surface components only:** armor (helmet, chest, pauldron, gauntlets, greaves, boots), weapons (sword, axe, bow, staff), shields, props (belts, pouches, backpacks, capes if rigid).
- **Single base skeleton:** one choice — UE5 Mannequin OR Mixamo standard rig. User picks at install; not auto-detected.
- **Local-only pipeline.** No server. No crowdsourcing. No remote retraining.
- **On-demand AI review.** User explicitly clicks "review this mesh" to invoke Claude API. No automatic tier cascade.
- **Per-user recipe book** in SQLite — remembers settings that worked for *this user*, not anyone else.
- **UE5 Editor plugin with one button:** "Generate from image."

### Out of V1 scope (explicitly deferred)
- Cloth and hair reconstruction — TripoSR/SF3D quality is too weak today. Revisit when models improve.
- Multiple base skeletons (male/female/creature variants) — pick one, do it well.
- Tier 2 local quality classifier — not enough data until a user has hundreds of generations.
- Global server, crowdsourced correction sharing, weekly retraining, style-fingerprint distribution.
- Automatic multi-tier cascade — replaced by explicit user-initiated review.
- Photorealistic output. Scope is **stylized modular assets**, not MetaHuman replacement.

---

## High-Level Pipeline

```
┌─────────────────────────────────────────────────────┐
│                  UE5 EDITOR PLUGIN                   │
│                                                     │
│  ┌──────────┐   ┌─────────────┐   ┌──────────────┐ │
│  │  Image   │→  │ Segmentation│→  │  Per-Part    │ │
│  │  Input   │   │ (SAM adapter)│  │  2D → 3D     │ │
│  └──────────┘   └─────────────┘   │(recon adapter)│ │
│                                   └──────┬───────┘ │
│                                          │          │
│  ┌──────────────────────────────────────┘          │
│  ↓                                                  │
│  ┌──────────┐   ┌─────────────┐   ┌──────────────┐ │
│  │  Mesh    │→  │   Fit to    │→  │    Export    │ │
│  │ Cleanup  │   │   Skeleton  │   │  (glTF/FBX)  │ │
│  │ + UV +   │   │  + Skinning │   └──────────────┘ │
│  │ Texture  │   └─────────────┘                    │
│  └──────────┘                                       │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  Quality Review (Tier 1 auto + Tier 3 API)  │   │
│  │         Local SQLite recipe book            │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

No external network dependency. The Claude API call is optional, user-initiated, and uses the user's own key.

---

## Adapter Boundaries (the one bet on the future)

Three pipeline stages are isolated behind stable interfaces. When better models ship, we replace the adapter — not the pipeline.

### Segmentation adapter
```cpp
// src/ml/segmentation/ISegmenter.h
class ISegmenter {
public:
    virtual ~ISegmenter() = default;
    virtual std::vector<MaskedRegion> Segment(const Image& input) = 0;
    virtual std::string ModelName() const = 0;
    virtual std::string ModelVersion() const = 0;
};
```
V1 implementation: `SAM2Segmenter`. Future: `SAM3Segmenter` drops in as a one-file swap.

### Reconstruction adapter
```cpp
// src/ml/reconstruction/IReconstructor.h
class IReconstructor {
public:
    virtual ~IReconstructor() = default;
    virtual ReconstructResult Reconstruct(const MaskedImage& part, const ReconstructSettings& s) = 0;
    virtual bool SupportsComponent(ComponentType t) const = 0;
    virtual std::string ModelName() const = 0;
    virtual std::string ModelVersion() const = 0;
};
```
V1 implementation: `TripoSRReconstructor`. Future: swap in whatever beats TripoSR on hard surface.

### Reviewer adapter
```cpp
// src/ml/reviewer/IReviewer.h
class IReviewer {
public:
    virtual ~IReviewer() = default;
    virtual ReviewResult Review(const MeshBundle& mesh, const QualityMetrics& tier1) = 0;
    virtual std::string ProviderName() const = 0;   // "anthropic" | "openai" | ...
    virtual std::string ModelName() const = 0;      // "claude-opus-4-7" | ...
};
```
V1 implementation: `ClaudeReviewer` (calls Claude API with mesh renders + Tier 1 metrics, returns structured fix suggestions). Future: swap to Opus 5, Sonnet 5, etc.

Every recipe-book row also stores `model_name + model_version` so that stale records (generated against older models) can be filtered as models are upgraded.

---

## Module Breakdown

### 1. Image Input Layer (`src/core/image/`)
- Formats: PNG, JPG, BMP, TIFF, WebP
- Multi-view support: front / side / back tagged as a view group
- Preprocessing: normalize, aspect-preserving letterbox to 512×512
- Style tagging: simple CLIP embedding stored per session. Used as a *tag* for the recipe book, not as a clustering key for cross-user learning.

### 2. Python Sidecar IPC (`src/core/ipc/`)
- C++ launches Python 3.11 subprocess
- JSON protocol over local socket (Unix domain) or named pipe (Windows)
- Heartbeat every 2s, auto-restart on loss
- All PyTorch/ONNX inference isolated to Python; C++ stays pure

### 3. Segmentation (`src/ml/segmentation/`)
- `ISegmenter` interface
- V1 impl: SAM2 via ONNX Runtime C++ API
- Followed by a small MobileNetV3 component classifier predicting: helmet, chest_armor, pauldron, gauntlets, greaves, boots, weapon_primary, shield, belt, backpack, cape, unknown
- Clothing/hair/dress masks, if detected, are flagged **unsupported in V1** and surfaced as a warning — not processed.

### 4. 2D → 3D Reconstruction (`python_sidecar/reconstruction/`)
- `IReconstructor` interface, called via sidecar
- V1 impl: TripoSR (hard-surface quality is acceptable)
- SF3D not included in V1 — kept as a future adapter slot

### 5. Mesh Processing (`src/mesh/`)
**5a. Cleanup** — remove degenerate faces, fill holes (watertight guarantee), fix non-manifold edges, smooth normals. Tech: OpenMesh + libigl.

**5b. UV Unwrapping** — xatlas per-component. Component-specific padding. Atlas packing target ≥ 75%.

**5c. Texture Projection** — project concept art onto mesh. Multi-angle blend when side/back views provided. Inpaint occluded regions. Output: 1024² LOD0, 512² LOD1, 256² LOD2.

### 6. Fitting & Rigging (`src/mesh/assembly/`)
The hardest part of the project. Budgeted accordingly in the implementation plan.

- **Base skeleton:** one, chosen at install. Read-only template FBX.
- **Fitting:** wrap generated mesh to base body surface using libigl mesh deformation. Offset per component type from small hand-curated defaults.
- **Skinning:** bounded biharmonic weights via libigl. Max 8 bone influences (UE5 limit). Post-corrections applied to shoulders, ankles.
- **Socket attachment:** rule-based (weapon → hand_r socket, helmet → head socket, etc.).
- **Output:** one Skeletal Mesh asset per component, all sharing the master skeleton. Material slots named consistently so Blueprints can swap variants.

### 7. Quality Review (`src/ml/feedback/`)

Two layers, not three:

**Tier 1 — Automatic local heuristics (always runs, free)**
- Watertight check
- UV overlap percentage
- Poly count vs target
- Normal consistency
- Texture stretch metric
- Silhouette match score (rendered mesh vs original mask)

Output: `QualityMetrics` struct, displayed to user.

**Tier 3 — User-initiated AI review (optional, costs API credits)**
- User clicks "review this mesh" in the plugin UI
- Plugin renders the mesh from a few angles, packages renders + Tier 1 metrics, sends to Claude via the `IReviewer` adapter
- API returns a fix suggestion **from a preset catalog** (not open-ended code generation):
  - `remesh_higher_density`
  - `repack_uv_larger_padding`
  - `refit_offset_increase`
  - `re-project_texture_different_angle`
  - `regenerate_with_different_backend`
  - `accept_as_is`
- C++ executor applies the chosen fix and re-runs the relevant stage
- Every review + outcome logged to the local recipe book

**No Tier 2 local classifier in V1.** Added later if a user accumulates enough personal history to make it worthwhile.

### 8. Recipe Book (`src/core/db/`)

Local SQLite. One database per user. No sync. No server.

```sql
CREATE TABLE generations (
    id INTEGER PRIMARY KEY,
    created_at INTEGER NOT NULL,
    component_type TEXT NOT NULL,
    style_tag TEXT,
    seg_model_name TEXT,
    seg_model_version TEXT,
    recon_model_name TEXT,
    recon_model_version TEXT,
    settings_json TEXT NOT NULL,
    tier1_watertight INTEGER,
    tier1_uv_overlap REAL,
    tier1_poly_count INTEGER,
    tier1_silhouette_match REAL,
    user_accepted INTEGER,         -- 1 = accepted, 0 = rejected, NULL = undecided
    confidence_weight REAL         -- computed: quality delta × user acceptance × recency decay
);

CREATE TABLE reviews (
    id INTEGER PRIMARY KEY,
    generation_id INTEGER REFERENCES generations(id),
    reviewer_provider TEXT,
    reviewer_model TEXT,
    fix_suggested TEXT,            -- one of the preset catalog entries
    fix_applied INTEGER,
    quality_before REAL,
    quality_after REAL,
    cost_usd REAL,
    created_at INTEGER
);
```

On a new generation, the pipeline queries: *"for this component_type + style_tag, what settings produced the highest-weighted accepted generations in the last 90 days against the current model versions?"* — and pre-applies them. That's the entirety of the "learning." No ML, no classifier, no magic.

---

## Technology Stack

| Layer | Technology | License |
|---|---|---|
| Build | CMake 3.26+, vcpkg | MIT |
| C++ standard | C++20 | — |
| Image loading | OpenCV, stb_image | Apache 2 / MIT |
| ONNX inference | ONNX Runtime C++ | MIT |
| Mesh processing | OpenMesh, libigl | LGPL / MPL2 |
| UV unwrapping | xatlas | MIT |
| 3D reconstruction | TripoSR (via Python sidecar) | MIT |
| Segmentation | SAM2 (via ONNX) | Apache 2 |
| Serialisation | nlohmann/json | MIT |
| Networking (API only) | libcurl | MIT |
| Database | SQLite3 | Public domain |
| Export | Assimp (glTF primary, FBX read) | BSD |
| UE5 integration | Unreal Engine 5 | Epic EULA |
| AI review | Anthropic Claude API | commercial, user's own key |

**Intentionally cut from V1:** CGAL (licensing risk + libigl covers it), FBX SDK (redistribution hassle, Assimp is sufficient), libsodium (no server means no anonymisation crypto), Rust/axum (no server), PyTorch training pipeline (no Tier 2), Boost.Asio (overkill for local IPC — use plain sockets).

---

## Performance Targets (V1, realistic)

| Operation | CPU Target | GPU Target |
|---|---|---|
| Image load + preprocess | < 200ms | — |
| SAM2 segmentation | < 10s | < 2s |
| TripoSR reconstruction (per part) | < 90s | < 20s |
| Mesh cleanup + UV (per part) | < 15s | — |
| Texture projection (per part) | < 8s | — |
| Fitting + skinning (per part) | < 30s | — |
| Tier 1 heuristics | < 1s | — |
| Full 4-part character, GPU | ~4–6 min | — |

These targets are *loose* on purpose. V1 prioritises "it works reliably" over "it's fast." Performance tuning happens after behaviour is correct.

---

## What changes over time, and how

| Thing | Mechanism for improvement |
|---|---|
| Segmentation quality | Swap `ISegmenter` impl when SAM3 ships |
| Reconstruction quality | Swap `IReconstructor` impl when better single-image 3D ships |
| AI review quality | Bump `ClaudeReviewer` model ID (e.g., `claude-opus-4-7` → `claude-opus-5`) |
| User's personal defaults | Recipe book grows naturally per generation |
| Supported component types | Add new types to the classifier + fit rules; add to recipe book schema with migration |
| Cloth / hair support | Unlocked when 2D→3D quality crosses the threshold — flip a feature flag |

The pipeline itself does not rewrite itself. It does not need to.
