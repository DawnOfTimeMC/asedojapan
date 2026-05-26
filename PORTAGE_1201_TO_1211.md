# Porting Guide: Structure Mod 1.20.1 to 1.21.1 (Multiloader - Fabric + NeoForge)

## Context

This guide documents the complete process for porting a multiloader structure mod (Fabric + Forge/NeoForge)
from Minecraft 1.20.1 to 1.21.1. It is structured as a step-by-step reference with exact file paths
and line numbers to allow systematic application on other projects.

Key architectural change: **Forge (LegacyForge) is replaced by NeoForge** as the second loader.
The `forge/` module is renamed `neoforge/` and the plugin switches from
`net.neoforged.moddev.legacyforge` to `net.neoforged.moddev`.

---

## Phase 1: Build Environment

### 1.1 Version Reference Table

| Parameter | 1.20.1 | 1.21.1 |
|-----------|--------|--------|
| Java | 17 | 21 |
| Minecraft | 1.20.1 | 1.21.1 |
| Parchment MC | 1.20.1 | 1.21.1 |
| Parchment version | 2023.09.03 | 2024.11.17 |
| Fabric API | 0.92.6+1.20.1 | 0.116.4+1.21.1 |
| Fabric Loader | 0.16.14 | 0.16.14 (unchanged) |
| Fabric Loom | 1.11-SNAPSHOT | 1.10.2 |
| Forge version | 47.2.20 | REMOVED |
| NeoForge version | N/A | 21.1.197 |
| NeoForge loader range | N/A | [4,) |
| NeoForm version | N/A | 1.21.1-20240808.144430 |
| moddev plugin | net.neoforged.moddev.legacyforge 2.0.106 | net.neoforged.moddev 2.0.105 |
| Data pack format | 15 | 48 |
| Mixin compatibility | JAVA_17 | JAVA_21 |

---

### 1.2 gradle.properties

**File:** `gradle.properties`

Changes:
- Line 8: `java_version=17` -> `java_version=21`
- Line 11: `minecraft_version=1.20.1` -> `minecraft_version=1.21.1`
- Line 18: `minecraft_version_range=[1.20.1, 1.22)` -> `minecraft_version_range=[1.21.1, 1.22)`
- Line 20: `parchment_minecraft=1.20.1` -> `parchment_minecraft=1.21.1`
- Line 21: `parchment_version=2023.09.03` -> `parchment_version=2024.11.17`
- Line 24: `fabric_version=0.92.6+1.20.1` -> `fabric_version=0.116.4+1.21.1`
- Lines 27-29 (Forge section): REMOVE entirely, REPLACE with NeoForge section:
  ```
  neoforge_version=21.1.197
  neoforge_loader_version_range=[4,)
  neo_form_version=1.21.1-20240808.144430
  ```

---

### 1.3 build.gradle (root)

**File:** `build.gradle`

Changes:
- Line 2: `id 'fabric-loom' version '1.11-SNAPSHOT' apply(false)` -> `id 'fabric-loom' version '1.10.2' apply false`
- Line 3: `id 'net.neoforged.moddev.legacyforge' version '2.0.106' apply(false)` -> `id 'net.neoforged.moddev' version '2.0.105' apply false`
- Line 15: `configure([project(":fabric"), project(":forge")])` -> `configure([project(":fabric"), project(":neoforge")])`

---

### 1.4 settings.gradle

**File:** `settings.gradle`

Changes:
- Lines 16-18 (nested `repositories {}` block): REMOVE the malformed nested block, REPLACE with flat maven entry:
  ```groovy
  maven {
      name = 'NeoForged'
      url = 'https://maven.neoforged.net/releases/'
  }
  ```
- Line 30: `include("forge")` -> `include("neoforge")`
- The old `maven { url = 'https://maven.parchmentmc.org' }` line inside a nested block is not needed
  since ParchmentMC is already handled by `exclusiveContent` in `multiloader-common.gradle`.

---

### 1.5 buildSrc/src/main/groovy/multiloader-common.gradle

**File:** `buildSrc/src/main/groovy/multiloader-common.gradle`

Changes:
- Line 51: REMOVE the extra capability line `capability("$group:${project.name}:$version")`
  (the 1.21.1 reference template does not have this line)
