# JVC.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.04
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The JVC.pm module handles JVC (Japan Victor Company) camera maker notes in two distinct formats: standard EXIF-based binary maker notes and proprietary text-based maker notes. This is a relatively simple manufacturer module with minimal complexity, supporting basic camera information extraction from JVC digital cameras and camcorders.

## Module Structure

### Tag Tables (2 total)

1. **Main** (`%Image::ExifTool::JVC::Main`):

   - EXIF-based binary maker notes
   - Uses standard EXIF write/check procedures
   - 3 defined tags (0x0002, 0x0003, plus undocumented 0x0001)
   - Standard MakerNotes group classification

2. **Text** (`%Image::ExifTool::JVC::Text`):
   - Text-based maker notes with custom processing
   - Uses `ProcessJVCText` function
   - 2 defined tags (VER, QTY) plus dynamic unknown tag support
   - TAG:VALUE text format parsing

### Key Data Structures

- **Tag ID formats**: Hexadecimal (0x0002, 0x0003) for binary format, string keys (VER, QTY) for text format
- **Value processing**: Custom CPU version string cleanup, quality level mapping
- **Format validation**: Text format must start with "VER:" prefix

## Processing Architecture

### 1. Main Processing Flow

**Binary Format (Main table):**

- Standard EXIF maker note processing
- Uses Image::ExifTool::Exif::WriteExif and CheckExif
- Direct tag lookup by hexadecimal ID

**Text Format (Text table):**

- Custom ProcessJVCText function (`JVC.pm:64-99`)
- Format validation against "VER:" prefix
- Regex-based TAG:VALUE extraction
- Dynamic tag creation for unknown entries

### 2. Special Processing Functions

**ProcessJVCText** (`JVC.pm:64-99`):

- Validates text maker notes format (must start with "VER:")
- Extracts TAG:VALUE pairs using regex `m/([A-Z]+):(.{3,4})/sg`
- Creates unknown tags dynamically with "JVC*Text*" prefix
- Handles verbose output and error reporting

## Key Features

1. **Dual Format Support**: Handles both binary EXIF and text-based maker notes
2. **CPU Version Processing**: Custom string cleanup for CPUVersions tag (removes trailing nulls/spaces, splits on nulls)
3. **Quality Level Mapping**: Standardizes quality values across different representations
4. **Dynamic Tag Creation**: Unknown text tags automatically added with consistent naming
5. **Format Validation**: Strict validation for text format integrity

## Special Patterns

1. **Text Format Parsing**: Uses regex with 3-4 character value extraction
2. **String Sanitization**: Custom ValueConv for CPU version string cleanup
3. **Quality Standardization**: Multiple representations (0/1/2 and STND/STD/FINE) mapped to consistent output
4. **Error Handling**: Graceful degradation with warning for malformed text maker notes

## Usage Notes

- JVC cameras may use either format depending on model and firmware
- Text format is specific to JVC/Victor brand cameras
- CPU version strings contain multiple version numbers separated by nulls
- Quality settings use different representations across camera models
- Module automatically handles unknown tags in text format

## Debugging Tips

1. **Text Format Issues**: Check if maker notes start with "VER:" - if not, format validation will fail
2. **Missing Tags**: Enable unknown tag extraction (`-u` option) to see all text-based tags
3. **CPU Version Problems**: Check for embedded nulls or spaces in version strings
4. **Quality Mapping**: Verify quality values match expected 0/1/2 or STND/STD/FINE patterns
5. **Verbose Mode**: Use `-v` option to see detailed text parsing process

## Implementation Details

**File Location**: `lib/Image/ExifTool/JVC.pm`
**Dependencies**: Image::ExifTool::Exif (for binary format processing)
**Line Count**: 131 lines (very compact module)
**Complexity**: Low - simple dual-format manufacturer module
**Maintenance**: Stable - minimal changes expected
