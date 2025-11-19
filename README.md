# Nova-Bone-Animation
Novalogic BHD, JO, X .bad File

# `.bad` File Format Specification

This document describes the layout and structure of the custom `.bad` animation file format. The format is used to store skeletal animation data including bones, frames, rotations, and optional translation and offset data.

## Tools

A work in progress exporter for 3ds max is [here](https://github.com/taylorfinnell/onbadexporter)

## Overview

A `.bad` file consists of:

1. A fixed-size animation header.
2. A series of frame table entries, one per bone.
3. Per-bone frame length arrays.
4. Per-bone rotation data arrays.
5. Optional offset entries for root motion or event data.
6. A list of bone records, defining the skeletal hierarchy and transforms.
7. Optional per-frame translation data, if enabled by flags.

All multi-byte values are stored in **little-endian** format.

## Flags

The `AnimationFlags` field in the header is a bitmask that modifies how data is interpreted:

- `0x01` (`ANIM_FLAG_LOOPED`): The animation loops.
- `0x02` (`ANIM_FLAG_TRANSLATION`): Translation frames are included for each bone.
- `0x04` (`ANIM_FLAG_UNK1`): Unknown purpose.
- `0x08` (`ANIM_FLAG_UNK2`): Unknown purpose.

## File Structure

**File Layout (in order):**

1. **AnimationHeader (80 bytes)**  
   Defines the overall animation properties, offsets into the file, number of bones, frames, etc.

2. **Frame Table Entries (`NumBones` entries, 12 bytes each)**  
   Each bone has a frame table entry, which references where that bone’s frame length data and rotation data start.

3. **Frame Length Arrays**  
   For each bone, a sequence of `NumFrames` unsigned 16-bit integers indicating the frame length or timing values per frame.

4. **Rotation Data Arrays**  
   For each bone, after the frame length array, `NumFrames` rotations are stored. Each rotation is 16 bytes (4 floats: x, y, z, w).

5. **Offset Entries (optional)**  
   If `NumOffsets > 0`, a series of `NumOffsets` offset entries follows. Each offset entry is 24 bytes and may represent per-frame root motion or event triggers.

6. **Bone Data (`NumBones` entries, 100 bytes each)**  
   Each bone record provides the bone name, hierarchy information, length, and a 3x3 transform matrix.

7. **Translation Data (optional)**  
   If `ANIM_FLAG_TRANSLATION` is set, per-frame translation data follows. This consists of `NumFrames * NumBones` sets of 3 floats (12 bytes) each, storing the positional offsets for each bone at each frame.

After all data is written, certain offsets within the file are updated to point to the correct locations.

---

## Detailed Structure Definitions

### AnimationHeader (80 bytes)

The header is always located at the start of the file.

| Field             | Type    | Size (bytes) | Description                                                     |
|-------------------|---------|--------------|-----------------------------------------------------------------|
| Version           | int32   | 4            | File version number.                                            |
| HeaderLen         | int32   | 4            | Size of this header in bytes (80).                              |
| FPS               | int32   | 4            | Frames per second.                                              |
| NumFrames         | int32   | 4            | Total number of frames in the animation.                        |
| AnimationFlags     | int32   | 4            | Bitmask of animation flags. See above.                          |
| NumBones          | int32   | 4            | Number of bones in the skeleton.                                |
| BoneOffset        | int32   | 4            | File offset to the start of the bone data section.              |
| FrameDataOffset   | int32   | 4            | File offset to the frame table entries (per-bone).              |
| Unknown1          | int32   | 4            | Unknown usage.                                                  |
| NumEventBlocks    | int32   | 4            | Possibly number of event blocks, often 0.                       |
| Unknown3          | int32   | 4            | Unknown usage. Often set to 8.                                  |
| sizeofBoneData    | int32   | 4            | Size of each bone record (generally 100 bytes).                 |
| sizeofFrameRecord | int32   | 4            | Size of each frame record reference (12 bytes).                 |
| sizeofRotation    | int32   | 4            | Size of a rotation entry (16 bytes).                            |
| PosOffsetsFrameLen| int32   | 4            | Possibly related to position offsets per frame (often 1).       |
| NumOffsets        | int32   | 4            | Number of offset entries following the frame/rotation data.     |
| rootOffsetsOffset | int32   | 4            | File offset to the offset entries block.                        |
| Unknown6          | int32   | 4            | Unknown usage.                                                  |
| Unknown7          | int32   | 4            | Unknown usage.                                                  |
| Unknown8          | int32   | 4            | Unknown usage.                                                  |

Total: 20 * 4 bytes = 80 bytes.

### FrameTableEntry (12 bytes per bone)

For each bone, one frame table entry references where to read its frame length array and rotation data.

| Field           | Type   | Size | Description                                     |
|-----------------|--------|------|-------------------------------------------------|
| NumFrames       | uint16 | 2    | The number of frames for this bone’s animation. |
| Padding         | uint16 | 2    | Padding or unknown usage.                       |
| frame_length_ptr| int32  | 4    | Offset in file to this bone’s frame length data.|
| quat_ptr        | int32  | 4    | Offset in file to this bone’s rotation data.    |

Total: 12 bytes.

### Frame Length Array (variable size)

- For each bone, `NumFrames` unsigned 16-bit values follow.
- Each value is presumably the length or time delta for that frame.
- Size: `NumFrames * 2` bytes per bone.

### Rotation Data (variable size)

- For each bone, `NumFrames` rotations follow after its frame length array.
- Each rotation is stored as 4 floats (x, y, z, w), each float 4 bytes, total 16 bytes per frame.
- Size: `NumFrames * 16` bytes per bone.

### OffsetEntryV1 (24 bytes per offset entry)

If `NumOffsets > 0`, these entries appear. They may store per-frame velocity or event information.

| Field  | Type  | Size | Description                     |
|--------|-------|------|---------------------------------|
| vel.x  | float | 4    | Velocity along X                |
| vel.y  | float | 4    | Velocity along Y                |
| vel.z  | float | 4    | Velocity along Z                |
| bottom | float | 4    | Possibly ground-level or event? |
| top    | float | 4    | Possibly upper-level or event?  |
| event  | int32 | 4    | Event ID or similar             |

Total: 24 bytes per offset entry.

### Bone (100 bytes per bone)

Describes a single bone in the skeleton.

| Field       | Type                 | Size  | Description                                    |
|-------------|----------------------|-------|------------------------------------------------|
| name        | char[32]            | 32    | ASCII name, null-terminated and padded.        |
| unknown1    | uint8               | 1     | Unknown usage.                                 |
| unknown2    | uint8               | 1     | Unknown usage.                                 |
| unknown3    | uint8               | 1     | Unknown usage.                                 |
| BoneNum     | uint8               | 1     | The bone’s index number.                       |
| NumChildren | int32               | 4     | Number of child bones.                         |
| ChildOffset | int32               | 4     | File offset to this bone’s first child bone.   |
| ParentOffset| int32               | 4     | File offset to this bone’s parent bone.        |
| BoneLength  | float               | 4     | Length of the bone.                            |
| Position.x  | float               | 4     | Bone’s position X (local space).               |
| Position.y  | float               | 4     | Bone’s position Y (local space).               |
| Position.z  | float               | 4     | Bone’s position Z (local space).               |
| Transform   | float[9] (3x3 mat)  | 36    | Bone’s orientation matrix (no translation row).|

Total: 100 bytes.

### Translation Frames (optional)

If `ANIM_FLAG_TRANSLATION` is set in `AnimationFlags`, translation data is included after the bone array. This data defines the per-frame positional offsets for each bone.

- Stored as `NumFrames * NumBones` sets of three floats (x, y, z).
- Each set: 3 floats * 4 bytes = 12 bytes.
- Total size: `NumFrames * NumBones * 12` bytes.

---
