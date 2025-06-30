# BMP.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 1.09  
**Document Version:** 1.0  
**Last Updated:** 2025-06-29

## Overview

The BMP module reads metadata from Windows Bitmap (BMP) files. It's a simple format module that demonstrates basic ExifTool patterns for fixed-structure binary formats.

## Module Structure

### Tag Tables

1. **%Image::ExifTool::BMP::Main** (lines 36-212)

   - Primary table for Windows V3/V4/V5 BMP headers
   - Uses ProcessBinaryData for fixed-offset tag extraction
   - Contains 23 tags covering image dimensions, color info, and profiles

2. **%Image::ExifTool::BMP::OS2** (lines 215-232)

   - Handles older OS/2 format BMP headers (12 or 64 bytes)
   - Simplified structure with only 5 basic tags

3. **%Image::ExifTool::BMP::Extra** (lines 234-248)
   - Handles additional extracted data (embedded images, profiles)
   - Not part of the BMP header structure

### Key Functions

**ProcessBMP()** (lines 254-317)

- Main entry point called by ExifTool
- Validates BMP signature and header size
- Determines BMP version from header size
- Extracts embedded JPEG/PNG images if present
- Processes ICC profiles for V5 BMPs

## Processing Flow

1. **File Validation**

   - Check "BM" signature at start
   - Read header size at offset 14
   - Validate header size (12, 16, 40-124 bytes)

2. **Version Detection**

   - Header size determines BMP version:
     - 12/16/64: OS/2 format
     - 40: Windows V3
     - 108: Windows V4
     - 124: Windows V5

3. **Tag Extraction**

   - ProcessBinaryData handles fixed-offset tags
   - Special handling for conditional tags (ColorSpace-dependent)
   - DataMember tags store values for later use

4. **Additional Processing**
   - Extract embedded JPEG (compression=4) or PNG (compression=5)
   - Process linked/embedded color profiles (V5 only)

## Special Features

### Hook Usage (line 119)

```perl
Hook => '$varSize += $size if $$self{BMPVersion} == 68'
```

Adjusts processing for AVI BMP structures which have different layouts.

### Conditional Tags (lines 157-188)

Multiple tags depend on ColorSpace value:

```perl
Condition => '$$self{BMPColorSpace} eq "0"'
```

### Fixed-Point Conversion (lines 22-33)

Handles 2.30 fixed-point format for color endpoints.

### Profile Handling (lines 299-315)

- LinkedProfileName: Null-terminated Latin-encoded string
- ICC_Profile: Binary ICC profile data
- 14-byte offset adjustment for profile location

## Common Patterns Demonstrated

1. **Binary Format Module**

   - Fixed tag offsets
   - ProcessBinaryData usage
   - Format specifications for each tag

2. **Version Handling**

   - Multiple tag tables for format variants
   - Version detection from structure size

3. **DataMember Pattern**

   - Store values during extraction for later use
   - BMPVersion, BMPCompression, BMPColorSpace, etc.

4. **Embedded Data Extraction**
   - ExtractBinary() for embedded images
   - Seek/Read for ICC profiles

## Usage Notes

- Simple format with minimal metadata
- No writing support (read-only module)
- Handles both Windows and OS/2 variants
- Good example of ProcessBinaryData usage
