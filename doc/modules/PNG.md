# PNG.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.71
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

PNG.pm handles reading and writing metadata in PNG (Portable Network Graphics), MNG (Multi-image Network Graphics), and JNG (JPEG Network Graphics) images. The module processes PNG chunks containing metadata including text chunks, embedded profiles, and format-specific image parameters. It features complex metadata positioning logic to ensure compatibility with applications that require XMP to appear before image data.

## Module Structure

### Tag Tables (8 total)

1. **Main** - Primary PNG chunk definitions with 40+ chunk types
2. **ImageHeader** - IHDR chunk data (dimensions, color type, compression)
3. **PrimaryChromaticities** - cHRM chunk (8 chromaticity values)
4. **PhysicalPixel** - pHYs chunk (resolution and units)
5. **CICodePoints** - cICP chunk (color space information)
6. **SubjectScale** - sCAL chunk (physical scale information)
7. **VirtualPage** - vpAg chunk (ImageMagick virtual page)
8. **StereoImage** - sTER chunk (stereoscopic image mode)
9. **TextualData** - tEXt/zTXt/iTXt chunks with extensive metadata support
10. **AnimationControl** - acTL chunk (APNG animation parameters)

### Key Data Structures

- **%pngLookup** - File signature to format mapping (PNG/MNG/JNG detection)
- **%pngMap** - Directory hierarchy mapping for write operations
- **%stdCase** - Case correction for historical chunk name variations
- **%isDatChunk** - Image data chunk identification (IDAT, JDAT, JDAA)
- **%isTxtChunk** - Text chunk identification (tEXt, zTXt, iTXt, eXIf)
- **%noLeapFrog** - Chunks that text chunks cannot be moved across
- **Global $colorType** - Current image color type for format decisions

## Processing Architecture

### 1. Main Processing Flow

**ProcessPNG()** - Primary entry point (lines 1410-1685):

- Signature validation and file type detection
- Two-pass processing for metadata positioning optimization
- Chunk-by-chunk sequential parsing with CRC validation
- Text chunk relocation logic for application compatibility
- Samsung trailer detection for mobile images

### 2. Special Processing Functions

- **ProcessPNG_tEXt()** - Uncompressed text chunks with Latin-1 encoding
- **ProcessPNG_iTXt()** - International text with UTF-8 and language codes
- **ProcessPNG_Compressed()** - zTXt and iCCP decompression using Compress::Zlib
- **ProcessPNG_eXIf()** - EXIF/zXIf chunk processing with TIFF structure
- **ProcessProfile()** - ImageMagick hex-encoded raw profiles (EXIF/ICC/IPTC/XMP)
- **FoundPNG()** - Universal PNG metadata extraction and writing handler

### 3. Text Chunk Processing Pipeline

1. **Encoding Detection**: Latin-1 (tEXt/zTXt) vs UTF-8 (iTXt)
2. **Decompression**: Automatic deflate decompression when Compress::Zlib available
3. **Language Handling**: RFC 3066 language codes with case normalization
4. **Profile Decoding**: Hex-to-binary conversion for embedded profiles
5. **Subdirectory Processing**: Recursive handling of EXIF/XMP/ICC/Photoshop data

## Key Features

### 1. Advanced Text Chunk Management

- **Language Support**: Full RFC 3066 language code handling with alternate language tags
- **Profile Embedding**: Support for hex-encoded EXIF, ICC, IPTC, XMP, and Photoshop profiles
- **Compression Options**: Automatic zTXt compression with fallback to tEXt
- **Character Encoding**: Proper Latin-1/UTF-8 handling with encoding conversion

### 2. Metadata Positioning Intelligence

- **Pre-IDAT Movement**: Text chunks relocated before image data for compatibility
- **Apple/Adobe Compatibility**: Special handling for Spotlight, Preview, and Photoshop
- **Two-Pass Algorithm**: Scan-ahead to identify text chunks after IDAT for relocation
- **NoLeapFrog Protection**: Respects MNG/JNG chunks that cannot be crossed

### 3. Profile Processing Capabilities

- **iCCP Handling**: Native ICC profile chunks with profile name support
- **Raw Profile Support**: ImageMagick-style hex-encoded profiles in text chunks
- **Multi-Format Support**: EXIF (APP1/TIFF), XMP, IPTC, ICC, Photoshop IRB
- **Validation**: Profile size verification and format detection

### 4. Specialized Format Support

- **APNG Animation**: acTL chunk processing with frame count and loop detection
- **Apple Extensions**: iDOT chunk support for Apple's multi-resolution images
- **Gain Map Support**: gdAT chunk for HDR gain map images
- **C2PA Support**: caBX chunk for content authenticity metadata

## Special Patterns

### 1. Chunk Case Sensitivity Management

PNG specification inconsistencies led to case variations in eXIf/zXIf chunks. Module maintains %stdCase lookup for historical compatibility while preferring current standard casing.

### 2. Compression Fallback Strategy

```perl
if (eval { require Compress::Zlib }) {
    # Use zTXt compression
} else {
    # Fall back to tEXt
    $noCompressLib = 1;
}
```

### 3. CRC Validation Modes

- **Reading**: Optional validation with Verbose/Validate options
- **Writing**: Mandatory validation unless FastScan bypasses
- **Error Handling**: Graceful degradation for corrupted chunks

### 4. Metadata Inheritance Pattern

Uses %pngMap to establish write hierarchy: IFD0→PNG, XMP→PNG, ensuring proper metadata organization during write operations.

## Usage Notes

### PNG Write Complexity Warning

The module documentation notes that "Writing meta information in PNG images is a pain in the butt" due to:

- Required decompression/recompression cycles
- ASCII/hex encoding transformations
- CRC recalculation requirements
- Application compatibility constraints

### Text Chunk Location Strategy

As of ExifTool 11.58+, implements two-pass writing to ensure text chunks appear before IDAT while maintaining compatibility with XMP-after-IDAT specifications.

### Profile Name Handling

iCCP profile names are managed through a special 'iCCP-name' fake tag that can only be set when writing the ICC_Profile, ensuring proper chunk structure.

## Debugging Tips

### Common Issues

1. **Compress::Zlib Missing**: Module gracefully degrades but warns about inability to process compressed chunks
2. **Text After IDAT**: Warnings issued for text chunks found after image data with fix/ignore guidance
3. **Invalid CRC**: Bad checksums trigger errors in write mode, warnings in read mode
4. **Non-standard Headers**: Apple iPhone CgBI chunks and other deviations detected with warnings

### Diagnostic Options

- **Verbose Mode**: Detailed chunk processing with size and CRC information
- **Validate Mode**: Enhanced CRC checking and format validation
- **HtmlDump**: Available for binary chunk structure visualization

### Profile Troubleshooting

- Check profile size matches declared length
- Verify hex encoding validity for raw profiles
- Ensure proper profile headers for format detection
- Monitor compression success/failure messages

## Trailer and Extension Support

### Samsung Mobile Extensions

Automatic detection and processing of Samsung SEFT trailer data in mobile images, providing access to proprietary metadata.

### Format Variants

- **PNG Plus**: OLE cpIp chunk detection triggers FileType change
- **APNG**: acTL chunk detection overrides FileType to "APNG"
- **iPhone PNG**: CgBI chunk detection with non-standard warning

The module's sophisticated handling of PNG's flexible chunk architecture makes it one of ExifTool's most complex format processors, balancing specification compliance with real-world application compatibility.
