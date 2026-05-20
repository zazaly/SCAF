# SCAF
We are designing a new digital comic book file format called ["Seekable Comic Archive Format (SCAF)"]. Based on Zstandard's seekable format principles: independent compressed frames + a seek table at the end (in a skippable frame) that enables random access to pages, panels, or specific assets without decompressing the entire file.

# The three sages
To kick this off, I’m going to simulate the Grill-for-Alignment Phase where our three experts hash out the constraints, identify the gaps in your prompt, and build the architectural consensus needed to generate the specification.

# 🎙️ The Alignment Grill (Simulated Discussion)
Expert 1 (Format Architect): "Alright, looking at the requirements for SCAF. We are building on the Zstandard Seekable Format. The biggest architectural constraint here is the network-mutable metadata frame. If Grimmory needs to update a cover or ComicInfo.xml on a 2GB file over the network without rebuilding the file, we cannot shift the byte offsets of the asset frames. If the offsets shift, the Zstd Seek Table at the end of the file becomes invalid."
Expert 3 (Implementation): "Exactly. The only way to guarantee O(1) in-place network mutation without rewriting the seek table is to lock the physical size of that metadata block. If the prompt says 'no larger than 10MB', we should enforce a fixed 10MB allocation at byte 0. We compress the metadata, and whatever is left of the 10MB, we fill with a Zstd skippable frame (Padding). This keeps the starting offset of Frame 2 strictly at 10,485,760 bytes."
Expert 2 (Domain Expert): "I love that for Grimmory's performance. But 10MB is a lot for a webtoon. Still, disk is cheap, and O(1) edits are a massive win for self-hosted libraries. Let's make sure we define a 'Spine' image properly — self-hosted apps use these for 3D bookshelf UIs. We also need to map logical pages to physical frames. Are we putting one page per frame? Or chunking them?"
Expert 1: "Zooming out, if we have a 1000-page graphic novel, 1000 frames mean a bloated seek table. We need a 'Deep Module' approach. Let's define a TOC (Table of Contents) inside that 10MB metadata frame. The TOC maps logical assets (page_01.jpg, spine.png) to uncompressed byte offsets. The Zstd Seek Table at the EOF only maps uncompressed offsets to physical frames. Our SeekableReader interface will hide this two-step resolution from the client."
Expert 3: "Testability constraint: What if the 10MB head is corrupted? We need checksums. Standard Zstd frame checksums will cover the metadata frame. If the TOC is valid, we can read the comic. Let's draft the spec."

# Seekable Comic Archive Format (SCAF) Specification

## 1. Context & Glossary (Ubiquitous Language)
To ensure the team and future maintainers are aligned, we strictly adhere to these definitions:

SCAF (Seekable Comic Archive Format): The overall file format, extension .scaf.

10MB Mutable Head: The strictly 10-megabyte (10,485,760 bytes) block at the beginning of the file containing the Metadata Frame and a Padding Frame.

Metadata Frame: A standard Zstd frame containing the file TOC, ComicInfo.xml, Cover Image, and Spine Image.

Padding Frame: A Zstd skippable frame (0x184D2A50) used to pad the Mutable Head to exactly 10MB.

Asset Frame: A compressed Zstd frame containing one or more serialized comic pages or resources.

Seek Table: The Zstd-standard skippable frame at the End of File (EOF) that maps compressed physical offsets to uncompressed logical offsets.

TOC (Table of Contents): A JSON or binary dictionary within the Metadata Frame mapping logical domain entities (e.g., "Page 5", "Variant Cover") to uncompressed byte offsets.

Guided View Region: Bounding box coordinates defined in ComicInfo.xml indicating focus areas for panel-by-panel reading.

## 2. High-Level Format Specification
SCAF is a strictly valid Zstandard Seekable file, allowing any standard Zstd tool to decompress it entirely, while SCAF-aware readers can randomly access it.

Physical Layout:

Byte 0 - 10MB: The Mutable Head

Magic Number: Standard Zstd Magic (0xFD2FB528).

Content: A TAR stream (uncompressed) containing toc.json, ComicInfo.xml, cover.jpg, and spine.jpg, compressed as a single Zstd frame.

Padding: Immediately followed by a Zstd skippable frame filling the exact remainder up to byte 10,485,759.

Byte 10MB - EOF (minus Seek Table): Asset Frames

A continuous stream of standard Zstd frames.

Inside the frames is a continuous TAR stream of the comic's images/assets.

Frames are chunked by size (e.g., every 4MB of uncompressed data) or by logical page boundaries, depending on the encoder.

EOF: Zstandard Seek Table

A Zstd skippable frame containing the jump table required to map uncompressed TAR offsets back to the physical frames.

## 3. Detailed Seek & Access Mechanics
To support partial loading and instant streaming (e.g., opening directly to page 50), the system uses a two-stage resolution process hidden behind a Deep Module interface.

The Resolution Flow:

Init: Client HTTP RANGE requests the last 8KB of the file to parse the Zstd Seek Table.

Metadata Load: Client HTTP RANGE requests the first 10MB. It decompresses the Metadata Frame, reading toc.json and ComicInfo.xml.

