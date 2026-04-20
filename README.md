# Modular Character Asset Generator

> A free, open-source UE5 plugin that generates rigged, modular game characters from concept art images — locally, privately, and with a self-improving pipeline.

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)
![UE5 Version](https://img.shields.io/badge/UE5-5.3%2B-blue.svg)
![Platform](https://img.shields.io/badge/Platform-Windows%20%7C%20Linux-lightgrey.svg)
![Status](https://img.shields.io/badge/Status-In%20Development-orange.svg)

---

## What It Does

Drop a concept art image into the UE5 Editor. Get back a fully rigged, UV-unwrapped, textured **modular skeletal mesh** — with separate swappable assets for hair, clothing, armor, weapons, and accessories — ready to animate.

```
Concept art image
      ↓
Auto-segmentation (hair / shirt / armor / boots / weapon...)
      ↓
Per-part 2D → 3D reconstruction (runs fully locally)
      ↓
Mesh cleanup + UV unwrap + texture projection
      ↓
Fitted to base body skeleton + skinning weights
      ↓
UE5 Modular Skeletal Mesh assets in your Content Browser
```

---

## Key Features

- **Fully local pipeline** — no cloud service required, no meshes or images ever leave your machine
- **Your own API key** — optionally use your own Claude/GPT-4V key for AI quality feedback; you control the cost
- **Self-improving** — the pipeline learns from corrections over time, reducing AI API calls toward zero
- **Crowdsourced intelligence** — opt-in to share anonymised correction patterns globally; everyone's pipeline improves
- **Modular output** — each component (hair, boots, sword...) is a separate UE5 Skeletal Mesh sharing one master skeleton
- **Component library** — browse, mix, and reuse all previously generated parts inside the Editor
- **Free & open source** — MIT license, always

---

## Repository Structure

```
CharacterAssetGenerator/
├── README.md
├── CONTRIBUTING.md
├── PRIVACY.md
├── LICENSE
│
├── docs/
│   ├── architecture.md          ← Full pipeline architecture
│   ├── implementation_plan.md   ← 7-phase build roadmap
│   ├── acceptance_criteria.md   ← Pass/fail criteria for every task
│   └── privacy_policy.md        ← Exactly what data is shared and when
│
├── src/
│   ├── core/                    ← Core C++ library (image, IPC, DB)
│   ├── ml/                      ← ONNX inference wrappers
│   ├── mesh/                    ← Mesh processing (cleanup, UV, fitting)
│   ├── export/                  ← FBX/glTF export layer
│   ├── server/                  ← Global sync client
│   └── ue5_plugin/              ← UE5 Editor plugin (Slate UI, build targets)
│
├── python_sidecar/              ← Python ML sidecar (TripoSR, SF3D, SAM2)
│
├── server/                      ← Global aggregation server (Rust/Go)
│
├── training/                    ← Tier 2 classifier training pipeline (PyTorch)
│
└── tests/
    ├── unit/                    ← Per-module unit tests
    ├── integration/             ← Pipeline integration tests
    └── test_assets/             ← Reference images and meshes for CI
```

---

## Prerequisites

| Requirement | Version |
|---|---|
| Unreal Engine | 5.3 or later |
| CMake | 3.26+ |
| Visual Studio | 2022 (Windows) |
| Python | 3.11+ |
| CUDA (optional) | 11.8+ for GPU acceleration |
| vcpkg | latest |

---

## Quick Start

```bash
# 1. Clone the repo
git clone https://github.com/YOUR_USERNAME/CharacterAssetGenerator.git
cd CharacterAssetGenerator

# 2. Install C++ dependencies
vcpkg install

# 3. Install Python sidecar dependencies
cd python_sidecar
pip install -r requirements.txt

# 4. Build
cmake -B build -DCMAKE_TOOLCHAIN_FILE=[vcpkg root]/scripts/buildsystems/vcpkg.cmake
cmake --build build --config Release

# 5. Install plugin into UE5 project
# Copy build/ue5_plugin/ into your UE5 project's Plugins/ folder
# Enable the plugin in Edit → Plugins → Character Asset Generator
```

Full setup guide: [`docs/setup.md`](docs/setup.md) *(coming soon)*

---

## Privacy

**Nothing sensitive ever leaves your machine.**

What stays local: your images, generated meshes, API key, and identity.  
What is optionally shared (opt-in, disabled by default): anonymised correction patterns — quality scores and settings that worked, with no geometry or imagery attached.

Full details: [`PRIVACY.md`](PRIVACY.md)

---

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for how to:
- Run the test suite
- Add a new fix type to the self-improvement engine
- Add a new component category
- Retrain the local quality classifier

---

## Roadmap

See [`docs/implementation_plan.md`](docs/implementation_plan.md) for the full 7-phase, 20-week build plan with acceptance criteria per task.

---

## License

MIT — see [`LICENSE`](LICENSE)
