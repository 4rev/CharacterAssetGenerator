# Implementation Plan (V1)

**Philosophy:** ship what today's tech does well. Hard-surface modular assets, local-only, on-demand AI review.

**Target:** 10–14 weeks solo, part-time (alongside a primary project). 6–8 weeks if full-time.

**Explicitly cut from V1:** cloth & hair reconstruction, multiple skeleton variants, Tier 2 local classifier, global server, crowdsourced retraining, opt-in anonymisation. These are revisited only after V1 ships and only if warranted.

---

## Phase 1 — Foundation (Weeks 1–3)
*Scaffold, build toolchain, adapter interfaces, UE5 integration spike*

### 1.1 CMake + vcpkg scaffold
Top-level `CMakeLists.txt`, per-module sub-lists (core, ml, mesh, export). `vcpkg.json` manifest. Proves the build toolchain works end-to-end with at least one non-trivial dep (OpenCV).
**Tech:** CMake 3.26+, vcpkg, MSVC 2022

### 1.2 UE5 plugin ↔ CMake library integration spike
**Budget a full week for this.** UBT linking against vcpkg-built binaries is a known pain point. Resolve it on empty stubs before writing real code. Deliverable: UE5 Editor loads the plugin, calls a stub function in the CMake library, returns "hello world."
**Tech:** UE5 5.3+, UBT, Third Party Module pattern

### 1.3 Adapter interfaces
Define `ISegmenter`, `IReconstructor`, `IReviewer` headers. Stub implementations that return fake data. Wire them through the pipeline skeleton end-to-end with fake data so the whole flow compiles.
**Tech:** C++20

### 1.4 Image Input Layer
Load PNG/JPG/BMP/TIFF. Multi-view grouping (front/side/back). Normalize + letterbox to 512². CLIP embedding for style_tag (from ONNX).
**Tech:** OpenCV, stb_image, ONNX Runtime

### 1.5 Python sidecar IPC
Spawn Python 3.11 subprocess. JSON protocol over local socket / named pipe. Heartbeat + auto-restart. Round-trip a "ping" / "pong" before any ML code ships.
**Tech:** plain sockets, nlohmann/json, Python 3.11

### 1.6 SQLite schema v1
`generations` and `reviews` tables (see architecture.md §8). Migration infrastructure from day one.
**Tech:** SQLite3

---

## Phase 2 — Segmentation + Classification (Weeks 4–5)
*SAM2 + component classifier, hard-surface only*

### 2.1 SAM2 ONNX via segmentation adapter
Export SAM2 to ONNX. Run via ONNX Runtime C++ behind the `ISegmenter` interface. Auto-segment mode.
**Tech:** SAM2, ONNX Runtime

### 2.2 Hard-surface component classifier
MobileNetV3-based ONNX model classifying masks into: helmet, chest_armor, pauldron, gauntlets, greaves, boots, weapon_primary, shield, belt, backpack, cape, other/unsupported.
**Tech:** MobileNetV3 (ONNX), OpenCV

### 2.3 Unsupported-component warning UI
When the classifier detects hair, cloth, dress, robe, beard — surface a clear warning in the Editor: *"Cloth/hair not supported in V1. These parts will be skipped."*
**Tech:** UE5 Slate

### 2.4 Mask post-processing
Feathering, background removal, per-component padding for reconstruction input.
**Tech:** OpenCV

---

## Phase 3 — 2D → 3D + Mesh Processing (Weeks 6–7)
*TripoSR + cleanup + UV + texture projection*

### 3.1 TripoSR via reconstruction adapter
Python sidecar runs TripoSR. Configurable per-component settings surfaced through `IReconstructor`. Returns .glb.
**Tech:** TripoSR, PyTorch, trimesh

### 3.2 Mesh cleanup pipeline
Degenerate face removal, hole filling (watertight), non-manifold fix, normal smoothing. Decimation to LOD0/LOD1/LOD2 targets.
**Tech:** OpenMesh, libigl

### 3.3 UV unwrapping
xatlas per-component. Padding config per type. Atlas packing ≥ 75%.
**Tech:** xatlas

### 3.4 Texture projection
Project concept art onto mesh. Multi-angle blend if multi-view provided. Inpaint occluded regions.
**Tech:** OpenCV, libigl, Pillow (Python side)

---

## Phase 4 — Fitting & Rigging (Weeks 8–11) — *the hard one*
*Budget 4 weeks. This is where V1 succeeds or fails.*

### 4.1 Base skeleton template
Pick **one** skeleton (UE5 Mannequin recommended). Define standard sockets. Ship as read-only template FBX bundled with the plugin.
**Tech:** FBX template, UE5 Skeleton asset

### 4.2 Mesh fitting / wrapping
Wrap generated mesh to base body surface. Offset distance per component type from hand-curated defaults. Detect and prevent interpenetration.
**Tech:** libigl deformation, distance fields

