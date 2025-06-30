# HP.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.04
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The HP.pm module handles Hewlett-Packard digital camera maker notes across multiple proprietary formats. This is a moderately complex manufacturer module supporting at least 6 different format types across various PhotoSmart camera models. The module demonstrates HP's evolution from EXIF-standard maker notes to highly proprietary formats, including a unique TDHD (hierarchical directory) format found in APP6 JPEG segments.

## Module Structure

### Tag Tables (5 total)

1. **Main** (`%Image::ExifTool::HP::Main`):

   - EXIF-format maker notes (PhotoSmart 720, Vivitar ViviCam)
   - Standard EXIF processing
   - PrintIM (Print Image Matching) support

2. **Type2** (`%Image::ExifTool::HP::Type2`):

   - Text-based format (PhotoSmart E427)
   - Custom ProcessHP function
   - Key-value text parsing

3. **Type4** (`%Image::ExifTool::HP::Type4`):

   - Binary format (PhotoSmart M627)
   - Structured binary data with known offsets
   - Multiple data formats (int16u, int32u, string)

4. **Type6** (`%Image::ExifTool::HP::Type6`):

   - Binary format (PhotoSmart M425, M525, M527)
   - Similar to Type4 with different field layout
   - Camera exposure and technical data

5. **TDHD** (`%Image::ExifTool::HP::TDHD`):
   - Proprietary hierarchical format (PhotoSmart R837)
   - APP6 JPEG segment processing
   - Recursive subdirectory structure

### Key Data Structures

- **TDHD Format**: 12-byte headers (4-byte tag + 4-byte type + 4-byte size)
- **Binary Formats**: Fixed offset structures with known field locations
- **Serial Number Processing**: Prefix stripping pattern ("SERIAL NUMBER:" removal)
- **Format Detection**: Type field (0x10001) for subdirectory identification

## Processing Architecture

### 1. Main Processing Flow

**EXIF Format (Main table):**

- Standard maker note processing
- PrintIM subdirectory handling

**Text Format (Type2 table):**

- ProcessHP function for text scanning
- Pattern matching for key-value pairs
- Brute-force preview image extraction

**Binary Formats (Type4/Type6 tables):**

- ProcessBinaryData with known field offsets
- Fixed format structures
- Data type conversion (scaling, time formatting)

**TDHD Format (TDHD table):**

- ProcessTDHD for hierarchical processing
- Recursive subdirectory parsing
- Dynamic tag creation for unknowns

### 2. Special Processing Functions

**ProcessTDHD** (`HP.pm:143-198`):

- Processes APP6 TDHD metadata segments
- Little-endian byte order (SetByteOrder('II'))
- 12-byte entry headers (tag + type + size)
- Recursive subdirectory processing (type 0x10001)
- Dynamic unknown tag creation with format guessing
- Binary flag for large data blocks (>80 bytes)

**ProcessHP** (`HP.pm:204-232`):

- Text-based maker note processing
- Brute-force JPEG preview extraction using regex `/(\xff\xd8\xff\xdb.*\xff\xd9)/gs`
- Text pattern matching for tag-value pairs
- Preview truncation for performance optimization

## Key Features

1. **Multi-Format Support**: 5+ distinct format types across camera generations
2. **TDHD Hierarchical Processing**: Unique recursive directory structure
3. **Preview Image Extraction**: Brute-force JPEG scanning and extraction
4. **Serial Number Normalization**: Consistent prefix stripping across formats
5. **Format Guessing**: Automatic data type detection based on size
6. **Performance Optimization**: Preview truncation to speed tag scanning
7. **Cross-Brand Support**: Vivitar cameras using HP format

## Special Patterns

1. **Serial Number Processing**: Pattern `s/^SERIAL NUMBER://` for normalization
2. **Preview Extraction**: JPEG signature scanning with performance optimization
3. **Format Evolution**: Progression from EXIF → Text → Binary → TDHD formats
4. **Binary Field Layout**: Different offset schemes (Type4 vs Type6) for similar data
5. **Hierarchical Metadata**: TDHD subdirectories with type-based identification
6. **Dynamic Tag Creation**: Unknown tag handling with format guessing algorithms

## Usage Notes

- HP cameras show significant format variation across model generations
- TDHD format is unique to PhotoSmart R837 (possibly other R-series models)
- Text format requires exact tag name matching (case-insensitive)
- Binary formats have fixed field layouts - offsets are model-specific
- Preview images are embedded as complete JPEG streams
- Unknown TDHD tags are automatically created when `-u` option is used

## Debugging Tips

1. **Format Identification**: Check maker note signature to determine which table applies
2. **TDHD Issues**: Verify APP6 segment presence and proper 12-byte header alignment
3. **Preview Problems**: Use verbose mode to see preview extraction process
4. **Serial Number**: Check for "SERIAL NUMBER:" prefix in raw data
5. **Binary Format**: Validate offset calculations match expected data types
6. **Unknown Tags**: Enable unknown tag extraction (`-u`) for TDHD format exploration
7. **Byte Order**: TDHD uses little-endian consistently

## Implementation Details

**File Location**: `lib/Image/ExifTool/HP.pm`
**Dependencies**: Image::ExifTool::ProcessBinaryData, Image::ExifTool::PrintIM
**Line Count**: 264 lines (medium complexity)
**Complexity**: Medium-High - multiple format types with proprietary TDHD processing
**Maintenance**: Active - multiple camera model support requires ongoing updates
