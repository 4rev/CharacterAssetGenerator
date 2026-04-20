# Implementation Plan

7 phases, ~20 weeks to a releasable open-source plugin.

---

## Phase 1 — Foundation (Weeks 1–2)
*Project scaffold & core infrastructure*

### 1.1 CMake Project Structure
Modular CMake setup: core lib, ML module, mesh module, export module, server module, UE5 plugin wrapper. Windows-first, Linux-compatible.  
**Tech:** CMake, C++20, vcpkg

### 1.2 Image Input Layer
Load PNG/JPG/BMP/TIFF concept art. Multi-view support (front/side/back). Basic preprocessing: normalize, resize, contrast enhance per-style.  
**Tech:** OpenCV, stb_image

### 1.3 Python Sidecar IPC
C++ launches Python subprocess. Communication via local socket or named pipe. JSON protocol for commands and results. Heartbeat + auto-restart.  
**Tech:** Boost.Asio, nlohmann/json, Python 3.11

### 1.4 SQLite Component Library Schema
Tables: components, corrections, style_fingerprints, pipeline_settings, quality_history. Migrations system from day one.  
**Tech:** SQLite3, C++ ORM wrapper

### 1.5 Basic UE5 Plugin Shell
Editor module, empty toolbar button, settings panel exposing API key field and budget cap. Plugin descriptor and build targets.  
**Tech:** UE5 C++, Slate UI

---

## Phase 2 — Segmentation (Weeks 3–4)
*SAM2-powered part isolation*

### 2.1 SAM2 ONNX Integration
Export SAM2 weights to ONNX. Run via ONNX Runtime C++ API. Auto-segment mode first, then prompt-guided refinement.  
**Tech:** ONNX Runtime, SAM2, CUDA optional

### 2.2 Component Classifier
Post-SAM2 classification of each mask: hair, shirt, pants, boots, armor, weapon, accessory. Small ONNX classifier.  
**Tech:** ONNX Runtime, OpenCV

### 2.3 Style Fingerprinting
Detect art style from input: realistic, anime, dark_fantasy, cartoon, stylized. Drives downstream parameter selection. Stored per-generation.  
**Tech:** CLIP ONNX, SQLite

### 2.4 Mask Post-processing
Clean isolated part images: feathering, background removal, padding for 3D reconstruction. Output per-part image set.  
**Tech:** OpenCV, libtiff

---

## Phase 3 — 2D → 3D (Weeks 5–7)
*Per-part mesh reconstruction*

### 3.1 TripoSR Python Sidecar
Python sidecar runs TripoSR locally. Receives masked part image, returns .glb. Configurable settings per component type.  
**Tech:** TripoSR, PyTorch, trimesh

### 3.2 SF3D Fallback
SF3D as alternative backend. Automatic selection based on component type. Fallback chain if primary fails.  
**Tech:** SF3D, Python sidecar

### 3.3 Mesh Cleanup Pipeline
Remove degenerate faces, fill holes, smooth normals, decimate to target poly count per LOD0/LOD1/LOD2.  
**Tech:** OpenMesh, CGAL, libigl

### 3.4 UV Unwrapping
xatlas per-component UV generation. Component-specific settings. Atlas packing per character.  
**Tech:** xatlas

### 3.5 Texture Projection
Project original concept art onto reconstructed mesh. Multi-angle blending. Inpaint occluded regions.  
**Tech:** OpenCV, libigl, Python/Pillow

---

## Phase 4 — Assembly (Weeks 8–10)
*Fitting, rigging & skinning*

### 4.1 Base Body Template System
Master skeleton + base meshes (male/female/creature). All generated parts fit to these. Read-only in pipeline.  
**Tech:** C++, Assimp, FBX SDK

### 4.2 Mesh Fitting / Wrapping
Wrap generated clothing to base body surface. Offset distance learned per component type. No interpenetration.  
**Tech:** libigl, CGAL, custom deformation

### 4.3 Skeleton Socket Attachment
Rule-based attachment per component type. Hair→head, weapon→hand, cape→spine. Learned offset corrections from DB.  
**Tech:** C++, UE5 Skeleton API

### 4.4 Skinning Weights
Bounded biharmonic weights via libigl per component. Post-process known problem zones (fingers, shoulders, ankles).  
**Tech:** libigl, Eigen

