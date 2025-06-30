# DNG.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.25
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The DNG (Digital Negative) module handles Adobe's open RAW format specification, processing both standard DNG metadata and proprietary Adobe DNGPrivateData containing manufacturer-specific information preserved from original RAW files. It manages complex data structures including compressed original RAW images, maker notes, and format-specific processing for various camera manufacturers.

## Module Structure

### Tag Tables (4 total)

1. **`%Image::ExifTool::DNG::OriginalRaw`** - Extracts embedded original RAW file data
2. **`%Image::ExifTool::DNG::AdobeData`** - Processes Adobe DNGPrivateData with manufacturer-specific handlers
3. **`%Image::ExifTool::DNG::ImageSeq`** - DNG 1.7 image sequence metadata
4. **`%Image::ExifTool::DNG::ProfileDynamicRange`** - DNG 1.7 dynamic range profiles

### Key Data Structures

- **OriginalRaw array structure**: 8 fixed entries (0-7) for compressed image data, resources, and file type information
- **AdobeData manufacturer map**: Format-specific handlers for CRW, MRW, SR2, RAF, Pano, Koda, Leaf formats
- **MakerNotes integration**: Dynamic copying from `Image::ExifTool::MakerNotes::Main` table
- **Compression headers**: Variable-length block structures with 65536-byte chunk boundaries

## Processing Architecture

### 1. Main Processing Flow

**OriginalRaw Processing** (`ProcessOriginalRaw`):

- Always uses big-endian byte order for pointer structures
- Handles 8 predefined data slots (images, resources, file types, creators)
- Supports Zlib compression for binary data extraction
- Optional extraction based on Binary/RequestedTags options

**AdobeData Processing** (`ProcessAdobeData`):

- Validates "Adobe\0" header signature
- Processes variable-length records with 4-byte tag + 4-byte length
- Dispatches to format-specific processors based on 4-character tags
- Handles padding for odd-length records

### 2. Special Processing Functions

**Manufacturer-Specific Processors**:

- `ProcessAdobeCRW()` - Canon CRW with entry count and reformatted structure
- `ProcessAdobeMRW()` - Minolta MRW with fake file wrapper construction
- `ProcessAdobeRAF()` - Fujifilm RAF with multiple directory support
- `ProcessAdobeSR2()` - Sony SR2 with offset adjustment calculations
- `ProcessAdobeIFD()` - Generic IFD for Panasonic/Kodak/Leaf (big-endian structure, native data order)
- `ProcessAdobeMakN()` - Maker notes with Adobe Camera Raw bug workarounds

**Write Support**:

- `WriteAdobeStuff()` - Universal write dispatcher for Adobe data structures
- Full write support for CRW format including subdirectories and value updates
- Limited write support for other formats (MRW supports structure preservation)

## Key Features

### Compression Handling

- **Zlib Integration**: Requires `Compress::Zlib` for compressed image extraction
- **Block Structure**: 65536-byte chunks with pointer tables and inflation
- **Conditional Extraction**: Only processes compressed data when explicitly requested

### Format Compatibility Matrix

| Format | Read | Write | Notes                                               |
| ------ | ---- | ----- | --------------------------------------------------- |
| CRW    | ✓    | ✓     | Full tag modification support, maker notes building |
| MRW    | ✓    | ✓     | Fake file wrapper, structure preservation           |
| SR2    | ✓    | ✗     | Offset adjustment, read-only                        |
| RAF    | ✓    | ✗     | Multiple directory support, read-only               |
| Others | ✓    | ✗     | IFD-based formats, read-only                        |

### Adobe Camera Raw Bug Mitigation

- **JPEG→DNG Conversion Bug**: Detects and handles 12-byte header offset error
- **Maker Notes Corruption**: Warns about lost external references in maker notes
- **Offset Recalculation**: Fixes base address calculations for repositioned data

### Dynamic Range Profiles (DNG 1.7)

- **Profile Versioning**: Version tracking for dynamic range capabilities
- **HDR Support**: Standard vs High dynamic range identification
- **Hint Values**: Output value recommendations for processing

## Special Patterns

### Adobe Data Block Structure

```
"Adobe\0" header (6 bytes)
Record 1: 4-byte tag + 4-byte length + data + optional padding
Record 2: 4-byte tag + 4-byte length + data + optional padding
...
```

### Maker Notes Preservation Pattern

```perl
# Dynamic table population from MakerNotes::Main
foreach $tagInfo (@Image::ExifTool::MakerNotes::Main) {
    my %copy = %$tagInfo;
    delete $copy{Groups};    # Remove group inheritance
    delete $copy{GotGroups}; # Remove processing flags
    delete $copy{Table};     # Remove table references
    push @$list, \%copy;
}
```

### Offset Fixup Pattern

```perl
# Base adjustment for repositioned maker notes
my $fix = $dataPos + $dirStart - $originalPos;
$subdirInfo{Base} += $fix;
$subdirInfo{DataPos} -= $fix;
```

## Usage Notes

### Performance Considerations

- **FastScan Optimization**: Skips maker notes processing when FastScan > 1
- **Conditional Binary Extraction**: Only extracts compressed images when needed
- **Memory Management**: Uses substr() for large data block handling

### Debugging Features

- **HtmlDump Integration**: Provides detailed hex dumps of Adobe data structures
- **Verbose Directory Listing**: Shows record structure and processing steps
- **Compression Status**: Reports Zlib inflation success/failure

### Common Pitfalls

- **Missing Zlib**: Graceful degradation with warning for compressed data
- **Corrupted Structures**: Comprehensive validation with specific error messages
- **Padding Requirements**: Automatic odd-byte padding for Adobe record alignment

## Debugging Tips

### Identifying Adobe Data Issues

```bash
# Check for Adobe DNGPrivateData
exiftool -v3 -adobedata image.dng

# Extract compressed original image
exiftool -b -originalrawimage image.dng > original.raw

# Verify maker notes preservation
exiftool -v2 -makernotes image.dng
```

### Common Error Patterns

- **"Adobe private data is corrupt"**: Usually indicates truncated or malformed Adobe section
- **"Error inflating compressed RAW image"**: Zlib decompression failure, check data integrity
- **"Maker notes could not be parsed"**: IFD structure issues in preserved maker notes
- **"Invalid DNG RAF data"**: Byte order validation failure in RAF section

### Validation Commands

```bash
# Verify DNG structure
exiftool -validate image.dng

# Check for format-specific issues
exiftool -G1 -a -s image.dng | grep -E "(CRW|MRW|SR2|RAF)"
```
