# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## Project Overview

This is **SeedsPlease: Lite Redux**, a RimWorld mod (package ID `Evyatar108.SeedsPleaseLiteRedux`) that adds a seed system to farming. It is a C# Harmony mod targeting RimWorld 1.5 and 1.6. The core design is **runtime seed generation**: rather than defining every seed statically in XML, the mod inspects all sowable plants at startup and auto-generates `ThingDef` seeds for any plant that lacks a pre-defined one.

## Build Commands

The project uses a standard .NET SDK-style project file targeting `net48`.

```bash
# Build Debug (default)
dotnet build Source/SeedsPleaseLiteRedux.csproj

# Build Release
dotnet build -c Release Source/SeedsPleaseLiteRedux.csproj

# Build the solution
dotnet build Source/SeedsPleaseLiteRedux.sln
```

Both Debug and Release configurations output the assembly to `1.6/Assemblies/SeedsPleaseLiteRedux.dll`.

### Build Dependencies

- `Krafs.Rimworld.Ref` (1.5.4063) — RimWorld assembly references (will need update for 1.6)
- `Lib.Harmony` (2.2.2) — runtime patching
- `Krafs.Publicizer` — publicizes `Assembly-CSharp` to access `internal` members
- `Microsoft.NETFramework.ReferenceAssemblies.net48`

There are no unit tests or lint tools in this repository.

## Code Architecture

### Runtime Seed Generation Pipeline

The mod's most important architectural characteristic is that it mutates the RimWorld def database at runtime. Understanding this pipeline is essential before making changes:

1. **`SeedsPleaseUtility` (static constructor, `[StaticConstructorOnStartup]`)**
   - Calls `Setup()` during game startup.
   - Scans `DefDatabase<ThingDef>` for all sowable plants.
   - For each plant without an associated seed, it dynamically constructs a new `ThingDef`, assigns a `Seed` `DefModExtension`, gives it a short hash, and injects it into `DefDatabase<ThingDef>`.
   - The **template seed** (`template` field) is the first pre-existing seed `ThingDef` found in the database. New seeds clone its visual/audio/category properties.

2. **`ProcessSeed(ThingDef, Seed, bool)`**
   - Links the plant's `blueprintDef` to the seed `ThingDef`.
   - Calculates auto-market-value based on plant growth time, yield, harvest product value, skill requirements, and user settings.
   - Updates seed descriptions to mirror the parent plant's description and creates bidirectional `descriptionHyperlinks` between seed, plant, and harvest product.

3. **`AddButchery(List<(ThingDef, Seed)>)`**
   - After all seeds are resolved, injects seed drops into the `butcherProducts` of harvested produce.
   - Plants with lower yield receive a multiplier to seed extraction output (configured via `extractionValue` and `harvestYield`).

### Harmony Patches (`Patches/Patches.cs`)

- **`Patch_PlantCollected`** — Prefix on `Plant.PlantCollected`. Drops seeds on harvest based on `Seed.baseChance` / `extraChance` / `seedFactor`.
- **`Patch_IsPlantAvailable`** — Postfix on `Command_SetPlantToGrow.IsPlantAvailable`. Hides plants from grow-zone menus unless the player possesses the corresponding seed.
- **`Patch_WorkGiver_GrowerSow_JobOnCell`** — Postfix on `WorkGiver_GrowerSow.JobOnCell` (and optional VE More Plants / VE Mushrooms variants). Redirects sow jobs to a custom `JobDriver_PlantSowWithSeeds` that requires carrying seeds.
- **`Patch_PossiblePodContentsDefs`** — Postfix on `ThingSetMaker_ResourcePod.PossiblePodContentsDefs`. Reduces seed saturation in random drop-pod events.
- **`Patch_GenerateThings`** — Prefix on `StockGenerator_Tag.GenerateThings`. Excludes "useless" seeds (biome-locked wild plants not present on the player's map) from trader stock when the setting is enabled.

### Custom JobDriver (`Workers/JobDriver_PlantSowWithSeeds.cs`)

Replaces vanilla sowing with a multi-step job that:
1. Reserves a seed stack.
2. Hauls seeds to the sowing site.
3. Consumes **exactly one seed** from the carried stack upon successful sow completion.
4. Attempts to chain to an adjacent valid planting cell (if `blockAdjacentSow` is false) to reduce pawn idle time.

### Mod Settings (`ModSettings_SeedsPleaseLiteRedux.cs` + `OptionsDrawUtility.cs`)

Settings are persisted via RimWorld's `ModSettings` system (`ExposeData`). The settings window includes:
- Sliders: `marketValueModifier`, `extractionModifier`, `seedFactorModifier`
- Checkboxes: `noUselessSeeds`, `clearSnow`, `edibleSeeds`
- Tabbed scroll lists (Seedless / Labels) for per-plant inversion toggles. These operate on `seedlessCache` (ushort hashes) and `seedlessInversions` (string defNames).

When the user toggles a plant's seedless status, `ProcessInversions()` reconciles the inversion set against the static seedless rules and warns that a reload is required.

## File Organization

- **`Source/`** — All C# source.
- **`1.5/` / `1.6/`** — Version-specific XML `Defs` and compiled assembly.
- **`Common/`** — Shared across versions: textures, language translations, and the Harmony patch file for the original `SeedsPleaseLite` (`patch.owlchemist.seedspleaselite.xml`).
- **`Mods/`** — Compatibility patches for ~100 third-party mods. Each subdirectory is named by the target mod's package ID and typically contains `Defs/` (seed definitions) and/or `Patches/` (XML patches). `LoadFolders.xml` conditionally loads these folders via `IfModActive`.
- **`SowGrass/`** — A conditional patch for `Vanilla Factions Expanded - Core` that enables sowing grass when that mod is active.

## Important Implementation Details

- **Namespace**: All C# files use `namespace SeedsPleaseLite`.
- **Seedless Logic**: A plant is considered seedless if it has the `Seedless` `DefModExtension` **or** `plant.harvestedThingDef == null`. Users can invert this per-plant via the settings UI.
- **Dynamic Def Mutation**: Because seeds are injected into `DefDatabase<ThingDef>` at runtime, any code that iterates defs (e.g., traders, resource counters) will see them. After generation, `ResolveReferences()` is called on affected categories and storage buildings to ensure consistency.
- **RimWorld Ref Publicizer**: The project uses `Krafs.Publicizer` to make `Assembly-CSharp` internals accessible. This means internal RimWorld fields/methods can be referenced directly without reflection.
- **No Tests**: There is no test infrastructure. Validation is done by building and running in RimWorld.

## Adding Compatibility for a New Mod

1. Create a folder under `Mods/` named after the new mod's package ID.
2. Add seed `Defs` in `Mods/<packageId>/Defs/` and/or XML patches in `Mods/<packageId>/Patches/`.
3. Register the folder in both `<v1.5>` and `<v1.6>` sections of `LoadFolders.xml` with `IfModActive="<packageId>"`.
4. A template exists at `Mods/__TEMPLATE/`.
