[clay ui in zig seems great](https://github.com/johan0A/clay-zig-bindings)

## expectations of a decent asset system
1. support both binary and text forms where applicable
2. binary form should be packable
3. minimum conversion overhead from binary to in-memory
4. able to load and unload bundles of assets
5. git merge
6. hot reload



the engine asset system will be based on modules,
wherein each module may contribute the following types of asset:
* raw image: png, bmp, jpg
* raw audio: wav? ogg??
* raw script: .bb, .frag, .vert, etc
* structured data: a mesh, a level structure, a shader program, a material, game save, etc

each asset a module contributes must provide:
* a unique name for the asset
* a binding to the data, ie a cwd path to a source file, the binary data itself, etc

modules are loaded in a specified order, with unique names overriding

modules can form interdependencies within this order for e.g. script code by explicitly specifying their dependencies


assets should be capable of giving their asset-wise dependencies, for example;
if we are going to load a new level, we should be able to get its asset dependencies, and load those.
this way, when the level is built the assets are ready


on init: catalog the assets available; do not load them into memory

in editor mode: export asset catalog, diff asset catalog, export diff, etc



## Core Concepts

1.  **Granularity (ECS Advantage):** Instead of merging monolithic objects (like a whole NPC record), often merging individual components or fields within components. This reduces the scope of potential conflicts.
2.  **Asset Identification:** Every mergeable asset needs a unique, persistent identifier.
    *   **Entities:** A unique Entity ID.
    *   **Components:** Identified by Entity ID + Component Type.
    *   **Files (Shaders, Scripts, Configs):** Canonical path (e.g., `scripts/player/movement.bb`, `shaders/standard_lit.frag`).
    *   **Binary Assets (Textures, Meshes, Audio):** Canonical path (e.g., `textures/props/barrel_d.png`).
3.  **Load Order:** Still crucial. It defines the *sequence* in which modifications are layered and provides the ultimate fallback mechanism when a merge fails or isn't applicable.
4.  **Merge Strategies:** Different asset types require different merge algorithms.
5.  **Conflict Resolution:** Define clear rules for what happens when an automatic merge isn't possible.

## Proposed Pipeline

1.  **Mod Discovery & Ordering:**
    *   Identify all available mods (e.g., from a specific directory).
    *   Read manifests (`modinfo.bb`, `mod.json`, etc.) to get dependencies and user-defined load order hints.
    *   Calculate the final linear load order (Base Game -> Mod A -> Mod B -> Mod C...).

2.  **Asset Collection & Grouping:**
    *   Iterate through mods in load order.
    *   For each mod, scan its contents for assets (data files, scripts, textures, etc.).
    *   Maintain a central registry (e.g., a dictionary/hash map) keyed by the unique Asset ID.
    *   For each Asset ID, store a list of *sources* (Mod + specific file/data chunk) in load order.
    *   *Example:*
        *   `AssetRegistry["Entity_Player.PositionComponent"] = [ (BaseGame, data), (ModB, data_override) ]`
        *   `AssetRegistry["scripts/player/jump.lua"] = [ (BaseGame, file_path), (ModA, file_path), (ModC, file_path) ]`
        *   `AssetRegistry["textures/weapons/sword.png"] = [ (BaseGame, file_path), (ModC, file_path) ]`

3.  **Merge Execution (per Asset ID):**
    *   Iterate through each Asset ID in the `AssetRegistry`.
    *   If an asset only has one source (the Base Game or a single mod added it), use it directly.
    *   If an asset has multiple sources, initiate the merge process:
        *   **Identify Asset Type:** Determine if it's Binary, Textual, Structured Data (Component), etc.
        *   **Select Merge Strategy:** Based on the type.
        *   **Perform Merge:** Execute the chosen algorithm.
        *   **Handle Conflicts:** Apply conflict resolution rules if the merge fails.
        *   **Store Result:** Place the final, merged asset into a "Live Asset Cache" that the game engine will use.

4.  **Runtime:**
    *   The game engine requests assets using their unique IDs.
    *   The Asset Manager returns the merged version from the Live Asset Cache.
## Algorithms and Strategies

### A. Binary Assets (Textures, Meshes, Audio, Compiled Code/Shaders)

*   **Strategy:** Load Order Fallback ("Last Loaded Wins").
*   **Algorithm:**
    1.  Identify all sources for the binary asset path (e.g., `textures/props/barrel_d.png`).
    2.  Select the source belonging to the mod highest in the load order.
    3.  Use that specific file.
*   **Rationale:** You cannot meaningfully "merge" the binary content of two different textures or meshes. The last loaded version represents the final intended override.

### B. Textual Assets (Scripts, Shaders Source, Config Files)

*   **Strategy:** 3-Way Merge (or K-Way Merge).
*   **Algorithm (Conceptual - using 3-Way Merge as the core building block):**
    1.  Identify the sources: `Base`, `Mod A`, `Mod B`, `Mod C`...
    2.  Start with the `Base` version.
    3.  **Merge `Base` with `Mod A`:**
        *   Perform a diff between `Base` and `Mod A` to find changes made by `Mod A`.
        *   Apply these changes to `Base` to get `Merged_Base_A`.
    4.  **Merge `Merged_Base_A` with `Mod B`:**
        *   Perform a 3-way merge:
            *   **Base:** `Base` (The original common ancestor for this step)
            *   **"Mine":** `Merged_Base_A` (The result of the previous merge)
            *   **"Theirs":** `Mod B` (The next set of changes)
        *   Use a standard 3-way merge algorithm (like the one used by `git merge`):
            *   Identify lines/hunks added, deleted, or modified in `Merged_Base_A` compared to `Base`.
            *   Identify lines/hunks added, deleted, or modified in `Mod B` compared to `Base`.
            *   Combine non-conflicting changes.
            *   **Conflict:** Occurs if both `Merged_Base_A` and `Mod B` modified the *same* line(s) *differently* compared to `Base`.
        *   Result is `Merged_Base_A_B`.
    5.  **Repeat:** Merge `Merged_Base_A_B` with `Mod C` (using `Base` as the ultimate ancestor, or potentially using the previous version as the base for the next 3-way merge - careful design needed here).
    6.  **Conflict Handling:**
        *   **Automatic:** If a conflict occurs, the merge fails *for this asset*.
        *   **Resolution:**
            *   **Option 1 (Fallback):** Use the version from the *last* mod involved in the conflict (respects load order priority).
            *   **Option 2 (Strict):** Fail the asset load, report error.
            *   **Option 3 (User Tool):** Mark the asset as conflicted and require user intervention via a dedicated merge tool (like `git mergetool`). This is powerful but complex for end-users.
            *   **Option 4 (Log & Default):** Log the conflict and default to the Base Game version or the last loaded version, accepting potential breakage.
*   **Underlying Algorithms:** `diff` (e.g., Myers diff algorithm, Patience diff), 3-way merge logic. Libraries exist for this in many languages.
*   **Caveats:** Merging code/shaders textually can easily break syntax or logic even if the text merge itself succeeds without line conflicts. Requires careful testing by modders.

### C. Structured Data (Component Data - JSON, YAML, Custom Binary)

*   **Strategy:** Field-level or Granular Merge with Type-Specific Rules. This is the most complex.
*   **Algorithm:**
    1.  Represent the component data as a structured object (like a dictionary/map or object instance).
    2.  Start with the `Base` version of the component data.
    3.  Iterate through the mods in load order (`Mod A`, `Mod B`, ...).
    4.  For each mod that modifies this component:
        *   Compare the mod's version (`ModVersion`) against the *original* `Base` version (or potentially the version from the *immediately preceding* mod, depending on desired semantics).
        *   Identify changed fields/properties.
        *   Apply these changes to the current `MergedVersion`.
        *   **Conflict Handling (Field Level):**
            *   If `Mod B` tries to change a field that was *already changed* by `Mod A` (compared to their common ancestor, likely `Base`):
                *   **Default Rule (Load Order):** `Mod B`'s value overwrites `Mod A`'s value for that specific field. This is often the most intuitive default for component data.
                *   **Configurable Rule:** Allow modders or component definitions to specify merge strategies *per field* (e.g., `health: "replace"`, `inventory_items: "union"`, `tags: "append"`).
        *   **Handling Complex Types:**
            *   **Lists/Arrays:** Concatenate? Replace? Union (unique elements)? Needs defined behavior (e.g., default to replace, allow "append" strategy).
            *   **Nested Objects/Dictionaries:** Recursively apply the merge logic? Replace the entire nested object? (Recursive is generally better but more complex).
*   **Example (`PositionComponent` with X, Y fields):**
    *   `Base`: `{ x: 0, y: 0 }`
    *   `Mod A`: `{ y: 10 }` (Changes Y)
    *   `Mod B`: `{ x: 5 }` (Changes X)
    *   Merge `Base` with `Mod A` -> `{ x: 0, y: 10 }`
    *   Merge result with `Mod B`: `Mod B` changes X (from base 0 to 5). The current merged Y is 10. No conflict on individual fields. Result: `{ x: 5, y: 10 }`.
*   **Example (`StatsComponent` with Health):**
    *   `Base`: `{ health: 100 }`
    *   `Mod A`: `{ health: 150 }`
    *   `Mod B`: `{ health: 120 }`
    *   Merge `Base` with `Mod A` -> `{ health: 150 }`
    *   Merge result with `Mod B`: `Mod B` also changes health. Conflict! Applying default "last write wins": Result -> `{ health: 120 }`.
*   **ECS Specifics:**
    *   **Adding Components:** If `Mod A` adds `ComponentX` to `Entity_123`, and `Mod B` also adds `ComponentX` (perhaps with different data), how is this handled? Probably treat the *second* addition as a modification/merge onto the first one.
    *   **Removing Components:** If `Mod A` adds `ComponentX` and `Mod B` *removes* it? Load order likely dictates the final state (component exists if last mod touching it adds it, doesn't exist if last mod removes it).

## Tooling Requirements

*   **Mod Manifests:** Need syntax for mods to declare dependencies and potentially hint at desired merge strategies for specific assets or data fields.
*   **Conflict Resolution Tool (Highly Recommended):** A GUI tool for users/modders to visualize merge conflicts (especially textual and structured data) and manually resolve them, similar to `git mergetool`. This tool would save the resolved merge decisions.
*   **Debugging Tools:** Ability to inspect the final merged state of any asset and trace back which mods contributed to it.

## Summary

1.  **Binaries:** Use load order fallback (simple).
2.  **Text:** Use 3-way merge algorithms, define clear conflict resolution (e.g., fallback to load order winner for the whole file on conflict). Be aware of potential semantic breakage.
3.  **Structured Data (Components):** Use field-level merging. Default to "last write wins" for conflicting field changes based on load order. Consider allowing type/field-specific merge strategies (append, union, etc.) defined in schemas or manifests. This is the most complex part to design robustly.

This "true merge" system is significantly more complex to implement than the Bethesda "Rule of One" but offers the potential for much better out-of-the-box compatibility between mods that touch different *parts* of the same asset or component. The key is defining clear, predictable merge rules and robust conflict handling.
