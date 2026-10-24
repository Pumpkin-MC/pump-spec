# PUMP Format Specification

[![License: MIT OR Apache-2.0](https://img.shields.io/badge/License-MIT%20OR%20Apache--2.0-blue.svg)](LICENSE-MIT)
[![Specification Version](https://img.shields.io/badge/spec-v1.0-green.svg)](SPEC.md)

Specification, documentation, and reference materials for **PUMP**, the high-performance, crash-resilient Minecraft world region and chunk format developed by [Pumpkin](https://github.com/Pumpkin-MC/Pumpkin).

---

## Overview

Minecraft's legacy **Anvil** (`.mca`) format has served the community for years, but poses significant bottlenecks for modern, multi-threaded Minecraft servers:

- **4 KiB Sector Padding Waste:** Even a 100-byte change forces an allocation of at least 4 KiB, causing massive internal fragmentation (often wasting 30–50% of disk space on empty padding).
- **Vulnerability to Torn Writes:** In-place updates can lead to corrupted chunk headers or sliced writes if the server crashes or loses power mid-save.
- **Java NBT Bottleneck:** Storing tabular section data (block palettes, block states, biomes, lighting nibbles, heightmaps) in verbose NBT creates high CPU serialization overhead and memory churn.

**PUMP** solves these issues with a modern storage architecture:

1. **256-Byte Sector Granularity:** 16× finer sector alignment than Anvil, reducing slack space waste to virtually zero.
2. **Copy-on-Write Crash Safety:** New chunk revisions are written to free sectors and synced before the 8 KiB file header updates. A sudden crash leaves every chunk intact in either its old or new version.
3. **Zstandard + XXH64:** High compression ratios and multi-threaded throughput with robust 64-bit integrity verification.
4. **`PCHK` Binary Chunk Format:** Replaces heavyweight NBT for sections, biomes, and lighting with direct, bit-packed structures, while retaining standard unnamed NBT exclusively for dynamic block entities and custom plugin data.

---

## Specification

The complete formal specification is documented in [**SPEC.md**](SPEC.md):

- [1. Overview & Comparison with Anvil/Linear](SPEC.md#1-overview)
- [2. Region Container (`.pump`)](SPEC.md#2-region-container-pump)
  - Sector layout, 256-byte superblock, 1024-entry offset/length table, allocation bitmap, and atomic flush sequence.
- [3. Chunk Compression Envelope](SPEC.md#3-chunk-compression-envelope)
  - 12-byte frame header (`uncompressed_len: u32` + `checksum: u64` XXH64) and Zstd compression.
- [4. Native Chunk Payload (`PCHK` v1)](SPEC.md#4-native-chunk-payload-pchk-v1)
  - Header, heightmaps, homogeneous/indexed/dense block and biome palettes, nibble-packed lighting, scheduled tick queues, block entities, and custom metadata.
- [5. Anvil Compatibility & Mapping](SPEC.md#5-anvil-compatibility--mapping)
- [6. Security & Resource Bounds](SPEC.md#6-security--resource-bounds)

---

## Implementations & Ecosystem

- **[Pumpkin](https://github.com/Pumpkin-MC/Pumpkin):** The flagship Minecraft server in Rust utilizes `.pump` as its default world format, including built-in bi-directional converters between Anvil, Linear, and PUMP.

---

## License

This specification and related materials are dual-licensed under either:

- **MIT License** ([LICENSE-MIT](LICENSE-MIT))
- **Apache License, Version 2.0** ([LICENSE-APACHE](LICENSE-APACHE))

at your option.
