# RICOH Signature Detection in MakerNotes

This document explains the exact conditions and flow for RICOH maker note detection in ExifTool, particularly focusing on the case where Make is "PENTAX RICOH" and the data starts with TIFF magic bytes.

## Overview

ExifTool has three different RICOH maker note detection patterns in MakerNotes.pm (lines 873-924), each handling different variations of RICOH camera formats.

## Detection Flow

### 1. MakerNoteRicohPentax (lines 873-882)

**Condition**: `$$valPt=~/^RICOH\0(II|MM)/`

This handles cameras like the Ricoh GR III that have a clear "RICOH\0" signature followed by TIFF byte order markers.

- **SubDirectory**: Uses `Pentax::Main` table (not Ricoh!)
- **Start**: `$valuePtr + 8` (skips "RICOH\0" + byte order)
- **Base**: `$start - 8` (compensates for the 8-byte skip)
- **ByteOrder**: 'Unknown' (will be determined from the II/MM markers)

### 2. MakerNoteRicoh (lines 884-897) - The Complex Case

**Condition** (multi-part):
```perl
$$self{Make} =~ /^(PENTAX )?RICOH/ and
$$valPt =~ /^(Ricoh|      |MM\0\x2a|II\x2a\0)/i and
$$valPt !~ /^(MM\0\x2a\0\0\0\x08\0.\0\0|II\x2a\0\x08\0\0\0.\0\0\0)/s and
$$self{Model} ne 'RICOH WG-M1'
```

This is the most complex detection, handling multiple RICOH formats:

1. **Make must match**: "RICOH" or "PENTAX RICOH"
2. **Data must start with one of**:
   - "Ricoh" (case-insensitive text signature)
   - "      " (6 spaces - seen in R50)
   - "MM\0\x2a" (Big-endian TIFF magic)
   - "II\x2a\0" (Little-endian TIFF magic)
3. **Data must NOT match**: Specific byte patterns that indicate Type2 format
4. **Model must NOT be**: "RICOH WG-M1" (handled by Type2)

**Processing**:
- **SubDirectory**: Uses `Ricoh::Main` table
- **Start**: `$valuePtr + 8` (skips 8 bytes regardless of signature)
- **ByteOrder**: 'Unknown' (auto-detected)

**Key Insight**: When the data starts with TIFF magic bytes (MM\0\x2a or II\x2a\0), ExifTool still skips 8 bytes. This suggests RICOH cameras may have 8 bytes of padding or header even when using standard TIFF format.

### 3. MakerNoteRicoh2 (lines 899-915)

**Condition**:
```perl
$$self{Make} =~ /^(PENTAX )?RICOH/ and ($$self{Model} eq 'RICOH WG-M1' or
$$valPt =~ /^(MM\0\x2a\0\0\0\x08\0.\0\0|II\x2a\0\x08\0\0\0.\0\0\0)/s)
```

This handles specific models (WG-M1) and TIFF-like data with extra null padding after the IFD entry count.

- **SubDirectory**: Uses `Ricoh::Type2` table
- **ProcessProc**: Uses `ProcessKodakPatch` (handles extra null bytes)
- **Base**: `$start - 8` (different base calculation)

### 4. MakerNoteRicohText (lines 917-924)

**Condition**: `$$self{Make}=~/^RICOH/`

This is a fallback for any RICOH camera not caught by previous conditions.

- **NotIFD**: 1 (indicates text-based format, not binary IFD)
- **SubDirectory**: Uses `Ricoh::Text` table

## Critical Implementation Details

### When Make is "PENTAX RICOH" and Data Starts with TIFF Magic

For the specific case mentioned, the flow is:

1. **First check**: MakerNoteRicohPentax condition fails (no "RICOH\0" prefix)
2. **Second check**: MakerNoteRicoh condition succeeds:
   - Make matches `/^(PENTAX )?RICOH/` ✓
   - Data matches `/^(MM\0\x2a|II\x2a\0)/` ✓
   - Data doesn't match the exclusion pattern ✓
   - Model isn't "RICOH WG-M1" ✓
3. **Processing**: Uses Ricoh::Main with 8-byte offset

### The 8-Byte Skip Mystery

The comment on line 885 provides a clue: "my test R50 image starts with '      \x02\x01'". This suggests RICOH cameras may use various 8-byte headers:
- 6 spaces + 2 bytes of data
- TIFF magic + 4 bytes of padding
- "Ricoh" text + padding

The consistent 8-byte skip handles all these variations uniformly.

### ByteOrder Detection

When ByteOrder is 'Unknown', ExifTool examines the data to determine endianness:
- For TIFF magic bytes, it's obvious from II (little) or MM (big)
- For other formats, it uses heuristics or defaults

## Implications for Implementation

1. **Order Matters**: The conditions are evaluated sequentially, so more specific patterns must come before general ones
2. **Negative Conditions**: The exclusion patterns prevent misidentification of Type2 format
3. **Flexible Headers**: RICOH uses multiple header formats, requiring flexible detection
4. **Table Choice**: Despite being RICOH cameras, some use Pentax::Main table (due to company merger/acquisition history)

## Historical Context

RICOH's acquisition of Pentax explains the mixed branding and table usage:
- Older RICOH cameras: Use Ricoh tables
- Pentax-era cameras: May use Pentax tables despite RICOH branding
- Modern cameras: Can use either depending on firmware heritage