### 4.5 Modular Mesh Assembly
Combine parts sharing master skeleton. Output as UE5 Modular Skeletal Mesh set — separate assets per component.  
**Tech:** UE5 C++, FBX SDK, Assimp

---

## Phase 5 — Self-Improvement (Weeks 11–13)
*Tiered learning & feedback loop*

### 5.1 Tier 1: Local Heuristics Engine
Free C++ quality checks: watertight, UV overlap, poly count, normal consistency, texture stretch, silhouette match.  
**Tech:** C++, libigl, OpenCV

### 5.2 Tier 2: Local ONNX Classifier
Small quality classifier trained on local correction history. Retrained weekly. Replaces most API calls over time.  
**Tech:** ONNX Runtime, PyTorch training, SQLite

### 5.3 Tier 3: API Feedback Integration
Vision model call for novel/low-confidence cases only. Structured JSON → C++ fix executor. Hard monthly budget cap.  
**Tech:** Anthropic API, nlohmann/json, C++

### 5.4 Correction Pattern DB
Log every correction: component type, style fingerprint, settings before/after, quality delta, source tier.  
**Tech:** SQLite3, C++

### 5.5 Fix Library (C++)
Executable fixes: remesh, repack UVs, adjust smoothing, refit clothing, rebuild skinning zone, re-project texture.  
**Tech:** C++, OpenMesh, libigl, xatlas

### 5.6 Batch Deferred Review
Queue uncertain meshes during session. End-of-session batch API call reviews all at once. Cost estimate shown to user.  
**Tech:** C++, JSON batching

---

## Phase 6 — Global Network (Weeks 14–16)
*Crowdsourced improvement system*

### 6.1 Anonymization Layer
Strip all user identity before upload. Rotating weekly hash. Upload code open-sourced for user verification.  
**Tech:** C++, libsodium, open source

### 6.2 Global Server
REST API server. Receives anonymised corrections, aggregates, serves model updates. ~$5–20/mo VPS.  
**Tech:** Rust/axum, PostgreSQL, REST

### 6.3 Global Tier 2 Model Retraining
Weekly automated retraining on all user data. ONNX export. Quality gate before publish. Versioned with rollback.  
**Tech:** PyTorch, cron/CI, ONNX export

### 6.4 Plugin Update Sync
Silent background version check on launch. SHA256 integrity verification. Hot-loaded without Editor restart.  
**Tech:** C++, libcurl, versioned CDN

### 6.5 Style Fingerprint Library
Monthly aggregated style→settings map published globally. Pre-tunes pipeline for art styles users haven't seen.  
**Tech:** JSON distribution, SQLite merge

---

## Phase 7 — Polish & Release (Weeks 17–20)
*UE5 integration, UI & open source*

### 7.1 Full UE5 Editor UI
Image drop zone, component type override, style tag display, quality score display, per-part approve/reject, Content Browser import.  
**Tech:** UE5 Slate, C++

### 7.2 Component Library Browser
In-editor browser of all generated parts. Filter by type, style, date. Mix-and-match. Thumbnail previews.  
**Tech:** UE5 Slate, SQLite, C++

### 7.3 API Key & Budget UI
API key stored in OS keychain. Monthly spend display. Budget cap control. Opt-in/out global sharing toggle.  
**Tech:** UE5 Slate, OS keychain

### 7.4 GitHub Open Source Release
MIT license. README, CONTRIBUTING, PRIVACY docs. Separate repos: plugin, server, training.  
**Tech:** GitHub, Markdown

### 7.5 Epic Marketplace Listing (Free)
Free listing for discoverability. Links to GitHub. 5+ screenshots, demo video, supported UE5 versions tagged.  
**Tech:** Epic Marketplace

---

## Timeline Summary

| Phase | Title | Weeks |
|---|---|---|
| 1 | Foundation | 1–2 |
| 2 | Segmentation | 3–4 |
| 3 | 2D → 3D | 5–7 |
| 4 | Assembly | 8–10 |
| 5 | Self-Improvement | 11–13 |
| 6 | Global Network | 14–16 |
| 7 | Polish & Release | 17–20 |

## Quality Milestones

| Milestone | Expected Output Quality |
|---|---|
| End of Phase 4 | ~65% — basic rigged character, rough textures |
| End of Phase 5 | ~75% — self-correcting, consistent quality |
| After 200 generations (learned) | ~85–88% |
| After 500+ generations (mature) | ~90–93% |
