# Acceptance Criteria

Every task in the implementation plan has measurable pass/fail criteria listed here. No criterion uses subjective language — each states *what* to verify and *how* to verify it.

**Legend:**
- 🟢 Easy task
- 🟡 Medium task  
- 🔴 Hard task

---

## Phase 1 — Foundation

### 1.1 CMake Project Structure 🟢
1. `cmake --build` succeeds with zero errors and zero warnings on Windows (MSVC) and Linux (GCC/Clang) from a fresh clone
2. Each module (core, ML, mesh, export, server, UE5) compiles independently — removing one does not break others
3. vcpkg installs all dependencies automatically via `vcpkg.json` manifest, no manual steps required
4. Plugin `.uplugin` descriptor loads in UE5 Editor without errors after CMake-generated build
5. A basic GitHub Actions workflow runs the build on push and reports pass/fail

### 1.2 Image Input Layer 🟢
1. Successfully loads PNG, JPG, BMP, TIFF, WebP without crash or corruption on a test set of 20 diverse images
2. Front/side/back images correctly tagged and stored as a view group in a single session struct
3. Output images are normalized to [0,1] float range, resized to 512×512 with aspect-preserving letterbox
4. Corrupt, missing, or unsupported files return a typed error — no unhandled exceptions or crashes
5. Loading and preprocessing a 4K source image completes in under 200ms on a mid-range CPU

### 1.3 Python Sidecar IPC 🟡
1. C++ successfully spawns and connects to Python sidecar in under 3 seconds on cold start
2. JSON command → response round-trip (excluding ML inference) completes in under 50ms
3. If sidecar crashes, C++ detects loss of heartbeat within 5 seconds and restarts automatically without user intervention
4. Malformed or unexpected JSON responses are caught, logged, and return a typed error — no undefined behaviour
5. Multiple sequential commands do not interleave or corrupt each other across 100 rapid-fire test calls

### 1.4 SQLite Component Library Schema 🟢
1. All 5 tables (components, corrections, style_fingerprints, pipeline_settings, quality_history) created with correct types, constraints, and indices
2. Schema version tracked in DB; migration scripts run automatically on open; downgrade path documented
3. Insert, read, update, delete verified for each table via unit tests with 100% pass rate
4. WAL mode enabled; simultaneous read+write from two threads does not corrupt DB across 1000 operations
5. Plugin initialises correctly on a brand new machine with no existing DB — creates fresh schema without user action

### 1.5 Basic UE5 Plugin Shell 🟡
1. Plugin activates in UE5 5.3+ Editor with no errors in Output Log on both blank and existing projects
2. Plugin button appears in main Editor toolbar and opens the main panel without crash
3. API key stored in OS credential store (Windows Credential Manager / libsecret), never written to `.ini` or project files
4. Monthly budget cap value survives Editor restart — stored in `EditorPerProjectUserSettings`, not source-controlled config
5. Disabling the plugin from Plugin Manager leaves the project in a clean state with no residual assets or errors

---

## Phase 2 — Segmentation

### 2.1 SAM2 ONNX Integration 🔴
1. SAM2 ONNX model loads without error on CPU; loads with GPU acceleration when CUDA is available
2. Auto-segment mode detects at least 80% of visible clothing regions on a test set of 30 diverse character concept images
3. Generated masks have IoU ≥ 0.75 compared to manually drawn reference masks on test set
4. Full auto-segment of a 512×512 image completes in under 8 seconds on CPU; under 1.5 seconds with GPU
5. If no CUDA device found, automatically falls back to CPU with a log message — no crash or missing feature

### 2.2 Component Classifier 🟡
1. ≥ 85% accuracy on a held-out test set of 200 pre-labelled character part images across all 9 component categories
2. Classifier returns a confidence score [0–1] per class; high confidence (>0.85) is actually correct ≥ 90% of the time
3. Ambiguous or non-character masks are classified as `unknown` and excluded from pipeline rather than misclassified
4. Classification of a single masked region completes in under 100ms on CPU
5. Classifier model path is config-driven — swapping in a retrained ONNX model requires no code change

### 2.3 Style Fingerprinting 🟡
1. ≥ 80% correct style classification on a test set of 50 images across 5 style categories
2. Style fingerprint (label + confidence vector) written to SQLite `style_fingerprints` table for every processed image
3. Each detected style maps to a distinct set of downstream parameters; verified by inspection of parameter table
4. Images not clearly matching any known style get `unknown` fingerprint; pipeline uses conservative defaults
5. CLIP inference completes in under 500ms on CPU per image

