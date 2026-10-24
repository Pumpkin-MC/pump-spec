# PUMP World Region & Chunk Format Specification

**Version:** 1.0  
**Status:** Stable / Standard  
**Authors:** Pumpkin Contributors  
**License:** MIT OR Apache-2.0  

---

## Table of Contents

1. [Overview](#1-overview)
   - [Design Goals](#design-goals)
   - [Comparison with Anvil & Linear](#comparison-with-anvil--linear)
2. [Region Container (`.pump`)](#2-region-container-pump)
   - [File Naming & Region Coordinates](#file-naming--region-coordinates)
   - [Sector Alignment](#sector-alignment)
   - [Header Structure](#header-structure)
   - [Sector Allocation & Fragmentation](#sector-allocation--fragmentation)
   - [Crash Resilience & Write Ordering](#crash-resilience--write-ordering)
3. [Chunk Compression Envelope](#3-chunk-compression-envelope)
   - [Frame Header](#frame-header)
   - [Compression Algorithm](#compression-algorithm)
   - [Integrity Checksum](#integrity-checksum)
4. [Native Chunk Payload (`PCHK` v1)](#4-native-chunk-payload-pchk-v1)
   - [Chunk Header](#chunk-header)
   - [Heightmaps](#heightmaps)
   - [Sections](#sections)
     - [Block Palette & States](#block-palette--states)
     - [Biome Palette & States](#biome-palette--states)
     - [Light Arrays](#light-arrays)
   - [Scheduled Ticks](#scheduled-ticks)
   - [Block Entities](#block-entities)
   - [Custom Data](#custom-data)
5. [Anvil Compatibility & Mapping](#5-anvil-compatibility--mapping)
6. [Security & Resource Bounds](#6-security--resource-bounds)

---

## 1. Overview

The **PUMP** format is a high-performance, crash-resilient world region and chunk storage system designed to replace the legacy Minecraft **Anvil** (`.mca`) format.

### Design Goals

1. **Sub-Kilobyte Sector Granularity:** Anvil's 4 KiB sectors lead to severe internal fragmentation (slack space) when storing compressed chunks. PUMP utilizes **256-byte sectors**, reducing tail padding waste by up to 16×.
2. **True Crash Resilience:** PUMP implements copy-on-write chunk allocation with strict write barriers. Chunk data is fully flushed to disk before updating the in-file allocation table, guaranteeing that a sudden crash or power failure leaves every chunk intact in either its previous or updated state without torn writes.
3. **No NBT Overhead for Bulk Data:** Java NBT is inefficient for bulk tabular data such as block states, biomes, heightmaps, and lighting arrays. PUMP introduces `PCHK`, a native binary representation with compact bit-packing, while preserving standard unnamed NBT exclusively for dynamic block entities and arbitrary custom data.
4. **Fast Modern Compression:** Uses **Zstandard (zstd)** with parallel compression and decompression on multi-core worker threads, coupled with **XXH64** checksums for rapid validation.

### Comparison with Anvil & Linear

| Feature | Anvil (`.mca`) | Linear (`.linear`) | PUMP (`.pump`) |
| :--- | :--- | :--- | :--- |
| **Sector Granularity** | 4096 bytes | None (stream-based) | **256 bytes** |
| **Slack Space Waste** | High (~2 KiB per chunk) | Zero | **Low (<128 bytes per chunk)** |
| **Write Amplification** | Low (sector-based) | High (entire region rewritten) | **Low (only modified chunks written)** |
| **Parallel I/O & Compaction** | Limited | Single stream | **Independent chunk compression** |
| **Bulk Chunk Payload** | Java NBT (heavy) | Java NBT (heavy) | **`PCHK` Binary (zero NBT overhead)** |
| **Block Entities / Custom Data** | Java NBT | Java NBT | **Unnamed Java NBT** |
| **Crash Safety** | Partial (unprotected overwrite) | Atomic rename (`.tmp`) | **Copy-on-write sector updates** |
| **Integrity Validation** | None (ad-hoc zlib/gzip) | XXH64 | **XXH64** |

---

## 2. Region Container (`.pump`)

A `.pump` file is a container format that stores up to **1024 chunks** organized in a 32×32 grid.

### File Naming & Region Coordinates

Region files are stored in the dimension's region directory and named:

$$\text{r.}X\text{.}Z\text{.pump}$$

Where:
- $X = \lfloor \text{chunk\_x} / 32 \rfloor$
- $Z = \lfloor \text{chunk\_z} / 32 \rfloor$

Chunk index within the region is calculated via:

$$\text{index} = (Z_{\text{chunk}} \ \& \ 31) \times 32 + (X_{\text{chunk}} \ \& \ 31)$$

Valid chunk indices range from `0` to `1023`.

### Sector Alignment

- **Sector Size (`SECTOR_BYTES`):** `256` bytes (`0x100`).
- Sector offsets are represented as unsigned 32-bit integers (`u32`).
- Maximum addressable file size per region file:

$$2^{32} \times 256 \text{ bytes} = 1 \text{ TiB}$$

- Any chunk payload written to disk is padded with zero bytes to align to the nearest multiple of `256` bytes:

$$\text{sector\_count}(\text{len}) = \lceil \text{len} / 256 \rceil$$

### Header Structure

The file header occupies exactly **33 sectors** ($33 \times 256 = 8,448$ bytes / `0x2100` bytes):

```
+---------------------------------------------------------------+
| Sector 0 (256 bytes): Superblock                              |
+---------------------------------------------------------------+
| Sectors 1..33 (8192 bytes): Chunk Location Table              |
| (1024 entries * 8 bytes each)                                 |
+---------------------------------------------------------------+
| Sector 33..EOF: Chunk Payload Data Sectors                    |
+---------------------------------------------------------------+
```

#### Sector 0: Superblock (256 Bytes)

All multi-byte numeric values are stored in **Big-Endian** byte order.

| Byte Offset | Size | Type | Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00..0x08` | 8 | `[u8; 8]` | `signature` | Magic signature and format version: `b"PUMP\x00\x00\x00\x01"` |
| `0x08..0x0C` | 4 | `i32` | `region_x` | Region X coordinate |
| `0x0C..0x10` | 4 | `i32` | `region_z` | Region Z coordinate |
| `0x10..0x100`| 240 | `[u8; 240]` | `reserved` | Reserved for future expansion; filled with `0x00` |

#### Sectors 1..33: Chunk Location Table (8,192 Bytes)

Contains 1024 contiguous 8-byte entries corresponding to chunk indices `0` to `1023`.

For index $i$:

| Relative Offset | Size | Type | Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `i * 8 + 0` | 4 | `u32` | `sector_offset` | Absolute sector index where chunk record starts. `0` if chunk is absent. Must be $\ge 33$ if present. |
| `i * 8 + 4` | 4 | `u32` | `byte_length` | Exact byte length of the chunk frame record (excluding sector padding). |

### Sector Allocation & Fragmentation

- Sector numbers `0..=32` are reserved for the header. Chunk payloads are allocated at sector offsets $\ge 33$.
- Implementations track sector allocations via a dynamic free-run bitmap:
  - When writing a chunk, search for the earliest contiguous run of free sectors of length $\text{sector\_count}(\text{byte\_length})$.
  - If no suitable gap exists, sectors are allocated at the current end of the file.
- Old sectors previously occupied by updated chunks are freed and made available for future chunk writes.
- When freed sectors occur at the end of the file, the file is shrunk via truncation (`ftruncate` / `set_len`).

### Crash Resilience & Write Ordering

To protect against corruption during unexpected shutdowns:

1. Staged chunks are compressed and prepared in memory.
2. New sectors are allocated from the free list, and the chunk records are written to these sectors.
3. The underlying storage is synced (`fsync` / `fdatasync`).
4. The 8 KiB location table (and superblock) is updated in memory and written to sectors `0..32` at file offset `0`.
5. The header update is synced to disk.
6. Old sectors are returned to the free pool, and unused trailing file sectors are truncated.

Because chunk data is guaranteed to reside on disk before the header pointers switch to point to it, a crash at any point preserves either the previous version or the newly committed version of every chunk.

---

## 3. Chunk Compression Envelope

Every non-empty chunk entry points to a compressed frame record:

```
+--------------------+---------------------+-------------------------------+
| uncompressed_len   | checksum            | compressed_payload            |
| (u32, 4 bytes)     | (u64, 8 bytes)      | (variable length)             |
+--------------------+---------------------+-------------------------------+
|<------- 12-byte Frame Header ----------->|
```

### Frame Header

The frame header is **12 bytes**:

| Offset | Size | Type | Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00..0x04` | 4 | `u32` | `uncompressed_len` | Size of uncompressed chunk data in bytes (Big-Endian). Maximum allowable size: 64 MiB (`0x04000000`). |
| `0x04..0x0C` | 8 | `u64` | `checksum` | 64-bit **XXH64** checksum (seed = 0) of the uncompressed data (Big-Endian). |

### Compression Algorithm

- Compressed using **Zstandard (`zstd`)**.
- Decompression streaming decoders verify that the decompressed output byte count exactly equals `uncompressed_len`.

### Integrity Checksum

- The checksum algorithm is **XXH64** with seed `0`.
- Decompressors **must** verify that `xxh64(uncompressed_data, 0) == checksum`. If the checksum or length does not match, the chunk is deemed corrupted.

---

## 4. Native Chunk Payload (`PCHK` v1)

The uncompressed payload begins with the magic identifier `PCHK`. All multi-byte numeric values are encoded in **Big-Endian**.

```
+-------------------+--------------------+--------------------+
| Magic ("PCHK")    | Version (u8 = 1)   | Header Fields...   |
+-------------------+--------------------+--------------------+
| Heightmaps (optional)                                       |
+-------------------------------------------------------------+
| Sections (Block Palette, Biome Palette, Lighting)           |
+-------------------------------------------------------------+
| Scheduled Block Ticks & Fluid Ticks                         |
+-------------------------------------------------------------+
| Block Entities (Unnamed Java NBT Compounds)                 |
+-------------------------------------------------------------+
| Custom Data (Unnamed Java NBT Compound, optional)           |
+-------------------------------------------------------------+
```

### Chunk Header

| Offset | Size | Type | Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00..0x04` | 4 | `[u8; 4]` | `magic` | ASCII bytes `b"PCHK"` (`0x50, 0x43, 0x48, 0x4B`) |
| `0x04` | 1 | `u8` | `version` | Payload schema version. Must be `1`. |
| `0x05..0x09` | 4 | `i32` | `data_version` | Minecraft World Data Version (e.g., `3955` for 1.21). |
| `0x09..0x0D` | 4 | `i32` | `x` | Chunk X coordinate |
| `0x0D..0x11` | 4 | `i32` | `z` | Chunk Z coordinate |
| `0x11..0x15` | 4 | `i32` | `min_section_y` | Lowest section Y index (e.g., `-4` for Overworld standard, `0` for Nether/End). |
| `0x15` | 1 | `u8` | `status` | Chunk status enum (`0..=9`):<br>`0`: Empty<br>`1`: StructureStarts<br>`2`: StructureReferences<br>`3`: Biomes<br>`4`: Terrain<br>`5`: Features<br>`6`: InitializeLight<br>`7`: Light<br>`8`: Spawn<br>`9`: Full |
| `0x16` | 1 | `u8` | `flags` | Bitfield flags:<br>• Bit 0 (`0x01`): `is_light_correct`<br>• Bit 1 (`0x02`): `has_world_surface`<br>• Bit 2 (`0x04`): `has_motion_blocking`<br>• Bit 3 (`0x08`): `has_motion_blocking_no_leaves`<br>• Bit 4 (`0x10`): `has_custom_data`<br>• Bits 5..7: Reserved (must be `0`). |
| `0x17..0x1F` | 8 | `u64` | `inhabited_time` | Total cumulative player inhabited time in ticks. |

### Heightmaps

Heightmaps are conditionally present based on `flags` bits 1, 2, and 3:

1. If `flags & 0x02 != 0`: **World Surface** heightmap (296 bytes)
2. If `flags & 0x04 != 0`: **Motion Blocking** heightmap (296 bytes)
3. If `flags & 0x08 != 0`: **Motion Blocking No Leaves** heightmap (296 bytes)

Each present heightmap consists of exactly **37 signed 64-bit integers** (`37 * 8 = 296` bytes), matching the standard packed representation (9 bits per entry for a 16×16 column).

### Sections

Directly follows the heightmaps:

| Size | Type | Name | Description |
| :--- | :--- | :--- | :--- |
| 2 | `u16` | `section_count` | Number of vertical 16×16×16 sub-chunk sections (Max: 256). |

Sections are serialized in ascending vertical order starting from `min_section_y` up to `min_section_y + section_count - 1`.

Each section sequentially contains:
1. Block Palette & States
2. Biome Palette & States
3. Light Arrays

---

#### Block Palette & States

Coordinates within a 16×16×16 section are ordered $(y, z, x)$ in row-major layout:

$$\text{index} = (y \ll 8) \mid (z \ll 4) \mid x \quad (0 \le \text{index} < 4096)$$

The block storage starts with a 1-byte mode flag:

##### Mode 0: Homogeneous Section (Single Block)
Used when the entire 16×16×16 section consists of identical block states (e.g. solid air or stone).

| Size | Type | Name | Description |
| :--- | :--- | :--- | :--- |
| 1 | `u8` | `mode` | `0` |
| 2 | `u16` | `block_state_id` | Global Minecraft block state ID. |

##### Mode 1: Indexed Palette
Used when the section contains between 1 and 256 unique block states.

| Size | Type | Name | Description |
| :--- | :--- | :--- | :--- |
| 1 | `u8` | `mode` | `1` |
| 2 | `u16` | `palette_len` | Palette length $N$ ($1 \le N \le 256$). |
| $N \times 4$ | Entries | `palette` | For each palette entry $0 \le k < N$:<br>• `block_state_id: u16` (Global block state ID)<br>• `block_count: u16` (Number of blocks in this section with this state) |
| 4096 | `[u8; 4096]` | `indices` | Flat array of 4096 byte indices into the palette, ordered by $(y, z, x)$. |

##### Mode 2: Dense Storage
Used when the section contains more than 256 unique block states.

| Size | Type | Name | Description |
| :--- | :--- | :--- | :--- |
| 1 | `u8` | `mode` | `2` |
| 8192 | `[u16; 4096]` | `states` | 4096 direct global `block_state_id: u16` entries, ordered by $(y, z, x)$. |

---

#### Biome Palette & States

Biomes in modern Minecraft are sampled on a 4×4×4 grid per 16×16×16 block section (64 biome cells total). Coordinates are ordered $(y, z, x)$ in row-major layout:

$$\text{index} = (y \ll 4) \mid (z \ll 2) \mid x \quad (0 \le \text{index} < 64)$$

The biome storage starts with a 1-byte mode flag:

##### Mode 0: Homogeneous Biome
Used when the entire section has a single biome.

| Size | Type | Name | Description |
| :--- | :--- | :--- | :--- |
| 1 | `u8` | `mode` | `0` |
| 1 | `u8` | `biome_id` | Biome registry numeric ID. |

##### Mode 1: Heterogeneous Palette
Used when multiple biomes are present in the section.

| Size | Type | Name | Description |
| :--- | :--- | :--- | :--- |
| 1 | `u8` | `mode` | `1` |
| 1 | `u8` | `palette_len` | Palette length $M$ ($1 \le M \le 64$). |
| $M \times 3$ | Entries | `palette` | For each palette entry $0 \le k < M$:<br>• `biome_id: u8`<br>• `biome_count: u16` |
| 64 | `[u8; 64]` | `biomes` | 64 direct biome IDs ordered by $(y, z, x)$. |

---

#### Light Arrays

Each section specifies its lighting data via a 1-byte bitmask:

| Size | Type | Name | Description |
| :--- | :--- | :--- | :--- |
| 1 | `u8` | `light_flags` | • Bit 0 (`0x01`): `has_block_light`<br>• Bit 1 (`0x02`): `has_sky_light` |
| 0 or 2048 | `[u8; 2048]` | `block_light` | Present only if `light_flags & 0x01 != 0`. 2048 bytes of 4-bit nibbles (4096 entries). |
| 0 or 2048 | `[u8; 2048]` | `sky_light` | Present only if `light_flags & 0x02 != 0`. 2048 bytes of 4-bit nibbles (4096 entries). |

If a light flag bit is `0`, all light levels in that section are implicitly zero.

---

### Scheduled Ticks

Following all chunk sections are the scheduled tick queues for blocks and fluids.

#### Block Ticks

| Size | Type | Name | Description |
| :--- | :--- | :--- | :--- |
| 4 | `u32` | `count` | Number of scheduled block ticks (Max: 65,536). |
| $\text{count} \times 16$ | Entries | `ticks` | 16 bytes per tick entry: |

Each block tick entry:
- `x: i32` (World coordinate)
- `y: i32` (World coordinate)
- `z: i32` (World coordinate)
- `delay: u8` (Remaining tick delay)
- `priority: i8` (Tick priority: `-3` = ExtremelyLow, `-2` = VeryLow, `-1` = Low, `0` = Normal, `1` = High, `2` = VeryHigh, `3` = ExtremelyHigh)
- `block_id: u16` (Block definition numeric ID)

#### Fluid Ticks

| Size | Type | Name | Description |
| :--- | :--- | :--- | :--- |
| 4 | `u32` | `count` | Number of scheduled fluid ticks (Max: 65,536). |
| $\text{count} \times 16$ | Entries | `ticks` | 16 bytes per tick entry: |

Each fluid tick entry:
- `x: i32` (World coordinate)
- `y: i32` (World coordinate)
- `z: i32` (World coordinate)
- `delay: u8` (Remaining tick delay)
- `priority: i8` (Tick priority)
- `fluid_id: u16` (Fluid definition numeric ID)

---

### Block Entities

Following the scheduled ticks is the list of block entities:

| Size | Type | Name | Description |
| :--- | :--- | :--- | :--- |
| 4 | `u32` | `count` | Number of block entities (Max: 65,536). |
| Variable | Entries | `block_entities` | Contiguous block entity entries. |

Each block entity is encoded as:
- `len: u32`: Exact byte length of the following NBT payload.
- `payload: [u8; len]`: Standard Java-format unnamed NBT Compound tag (`TAG_Compound`) containing:
  - `x: TAG_Int`
  - `y: TAG_Int`
  - `z: TAG_Int`
  - `id: TAG_String` (Entity identifier, e.g. `"minecraft:chest"`)
  - Any item inventories, custom names, or container properties.

---

### Custom Data

If bit 4 (`0x10`) of `flags` is set, custom chunk metadata is encoded at the end of the payload:

| Size | Type | Name | Description |
| :--- | :--- | :--- | :--- |
| 4 | `u32` | `len` | Exact byte length of custom data NBT payload. |
| `len` | `[u8; len]` | `payload` | Standard Java-format unnamed NBT Compound tag (`TAG_Compound`). |

This tag allows server implementations and plugins to persist custom properties, modded tags, or arbitrary auxiliary data without breaking standard chunk readers.

---

## 5. Anvil Compatibility & Mapping

A `.pump` file losslessly represents all vanilla Minecraft world data:

- **Coordinate Systems:** Identical to Anvil (`r.X.Z` maps to the exact same $32 \times 32$ chunks).
- **Block State & Biome IDs:** Mapped to their respective global Minecraft runtime IDs. Converters parse vanilla block states (`Name` + `Properties` compounds) into global IDs and vice-versa.
- **Heightmaps:** The three standard vanilla heightmaps (`WORLD_SURFACE`, `MOTION_BLOCKING`, `MOTION_BLOCKING_NO_LEAVES`) map directly to the 37 longs representation.
- **Lighting:** Vanilla nibble arrays map 1:1 to the 2048-byte full light buffers.
- **Ticks:** Anvil `block_ticks` and `fluid_ticks` NBT lists map 1:1 with the scheduled tick records.
- **Block Entities:** Retain their identical NBT compound tags.

When converting from Anvil to PUMP, any non-standard chunk-level root tags that do not belong to the standard schema are preserved in the `custom_data` compound.

---

## 6. Security & Resource Bounds

To ensure robust operation in multi-tenant or untrusted environments, parsers **must** enforce the following bounds:

1. **Maximum Chunk Decompressed Size:** Maximum $64\text{ MiB}$ (`67,108,864` bytes). Any chunk record advertising a higher `uncompressed_len` must be rejected immediately before allocating decompression memory.
2. **Checksum Verification:** The XXH64 checksum must be validated against the uncompressed payload.
3. **Maximum Section Count:** Maximum 256 sections per chunk.
4. **Maximum Ticks Count:** Maximum 65,536 block ticks and 65,536 fluid ticks per chunk.
5. **Maximum Block Entities:** Maximum 65,536 block entities per chunk.
6. **Sector Offset Sanity:** In `.pump` containers, `sector_offset` must be either `0` (empty) or $\ge 33$. The calculated byte range $(\text{sector\_offset} \times 256 + \text{byte\_length})$ must not exceed the actual file size.
