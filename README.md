# ReShade Config – Lego Star Wars: The Complete Saga

A ReShade post-processing configuration for *Lego Star Wars: The Complete Saga* built around a layered signal processing pipeline. Each shader stage operates on the output of the last — sharpening feeds anti-aliasing, HDR expansion feeds ambient occlusion — producing a result that is meaningfully different from stacking independent filters.

The goal is not to change the art style. TCS has a specific aesthetic — flat cel shading, saturated primaries, exaggerated proportions — and the pipeline is tuned to enhance it rather than override it. SSAO adds contact shadow depth to geometry that was designed without it. HDR expansion recovers highlight separation that the SDR framebuffer compresses. DAA cleans aliased edges without softening the linework that defines the look.

<p align="center"><em>Before (left) &nbsp;|&nbsp; After (right)</em></p>

<p align="center">
  <img src="images/comparison1.png" width="440"/>
  <img src="images/comparison2.png" width="440"/>
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

**Role:** Input sharpening — recovers detail softened by the renderer before any other processing touches the signal.

CAS uses local contrast to determine sharpening strength per-pixel — areas with high existing contrast receive less sharpening, soft regions receive more. This avoids haloing artifacts on already-sharp edges while recovering softness in flat regions. At `Sharpening=1.0` with `Contrast=0.5`, the pass is aggressive but the adaptive weighting keeps it artifact-free on TCS's hard-edged geometry.

Running CAS first means every downstream stage — AA, HDR, MXAO — operates on the sharpened signal. This matters particularly for DAA and MXAO, both of which are edge-aware and produce better results when edge information in the source is clean and well-defined.

| Parameter | Value |
|-----------|-------|
| `Sharpening` | 1.0 |
| `Contrast` | 0.5 |

---

### 2. Advanced Auto HDR

**Role:** Luminance expansion — maps the SDR framebuffer into HDR headroom to separate highlight detail that is otherwise clipped.

TCS renders to a standard SDR framebuffer. In practice this means specular highlights, bright surfaces, and light bloom all clip to the same maximum white value — there is no luminance separation between a barely-lit surface and a fully saturated light source. AdvancedAutoHDR performs an inverse tonemapping pass to reconstruct an extended luminance range from the SDR signal, then applies a shoulder curve to roll off highlights naturally rather than hard-clipping them.

The configuration uses Method 1 (scene-referred HDR reconstruction) with a shoulder power of 2.5 — aggressive enough to meaningfully separate highlights without pushing the image into an unnatural HDR look. Peak white is set to 750 nits with a 400 nit max cap, calibrated for the display output level. Shadow tuning at 0.28 lifts the black floor slightly to recover detail in TCS's darker interior scenes.

This stage runs before DAA and MXAO because both passes are luminance-sensitive. MXAO in particular uses screen-space luminance to weight occlusion blending — running it on a clipped SDR signal would cause it to incorrectly treat clipped highlights as uniformly lit surfaces.

| Parameter | Value |
|-----------|-------|
| `AUTO_HDR_METHOD` | 1 |
| `AUTO_HDR_SHOULDER_POW` | 2.5 |
| `HDR_PEAK_WHITE` | 750 nits |
| `AUTO_HDR_MAX_NITS` | 400 nits |
| `SHADOW_TUNING` | 0.28 |

---

### 3. Directional Anti-Aliasing (DAA)

**Role:** Edge smoothing — removes staircase aliasing from geometry edges without softening the sharpened detail from CAS.

DAA analyzes edge directionality in the image and applies sub-pixel smoothing aligned to the detected edge angle rather than blending uniformly. The temporal accumulation pass (`AccumFrames=8`) builds a running average across 8 frames, allowing the AA to resolve sub-pixel edge positions that a single-frame pass would miss. This is particularly effective on TCS's hard geometric edges — the kind of long, straight silhouette lines that produce the worst aliasing artifacts.

`EdgeThreshold=0.39` and `EdgeFalloff=0.159` are tuned conservatively — the pass targets obvious staircase edges and leaves smooth regions untouched. Running DAA after CAS rather than before means the sharpened linework is what gets anti-aliased, not the soft pre-sharpened signal — the result is a clean, sharp edge rather than a smoothed-then-sharpened edge.

| Parameter | Value |
|-----------|-------|
| `AccumFrames` | 8 |
| `EdgeThreshold` | 0.39 |
| `EdgeFalloff` | 0.159 |
| `DirectionalStrength` | 1.0 |
| `EnableTemporalAA` | 1 |

---

### 4. Glamarye Fast Effects

**Role:** Localized contrast sharpening + global tone shaping via fake GI.

Two sub-passes are active in this stage. The edge-detect sharpen pass (`edge_detect_sharpen=1`) applies contrast enhancement specifically at detected edges — complementing CAS's global sharpen with a targeted pass that boosts local contrast at geometry boundaries. The result sharpens the perceived separation between objects without increasing noise in flat regions.

The fake GI tone mapping (`tone_map=3.0`) applies a global tonal response curve that approximates the luminance distribution of a scene with indirect bounce lighting. On TCS's flat-lit geometry this adds perceived depth — the overall contrast response curves in a way that suggests volumetric lighting even though none is present. No AO or DOF sub-passes are enabled; those functions are handled by MXAO in the next stage.

| Parameter | Value |
|-----------|-------|
| `edge_detect_sharpen` | 1 |
| `tone_map` | 3.0 |
| `ao_enabled` | 0 (handled by MXAO) |
| `fxaa_enabled` | 0 (handled by DAA) |

---

### 5. MXAO

**Role:** Screen-space ambient occlusion — adds contact shadows and geometric depth to surfaces with no baked lighting data.

MXAO runs a two-layer SSAO pass (`MXAO_TWO_LAYER=2`) with smooth normals reconstruction (`MXAO_SMOOTHNORMALS=2`) and high-quality sampling (`MXAO_GLOBAL_SAMPLE_QUALITY_PRESET=5`, `MXAO_HIGH_QUALITY=2`). The fine and coarse AO layers are both at full strength (1.0) with the SSAO amount at 4.0 — a strong occlusion signal appropriate for a game that has no ambient occlusion of its own.

Running MXAO last is important. Occlusion is composited as a darkening pass on the final image — it needs to operate on the fully processed signal to blend correctly. Running it before HDR expansion would cause the occlusion to be mapped incorrectly when luminance is later expanded. Running it before Glamarye's tone shaping would cause the occlusion depth to be altered by the subsequent tone curve. At the end of the pipeline it composites cleanly onto a stable, fully-processed signal.

The sample radius is kept small (0.5 primary, 0.2 secondary) to confine occlusion to contact areas and avoid the large halo artifacts that wide-radius SSAO produces on cel-shaded geometry. Full fade depth range (`FADE_DEPTH_START=1.0`, `FADE_DEPTH_END=1.0`) keeps occlusion active at all scene depths.

| Parameter | Value |
|-----------|-------|
| `MXAO_SSAO_AMOUNT` | 4.0 |
| `MXAO_AMOUNT_COARSE` | 1.0 |
| `MXAO_AMOUNT_FINE` | 1.0 |
| `MXAO_SAMPLE_RADIUS` | 0.5 |
| `MXAO_SAMPLE_RADIUS_SECONDARY` | 0.2 |
| `MXAO_GLOBAL_SAMPLE_QUALITY_PRESET` | 5 |
| `MXAO_SMOOTHNORMALS` | 2 |
| `MXAO_TWO_LAYER` | 2 |

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