### 2.4 Mask Post-processing 🟢
1. Output masks have smooth, anti-aliased edges — no jagged pixel staircasing visible at 1:1 zoom
2. Isolated part images have clean transparent background with no colour bleed from adjacent regions
3. Each part image padded to square with configurable margin (default 10%) — TripoSR receives correct input dimensions
4. Part images retain full detail of the original concept art region with no blurring or compression artefacts
5. All parts from the same input image processed in under 2 seconds total

---

## Phase 3 — 2D → 3D

### 3.1 TripoSR Python Sidecar 🟡
1. Every successful call returns a valid, parseable `.glb` file with at least one mesh and one texture
2. Output mesh is watertight, has correct face winding, and roughly matches input silhouette on 90% of test cases
3. Mesh density and inference resolution differ between hard-surface and soft component types — verified by poly count comparison
4. If TripoSR fails (GPU OOM, corrupt input), sidecar returns typed error JSON — C++ logs and skips part rather than crashing
5. Reconstruction of a single part completes in under 60 seconds on CPU; under 15 seconds with GPU

### 3.2 SF3D Fallback 🟡
1. Clothing/hair components automatically route to SF3D; hard-surface components route to TripoSR — verified by pipeline log
2. SF3D and TripoSR both return identically structured JSON + `.glb` — downstream C++ cannot distinguish backends
3. User can force a specific backend via config flag — takes effect without restarting the plugin
4. If primary backend fails, automatically retries with the other backend before returning an error
5. On a 20-image clothing test set, SF3D output scores ≥ 10% higher silhouette match than TripoSR baseline

### 3.3 Mesh Cleanup Pipeline 🟡
1. 100% of output meshes pass watertight check after cleanup — zero holes detected by libigl boundary edge test
2. Zero zero-area faces, zero duplicate vertices, zero non-manifold edges in output mesh
3. LOD0 ≤ target poly count, LOD1 ≤ 50% of LOD0, LOD2 ≤ 20% of LOD0 — verified programmatically per component
4. After decimation, silhouette match score vs input drops by no more than 15% between LOD0 and LOD2
5. Full cleanup + 3-LOD generation for a single mesh completes in under 10 seconds on CPU

### 3.4 UV Unwrapping 🟢
1. xatlas output has zero overlapping UV islands — verified by automated overlap detection test
2. UV atlas utilisation ≥ 75% of available texture space on all test meshes
3. UV seams placed in visually inconspicuous locations (back of head, inside of limbs) — verified by visual inspection checklist
4. Armor parts use 4px island padding; clothing uses 2px — configurable per component type
5. UV unwrap of a 5000-poly mesh completes in under 3 seconds

### 3.5 Texture Projection 🔴
1. Projected texture average pixel colour error vs source image ≤ 15% on visible surfaces of test set
2. When front+side views provided, seam between projection zones is invisible at normal viewing distance
3. Occluded/backface regions are inpainted with plausible colour rather than black or UV-default grey
4. Output texture is minimum 1024×1024 for LOD0, 512×512 for LOD1, 256×256 for LOD2
5. Straight lines in source concept art remain straight on flat mesh surfaces after projection

---

## Phase 4 — Assembly

### 4.1 Base Body Template System 🟡
1. Master skeleton imports into UE5 with correct bone hierarchy, compatible with UE5 Mannequin for retargeting
2. Male, female, and creature base meshes each bind cleanly to master skeleton with no skinning errors
3. Base templates are read-only in the pipeline — no generation run can modify them, only read them
4. All standard sockets defined (`head`, `hand_l`, `hand_r`, `spine_01/02/03`, `foot_l`, `foot_r`) with correct transforms
5. Template exported as FBX, re-imported into UE5, re-exported — skeleton and mesh remain bit-identical after round-trip

### 4.2 Mesh Fitting / Wrapping 🔴
1. Fitted clothing mesh does not intersect base body mesh — zero interpenetrating faces on a test set of 20 clothing items
2. Tight clothing sits within 3mm of body surface; armor within 12mm — verified by ray-cast distance measurement
3. After fitting, clothing silhouette matches pre-fit mesh silhouette within 5% area difference
4. When SQLite has ≥ 3 corrections for a component type, learned offset used automatically — verified by log output
5. Mesh fitting for a single clothing component completes in under 20 seconds on CPU

### 4.3 Skeleton Socket Attachment 🟡
1. All 9 component types map to correct socket — verified against defined mapping table with 100% match
2. Attached components have correct position/orientation in T-pose — no visible offsets or rotations in UE5 viewport
3. Cape correctly attaches to multiple spine sockets simultaneously with correct weight distribution
4. Per-component attachment offset corrections from SQLite applied automatically when available
5. Swapping one hair mesh for another at runtime in UE5 Blueprint works without requiring editor restart

