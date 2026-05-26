# Data Pack Porting Reference: 1.20.1 -> 1.21.1

Complement to PORTAGE_1201_TO_1211.md.
Covers all internal resource file changes: folder paths, loot tables, worldgen, biome tags,
structures, template pools. Reference for future portages.

Reference mod analyzed: `1.21.1_ancientstructures` (namespace: `ancientstructures`)
Target mod: `illagerstructures` (namespace: `illagerstructures`)

---

## 1. CRITICAL PATH RENAMES

These are **breaking changes** introduced between 1.20.x and 1.21 -- files in the old paths are
silently ignored by the game engine.

| Directory (1.20.1) | Directory (1.21.1) | Scope |
|---|---|---|
| `data/<ns>/loot_tables/` | `data/<ns>/loot_table/` | All namespaces |
| `data/<ns>/structures/` | `data/<ns>/structure/` | All namespaces |

### Applied to illagerstructures

```
Before: common/src/main/resources/data/illagerstructures/loot_tables/chest/
After:  common/src/main/resources/data/illagerstructures/loot_table/chest/

Before: common/src/main/resources/data/illagerstructures/structures/
After:  common/src/main/resources/data/illagerstructures/structure/
```

**No JSON content changes are needed in template_pool files.** The `"location"` field uses the
resource location (`namespace:name`) which Minecraft resolves to the new `structure/` path
automatically. Example -- `"location": "illagerstructures:balloon_tower"` resolves to
`data/illagerstructures/structure/balloon_tower.nbt` in 1.21.1.

---

## 2. PACK FORMAT

**File:** `common/src/main/resources/pack.mcmeta`

```json
"pack_format": 15   // 1.20.1
"pack_format": 48   // 1.21.1
```

Already documented in PORTAGE_1201_TO_1211.md section 2.5.

---

## 3. LOOT TABLE FORMAT

### 3.1 Path reference

```
data/<ns>/loot_table/chest/<name>.json    // 1.21.1 (was loot_tables/)
```

### 3.2 JSON format comparison

Both formats below are **valid in 1.21.1**. The `minecraft:` prefix is optional everywhere.
The compact roll format `{ "min": X, "max": Y }` is valid as shorthand for uniform.

**1.20.1 illagerstructures style (still accepted in 1.21.1):**
```json
{
  "type": "minecraft:chest",
  "pools": [{
    "rolls": { "min": 3, "max": 6 },
    "entries": [{
      "type": "minecraft:item",
      "name": "minecraft:apple",
      "weight": 45,
      "functions": [{
        "function": "minecraft:set_count",
        "count": { "min": 1, "max": 3 }
      }]
    }]
  }]
}
```

**1.21.1 ancientstructures style (reference):**
```json
{
  "type": "chest",
  "pools": [{
    "rolls": 6.0,
    "bonus_rolls": 2.0,
    "functions": [{
      "function": "set_count",
      "count": { "type": "uniform", "min": 1, "max": 3 }
    }],
    "entries": [{
      "type": "item",
      "name": "apple",
      "weight": 3
    }]
  }]
}
```

### 3.3 Differences (style, not breaking)

| Aspect | 1.20.1 style | 1.21.1 reference style |
|---|---|---|
| `minecraft:` prefix | Explicit everywhere | Omitted (optional) |
| `rolls` (range) | `{ "min": X, "max": Y }` | `X.0` (fixed) or `{ "type": "uniform", ... }` |
| `bonus_rolls` | Not used | Used for extra randomness |
| `count` type field | Omitted (shorthand) | `"type": "uniform"` explicit |
| Functions position | Inside each entry | At pool level (applies to all entries) |
| `conditions` pool | `minecraft:random_chance` | Same |

### 3.4 Functions validated in 1.21.1

These function IDs are confirmed valid (from illagerstructures files):
- `minecraft:set_count`
- `minecraft:set_damage`
- `minecraft:enchant_randomly`
- `minecraft:random_chance` (condition)

---

## 4. WORLDGEN/STRUCTURE

**Path:** `data/<ns>/worldgen/structure/<name>.json` -- unchanged

### Format (identical in 1.20.1 and 1.21.1)

