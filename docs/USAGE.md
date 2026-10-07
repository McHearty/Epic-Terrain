# Epic-Terrain

## Configuration and Usage Guide

This document is the user-facing configuration reference for the Epic-Terrain datapack.

It describes the current `versions/1.20.5-1.21.4/` implementation and reconciles the older configuration concepts documented in `docs/config-guide.pdf`.

> **Important:** The current datapack does **not** contain a single centralized configuration file. Configuration is currently distributed across the datapack's JSON resources. This document identifies the exact file and property for every supported configuration point. The historical guide is not a reliable map of the current implementation.

---

## Table of Contents

1. [Compatibility and Scope](#1-compatibility-and-scope)
2. [Quick Start](#2-quick-start)
3. [Configuration Categories](#3-configuration-categories)
4. [Active User Configuration](#4-active-user-configuration)
   - [4.1 Continentalness Sampling Scale](#41-continentalness-sampling-scale)
   - [4.2 Temperature Sampling Scale](#42-temperature-sampling-scale)
   - [4.3 Temperature Local Variation](#43-temperature-local-variation)
   - [4.4 Vegetation Sampling Scale](#44-vegetation-sampling-scale)
   - [4.5 Vegetation Local Variation](#45-vegetation-local-variation)
   - [4.6 Surface Lake Placement](#46-surface-lake-placement)
   - [4.7 Sea Level](#47-sea-level)
5. [Advanced Configuration](#5-advanced-configuration)
   - [5.1 Continentalness Detail](#51-continentalness-detail)
   - [5.2 Erosion Detail](#52-erosion-detail)
   - [5.3 River Noise Scale](#53-river-noise-scale)
   - [5.4 River Surface Profile](#54-river-surface-profile)
   - [5.5 River Elevation](#55-river-elevation)
   - [5.6 Cave Vertical Range](#56-cave-vertical-range)
   - [5.7 Magma Cave Field](#57-magma-cave-field)
   - [5.8 Zenith Lake Biome](#58-zenith-lake-biome)
   - [5.9 Biome Distribution](#59-biome-distribution)
   - [5.10 World Vertical Configuration](#510-world-vertical-configuration)
6. [Implementation Parameters](#6-implementation-parameters)
7. [Parameter Interaction Rules](#7-parameter-interaction-rules)
8. [Historical Configuration](#8-historical-configuration)
9. [File Reference](#9-file-reference)
10. [Troubleshooting](#10-troubleshooting)

---

# 1. Compatibility and Scope

Epic-Terrain is implemented as a Minecraft datapack rather than a Java mod.

The active version tree covered by this guide is:

```text
versions/1.20.5-1.21.4/
```

The active pack metadata currently specifies:

```json
{
  "pack_format": 48
}
```

The guide therefore describes the resources currently present in that version tree rather than assuming that resources from older revisions still exist.

## Important distinction

There are three different kinds of values in the datapack:

1. **User configuration**
   - Intended for deliberate world-generation customization.
   - Changes should normally be made here.

2. **Advanced configuration**
   - Legitimate tuning controls that affect a specific subsystem.
   - Useful for experienced users or major terrain redesigns.
   - Changes can interact with other systems.

3. **Implementation constants**
   - Internal noise, spline, threshold, multiplier, and routing values.
   - These define how Epic-Terrain works.
   - They are documented for completeness but should generally **not** be changed.

The older `docs/config-guide.pdf` contains additional parameters that belonged to an earlier implementation. Those are documented separately in [Historical Configuration](#8-historical-configuration).

---

# 2. Quick Start

## 2.1 Before changing anything

Make a backup of the datapack before changing world-generation resources.

Terrain-generation changes should normally be tested in a **new world**.

Already-generated chunks retain their existing terrain. Changing a density function does not retroactively regenerate those chunks.

## 2.2 Finding a parameter

Every current configuration value in this document gives:

- the file containing it;
- the JSON property or structure to edit;
- the current value;
- the effect of changing it;
- a reasonable operating range;
- a recommended range where one can be given responsibly.

The ranges in this document are **engineering guidance**, not Minecraft-enforced limits. Minecraft will not necessarily reject a value outside the suggested range.

## 2.3 Editing JSON

All configuration is currently performed by editing the corresponding JSON resource.

For example:

```text
versions/1.20.5-1.21.4/data/etn/worldgen/density_function/overworld/continents/1.json
```

contains:

```json
"xz_scale": 1
```

Do not add comments to JSON files.

After editing, validate that:

- the JSON remains syntactically valid;
- referenced resources still exist;
- numeric values have the expected type;
- the world loads without datapack errors.

## 2.4 Recommended workflow

For terrain tuning:

1. Change one parameter or tightly related group.
2. Create a new test world.
3. Inspect several regions rather than a single location.
4. Test both positive and negative terrain cases.
5. Test transitions between affected regions.
6. Only then make another change.

Changing many unrelated density functions simultaneously makes it difficult to determine which change produced an observed result.

---

# 3. Configuration Categories

| Category | Intended user | Change frequency | Risk |
|---|---|---:|---|
| Active User Configuration | Normal configuration | Occasional | Low–Moderate |
| Advanced Configuration | Experienced users | Occasional | Moderate–High |
| Implementation Parameters | Developers | Rare | High |
| Historical Configuration | Reference only | Do not use directly | N/A |

A parameter being documented does **not** mean that it is recommended for routine modification.

In particular, the large spline graphs and mathematical constants in the implementation section are part of the terrain-generation algorithm, not an intended public configuration API.

---

# 4. Active User Configuration

These are the parameters most appropriate for users who want to change the character of Epic-Terrain without redesigning its underlying generation algorithm.

---

## 4.1 Continentalness Sampling Scale

### Location

```text
versions/1.20.5-1.21.4/data/etn/worldgen/density_function/overworld/continents/1.json
```

### Property

```json
"xz_scale"
```

### Current value

```text
1.0
```

### Purpose

Controls the horizontal sampling scale of the Epic-Terrain continentalness noise.

Continentalness is one of the major inputs determining whether an area behaves as ocean, coast, lowland, or continental terrain.

### Effect

Because this is Minecraft's raw `xz_scale` value:

- **Lower values** sample the noise more slowly and produce broader/larger continental features.
- **Higher values** sample the noise more rapidly and produce smaller/more localized continental features.

This direction is important. A parameter named "scale" does not necessarily mean that increasing it produces larger features.

### Reasonable range

```text
0.25 – 4.0
```

### Recommended range

```text
0.5 – 2.0
```

The current value of `1.0` should be treated as the baseline.

### Do not confuse with the historical configuration

The older guide described a continental/ocean scale using a value of `0.2`. That was associated with a different resource and does **not** represent the current value.

The current implementation uses:

```text
continents/1.json → xz_scale = 1
```

---

## 4.2 Temperature Sampling Scale

### Location

```text
versions/1.20.5-1.21.4/data/etn/worldgen/density_function/excessive_biomes/temperature1.json
```

### Property

The large-scale temperature noise:

```json
"xz_scale": 0.2
```

### Current value

```text
0.2
```

### Purpose

Controls the horizontal scale at which the additional Epic-Terrain temperature variation is sampled.

This affects the size of temperature regions used by biome selection.

### Effect

For the raw Minecraft noise scale:

- **Lower values** produce broader temperature regions.
- **Higher values** produce smaller, more rapidly changing temperature regions.

### Reasonable range

```text
0.01 – 4.0
```

### Recommended range

```text
0.05 – 1.0
```

Values substantially above the recommended range can make climate transitions unnecessarily localized.

---

## 4.3 Temperature Local Variation

### Location

```text
versions/1.20.5-1.21.4/data/etn/worldgen/density_function/excessive_biomes/temperature1.json
```

### Property

The multiplier applied to the secondary temperature noise:

```json
"constant": 0.05
```

### Current value

```text
0.05
```

### Purpose

Adds smaller-scale temperature variation to the large-scale temperature field.

### Effect

- **0** removes this additional local variation.
- Higher values increase local temperature variation.
- Excessively high values can make biome transitions noisy.

### Reasonable range

```text
0.0 – 0.25
```

### Recommended range

```text
0.0 – 0.10
```

The current value of `0.05` is a conservative local-variation contribution.

---

## 4.4 Vegetation Sampling Scale

### Location

```text
versions/1.20.5-1.21.4/data/etn/worldgen/density_function/excessive_biomes/vegetation.json
```

### Property

The large-scale vegetation noise:

```json
"xz_scale": 0.2
```

### Current value

```text
0.2
```

### Purpose

Controls the horizontal scale of vegetation variation used by the biome distribution.

### Effect

- **Lower values** produce broader vegetation regions.
- **Higher values** produce smaller, more rapidly changing vegetation regions.

### Reasonable range

```text
0.01 – 4.0
```

### Recommended range

```text
0.05 – 1.0
```

---

## 4.5 Vegetation Local Variation

### Location

```text
versions/1.20.5-1.21.4/data/etn/worldgen/density_function/excessive_biomes/vegetation.json
```

### Property

The secondary vegetation-noise multiplier:

```json
"constant": 0.05
```

### Current value

```text
0.05
```

### Purpose

Adds smaller-scale vegetation variation to the main vegetation field.

### Effect

- `0` removes the additional local variation.
- Increasing the value increases local vegetation variation.
- Excessive values produce noisy biome boundaries.

### Reasonable range

```text
0.0 – 0.25
```

### Recommended range

```text
0.0 – 0.10
```

---

## 4.6 Surface Lake Placement

### Location

```text
versions/1.20.5-1.21.4/data/etn/worldgen/placed_feature/lake_water_surface.json
```

### Current placement structure

The current placement uses a weighted distribution containing:

```text
weight 2 → biased_to_bottom, 5–15
weight 3 → count 1
```

followed by:

```text
in_square
WORLD_SURFACE_WG
biome
```

### Purpose

Controls placement of the Epic-Terrain surface lake feature.

The current implementation does **not** expose lake frequency as a single probability.

The weighted placement structure determines how frequently the feature receives each placement-count behavior.

### Important distinction

Do not treat:

```text
weight 2
weight 3
```

as percentages.

They are relative weights within the placement distribution.

Likewise, the `5–15` biased-to-bottom range is a placement parameter rather than a lake depth.

### Recommended modification strategy

If increasing lake density:

- increase the relative contribution of the higher-count branch;
- avoid making the placement distribution overwhelmingly concentrated in one branch.

If reducing lake density:

- reduce the relative contribution of placement-count rules;
- do not remove required placement infrastructure unless the entire feature is intentionally being disabled.

### Reasonable operating guidance

Because this is a structured placement distribution rather than a single scalar, there is no meaningful universal minimum/maximum frequency.

For the current structure, modest changes are preferable:

```text
weight: approximately 1–10
count: approximately 0–4
vertical bias range: approximately 0–32
```

These are tuning guidance, not tested hard limits.

### Lake contents

The actual lake feature is defined in:

```text
versions/1.20.5-1.21.4/data/etn/worldgen/configured_feature/lake_water.json
```

It uses water and a weighted mud/grass-block barrier.

Changing those values changes the **composition or generation behavior** of the lake rather than its placement frequency.

---

## 4.7 Sea Level

### Location

```text
versions/1.20.5-1.21.4/data/minecraft/worldgen/noise_settings/overworld.json
```

### Property

```json
"sea_level": 63
```

### Current value

```text
63
```

### Purpose

Defines the nominal overworld sea level used by the noise settings.

It affects the relationship between terrain elevation, oceans, rivers, and surface generation.

### Reasonable range

```text
0 – 128
```

### Recommended range

```text
48 – 96
```

The current value of `63` should normally be retained unless deliberately changing the global vertical character of the world.

### Important

Sea level interacts with the river surface rules described in [River Elevation](#55-river-elevation).

Changing sea level without considering the river surface rules can produce rivers that are incorrectly positioned relative to oceans.

---

# 5. Advanced Configuration

These parameters are legitimate tuning points but should only be changed when the active user parameters are insufficient.

---

## 5.1 Continentalness Detail

### Location

```text
versions/1.20.5-1.21.4/data/etn/worldgen/density_function/excessive_biomes/continents1.json
```

### Current contribution

The secondary continentalness noise is multiplied by:

```text
0.05
```

before being added to the primary continentalness field.

### Purpose

Adds smaller-scale continentalness variation.

### Effect

- Lower values produce smoother continentalness.
- Higher values increase local continentalness variation.

### Reasonable range

```text
0.0 – 0.25
```

### Recommended range

```text
0.02 – 0.10
```

The surrounding `range_choice` also determines where this contribution is applied.

Do not modify the range-choice boundaries unless deliberately changing the continentalness algorithm.

---

## 5.2 Erosion Detail

### Location

```text
versions/1.20.5-1.21.4/data/etn/worldgen/density_function/excessive_biomes/erosion1.json
```

### Current values

The active branch adds the secondary erosion noise with:

```text
multiplier = 0.35
```

The alternate branch uses:

```text
0.2
```

### Purpose

Controls the contribution of additional erosion detail to the terrain climate/landform system.

### Effect

Increasing these contributions increases the influence of fine erosion variation.

Decreasing them produces smoother erosion fields.

### Reasonable range

```text
0.0 – 1.0
```

### Recommended range

```text
0.1 – 0.5
```

The two values are conditional branches. They should normally be tuned together rather than independently.

The surrounding range choice is:

```text
etn:overworld/continents/c1
```

with a range approximately covering:

```text
-1.0 – 0.6
```

Changing that condition changes **where** the detail contribution occurs, while changing the multiplier changes **how strongly** it contributes.

---

## 5.3 River Noise Scale

Current river noise resources are:

```text
versions/1.20.5-1.21.4/data/etn/worldgen/noise/r/1.json
versions/1.20.5-1.21.4/data/etn/worldgen/noise/r/2.json
versions/1.20.5-1.21.4/data/etn/worldgen/noise/r/3.json
```

### Current values

| Noise | `firstOctave` | Role |
|---|---:|---|
| `r/1` | `-13` | Primary river surface signal |
| `r/2` | `-12` | Secondary river signal |
| `r/3` | `-14` | Additional river detail |

The current surface rules primarily use:

```text
etn:r/1
```

### Effect of `firstOctave`

More negative values represent larger-scale noise.

Therefore:

- more negative → broader/larger river-scale variation;
- less negative → smaller/more frequent variation.

### Reasonable range

For individual river noise definitions:

```text
-16 – -8
```

### Recommended range

```text
-15 – -11
```

These values should not be changed independently without checking the resulting surface-rule behavior.

---

## 5.4 River Surface Profile

### Location

```text
versions/1.20.5-1.21.4/data/minecraft/worldgen/noise_settings/overworld.json
```

The surface rules contain several `noise_threshold` conditions using:

```text
etn:r/1
```

with thresholds around:

```text
±0.027
±0.028
±0.029
±0.030
±0.038
```

Additional `h/1`, `h/2`, and `h/3` noise conditions participate in the river surface selection.

### Purpose

These rules determine when the river surface treatment is applied.

They are the current implementation replacing the simpler river-width parameters from the historical configuration.

### General effect

Broadly:

- widening the threshold bands causes more terrain to satisfy river-related conditions;
- narrowing them makes the river surface treatment more selective.

However, the rules are layered, so a single threshold cannot be treated as a standalone river-width slider.

### Recommendation

Do not tune individual thresholds in isolation.

If changing river appearance, treat the river surface-rule block as one configuration system.

### Reasonable range

There is no reliable single min/max for these thresholds.

The current values are intentionally close to zero. Large changes should be considered an algorithm change rather than ordinary configuration.

---

## 5.5 River Elevation

### Location

```text
versions/1.20.5-1.21.4/data/minecraft/worldgen/noise_settings/overworld.json
```

The river surface rules contain vertical conditions around:

```text
Y = 63
Y = 58
```

### Purpose

Controls the vertical relationship between river surface rules and terrain.

### Interaction with sea level

The current sea level is:

```text
63
```

The principal river rules therefore operate close to sea level.

### Recommendation

Keep river elevation closely related to sea level.

A practical tuning range is approximately:

```text
sea_level ± 16
```

unless deliberately redesigning the river system.

Changing these values independently from sea level can cause rivers to become disconnected from oceans or appear at inappropriate elevations.

---

## 5.6 Cave Vertical Range

### Location

```text
versions/1.20.5-1.21.4/data/etn/worldgen/density_function/overworld/caves/noodle1.json
```

### Current vertical range

The noodle cave density function uses approximately:

```text
Y = -60
through
Y = 321
```

### Purpose

Defines the vertical region in which the Epic-Terrain noodle cave contribution is active.

### Reasonable range

Bottom:

```text
-64 – 64
```

Top:

```text
64 – 384
```

The bottom must remain below the top.

### Recommendation

Do not change the vertical range unless there is a specific cave-generation goal.

The current range is deliberately broad.

### Other cave constants

The same resource contains:

- noodle thickness adjustment;
- noodle multiplier;
- ridge sampling scale;
- ridge multiplier.

These are implementation parameters and are covered in [Implementation Parameters](#6-implementation-parameters).

---

## 5.7 Magma Cave Field

### Location

```text
versions/1.20.5-1.21.4/data/etn/worldgen/density_function/overworld/caves/magma_caves.json
```

### Current field scale

The Epic-Terrain cave noise is sampled with approximately:

```text
xz_scale = 0.6
y_scale = 1
```

### Purpose

Controls the spatial character of the magma-cave field.

### Effect

For the raw `xz_scale`:

- lower values → broader structures;
- higher values → smaller structures.

### Reasonable range

```text
0.25 – 1.5
```

### Recommended range

```text
0.4 – 1.0
```

The vertical gradients and clamping in this resource are part of the implementation and should generally be left unchanged.

---

## 5.8 Zenith Lake Biome

### Biome

```text
etn:zenith_lake
```

### Main definition

```text
versions/1.20.5-1.21.4/data/etn/worldgen/biome/zenith_lake.json
```

### Distribution

The biome is referenced by the overworld multi-noise biome source:

```text
versions/1.20.5-1.21.4/data/minecraft/dimension/overworld.json
```

The current mapping places `etn:zenith_lake` within a specific erosion range centered around approximately:

```text
1.15 – 1.5
```

### Purpose

This is a specialized biome-selection band rather than a normal terrain height parameter.

### Recommendation

Do not treat its erosion boundaries as a normal user slider.

Changing the boundaries changes biome classification and can substantially alter where the Zenith Lake biome appears.

If tuning this system, inspect the entire multi-noise biome mapping rather than changing only one boundary.

---

## 5.9 Biome Distribution

### Location

```text
versions/1.20.5-1.21.4/data/minecraft/dimension/overworld.json
```

### Purpose

Defines the overworld multi-noise biome source and its mappings between:

- temperature;
- humidity/vegetation;
- continentalness;
- erosion;
- weirdness/ridges;
- depth.

### Important

This is not a conventional configuration table.

The values define Minecraft's multi-dimensional biome-selection space.

Changing one boundary can have effects across many biomes.

### Recommendation

Do not expose individual biome boundaries as ordinary user parameters.

For normal customization, change:

- temperature sampling scale;
- temperature local variation;
- vegetation sampling scale;
- vegetation local variation.

Only modify the biome distribution directly when deliberately redesigning the biome layout.

---

## 5.10 World Vertical Configuration

The world-generation vertical architecture is defined in multiple places.

### Noise settings

```text
versions/1.20.5-1.21.4/data/minecraft/worldgen/noise_settings/overworld.json
```

Current values include:

```text
min_y = -64
height = 512
```

### Dimension type

```text
versions/1.20.5-1.21.4/data/minecraft/dimension_type/overworld.json
```

Current vertical settings include:

```text
min_y = -64
height = 576
logical_height = 384
```

### Important

These values are **not independent terrain sliders**.

Changing the world vertical architecture requires keeping the dimension and noise settings consistent.

### Recommendation

Do not change these values for ordinary terrain customization.

Changing them should be treated as a deliberate world-generation architecture change requiring comprehensive testing.

---

# 6. Implementation Parameters

This section documents important values that are present in the active implementation but should generally **not** be modified by ordinary users.

The purpose of documenting them is completeness and maintainability.

Arbitrary constants are not configuration merely because they are numeric.

## 6.1 Continentalness Noise

Current resources include:

```text
data/etn/worldgen/noise/continentalness.json
data/etn/worldgen/noise/continentalness1.json
```

The primary continentalness noise uses approximately:

```text
firstOctave = -13
amplitudes = [1, 1, 2, 2, 2, 1]
```

The secondary continentalness noise uses:

```text
firstOctave = -3
amplitudes = [1, 1, 2, 2, 2, 1]
```

These values define the underlying noise spectrum.

**Recommendation:** Do not change them unless modifying the continentalness algorithm itself.

---

## 6.2 Erosion Noise

Erosion resources include:

```text
data/etn/worldgen/noise/e/a/1.json
data/etn/worldgen/noise/e/a/2.json
data/etn/worldgen/noise/e/b/1.json
data/etn/worldgen/noise/e/b/2.json
data/etn/worldgen/noise/e/c/1.json
data/etn/worldgen/noise/e/c/2.json
data/etn/worldgen/noise/e/d/1.json
data/etn/worldgen/noise/e/e/1.json
data/etn/worldgen/noise/e/m.json
```

Their octave values range approximately from:

```text
-11 through -6
```

with amplitude arrays defining the noise spectrum.

**Recommendation:** Do not change individual erosion noise resources when a change to erosion detail strength can be achieved through `erosion1.json`.

---

## 6.3 Continentalness Splines

Important resources include:

```text
data/etn/worldgen/density_function/overworld/continents/c.json
data/etn/worldgen/density_function/overworld/continents/c1.json
data/etn/worldgen/density_function/overworld/continents/c2.json
```

These contain nested spline structures controlling the conversion from continentalness/erosion/ridge inputs into terrain-related density values.

They contain multiple knots, nested splines, and terrain-specific values.

### Recommendation

Do not treat individual spline knots as configuration parameters.

The spline graph is the implementation of the terrain-shaping algorithm.

---

## 6.4 Depth Gradient

### Location

```text
data/etn/worldgen/density_function/overworld/depth.json
```

The current vertical gradient spans approximately:

```text
from_y    = -64
to_y      = 512
from_value = 1.5
to_value   = -1.5
```

The resulting gradient is combined with:

```text
etn:overworld/continents/c2
```

### Recommendation

Leave these values unchanged unless redesigning vertical terrain shaping.

---

## 6.5 Ridge Functions

Current ridge resources include:

```text
data/etn/worldgen/density_function/overworld/ridge/1.json
data/etn/worldgen/density_function/overworld/ridge/2.json
data/etn/worldgen/density_function/overworld/ridge/r.json
data/etn/worldgen/density_function/overworld/ridge/r1.json
```

They contain:

- ridge-noise combinations;
- splines;
- narrow range-choice transitions;
- shifted-noise sampling.

Some transitions occur over very narrow intervals, such as approximately:

```text
0.100 – 0.105
```

### Recommendation

Do not modify these knots or transition widths as normal configuration.

They are part of the shape-preserving behavior of the ridge system.

---

## 6.6 Climate Noise

Minecraft climate noise resources include:

```text
data/minecraft/worldgen/noise/temperature.json
data/minecraft/worldgen/noise/temperature1.json
data/minecraft/worldgen/noise/vegetation.json
data/minecraft/worldgen/noise/vegetation1.json
```

Current octave structures include:

```text
temperature       firstOctave = -10
temperature1      firstOctave = -3
vegetation        firstOctave = -10
vegetation1       firstOctave = -3
```

The Epic-Terrain configuration layer already provides more appropriate places to alter climate behavior:

```text
etn:excessive_biomes/temperature1
etn:excessive_biomes/vegetation
```

**Recommendation:** Do not modify the underlying Minecraft noise definitions unless intentionally changing the entire climate noise architecture.

---

## 6.7 River Noise Definitions

Current resources:

```text
data/etn/worldgen/noise/r/1.json
data/etn/worldgen/noise/r/2.json
data/etn/worldgen/noise/r/3.json
```

Current octave values:

```text
r/1 = -13
r/2 = -12
r/3 = -14
```

All currently use similar multi-octave amplitude structures.

These are the underlying river fields.

**Recommendation:** If river tuning is necessary, start with the river surface configuration before changing the noise definitions themselves.

---

## 6.8 Noodle Cave Constants

### Location

```text
data/etn/worldgen/density_function/overworld/caves/noodle1.json
```

The implementation includes:

- vertical range approximately `-60` to `321`;
- noodle thickness offset approximately `-0.175`;
- noodle multiplier approximately `-0.025`;
- ridge sampling scale approximately `2.6666666666666665`;
- ridge multiplier approximately `1.5`.

These values work together.

**Recommendation:** Do not expose them as independent public configuration parameters.

---

## 6.9 Vanilla Cave Density Functions

The active datapack references Minecraft cave density functions including:

```text
minecraft/worldgen/density_function/overworld/caves/entrances.json
minecraft/worldgen/density_function/overworld/caves/entrances1.json
minecraft/worldgen/density_function/overworld/caves/entrances12.json
minecraft/worldgen/density_function/overworld/caves/noodle.json
minecraft/worldgen/density_function/overworld/caves/pillars.json
```

These are part of the Minecraft cave-generation pipeline.

**Recommendation:** Do not modify them unless the goal is to replace or substantially redesign vanilla cave generation.

---

## 6.10 Magma Cave Gradients

### Location

```text
data/etn/worldgen/density_function/overworld/caves/magma_caves.json
```

The magma cave function contains vertical gradients around:

```text
-27 → 48
128 → -62
```

and clamps the resulting field approximately to:

```text
-1 → 1
```

These values determine the vertical envelope of the magma cave field.

**Recommendation:** Leave unchanged for normal configuration.

---

## 6.11 Surface Rules

### Location

```text
data/minecraft/worldgen/noise_settings/overworld.json
```

The surface rules contain multiple nested conditions involving:

- river noise;
- height;
- secondary river noise;
- surface state selection.

These rules determine whether terrain receives river-specific surface treatment.

**Recommendation:** Treat the complete surface-rule block as implementation logic.

---

## 6.12 `minecraft:offset` Noise

The current resource:

```text
data/minecraft/worldgen/noise/offset.json
```

is a Minecraft **noise definition**.

It currently uses:

```text
firstOctave = -4
amplitudes = [1, 1, 1, 0]
```

This is **not** the same thing as the historical:

```text
minecraft/worldgen/density_function/overworld/offset.json
```

described in the old Epic-Terrain guide.

The old density function no longer exists in the current implementation.

---

## 6.13 Noise Settings Infrastructure

The active overworld noise settings contain standard Minecraft infrastructure including:

- noise router definitions;
- aquifer configuration;
- ore-vein configuration;
- surface rules;
- world vertical settings.

These values are necessary for the world-generation pipeline but are not ordinary Epic-Terrain user configuration.

**Recommendation:** Leave them unchanged unless performing a deliberate engine-level modification.

---

# 7. Parameter Interaction Rules

Epic-Terrain parameters should not always be interpreted independently.

## 7.1 Continentalness

These components work together:

```text
continents/1.json
continents/c.json
continents/c1.json
continents/c2.json
excessive_biomes/continents1.json
```

Changing the primary continentalness sampling scale changes the input to the spline system.

Do not expect the spline output to remain visually identical when continentalness scale is changed significantly.

---

## 7.2 Climate

Temperature and vegetation each consist of:

```text
large-scale field
+
small-scale variation
```

Therefore:

- changing sampling scale changes the size of climate regions;
- changing the local multiplier changes variation inside those regions.

A large sampling scale plus a large local multiplier can produce a visually noisy climate even if either value is reasonable independently.

---

## 7.3 Sea Level and Rivers

The current sea level is:

```text
63
```

while river surface rules contain conditions around:

```text
63
58
```

These values are coupled.

If sea level is moved substantially, river surface rules should be reviewed at the same time.

---

## 7.4 River Noise and River Thresholds

River appearance depends on both:

```text
etn:r/1
```

and the threshold rules consuming it.

Changing only the noise scale changes the spatial field.

Changing only the threshold changes which parts of that field are classified as river terrain.

Changing both simultaneously makes the result significantly harder to predict.

---

## 7.5 Lakes and Biomes

Lake placement is controlled by:

```text
etn/worldgen/placed_feature/lake_water_surface.json
```

but the placement is also subject to biome filtering.

Therefore increasing lake placement frequency does not necessarily increase lakes equally across every biome.

---

## 7.6 Cave Vertical Range and World Height

The cave ranges operate within the world-generation vertical envelope.

Current world generation uses:

```text
min_y = -64
```

Changing cave boundaries beyond the world's usable vertical region will not produce useful additional terrain.

---

## 7.7 World Vertical Architecture

These values must be considered together:

```text
data/minecraft/dimension_type/overworld.json
data/minecraft/worldgen/noise_settings/overworld.json
```

Do not change `min_y`, `height`, or `logical_height` independently.

These are structural Minecraft settings, not ordinary Epic-Terrain tuning parameters.

---

# 8. Historical Configuration

The historical configuration guide in:

```text
docs/config-guide.pdf
```

describes an earlier Epic-Terrain implementation.

Several of its parameters no longer correspond to individual resources in the active version.

This section preserves that information so that old documentation and configuration discussions can be reconciled with the current datapack.

---

## 8.1 Historical Continental/Ocean Scale

### Historical location

```text
data/etn/worldgen/density_function/overworld/c/1.json
```

### Historical value

```text
xz_scale = 0.2
```

### Historical purpose

Controlled the scale of continental/ocean terrain.

### Current status

The old resource is no longer present.

The current implementation instead uses:

```text
data/etn/worldgen/density_function/overworld/continents/1.json
```

with:

```text
xz_scale = 1
```

This is the current control to use for broad continentalness sampling.

---

## 8.2 Historical Ocean/Continent Ratio

### Historical location

```text
data/etn/worldgen/density_function/overworld/c/1a.json
```

### Historical value

Approximately:

```text
1.35
```

### Historical purpose

Adjusted the relative balance between ocean and continental terrain.

The historical documentation described approximately `1.35` as producing a broadly balanced land/sea distribution and indicated that increasing the value increased ocean influence.

### Current status

The old resource is absent.

There is no current one-to-one replacement.

The equivalent behavior is now distributed through:

```text
continents/c.json
continents/c1.json
continents/c2.json
```

and their associated spline structures.

### Recommendation

Do not attempt to recreate this parameter by changing a single current spline knot.

---

## 8.3 Historical Large River Width

### Historical location

```text
data/etn/worldgen/density_function/overworld/r/1/1.json
```

### Historical value

Approximately:

```text
0.008
```

The historical guide described this as a large-river width control and noted an approximate transition width of:

```text
0.008 × 6 = 0.048
```

### Current status

The resource no longer exists.

River width is now represented by the combination of:

```text
etn:r/1
```

and the river surface rules in:

```text
minecraft/worldgen/noise_settings/overworld.json
```

### Recommendation

Do not create the historical file to restore the old parameter.

---

## 8.4 Historical Small River Width

### Historical location

```text
data/etn/worldgen/density_function/overworld/r/2/1.json
```

### Historical value

Approximately:

```text
0.008
```

### Current status

The resource no longer exists.

Small-river behavior is now part of the current river-noise and surface-rule system.

---

## 8.5 Historical Terrain Offset Parameters

### Historical location

```text
data/minecraft/worldgen/density_function/overworld/offset.json
```

The historical guide described independent terrain concepts including:

- deep ocean depth;
- shallow ocean depth;
- trench depth;
- maximum ocean depth;
- coast height;
- minimum river depth;
- deepest river value;
- river boundary;
- ground height;
- mountain height.

### Current status

The historical density function no longer exists.

The current terrain pipeline distributes these effects across:

```text
etn:overworld/continents/*
etn:overworld/depth
etn:overworld/ridge/*
minecraft:overworld/sloped_cheese
minecraft:overworld/depth
```

and the active noise router.

### Important

There is no current independent:

```text
deepOceanDepth
shallowOceanDepth
trenchDepth
oceanMax
coastHeight
riverMin
riverDeepest
riverBoundary
groundHeight
mountainHeight
```

parameter.

These should therefore be considered **historical concepts**, not current configuration keys.

---

## 8.6 Historical Cave Top Boundary

### Historical location

```text
data/etn/worldgen/density_function/overworld/caves/2_top.json
```

### Historical values

The historical function used a vertical transition involving approximately:

```text
from_y = 40
to_y   = -100
```

### Current status

The file is absent.

The current cave implementation instead contains vertical logic in:

```text
data/etn/worldgen/density_function/overworld/caves/noodle1.json
```

with an active range approximately:

```text
-60 → 321
```

---

## 8.7 Historical Cave Bottom Boundary

### Historical location

```text
data/etn/worldgen/density_function/overworld/caves/3_down.json
```

### Historical value

The historical function contained a boundary around:

```text
to_y = -96
```

### Current status

The resource no longer exists.

The current cave system should be configured through its active cave density functions rather than restoring this historical resource.

---

## 8.8 Historical Temperature Scale

### Historical location

```text
data/minecraft/worldgen/density_function/overworld/temperature.json
```

### Historical value

Approximately:

```text
xz_scale = 0.2
```

### Current status

The active Epic-Terrain temperature customization is represented in:

```text
data/etn/worldgen/density_function/excessive_biomes/temperature1.json
```

which still contains:

```text
xz_scale = 0.2
```

Therefore the underlying concept remains active, although its implementation location has changed.

---

## 8.9 Historical Temperature Fine Variation

### Historical location

```text
data/etn/worldgen/density_function/overworld/temperature_1.json
```

### Historical value

Approximately:

```text
0.05
```

### Current status

The corresponding active value remains represented in:

```text
data/etn/worldgen/density_function/excessive_biomes/temperature1.json
```

with a secondary contribution of:

```text
0.05
```

This is therefore still an active concept.

---

## 8.10 Historical Vegetation Scale

### Historical location

```text
data/minecraft/worldgen/density_function/overworld/vegetation.json
```

### Historical value

Approximately:

```text
xz_scale = 0.2
```

### Current status

The active Epic-Terrain implementation is:

```text
data/etn/worldgen/density_function/excessive_biomes/vegetation.json
```

with the corresponding large-scale value:

```text
xz_scale = 0.2
```

---

## 8.11 Historical Vegetation Fine Variation

### Historical location

```text
data/minecraft/worldgen/density_function/overworld/vegetation_1.json
```

### Historical value

Approximately:

```text
0.05
```

### Current status

The active implementation retains the same conceptual secondary contribution in:

```text
data/etn/worldgen/density_function/excessive_biomes/vegetation.json
```

---

## 8.12 Historical River Lake Probability

### Historical location

```text
data/etn/worldgen/placed_feature/river/lake.json
```

### Purpose

Controlled river wetland/lake placement probability.

### Current status

The resource does not exist in the active implementation.

There is no current independent river-lake probability parameter.

The current lake implementation is centered around:

```text
data/etn/worldgen/configured_feature/lake_water.json
data/etn/worldgen/placed_feature/lake_water_surface.json
```

---

## 8.13 Historical Swamp Lake Probability

### Historical location

```text
data/etn/worldgen/placed_feature/swamp/lake.json
```

### Purpose

Controlled swamp lake placement probability.

### Current status

The resource does not exist in the active implementation.

The current implementation uses the general Epic-Terrain lake feature and biome placement system instead.

---

# 9. File Reference

The following map identifies the principal active resources.

## Terrain

```text
data/etn/worldgen/density_function/overworld/continents/1.json
```

Primary continentalness sampling.

```text
data/etn/worldgen/density_function/overworld/continents/c.json
data/etn/worldgen/density_function/overworld/continents/c1.json
data/etn/worldgen/density_function/overworld/continents/c2.json
```

Continentalness terrain-shaping splines.

```text
data/etn/worldgen/density_function/overworld/depth.json
```

Vertical depth shaping.

```text
data/etn/worldgen/density_function/overworld/ridge/
```

Ridge terrain shaping.

---

## Climate and Biomes

```text
data/etn/worldgen/density_function/excessive_biomes/temperature1.json
```

Temperature customization.

```text
data/etn/worldgen/density_function/excessive_biomes/vegetation.json
```

Vegetation customization.

```text
data/etn/worldgen/density_function/excessive_biomes/continents1.json
```

Continentalness detail.

```text
data/etn/worldgen/density_function/excessive_biomes/erosion1.json
```

Erosion detail.

```text
data/etn/worldgen/biome/zenith_lake.json
```

Zenith Lake biome definition.

```text
data/minecraft/dimension/overworld.json
```

Overworld biome-source and multi-noise mapping.

---

## Rivers

```text
data/etn/worldgen/noise/r/1.json
data/etn/worldgen/noise/r/2.json
data/etn/worldgen/noise/r/3.json
```

River noise fields.

```text
data/minecraft/worldgen/noise_settings/overworld.json
```

River surface rules and elevation conditions.

---

## Lakes

```text
data/etn/worldgen/configured_feature/lake_water.json
```

Lake generation contents.

```text
data/etn/worldgen/placed_feature/lake_water_surface.json
```

Lake placement.

---

## Caves

```text
data/etn/worldgen/density_function/overworld/caves/noodle1.json
```

Epic-Terrain noodle cave contribution.

```text
data/etn/worldgen/density_function/overworld/caves/magma_caves.json
```

Magma cave field.

```text
data/minecraft/worldgen/density_function/overworld/caves/
```

Vanilla cave density functions used by the generation pipeline.

---

## World Generation Infrastructure

```text
data/minecraft/worldgen/noise_settings/overworld.json
```

Noise router, sea level, surface rules, aquifers, ore veins, and world-generation settings.

```text
data/minecraft/dimension_type/overworld.json
```

Dimension vertical architecture.

```text
data/minecraft/dimension/overworld.json
```

Overworld dimension and biome-source configuration.

---

# 10. Troubleshooting

## The world fails to load after editing a JSON file

Check:

1. JSON syntax.
2. Missing commas.
3. Incorrect numeric/string types.
4. Misspelled resource identifiers.
5. References to resources that do not exist.

If the error identifies a resource path, restore that resource from the original datapack and reapply the change more carefully.

---

## My terrain did not change

Terrain-generation changes do not rewrite already-generated chunks.

Create a new world or travel into previously ungenerated terrain.

---

## My continents became much smaller/larger than expected

Check:

```text
data/etn/worldgen/density_function/overworld/continents/1.json
```

Remember that the raw `xz_scale` direction is:

```text
lower xz_scale → larger features
higher xz_scale → smaller features
```

---

## Climate became noisy

Check both the sampling scale and local variation.

Temperature:

```text
data/etn/worldgen/density_function/excessive_biomes/temperature1.json
```

Vegetation:

```text
data/etn/worldgen/density_function/excessive_biomes/vegetation.json
```

First return the local variation multiplier toward:

```text
0.05
```

before changing the underlying Minecraft noise definitions.

---

## Rivers look wrong after changing sea level

Review the river conditions in:

```text
data/minecraft/worldgen/noise_settings/overworld.json
```

The current river rules contain vertical conditions near:

```text
63
58
```

These were designed around the current sea level of:

```text
63
```

---

## Changing an old parameter has no effect

Verify that the file actually exists in the current version tree.

Several resources from the historical guide no longer exist, including:

```text
data/minecraft/worldgen/density_function/overworld/offset.json
data/etn/worldgen/density_function/overworld/c/1.json
data/etn/worldgen/density_function/overworld/c/1a.json
data/etn/worldgen/density_function/overworld/r/1/1.json
data/etn/worldgen/density_function/overworld/r/2/1.json
data/etn/worldgen/placed_feature/river/lake.json
data/etn/worldgen/placed_feature/swamp/lake.json
```

Do not recreate those files merely because they appear in historical documentation.

Use the current implementation described in this guide.

---

# Configuration Summary

For normal customization, start with these parameters:

| Parameter | Current value | Location | Recommended use |
|---|---:|---|---|
| Continentalness `xz_scale` | `1.0` | `etn/.../continents/1.json` | Change continental feature size |
| Temperature `xz_scale` | `0.2` | `etn/.../temperature1.json` | Change climate-region size |
| Temperature local variation | `0.05` | `etn/.../temperature1.json` | Add/remove local climate variation |
| Vegetation `xz_scale` | `0.2` | `etn/.../vegetation.json` | Change vegetation-region size |
| Vegetation local variation | `0.05` | `etn/.../vegetation.json` | Add/remove local vegetation variation |
| Surface lake placement | weighted placement | `etn/.../lake_water_surface.json` | Change lake density |
| Sea level | `63` | `minecraft/.../noise_settings/overworld.json` | Change global water level |

For more specialized tuning:

| System | Primary advanced location |
|---|---|
| Continentalness detail | `etn/.../excessive_biomes/continents1.json` |
| Erosion detail | `etn/.../excessive_biomes/erosion1.json` |
| River scale | `etn/.../noise/r/*.json` |
| River surface | `minecraft/.../noise_settings/overworld.json` |
| Cave vertical range | `etn/.../caves/noodle1.json` |
| Magma caves | `etn/.../caves/magma_caves.json` |
| Zenith Lake distribution | `minecraft/.../dimension/overworld.json` |
| World vertical architecture | `minecraft/.../dimension_type/overworld.json` + `noise_settings/overworld.json` |

The general rule is:

> **Change the highest-level parameter that expresses the behavior you want. Do not modify lower-level implementation constants when an appropriate higher-level control already exists.**

The current datapack does not yet provide a single centralized configuration file. Until such a configuration layer exists, the file locations in this document are the authoritative practical configuration map for the active implementation.
