# PUMP Format Specification

[![License: MIT OR Apache-2.0](https://img.shields.io/badge/License-MIT%20OR%20Apache--2.0-blue.svg)](LICENSE-MIT)
[![Specification Version](https://img.shields.io/badge/spec-v1.0.0-green.svg)](SPEC.md)

This repository contains the normative standard specification, data layout definitions, and architectural documentation for **PUMP**, a high-performance, crash-resilient Minecraft world region container and chunk storage architecture developed by [Pumpkin](https://github.com/Pumpkin-MC/Pumpkin).

---

## Architectural Highlights

Minecraft's legacy **Anvil** (`.mca`) format has served the community for years, but poses significant bottlenecks for modern, multi-threaded Minecraft servers:

- **4 KiB Sector Padding Waste:** Even a 100-byte modification forces an allocation of at least 4 KiB, causing massive internal fragmentation (often wasting 30–50% of disk space on empty padding).
- **Vulnerability to Torn Writes:** In-place sector overwriting can lead to corrupted chunk headers or sliced writes if the host process crashes or suffers power failure during save operations.
- **Java NBT Serialization Overhead:** Storing tabular section data (block palettes, block states, biomes, lighting nibbles, heightmaps) in verbose NBT creates high CPU serialization overhead and excessive memory allocation churn.

**PUMP** addresses these architectural limitations:

1. **256-Byte Sector Granularity:** 16× finer sector alignment than Anvil, reducing internal fragmentation and slack space to virtually zero.
2. **Copy-on-Write Crash Safety:** New chunk revisions are written to unassigned sectors and synced before the 8 KiB file header commits. A sudden failure leaves every chunk intact in either its previous or committed state.
3. **Zstandard + XXH64:** High compression ratios and multi-threaded throughput with robust 64-bit integrity verification.
4. **`PCHK` Binary Chunk Schema:** Eliminates NBT overhead for tabular section, biome, and lighting arrays through direct, bit-packed binary structures, while retaining standard unnamed NBT exclusively for dynamic block entities and auxiliary metadata.

---

## Normative Specification

The complete, formal specification is maintained in [**SPEC.md**](SPEC.md):

- [Chapter 1: Document Conventions & Conformance](SPEC.md#chapter-1-document-conventions--conformance)
- [Chapter 2: Numerical Representations & Coordinate Systems](SPEC.md#chapter-2-numerical-representations--coordinate-systems)
- [Chapter 3: Region Container Architecture (`.pump`)](SPEC.md#chapter-3-region-container-architecture-pump)
  - Sector layout, 256-byte superblock (`PumpSuperblock`), 1024-entry location table (`PumpChunkEntry`), allocation bitmap, and file truncation.
- [Chapter 4: Compression Envelope & Frame Format](SPEC.md#chapter-4-compression-envelope--frame-format)
  - 12-byte frame header (`PumpChunkFrameHeader`), Zstandard streaming decompression, and XXH64 integrity validation.
- [Chapter 5: Native Chunk Payload Schema (`PCHK` v1)](SPEC.md#chapter-5-native-chunk-payload-schema-pchk-v1)
  - Header (`PumpChunkHeader`), packed heightmaps, homogeneous/indexed/dense block and biome palettes, nibble-packed lighting buffers, scheduled tick queues, block entities, and custom extension compounds.
- [Chapter 6: Transaction Lifecycle & Consistency Model](SPEC.md#chapter-6-transaction-lifecycle--consistency-model)
  - Formal read sequence, phased atomic commit protocol with hardware write barriers, and crash consistency proofs.
- [Chapter 7: Security Considerations & Resource Bounds](SPEC.md#chapter-7-security-considerations--resource-bounds)
  - Memory exhaustion limits (64 MiB decompressed ceiling, maximum section counts) and structural validation invariants.
- [Appendix A: Rust Data Type Bindings](SPEC.md#appendix-a-rust-data-type-bindings)
  - Reference Rust constants and layout type definitions.
- [Appendix B: Anvil Compatibility and Transcoding](SPEC.md#appendix-b-anvil-compatibility-and-transcoding)

---

## Reference Implementations & Tooling

* **[Pumpkin](https://github.com/Pumpkin-MC/Pumpkin):** The high-performance Minecraft server in Rust utilizes `.pump` as its default world storage engine, featuring full bi-directional transcoding between Anvil, Linear, and PUMP.

---

## License

This specification and associated definitions are dual-licensed under either:

* **MIT License** ([LICENSE-MIT](LICENSE-MIT))
* **Apache License, Version 2.0** ([LICENSE-APACHE](LICENSE-APACHE))

at your option.