```json
{
  "type": "minecraft:jigsaw",
  "biomes": "#<ns>:<biome_tag_name>",
  "step": "surface_structures",
  "terrain_adaptation": "none",
  "spawn_overrides": {},
  "start_pool": "<ns>:<pool_name>",
  "size": 1,
  "start_height": { "absolute": -7 },
  "project_start_to_heightmap": "WORLD_SURFACE_WG",
  "max_distance_from_center": 116,
  "use_expansion_hack": false
}
```

### terrain_adaptation values (valid in 1.21.1)

| Value | Effect | Use case |
|---|---|---|
| `"none"` | No terrain modification | Simple/small structures |
| `"beard_thin"` | Thin underground beard | Medium structures (farms, towns) |
| `"beard_box"` | Box-shaped underground beard | Large complex structures |
| `"bury"` | Buries structure underground | Underground dungeons |

### Notes

- `"biomes"` field references a biome tag with `#` prefix -- must point to a valid tag file
- `"start_pool"` value must exactly match the `"name"` field in the corresponding template_pool JSON
- `"size"` = number of jigsaw expansion steps (1 for single-piece structures)
- `"start_height"` `"absolute": -7` is typical for surface structures that sit on terrain

---

## 5. WORLDGEN/TEMPLATE_POOL

**Path:** `data/<ns>/worldgen/template_pool/<name>.json` -- unchanged

### Format (identical in 1.20.1 and 1.21.1)

```json
{
  "name": "<ns>:<pool_name>",
  "fallback": "minecraft:empty",
  "elements": [{
    "element": {
      "element_type": "minecraft:single_pool_element",
      "processors": "minecraft:empty",
      "projection": "rigid",
      "location": "<ns>:<structure_nbt_name>"
    },
    "weight": 1
  }]
}
```

### Field reference

| Field | Value | Description |
|---|---|---|
| `name` | `"<ns>:<pool_name>"` | Must match `start_pool` in worldgen/structure JSON |
| `fallback` | `"minecraft:empty"` | Fallback pool if no elements match |
| `element_type` | `"minecraft:single_pool_element"` | Standard single NBT piece |
| `processors` | `"minecraft:empty"` | No post-processing (use processor_list for aging, etc.) |
| `projection` | `"rigid"` | Fixed placement (use `"terrain_matching"` to snap to terrain) |
| `location` | `"<ns>:<nbt_filename>"` | Resolves to `data/<ns>/structure/<nbt_filename>.nbt` in 1.21.1 |

**Important:** The `location` field value does NOT change when renaming `structures/` to
`structure/`. Minecraft resolves the resource location to the new path automatically.

---

## 6. WORLDGEN/STRUCTURE_SET

**Path:** `data/<ns>/worldgen/structure_set/<name>.json` -- unchanged

### Format (identical in 1.20.1 and 1.21.1)

```json
{
  "structures": [
    { "structure": "<ns>:<structure_name>", "weight": 1 }
  ],
  "placement": {
    "type": "minecraft:random_spread",
    "spacing": 50,
    "separation": 20,
    "spread_type": "triangular",
    "salt": 51179220
  }
}
```

### Placement field reference

| Field | Description | illagerstructures values |
|---|---|---|
| `spacing` | Chunk distance between structure attempts | 50 |
| `separation` | Minimum chunk distance between two instances | 20 |
| `spread_type` | `"triangular"` (clustered) or `"linear"` (uniform) | `"triangular"` |
| `salt` | Unique per-mod seed offset -- MUST be unique across all mods | varies per set |

**Rule:** `separation` must always be less than `spacing`.

---

## 7. BIOME TAGS

### 7.1 Two-level tag system (illagerstructures pattern)

illagerstructures uses two tag levels:

**Level 1 -- Group tags** (referenced by worldgen/structure JSON):
```
data/illagerstructures/tags/worldgen/biome/monks.json
data/illagerstructures/tags/worldgen/biome/grounds.json
data/illagerstructures/tags/worldgen/biome/ice.json
data/illagerstructures/tags/worldgen/biome/pirates.json
data/illagerstructures/tags/worldgen/biome/ruins.json
```
These contain concrete biome IDs:
```json
{ "values": ["minecraft:stony_peaks", "minecraft:jagged_peaks"] }
```