### 4.4 Skinning Weights 🔴
1. All vertex weights sum to exactly 1.0 — verified programmatically across all vertices of all test meshes
2. Mesh deforms without self-intersection or extreme distortion across a standard 10-pose animation test suite
3. Known bad zones (boot ankle, shoulder pauldron, cape collar) have correction rules applied when confidence ≥ 0.7
4. No vertex influenced by more than 8 bones — UE5 hardware skinning limit respected
5. Skinned mesh imports into UE5 with no skinning warnings in Output Log

### 4.5 Modular Mesh Assembly 🟡
1. All exported component meshes reference the same master skeleton asset in UE5
2. Each component is a separate Skeletal Mesh asset that can be replaced without affecting others
3. All components align correctly in T-pose with no visible gaps, overlaps, or misalignments in UE5 viewport
4. Material slots follow consistent naming convention compatible with runtime material swapping in Blueprint
5. Full assembled character plays a standard UE5 Mannequin animation with no skinning errors or mesh explosions

---

## Phase 5 — Self-Improvement

### 5.1 Tier 1: Local Heuristics Engine 🟡
1. All 6 heuristic checks implemented and returning numeric scores (watertight, UV overlap, poly count, normals, texture stretch, silhouette)
2. Heuristics flag fewer than 10% of visually acceptable meshes as failing on a 50-mesh human-rated test set
3. Heuristics catch at least 60% of visually bad meshes flagged by human review on same test set
4. All 6 checks on a single mesh complete in under 500ms
5. All heuristic scores written to SQLite `quality_history` table per generation

### 5.2 Tier 2: Local ONNX Classifier 🔴
1. Classifier can be retrained from SQLite correction history via a single CLI command
2. Classifier trains successfully with as few as 50 correction records — degrades gracefully below threshold
3. After 100 corrections, classifier achieves ≥ 75% accuracy on a held-out validation split
4. Cases where classifier confidence < 60% correctly escalate to Tier 3 — verified on 20 ambiguous test meshes
5. Updated classifier ONNX weights load at runtime without plugin restart

### 5.3 Tier 3: API Feedback Integration 🟡
1. 100% of valid API responses parsed into typed C++ structs with no runtime errors across 30-response test suite
2. Plugin makes zero API calls once monthly budget is reached — verified by mock billing counter test
3. At least 8 distinct fix types implemented and callable from JSON instruction
4. Malformed or empty API responses caught — pipeline continues with unmodified mesh rather than crashing
5. Every API call logged to SQLite with timestamp, component type, cost estimate, and response summary

### 5.4 Correction Pattern DB 🟢
1. Every record contains: `component_type`, `style_fingerprint_id`, `settings_before`, `settings_after`, `quality_before`, `quality_after`, `source`
2. Automated scan of DB confirms zero file paths, usernames, IPs, or API key fragments in any correction record
3. Similarity query (top-5 for a given fingerprint) returns in under 100ms at 10,000 records
4. Corrections exportable to anonymised JSON in a documented schema
5. Quality delta values reflect real improvement — validated on 20 before/after mesh pairs, ≥ 80% human agreement

### 5.5 Fix Library (C++) 🔴
1. Minimum 8 distinct fixes implemented, each callable by string name from the JSON fix executor
2. Applying the same fix twice produces the same result as applying it once — no compounding degradation
3. Every fix re-runs Tier 1 heuristics after execution and logs whether quality improved
4. No fix makes a passing heuristic fail — each fix only modifies the metric it targets
5. Receiving an unknown fix name returns a typed error and leaves the mesh unchanged

### 5.6 Batch Deferred Review 🟢
1. Queued meshes survive plugin reload — queue stored in SQLite, not in memory only
2. Batch capped at configurable max (default 10 items) — overflow rolls to next session
3. UI shows estimated API cost for pending batch before user triggers review
4. If API returns results for 4 of 5 queued items, the 4 are processed and the 1 failure is re-queued, not lost
5. User can trigger batch review manually at any time, not only at session end

---

## Phase 6 — Global Network

### 6.1 Anonymization Layer 🟡
1. Automated PII scan finds zero hits (file paths, usernames, IPs, API keys, machine IDs) in 100 synthetic upload payloads
2. Anonymous user hash rotates weekly — two uploads from the same user in different weeks cannot be linked
3. Anonymization code in a standalone, dependency-minimal file readable without building the full project
4. Global sharing is opt-in, disabled by default — verified by inspecting default config
5. Disabling sharing stops all uploads immediately — verified by network monitor showing zero outbound correction calls after opt-out