Seek: To render "Page 50", the client looks up page_50.jpg in toc.json. The TOC says it begins at uncompressed offset 512,000,000.

Frame Resolution: The client queries the Seek Table: "Which physical frame contains uncompressed offset 512,000,000?"

Fetch & Render: The client HTTP RANGE requests that specific compressed Asset Frame, decompresses it in memory, extracts the TAR bytes for the JPEG, and renders it.

## 4. Metadata & Extensibility
The format delegates all comic-specific domain logic to ComicInfo.xml (the industry standard) but extends it implicitly by providing structured access to assets.

Cover & Spine Routing: By explicitly storing cover and spine in the 10MB Mutable Head, library managers like Grimmory can serve bookshelf UI requests simply by reading the first 10MB, entirely bypassing the multi-gigabyte payload.

Guided View & Webtoon Scrolling: ComicInfo.xml can store an array of <PanelRegion> or <ScrollAnchor> tags. Because the TOC maps these assets to specific frames, a webtoon reader can pre-fetch the next 3 frames down the vertical scroll without keeping the previous 50 frames in memory.

In-Place Editing: To change the cover, Grimmory decompresses the Mutable Head, replaces cover.jpg, recompresses, recalculates the padding size to hit exactly 10MB, and overwrites bytes 0-10MB. No other bytes in the file are touched. The EOF Seek Table remains 100% valid.

## 5. Implementation Considerations
To ensure high reliability and developer ergonomics, we define a deep module: SeekableComicReader.

TypeScript
// Deep Module Interface
interface SeekableComicReader {
  // Parses TOC and Seek table. O(1) network requests.
  init(dataSource: RangedDataSource): Promise<void>; 
  
  // Retrieves domain metadata without loading images
  getMetadata(): ComicInfo; 
  
  // Streams a specific page into memory
  getPageStream(pageIndex: number): ReadableStream; 
  
  // O(1) in-place update for self-hosted admins
  updateMutableHead(newMetadata: ComicInfo, cover: Buffer, spine: Buffer): Promise<void>;
}
Encoder/Decoder Guidance: Encoders MUST use Zstd's --rsyncable or chunked frame boundaries to ensure frames don't span massive byte ranges. We recommend a maximum uncompressed Asset Frame size of 8MB.

Memory Constraints: Mobile readers only need to hold 1-2 decompressed frames in RAM at a time (~16MB total), rather than unzipping an entire CBZ to mobile storage.

## 6. Architectural Decision Records (ADRs)
ADR-001: The 10MB Fixed Padded Head

Problem: Grimmory needs to update covers/metadata instantly over the network without recompressing/rewriting gigabytes of data or invalidating the EOF Seek Table.

Options Considered: 1) Separate sidecar files. 2) Append metadata to EOF. 3) Fixed-size padded head frame.

Decision: Implement a strictly 10MB reserved block at byte 0 using Zstd skippable frames for padding.

Consequences: Wastes a few megabytes per file, but achieves guaranteed O(1) mutation for administrative operations. Simplifies self-hosted database syncing immensely.

ADR-002: Internal Archive Format (TAR vs ZIP)

Problem: How to serialize the raw images inside the uncompressed Zstd stream.

Options Considered: 1) ZIP inside Zstd. 2) Raw bytes with custom headers. 3) Uncompressed TAR.

Decision: Uncompressed TAR.

Consequences: TAR is natively streamable and carries zero compression overhead (relying purely on Zstd for compression). It provides a standard file-system-like abstraction that all programming languages can parse trivially.

## 7. Open Questions & Next Steps
10MB Exceedance: What if a publisher embeds a massive 12MB uncompressed GIF as a variant cover, exceeding the Mutable Head limit? Proposed mitigation: Enforce a strict warning in the CLI encoder, pushing oversized covers into the Asset Frames and keeping a low-res thumbnail in the Head.

Prototyping: Build a node-zstd-scaf prototype that proves the byte-range HTTP streaming works against an AWS S3 hosted .scaf file.

## 8. Potential Risks & Mitigations
Risk: Tooling Ecosystem. Standard comic readers (YACReader, Panels) expect CBZ/CBR (Zip/Rar).

Mitigation: SCAF's reliance on standard Zstd + TAR means we can write a lightweight FUSE driver or local proxy that mounts a .scaf file as an uncompressed local directory to the OS, granting immediate legacy compatibility.

Risk: Zstd Seek Table Corruption. If the EOF is truncated during a download, the file cannot be navigated.

Mitigation: Because SCAF uses a standard TAR stream across its frames, an emergency recovery tool can simply pipe the file through standard zstd -d to recover all assets, ignoring the seek table entirely.

We made a hard call in ADR-001 to mandate TAR inside Zstd rather than rolling a custom byte-offset manifest, to keep the ecosystem standard. Given your goals with Grimmory, does an internal uncompressed TAR stream combined with the TOC JSON provide the right balance of simplicity for your Go/Rust backend, or would you prefer a purely custom binary offset map to avoid TAR parsing overhead?