**Level 2 -- Per-structure has_structure tags** (for /locate and exploration maps):
```
data/illagerstructures/tags/worldgen/biome/has_structure/<structure_name>.json
```
These reference the group tags:
```json
{ "values": ["#illagerstructures:monks"] }
```

### 7.2 Single-level tag system (ancientstructures 1.21.1 reference pattern)

ancientstructures uses one level -- no group tags, no has_structure subdirectory:
```
data/ancientstructures/tags/worldgen/biome/german_farm.json
```
The worldgen/structure JSON references this tag directly:
```json
{ "biomes": "#ancientstructures:german_farm" }
```
Content:
```json
{ "replace": false, "values": ["minecraft:plains", "minecraft:sunflower_plains"] }
```

### 7.3 `replace` field

| Value | Behavior |
|---|---|
| `false` | Appends to existing tag values (standard for mods) |
| `true` | Replaces all existing values (dangerous in mod context) |
| omitted | Defaults to `false` |

### 7.4 Biome IDs validated in 1.21.1

These IDs confirmed present in 1.21.1 (used in illagerstructures):
- `minecraft:stony_peaks`, `minecraft:jagged_peaks`, `minecraft:frozen_peaks`
- `minecraft:plains`, `minecraft:sunflower_plains`, `minecraft:meadow`
- `minecraft:deep_frozen_ocean`, `minecraft:frozen_ocean`, `minecraft:cold_ocean`

**Removed in 1.19+** (would break tags if present):
- `minecraft:tall_birch_forest` -> use `minecraft:birch_forest` or `minecraft:old_growth_birch_forest`
- `minecraft:giant_tree_taiga` -> use `minecraft:old_growth_spruce_taiga`
- `minecraft:gravelly_mountains` -> use `minecraft:windswept_gravelly_hills`

---

## 8. DIRECTORY STRUCTURE REFERENCE (1.21.1)

Full expected path layout for a structure mod namespace `<ns>`:

```
data/<ns>/
    loot_table/                         # singular (was loot_tables/ in 1.20.1)
        chest/
            <chest_name>.json
    structure/                          # singular (was structures/ in 1.20.1)
        <structure_name>.nbt
    tags/
        worldgen/
            biome/
                <biome_tag_name>.json   # group tag OR per-structure tag
                has_structure/          # optional -- for /locate support
                    <structure_name>.json
    worldgen/
        structure/
            <structure_name>.json       # jigsaw structure definition
        structure_set/
            <set_name>.json             # placement + grouping
        template_pool/
            <structure_name>.json       # NBT piece selection
```

---

## 9. CROSS-FILE REFERENCE CHAIN

For a structure named `monastery` in namespace `illagerstructures`:

```
worldgen/structure_set/monks.json
    "structure": "illagerstructures:monastery"
        -> worldgen/structure/monastery.json
            "biomes": "#illagerstructures:monks"
                -> tags/worldgen/biome/monks.json (concrete biome IDs)
                -> tags/worldgen/biome/has_structure/monastery.json (for /locate)
            "start_pool": "illagerstructures:monastery"
                -> worldgen/template_pool/monastery.json
                    "location": "illagerstructures:monastery"
                        -> structure/monastery.nbt  (1.21.1 path)
```

Any broken link in this chain silently prevents the structure from spawning.

---

## 10. CHANGES APPLIED TO illagerstructures

| Action | From | To |
|---|---|---|
| Rename dir | `data/illagerstructures/loot_tables/` | `data/illagerstructures/loot_table/` |
| Rename dir | `data/illagerstructures/structures/` | `data/illagerstructures/structure/` |
| No change | `worldgen/structure/*.json` | -- same format -- |
| No change | `worldgen/template_pool/*.json` | -- same format -- |
| No change | `worldgen/structure_set/*.json` | -- same format -- |
| No change | `tags/worldgen/biome/**/*.json` | -- same format -- |
| Already done | `pack.mcmeta` pack_format | 48 |
