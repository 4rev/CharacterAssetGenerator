# Privacy Policy

**Last updated:** 2026-04  
**Applies to:** Modular Character Asset Generator UE5 Plugin

---

## The Short Version

**Your images, meshes, and API key never leave your machine. Ever.**

Global sharing is opt-in and disabled by default. If you enable it, only anonymised quality metrics and settings are uploaded — no geometry, no imagery, no identity.

You can read and verify the upload code yourself. It is in a single, standalone file with no hidden dependencies.

---

## What Stays on Your Machine (Always)

The following data is **never transmitted anywhere**, regardless of your settings:

| Data | Where it lives |
|---|---|
| Input concept art images | Your local disk only |
| Generated 3D meshes | Your local disk only |
| Texture files | Your local disk only |
| API key (Claude / GPT-4V) | OS credential store only (Windows Credential Manager / libsecret) |
| Your username or identity | Never collected |
| Machine identifiers (MAC, hostname, hardware ID) | Never collected |
| File paths | Never transmitted |
| UE5 project name or content | Never collected |

---

## Global Sharing (Opt-In, Off by Default)

If you choose to enable global sharing in the plugin settings, the following anonymised data is uploaded to the global aggregation server:

### What is uploaded

```json
{
  "style": "dark_fantasy",
  "component": "chest_armor",
  "quality_before": 0.51,
  "quality_after": 0.84,
  "settings_applied": {
    "remesh_density": 0.8,
    "uv_padding_px": 4,
    "smoothing_iterations": 3
  },
  "anon_id": "a3f7b2c9..."
}
```

This is a **correction record** — it describes what settings improved quality for a given style and component type. Nothing about your geometry or imagery is included.

### What the `anon_id` is

The `anon_id` is a cryptographic hash derived from a randomly generated local seed, combined with a weekly salt. It rotates every 7 days. Two records uploaded in different weeks from the same machine **cannot be linked to each other** by anyone, including us.

The seed is stored locally in your OS credential store alongside your API key. It is never transmitted.

### What is never uploaded (even with sharing enabled)

- Vertices, faces, or any mesh geometry
- Images or textures of any kind
- Your API key or any fragment of it
- File system paths
- Your username, email, or any account information
- IP address (the server receives it transiently as part of HTTP but does not log or store it)
- Machine hardware identifiers

---

## How to Verify This Yourself

The upload code is isolated in a single file:

```
src/server/upload.cpp
```

This file has no hidden dependencies — it uses only the standard HTTP library (libcurl) and the JSON serialiser (nlohmann/json). You can read it in under 10 minutes and confirm exactly what is serialised into each request.

If you want to verify at the network level, you can use a proxy tool (e.g. Wireshark, Fiddler) to inspect outbound traffic from the plugin process. The upload endpoint is documented in `src/server/config.h`.

---

## Data Retention on the Global Server

- Correction records are retained for up to **2 years** for model training purposes
- After 2 years, records are deleted from the server
- You can request deletion of all records associated with your `anon_id` at any time — contact details in the GitHub repository
- Because the `anon_id` rotates weekly, records older than 7 days cannot be linked to your current identity even by us

---

## The Global Server

The global aggregation server:
- Receives anonymised correction records
- Aggregates them into a training dataset
- Retrains the Tier 2 quality classifier weekly
- Distributes updated ONNX model weights to all users
- Does **not** store IP addresses beyond the standard transient HTTP connection
- Does **not** use cookies, tracking pixels, or any analytics

Server source code is open and available in the `/server` directory of this repository.

---

## API Key Usage

Your API key (if provided) is used **only** for Tier 3 quality feedback calls — sending a rendered image of a generated mesh to a vision model for quality analysis. It is:

- Stored in your OS credential store, never in files
- Never transmitted to our servers
- Used only when your own local pipeline (Tier 1 + Tier 2) cannot confidently assess quality
- Subject to the monthly budget cap you set — the plugin will not spend beyond this cap

You can disable Tier 3 entirely by setting a budget cap of $0.00 in the plugin settings.

---

## Changes to This Policy

If this policy changes in a way that affects what data is collected or transmitted, the change will be clearly documented in the repository's git history with a descriptive commit message, and announced in the GitHub releases page.

---

## Contact

For privacy concerns or deletion requests: open an issue in the GitHub repository tagged `privacy`.
