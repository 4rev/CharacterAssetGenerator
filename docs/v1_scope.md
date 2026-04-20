# V1 Scope — the realistic first release

This document defines **what we actually ship first**, as a subset of the full vision in [`architecture.md`](architecture.md) and [`implementation_plan.md`](implementation_plan.md).

The end goal remains: **high-quality 2D → 3D modular characters for UE5**, improving as AI models improve. V1 is the foundation that gets us there without overcommitting while today's 2D→3D models still have real limits.

## Philosophy

**Do well what today's tech does well. Grow the scope as AI models improve.** Every piece deferred from V1 is deferred for a *specific* reason — usually because the underlying model quality isn't there yet. When it gets there, we unlock that piece.

The architecture invests in clean **model-swap adapter boundaries** so upgrading SAM2 → SAM3, TripoSR → next-gen reconstruction, Claude Opus 4.7 → Opus 5/6/7 is a one-file change — not a rewrite.

## What V1 ships

- **Hard-surface components only:** helmet, chest_armor, pauldron, gauntlets, greaves, boots, weapon_primary, shield, belt, backpack, cape
- **Single base skeleton** (UE5 Mannequin recommended) — chosen at install, not auto-detected
- **Local-only pipeline** — no server, no crowdsourced correction sharing
- **On-demand Claude API review** — user clicks "review this mesh"; Claude picks from a preset fix catalog (not open-ended fix generation)
- **Per-user SQLite recipe book** — remembers settings that worked for this user on `(component_type, style_tag, model_versions)`; decays stale records as models upgrade
- **Three adapter interfaces** — `ISegmenter`, `IReconstructor`, `IReviewer` — so future model upgrades are drop-in replacements
- **UE5 editor plugin** with one-button "Generate from image" + a component library browser

## What V1 defers (and why)

| Deferred feature | Why deferred | When it unlocks |
|---|---|---|
| Cloth and hair reconstruction | Single-image 3D models (TripoSR, SF3D) produce low-quality cloth and hair today. Not solvable with parameter tuning. | When a stronger single-image soft-surface reconstruction model ships. Flip a feature flag; the adapter is already in place. |
| Multiple base skeletons (male/female/creature) | Auto-fitting + skinning quality on one skeleton is already the hardest part. Validate one before adding more. | Once V1 fitting quality is proven in real use. |
| Tier 2 local quality classifier | Needs ~300+ personal generations to train usefully. V1 users won't have that. | When an early user reports hitting the threshold. |
| Global server + crowdsourced correction sharing | Overengineered for ~0 users at launch. Aggregated style-fingerprint data only starts being useful past ~10K records (~1 year of sustained usage). | Only if/when the plugin has sustained usage and user demand. |
| Automatic tier cascade (Tier 1 → 2 → 3) | Tier 2 doesn't exist in V1; cascading across just two tiers adds control complexity without value. Replaced by explicit user-initiated review. | When Tier 2 is added. |
| Photorealistic quality ambition | Ceiling is set by TripoSR/SF3D. No pipeline tuning breaks through it. | As reconstruction models improve. |

## What V1 explicitly won't promise

- **No "90%+ quality after N generations"** marketing claims. The quality ceiling is the 2D→3D model's ceiling. The recipe book gets users to that ceiling *faster* on their own art style — it doesn't raise the ceiling.
- **No guarantee that Claude review produces usable output.** Claude suggests fixes from the preset catalog; some problems have no available fix. The API is an accelerant, not a safety net.
- **No stealth network activity.** Zero outbound network calls unless the user explicitly triggers a Claude review with their own API key.

## V1 timeline (6 phases, ~14 weeks part-time)

Tighter than the full-scope 20-week plan because of the deferrals. Phase 4 gets the lion's share because fitting + skinning is the hardest problem and where V1 lives or dies.

| Phase | Focus | Weeks |
|---|---|---|
| 1 | Foundation (CMake/vcpkg, UE5/UBT integration spike, adapter interfaces, IPC, SQLite) | 1–3 |
| 2 | Segmentation + hard-surface component classifier | 4–5 |
| 3 | 2D→3D (TripoSR) + mesh cleanup/UV/texture projection | 6–7 |
| 4 | Fitting & rigging (bounded biharmonic skinning on single skeleton) | 8–11 |
| 5 | Quality review (Tier 1 automatic + Tier 3 on-demand) + recipe book | 12–13 |
| 6 | UE5 Editor UI & v0.1.0 release | 14 |

## Adapter boundaries — the one bet on the future

Three pipeline stages are isolated behind stable interfaces:

```cpp
// src/ml/segmentation/ISegmenter.h
class ISegmenter {
    virtual std::vector<MaskedRegion> Segment(const Image& input) = 0;
    virtual std::string ModelName() const = 0;
    virtual std::string ModelVersion() const = 0;
};

// src/ml/reconstruction/IReconstructor.h
class IReconstructor {
    virtual ReconstructResult Reconstruct(const MaskedImage&, const ReconstructSettings&) = 0;
    virtual bool SupportsComponent(ComponentType) const = 0;
    virtual std::string ModelName() const = 0;
    virtual std::string ModelVersion() const = 0;
};

// src/ml/reviewer/IReviewer.h
class IReviewer {
    virtual ReviewResult Review(const MeshBundle&, const QualityMetrics& tier1) = 0;
    virtual std::string ProviderName() const = 0;
    virtual std::string ModelName() const = 0;
};
```

V1 implementations: `SAM2Segmenter`, `TripoSRReconstructor`, `ClaudeReviewer`.

Every recipe-book row stores `model_name + model_version` so that stale records (generated against older models) are automatically filtered once adapters are upgraded.

## V1 → full-vision transition path

Once V1 ships and has real users:

1. **Measure what's actually painful** — users will tell us whether cloth/hair absence, single-skeleton limit, or lack of Tier 2 is the biggest friction. Prioritise based on that, not speculation.
2. **Unlock deferred features when their unlock condition triggers** (see table above).
3. **The full `architecture.md` and `implementation_plan.md` docs** describe the destination. This `v1_scope.md` describes the first step. They don't conflict — V1 is a strict subset of the full vision.
