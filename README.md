<p align="center">
	<picture>
		<source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/Falcon-MC/.github/main/profile/logo-white.png">
		<img src="https://raw.githubusercontent.com/Falcon-MC/.github/main/profile/logo.png" alt="Falcon" width="200">
	</picture>
	<br>
	<b>Falcon BedrockData</b>
	<br>
	Minecraft: Bedrock Edition game data files, versioned by protocol
</p>

<p align="center">
	<img src="https://img.shields.io/badge/minecraft-v1.26.60%20(Bedrock)-56383E" alt="Minecraft">
	<img src="https://img.shields.io/badge/protocol-2225-blue" alt="Protocol">
</p>

## What is this?

The game data the [Falcon](https://github.com/Falcon-MC/Falcon) server needs and cannot compute itself: block
and item palettes, recipes, creative inventory, loot tables, biomes, spawn rules and more. Every file is
generated from the official dedicated server, one directory per network protocol.

A protocol bump (for example 2193 to 2225) becomes a data update in this single repository: add a
new `data/<protocol>/` directory, register it in `versions.json`, and every consumer picks it up.

- **Manifest** - `versions.json` lists every protocol with its Minecraft version, files and SHA-256
- **Verified** - the CMake module refuses a file whose hash does not match the manifest
- **Embeddable** - CMake functions turn any file into a C++ header
- **Generated** - new versions are produced automatically from the dedicated server

## Layout

```
versions.json            manifest: latest protocol, Minecraft version, files and SHA-256 per protocol
data/<protocol>/<file>   the data files for one protocol version
cmake/                   CMake module that locates, verifies and embeds the files
```

`versions.json` is the single source of truth:

```json
{
  "latest": 2225,
  "versions": [
    {
      "protocol": 2225,
      "minecraftVersion": "1.26.60",
      "files": [
        { "name": "biome_definitions.nbt", "format": "gzip, big-endian NBT", "sha256": "..." }
      ],
      "pending": [ "block_palette.nbt" ]
    }
  ]
}
```

- `files` lists the files present in `data/<protocol>/` with their SHA-256. CMake refuses a file whose
  hash does not match.
- `pending` lists files the server needs for that protocol that are not in the repository yet.

A manifest was chosen over a `data/latest` symbolic link because links are not portable to every
Git checkout on Windows, and because the manifest also carries checksums and the Minecraft version.

## Current contents

| Protocol | Minecraft |
|----------|-----------|
| 2225     | 1.26.60   |
| 2193     | 1.26.52   |
| 2169     | 1.26.45   |
| 2168     | 1.26.43   |

Every protocol ships the 17 files described below, none pending.

## File formats

| File | Format |
|------|--------|
| `biome_definitions.nbt` | gzip-compressed big-endian NBT, root compound with `biomeStringList` and per-biome definitions including `chunkGenData` |
| `block_definitions.nbt` | gzip-compressed big-endian NBT, root compound with a `blocks` list; each entry has `name` (string) and `properties` (compound), the data-driven block definitions sent in the start game block palette, CC0 |
| `block_light.json` | JSON object with a `blocks` array; each entry has `name`, the most common `emission` and `filter` (0 to 15) of its states, and an optional `overrides` array of `{ states, emission, filter }` for the states that differ |
| `block_palette.nbt` | gzip-compressed big-endian NBT, root compound with a `blocks` list; each entry has `network_id` (int), `name_hash` (long), `name` (string), `version` (int), `states` (compound) |
| `command_translations.json` | JSON object `locale -> { translation key -> text }` with the `commands.*` strings of every language of the vanilla resource pack |
| `creative_items.json` | JSON object with `groups` (`creative_category`, `name`, `icon`) and `items` (`id`, `block_state_b64`, `group_index`), as sent in the creative content packet |
| `entity_definitions.json` | JSON object `entity identifier -> { description, components, component_groups, events }` from the vanilla behavior packs, versioned packs applied in version order |
| `entity_despawn.json` | JSON object `entity identifier -> minecraft:despawn component` |
| `entity_loot_tables.json` | JSON object `entity identifier -> loot table path` from each entity's default `minecraft:loot` component, versioned behavior packs applied in version order |
| `item_components.nbt` | gzip-compressed big-endian NBT, root compound `item identifier -> { components }` for component-based items, CC0 |
| `item_palette.json` | JSON object with an `items` array; each entry has `name` (string), `id` (network id), `version` (int), `component_based` (bool), as sent in the item registry packet, CC0 |
| `item_tags.json` | JSON object `tag -> [item identifiers]`, keys and lists sorted, two-space indentation, trailing newline |
| `loot_tables.json` | JSON object `loot table path -> loot table`, paths relative to the vanilla behavior pack `loot_tables` folder (for example `chests/spawn_bonus_chest.json`), versioned behavior packs applied over the base pack in version order, tables unchanged |
| `r16_to_current_item_map.json` | JSON object with `simple` (`old identifier -> identifier`) and `complex` (`old identifier -> { meta -> identifier }`) for pre 1.16 item names and data values, CC0 |
| `recipes.json` | JSON object with `version` and a `recipes` array as sent in the crafting data packet |
| `spawn_rules.json` | JSON object `entity identifier -> { population_control, conditions }` from the vanilla spawn rules |
| `voxel_shapes.json` | JSON array of `{ "identifier": string, "boxes": [[[minX, minY, minZ], [maxX, maxY, maxZ]], ...] }` in 1/16 block units |

## Adding a protocol version

1. Create `data/<protocol>/` and put the new files in it.
2. Compute the SHA-256 of every file (`sha256sum`, or `Get-FileHash -Algorithm SHA256` on Windows,
   lower-case).
3. Add an entry to `versions` in `versions.json` with `protocol`, `minecraftVersion`, `files` and
   `pending`, then point `latest` at it.
4. Keep older protocol directories until no consumer pins them any more.

## Consuming from CMake

Requires CMake 3.19 or newer (`string(JSON)`).

```cmake
include(FetchContent)
FetchContent_Declare(
    FalconBedrockData
    GIT_REPOSITORY https://github.com/Falcon-MC/BedrockData.git
    GIT_TAG main
)
FetchContent_MakeAvailable(FalconBedrockData)
include("${falconbedrockdata_SOURCE_DIR}/cmake/FalconBedrockData.cmake")
```

Set `FALCON_BEDROCK_DATA_PROTOCOL` before the `include` to pin a protocol; otherwise `latest` from
`versions.json` is used.

Functions:

| Function | Result |
|----------|--------|
| `falcon_bedrock_data_directory(<var>)` | absolute path of `data/<protocol>` |
| `falcon_bedrock_data_minecraft_version(<var>)` | Minecraft version of the selected protocol |
| `falcon_bedrock_data_file(<name> <var>)` | absolute path of one file, after checking its SHA-256 |
| `falcon_bedrock_data_embed_binary(FILE <name> NAMESPACE <ns> SYMBOL <sym> OUTPUT <header>)` | header with `unsigned char <sym>[]` and `std::size_t <sym>Size` |
| `falcon_bedrock_data_embed_text(FILE <name> NAMESPACE <ns> SYMBOL <sym> DELIMITER <d> OUTPUT <header>)` | header with `const char <sym>[]` as a raw string literal |

Example producing the same header the server uses today for biome definitions:

```cmake
set(GENERATED_DIR "${CMAKE_CURRENT_BINARY_DIR}/generated")
falcon_bedrock_data_embed_binary(
    FILE biome_definitions.nbt
    NAMESPACE FalconBiomeData
    SYMBOL kBiomeDefinitionsNbt
    OUTPUT "${GENERATED_DIR}/BiomeDefinitionsNbt.h"
)
target_include_directories(MyTarget PRIVATE "${GENERATED_DIR}")
```

Each file used through these functions is added to `CMAKE_CONFIGURE_DEPENDS`, so editing a data file
re-runs the configure step.

## Generating

The files are produced by Falcon DataGen, which downloads the dedicated server, joins it with a headless client
and dumps the registries. The [Generate data](.github/workflows/generate.yml) workflow runs it every three hours
for the stable and preview channels and opens a draft pull request when a new server version is out.

## Related repositories

- [Falcon](https://github.com/Falcon-MC/Falcon) - the server
- [Protocol](https://github.com/Falcon-MC/Protocol) - packets and network types
- [Network](https://github.com/Falcon-MC/Network) - RakNet and NetherNet transport
- [NBT](https://github.com/Falcon-MC/NBT) - NBT tags and binary streams
- [BlockStateUpdater](https://github.com/Falcon-MC/BlockStateUpdater) - block state upgrade schemas

## Licensing information

The CMake module and the manifest are licensed under the
[GNU Lesser General Public License v3.0](LICENSE), which supplements the
[GNU General Public License v3.0](COPYING).

The data files come from different sources:

- `r16_to_current_item_map.json`, `item_palette.json`, `item_components.nbt` and `block_definitions.nbt` are released under CC0
- every other file is extracted from the official dedicated server and remains the property of Mojang;
  it is redistributed only so that Falcon can interoperate with the game

Falcon is not affiliated with Mojang. All brands and trademarks belong to their respective owners.