### 4.3 Skinning weights
Bounded biharmonic weights via libigl. Max 8 influences per vertex. Post-corrections for shoulders, ankles.
**Tech:** libigl, Eigen

### 4.4 Socket attachment
Rule-based (weapon → hand_r, helmet → head, shield → hand_l, backpack → spine_03, etc.). Offsets read from defaults, overridable via recipe book.
**Tech:** UE5 Skeleton API

### 4.5 Modular mesh assembly + export
Output one Skeletal Mesh per component, all referencing the master skeleton. Material slot naming convention for Blueprint swapping. Export via Assimp to glTF primarily; FBX optional.
**Tech:** Assimp, UE5 Skeletal Mesh factory

---

## Phase 5 — Quality Review + Recipe Book (Weeks 12–13)
*Tier 1 heuristics + Tier 3 on-demand Claude review + local recipe book*

### 5.1 Tier 1 heuristics
Watertight, UV overlap, poly count, normal consistency, texture stretch, silhouette match. All deterministic, free, always runs.
**Tech:** C++, libigl, OpenCV

### 5.2 Claude reviewer adapter (Tier 3)
`IReviewer` implementation. Renders mesh from ~4 angles, packages renders + Tier 1 metrics + component type + style tag, sends to Claude API. Parses structured response (fix from preset catalog). User-initiated only.
**Tech:** libcurl, Claude API, nlohmann/json

### 5.3 Fix catalog executor
C++ executor for the preset fix operations: `remesh_higher_density`, `repack_uv_larger_padding`, `refit_offset_increase`, `reproject_texture_different_angle`, `regenerate_with_different_backend`, `accept_as_is`. Each is a well-defined pipeline re-run on the affected stage.
**Tech:** C++, OpenMesh, libigl, xatlas

### 5.4 Recipe book read/write
On generation: lookup best-weighted settings for `(component_type, style_tag, current_model_versions)`. Pre-apply. After generation: log full record including user accept/reject.
**Tech:** SQLite3

### 5.5 API budget + key management
API key stored in OS keychain. Monthly spend display. Budget cap enforced client-side. All Tier 3 calls are user-initiated so budget can be hit only through explicit user action.
**Tech:** OS keychain, UE5 Slate

---

## Phase 6 — UI & Release Polish (Week 14)
*Editor UI, docs, GitHub release*

### 6.1 Editor UI
Image drop zone. Style tag display. Component preview with Tier 1 metrics. "Review with AI" button per component. Accept / reject buttons. Content Browser import on accept.
**Tech:** UE5 Slate

### 6.2 Component library browser
Simple in-editor browser of previously generated components (queried from the recipe book). Filter by type, style, date. Thumbnail previews.
**Tech:** UE5 Slate, SQLite

### 6.3 Docs + release
README, CONTRIBUTING, PRIVACY, setup guide, known limitations section (explicitly lists what's unsupported). GitHub release tagged v0.1.0.
**Tech:** Markdown, GitHub

---

## Timeline Summary

| Phase | Title | Weeks | Part-time est. |
|---|---|---|---|
| 1 | Foundation | 1–3 | 3 wk |
| 2 | Segmentation | 4–5 | 2 wk |
| 3 | 2D→3D + Mesh | 6–7 | 2 wk |
| 4 | Fitting & Rigging | 8–11 | 4 wk *(the hard one)* |
| 5 | Quality Review + Recipe Book | 12–13 | 2 wk |
| 6 | UI & Release | 14 | 1 wk |

**Total V1 part-time estimate: ~14 weeks** (vs original plan's 20). Shorter despite being more honest, because the cuts (no Tier 2, no server, no cloth/hair, no multi-skeleton) remove ~6 weeks of work.

---

## Quality Expectations

| Milestone | Expected Output Quality |
|---|---|
| End of Phase 4 | Rigged hard-surface assets that fit the base skeleton and don't interpenetrate. Usable but rough textures. |
| End of Phase 5 | Usable assets with Tier 1 gating obvious failures + optional AI review for suggested fixes. |
| After 50 personal generations | Pre-applied settings from the recipe book give noticeable time savings on repeat styles. |

**No promises about "90%+ quality after N generations."** The quality ceiling is set by TripoSR and SAM2. No recipe-book learning exceeds that ceiling. What the recipe book does is **get the user to that ceiling faster on their own art style**.

When TripoSR is replaced by a better model, the ceiling rises — and because of the adapter boundary, that upgrade is a one-file change.

---

## Post-V1 Candidates (*not committed*)

Only considered if V1 ships and has users:

- **Cloth & hair support** — when single-image reconstruction for soft surfaces becomes reliable.
- **Multiple base skeletons** — once fitting quality is proven on the first one.
- **Tier 2 local classifier** — once a user has 300+ personal generations to train on.
- **Opt-in anonymised sharing** — only if demanded; server infrastructure stays deferred.
- **Additional component types** — based on actual user requests, not speculative coverage.