- Lines 86-116 (`processResources` block): Full replacement needed:
  - Change `var expandProps` to `def expandProps` (Groovy `var` is unnecessary here)
  - Replace forge vars with neoforge vars in expandProps:
    - REMOVE: `"forge_version": forge_version`
    - REMOVE: `"forge_loader_version_range": forge_loader_version_range`
    - ADD: `'neoforge_version': neoforge_version`
    - ADD: `'neoforge_loader_version_range': neoforge_loader_version_range`
  - REMOVE the separate `jsonExpandProps` map and its `collectEntries` transform
  - Merge `filesMatching` calls into one, adding `META-INF/neoforge.mods.toml`:
    ```groovy
    filesMatching(['pack.mcmeta', 'fabric.mod.json', 'META-INF/mods.toml', 'META-INF/neoforge.mods.toml', '*.mixins.json']) {
        expand expandProps
    }
    ```

---

### 1.6 common/build.gradle

**File:** `common/build.gradle`

Full replacement:
- Line 3: `id 'net.neoforged.moddev.legacyforge'` -> `id 'net.neoforged.moddev'`
- Lines 6-15 (`legacyForge` block): REPLACE with `neoForge` block:
  ```groovy
  neoForge {
      neoFormVersion = neo_form_version
      def at = file('src/main/resources/META-INF/accesstransformer.cfg')
      if (at.exists()) {
          accessTransformers.add(at.absolutePath)
      }
      parchment {
          minecraftVersion = parchment_minecraft
          mappingsVersion = parchment_version
      }
  }
  ```
- Lines 17-19 (dependencies block): REPLACE `implementation group: 'com.google.code.findbugs' ...` with mixin dependencies:
  ```groovy
  dependencies {
      compileOnly group: 'org.spongepowered', name: 'mixin', version: '0.8.5'
      compileOnly group: 'io.github.llamalad7', name: 'mixinextras-common', version: '0.3.5'
      annotationProcessor group: 'io.github.llamalad7', name: 'mixinextras-common', version: '0.3.5'
  }
  ```

---

### 1.7 fabric/build.gradle

**File:** `fabric/build.gradle`

Changes:
- Line 4: REMOVE `id 'me.modmuss50.mod-publish-plugin'` from plugins block
  (publishing plugin is applied from root build.gradle, no need to declare again)
- Lines 7-26 (repositories block): REMOVE unused repos (Terraformers, isxander.dev), keep only Modrinth
- Line 36: REMOVE `implementation group: 'com.google.code.findbugs', name: 'jsr305', version: '3.0.1'`
  (no longer needed; NeoForge/Fabric both bundle what was needed from this)
- Line 37: REMOVE `implementation project(":common")` (already handled by multiloader-loader.gradle)
- Lines 49-60 (loom runs): Update run directories:
  - `runDir("run")` -> `runDir('runs/client')` for client
  - `runDir("run")` -> `runDir('runs/server')` for server
- Lines 63-74 (publishMods block): REMOVE entirely (handled by root build.gradle)

---

### 1.8 neoforge/build.gradle (NEW FILE)

**File:** `neoforge/build.gradle` (created - replaces `forge/build.gradle`)

```groovy
plugins {
    id 'multiloader-loader'
    id 'net.neoforged.moddev'
}

neoForge {
    version = neoforge_version

    def at = project(':common').file('src/main/resources/META-INF/accesstransformer.cfg')
    if (at.exists()) {
        accessTransformers.add(at.absolutePath)
    }
    parchment {
        minecraftVersion = parchment_minecraft
        mappingsVersion = parchment_version
    }

    runs {
        configureEach {
            systemProperty('neoforge.enabledGameTestNamespaces', mod_id)
            ideName = "NeoForge ${it.name.capitalize()} (${project.path})"
        }
        client {
            client()
        }
        data {
            data()
            programArguments.addAll '--mod', mod_id, '--all',
                '--output', file('src/generated/resources/').absolutePath,
                '--existing', file('src/main/resources/').absolutePath
        }
        server {
            server()
        }
    }

    mods {
        "${mod_id}" {
            sourceSet sourceSets.main
        }
    }
}

sourceSets.main.resources { srcDir 'src/generated/resources' }
```

Note: NO external mod dependencies (dawnoftimebuilder, armoroftheages) since illagerstructures
has no runtime dependencies on other mods.

---

## Phase 2: Resource Files

### 2.1 fabric/src/main/resources/fabric.mod.json

**File:** `fabric/src/main/resources/fabric.mod.json`

Changes:
- Line 3: `"id": "mod_id"` -> `"id": "illagerstructures"` (MUST be hardcoded, template expansion not supported for this field)
- Line 30: `"java": ">=17"` -> `"java": ">=21"`
- Lines 31-32: REMOVE dependency entries for `dawnoftimebuilder` and `armoroftheages`
- Line 35 (suggests block): REMOVE `"another-mod": "*"` template placeholder

---

### 2.2 neoforge/src/main/resources/META-INF/neoforge.mods.toml (NEW FILE)

