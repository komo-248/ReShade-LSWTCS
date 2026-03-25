# ReShade Config – Lego Star Wars: The Complete Saga

A ReShade post-processing configuration for *Lego Star Wars: The Complete Saga* built around a layered signal processing pipeline. Each shader stage operates on the output of the last — sharpening feeds anti-aliasing, HDR expansion feeds ambient occlusion — producing a result that is meaningfully different from stacking independent filters.

The goal is not to change the art style. TCS has a specific aesthetic — flat cel shading, saturated primaries, exaggerated proportions — and the pipeline is tuned to enhance it rather than override it. SSAO adds contact shadow depth to geometry that was designed without it. HDR expansion recovers highlight separation that the SDR framebuffer compresses. DAA cleans aliased edges without softening the linework that defines the look.

<p align="center"><em>Before (left) &nbsp;|&nbsp; After (right)</em></p>

<p align="center">
  <img src="images/comparison1.png" width="400"/>
  <img src="images/comparison2.png" width="400"/>
</p>
<p align="center">
  <img src="images/comparison3.png" width="400"/>
  <img src="images/comparison4.png" width="400"/>
</p>
<p align="center">
  <img src="images/comparison5.png" width="400"/>
  <img src="images/comparison6.png" width="400"/>
</p>

---

## Table of Contents

- [Pipeline Overview](#pipeline-overview)
- [Stage Breakdown](#stage-breakdown)
  - [1. Contrast Adaptive Sharpen (CAS)](#1-contrast-adaptive-sharpen-cas)
  - [2. Advanced Auto HDR](#2-advanced-auto-hdr)
  - [3. Directional Anti-Aliasing (DAA)](#3-directional-anti-aliasing-daa)
  - [4. Glamarye Fast Effects](#4-glamarye-fast-effects)
  - [5. MXAO](#5-mxao)
- [Installation](#installation)
- [Demonstration](#demonstration)
- [Credits](#credits)

---

## Pipeline Overview

```
Game framebuffer (SDR, aliased, flat lighting)
     │
     ▼
[CAS — Contrast Adaptive Sharpen]
  Recover high-frequency detail compressed by the game's internal renderer
  Output: sharper edges, tighter linework
     │
     ▼
[Advanced Auto HDR]
  Expand SDR luminance range into HDR headroom
  Output: separated highlights, recovered shadow detail, increased perceived depth
     │
     ▼
[DAA — Directional Anti-Aliasing]
  Temporal + directional AA pass on the sharpened, HDR-expanded signal
  Output: smooth edges without blurring the sharpened detail from CAS
     │
     ▼
[Glamarye Fast Effects]
  Edge-detect sharpen pass + fake GI tone mapping
  Output: localized contrast enhancement at geometry boundaries, global tone shaping
     │
     ▼
[MXAO — Multi-level Screen Space Ambient Occlusion]
  Two-layer SSAO on the fully processed signal
  Output: contact shadows and occlusion depth on geometry with no baked lighting
     │
     ▼
Final output
```

The ordering is deliberate. CAS runs first so DAA has a clean high-frequency signal to work with — running AA before sharpening would immediately undo the sharpening. HDR expansion runs before MXAO so the occlusion pass has accurate luminance values to blend against rather than clipped SDR output. Glamarye's tone mapping sits between AA and MXAO to shape the global tonal response before the occlusion layer is composited on top.

---

## Stage Breakdown

### 1. Contrast Adaptive Sharpen (CAS)

Per-pixel adaptive sharpening weighted by local contrast — recovers high-frequency detail softened by the renderer so every downstream stage operates on a clean, well-defined edge signal.

### 2. Advanced Auto HDR

Inverse tonemapping pass that reconstructs extended luminance range from the SDR framebuffer, separating clipped highlights and lifting shadow detail before the occlusion and AA passes run their luminance-sensitive analysis.

### 3. Directional Anti-Aliasing (DAA)

Edge-directional sub-pixel smoothing with 8-frame temporal accumulation — targets staircase aliasing on hard geometry silhouettes without blurring the sharpened linework from CAS.

### 4. Glamarye Fast Effects

Edge-detect contrast sharpening at geometry boundaries combined with a fake GI tone curve that adds perceived depth to TCS's flat-lit surfaces without AO or FXAA, which are handled by dedicated stages.

### 5. MXAO

Two-layer screen-space ambient occlusion with smooth normals reconstruction — composited last so contact shadows blend correctly against the fully processed, tone-mapped signal rather than raw SDR output.

---

## Installation

1. Install [ReShade](https://reshade.me/) for *Lego Star Wars: The Complete Saga*
2. During setup, select the following shader repositories when prompted:
   - [qUINT](https://github.com/martymcmodding/qUINT) — MXAO
   - [Glamarye Fast Effects](https://github.com/rj200/Glamarye_Fast_Effects_for_ReShade) — Glamarye
   - [iMMERSE](https://github.com/martymcmodding/iMMERSE) or standard ReShade repo — CAS, DAA
   - [AdvancedAutoHDR](https://github.com/BarbatosBachiko/AdvancedAutoHDR-ReShade) — HDR
3. Copy `LSWTCS.ini` into your ReShade presets folder
4. Launch the game, open the ReShade overlay, and select the preset from the preset dropdown

---

## Credits

**ReShade** — [reshade.me](https://reshade.me). Copyright © 2014–2024 Patrick Mours. BSD 3-Clause License.

**Shaders:**
- **CAS (Contrast Adaptive Sharpen)** — AMD / [CeeJayDK](https://github.com/CeeJayDK/SweetFX)
- **Advanced Auto HDR** — Barbatos. [BarbatosBachiko/AdvancedAutoHDR-ReShade](https://github.com/BarbatosBachiko/AdvancedAutoHDR-ReShade)
- **DAA (Directional Anti-Aliasing)** — [luluco250](https://github.com/luluco250)
- **Glamarye Fast Effects** — rj200. [rj200/Glamarye_Fast_Effects_for_ReShade](https://github.com/rj200/Glamarye_Fast_Effects_for_ReShade)
- **MXAO** — Pascal Gilcher (Marty McFly). [martymcmodding/qUINT](https://github.com/martymcmodding/qUINT)

**Game:** *Lego Star Wars: The Complete Saga* — TT Games / Warner Bros. Interactive Entertainment. All game assets and trademarks are property of their respective owners.