### 6.2 Global Server 🟡
1. Server achieves 99.5% uptime over a 30-day period — plugin gracefully queues uploads during downtime
2. Server rejects malformed payloads with HTTP 400; valid payloads return HTTP 201 in under 200ms
3. Each correction record stored in under 2KB — 1 million records consume under 2GB storage
4. Per-client rate limit of 100 uploads/hour enforced — excess requests return HTTP 429 without crashing server
5. Admin dashboard shows total records, records per day, top component types, and model version history

### 6.3 Global Tier 2 Model Retraining 🔴
1. Retraining runs on a weekly schedule without manual intervention — success/failure notification via CI
2. Every model tagged with version, training date, record count, and validation accuracy in model registry
3. New model only published if validation accuracy ≥ previous version − 2% — automatic rollback if quality drops
4. Exported ONNX model loads correctly in ONNX Runtime C++ API without opset errors
5. Model trained on only 50 records still produces better-than-random results — degraded but functional

### 6.4 Plugin Update Sync 🟢
1. Version check on launch completes in under 1 second — uses lightweight HEAD request, not full download
2. Model file downloads in a background thread — Editor remains fully responsive during download
3. Downloaded file hash verified against server-provided SHA256 before applying — corrupt downloads discarded and retried
4. Updated model hot-loaded into ONNX Runtime in-process — verified by running inference before and after in same session
5. If server unreachable, plugin starts normally with last downloaded model — no blocking dialog or failure

### 6.5 Style Fingerprint Library 🟡
1. Published library contains settings entries for at least 10 distinct style categories after first month of real usage
2. Local SQLite correctly merges incoming global library without overwriting user's own learned corrections
3. Characters using global library settings score ≥ 10% higher on Tier 1 heuristics than defaults, on unseen styles
4. When global and local settings conflict for same style+component, local takes priority — documented and tested
5. Monthly library update package under 500KB — JSON, not binary, human-readable

---

## Phase 7 — Polish & Release

### 7.1 Full UE5 Editor UI 🟡
1. Images drag-dropped from Windows Explorer are loaded correctly — supports PNG, JPG, TIFF
2. User can individually approve or reject each component before import — rejected parts re-queued, not silently discarded
3. Tier 1 heuristic scores displayed numerically per component with colour coding (green/amber/red)
4. Approved components import to a user-specified Content Browser folder with correct asset names — no file browser dialog
5. UI remains interactive during generation — generation runs in a background thread with progress bar

### 7.2 Component Library Browser 🟡
1. Filtering by component type, style tag, and date range all work and update results in under 500ms at 1000 entries
2. Thumbnails auto-generated for every component on import and cached — displayed without reimporting
3. User can select one part of each component type from library and trigger assembly — completes without manual rigging
4. Deleting a component prompts confirmation and removes both SQLite record and UE5 asset atomically
5. Browser remains responsive with 500+ library entries — no UI freeze when scrolling or filtering

### 7.3 API Key & Budget UI 🟢
1. API key written to OS keychain — confirmed absent from all project files, logs, and SQLite
2. Displayed monthly spend matches actual API call log total within 5%
3. Changing the budget cap in UI takes effect immediately — no restart required
4. Global sharing toggle correctly reflected in upload behaviour within the same session as the change
5. Entering an invalid API key shows a clear error message within 3 seconds — does not silently fail later

### 7.4 GitHub Open Source Release 🟢
1. Developer with no prior knowledge can produce a working plugin build by following the README alone
2. `PRIVACY.md` clearly documents exactly what data is uploaded, when, and how to verify it — reviewed against upload code
3. `CONTRIBUTING.md` covers: running tests, adding a new fix type, adding a component category, retraining the classifier
4. All third-party dependencies are MIT/Apache/BSD compatible — `LICENSES` file lists each with its license
5. GitHub repo has issue templates for: bug report, feature request, and new fix type proposal

### 7.5 Epic Marketplace Listing (Free) 🟢
1. Plugin passes Epic's technical review and is live on Marketplace as a free listing
2. Listing includes: feature description, 5+ screenshots of real results, 60-second demo video, link to GitHub
3. Listing correctly tagged for supported UE5 versions (minimum 5.3) — installation works from Marketplace without manual steps
4. Listing links to GitHub Discussions as the support channel
5. Marketplace version number matches GitHub release tag — no discrepancy between Marketplace and GitHub

---

## Summary

| Phase | Tasks | Criteria |
|---|---|---|
| 1 Foundation | 5 | 25 |
| 2 Segmentation | 4 | 20 |
| 3 2D → 3D | 5 | 25 |
| 4 Assembly | 5 | 25 |
| 5 Self-Improvement | 6 | 30 |
| 6 Global Network | 5 | 25 |
| 7 Polish & Release | 5 | 25 |
| **Total** | **35** | **175** |