**File:** `neoforge/src/main/resources/META-INF/neoforge.mods.toml` (created - replaces forge `mods.toml`)

Key differences from 1.20.1 `mods.toml`:
- `loaderVersion` uses `${neoforge_loader_version_range}` (was `${forge_loader_version_range}`)
- `mandatory = true` syntax is REMOVED, replaced by `type = "required"`
- The forge dependency entry (`modId = "forge"`) is replaced by `modId = "neoforge"`
- Mixin configs are declared via `[[mixins]]` blocks at the top level
- `description` no longer needs triple-quote syntax

---

### 2.3 neoforge/src/main/resources/illagerstructures.neoforge.mixins.json (NEW FILE)

**File:** `neoforge/src/main/resources/illagerstructures.neoforge.mixins.json`

Same structure as `illagerstructures.forge.mixins.json` but:
- `"compatibilityLevel": "JAVA_17"` -> `"JAVA_21"`

---

### 2.4 common/src/main/resources/illagerstructures.mixins.json

**File:** `common/src/main/resources/illagerstructures.mixins.json`

- Line 6: `"compatibilityLevel": "JAVA_17"` -> `"JAVA_21"`

---

### 2.5 common/src/main/resources/pack.mcmeta

**File:** `common/src/main/resources/pack.mcmeta`

- Line 3: `"pack_format": 15` -> `"pack_format": 48`
  (1.21.1 data pack format = 48; see https://wiki.vg/Pack_format for the full version table)

---

## Phase 3: Java Source Code

> Documented separately as changes are more complex and mod-specific.

### 3.1 Module restructuring

The `forge/` module directory is replaced by `neoforge/`. Java source files must be moved:
- `forge/src/main/java/` -> `neoforge/src/main/java/`

Entry point class rename convention (recommended):
- `IllagerStructuresForge.java` -> `IllagerStructuresNeoForge.java`
- Class annotation `@Mod` remains the same syntax in NeoForge 1.21.1

### 3.2 API changes (1.20.1 Forge -> 1.21.1 NeoForge)

Key breaking changes to address in Java code:

| Old (1.20.1 Forge) | New (1.21.1 NeoForge) |
|--------------------|-----------------------|
| `net.minecraftforge.*` imports | `net.neoforged.*` imports |
| `FMLJavaModLoadingContext.get()` | Injected via `@Mod` constructor parameter |
| `RegistryObject<T>` | `DeferredHolder<T, T>` or `Supplier<T>` |
| `ForgeRegistries.*` | `BuiltInRegistries.*` or `NeoForgeRegistries.*` |
| `IForgeRegistry` | `Registry` (vanilla) |
| `MinecraftForge.EVENT_BUS` | `NeoForge.EVENT_BUS` |
| `@SubscribeEvent` class-level | Same, but bus registration changed |

### 3.3 Datapack JSON format changes (1.20.1 -> 1.21.1)

Structure JSON files in `common/src/main/resources/data/` may need updates:

- **Structure files** (`worldgen/structure/*.json`): Check if `terrain_adaptation` values are still valid
- **Template pools** (`worldgen/template_pool/*.json`): `element_type` strings unchanged for `minecraft:single_pool_element`
- **Loot tables** (`loot_tables/**/*.json`): Some item IDs changed between 1.20 and 1.21
- **Biome tags**: Tag paths unchanged, but verify biome IDs are still valid in 1.21.1

---

## File Change Summary (Phase 1+2)

| File | Action | Critical change |
|------|--------|-----------------|
| `gradle.properties` | Modify | versions, forge->neoforge vars |
| `build.gradle` | Modify | plugin IDs and versions |
| `settings.gradle` | Modify | NeoForged repo, include forge->neoforge |
| `buildSrc/.../multiloader-common.gradle` | Modify | expandProps, processResources |
| `common/build.gradle` | Modify | legacyForge->neoForge plugin block |
| `fabric/build.gradle` | Modify | cleanup repos, run dirs, remove publish |
| `fabric/.../fabric.mod.json` | Modify | hardcode mod id, java version, remove deps |
| `common/.../illagerstructures.mixins.json` | Modify | JAVA_17->JAVA_21 |
| `common/.../pack.mcmeta` | Modify | pack_format 15->48 |
| `neoforge/build.gradle` | CREATE | NeoForge module definition |
| `neoforge/.../neoforge.mods.toml` | CREATE | NeoForge mod descriptor |
| `neoforge/.../illagerstructures.neoforge.mixins.json` | CREATE | NeoForge mixin config |
| `forge/` directory | ORPHANED | No longer referenced by settings.gradle |
