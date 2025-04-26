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
