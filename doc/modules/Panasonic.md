# Panasonic.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 2.24
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The Panasonic.pm module handles maker notes for both Panasonic and Leica cameras, providing comprehensive metadata extraction and writing capabilities. This dual-brand module is one of the most complex in ExifTool, supporting traditional photography, video formats, and specialized Leica rangefinder systems.

## Module Structure

### Tag Tables (20 total)

**Primary Tables:**

- **Main** (line 265) - Primary Panasonic maker notes with 100+ tags
- **Leica2-9** (lines 1566-2155) - 8 specialized Leica tag tables for different camera models
- **Type2** (line 2221) - Binary data tags for newer models

**Specialized Tables:**

- **FaceDetInfo/FaceRecInfo** (lines 2241, 2294) - Face detection and recognition data
- **PANA** (line 2362) - MP4 video metadata extraction
- **SerialInfo/TimeInfo** (lines 1686, 1901) - Binary data structures
- **Composite** (line 2563) - Derived tags combining multiple values

### Key Data Structures

**Lens Identification System:**

- **leicaLensTypes** hash (line 46) - 133+ Leica lens codes with model numbers
- **frameSelectorBits** hash (line 139) - Frame selector positions for M9 lens identification
- Complex lens code decoding using bit manipulation and multiple lookup stages

**Mode Conversion Tables:**

- **shootingMode** hash (line 181) - 90+ shooting mode definitions
- Extensive PrintConv tables for camera settings and scene modes

## Processing Architecture

### 1. Main Processing Flow

**Standard EXIF Processing:**

- Uses `WriteExif`/`CheckExif` procedures for most tag tables
- Supports full read/write operations with validation
- Maintains EXIF compliance while handling proprietary extensions

**Binary Data Processing:**

- 10 tag tables use `ProcessBinaryData` for complex binary structures
- Handles packed data formats with precise offset management
- Supports variable-length records and conditional processing

### 2. Special Processing Functions

**ProcessLeicaLEIC** (line 2709):

- Processes Leica maker notes in MP4 video files
- Handles LEIC atom restructuring for Leica T and X Vario
- Manages TIFF IFD reorganization within video containers

**ProcessLeicaTrailer** (line 2746):

- Processes complex MakerNote trailers from Leica S2
- Implements validation and multiple-call handling
- Manages trailer-specific offset calculations

**WhiteBalanceConv** (line 2694):

- Bidirectional Kelvin white balance conversion
- Handles Leica-specific color temperature encoding

**LensTypeConvInv** (line 2676):

- Inverse conversion for Leica M9 lens identification
- Manages complex lens code bit manipulation

## Key Features

### Dual Brand Architecture

- Seamless handling of both Panasonic and Leica formats
- Brand-specific tag tables with appropriate groupings
- Unified processing while maintaining manufacturer distinctions

### Sophisticated Lens Identification

- Multi-stage lens detection using primary codes and frame selector bits
- Support for zoom lenses with focal length detection
- Manual coding support for uncoded lenses on M9 systems

### Video Metadata Support

- PANA atom processing for MP4 video files
- Extraction of camera settings from video metadata
- Support for motion picture recording parameters

### Face Detection Systems

- Binary parsing of face detection coordinates
- Face recognition data with multiple subject support
- Position data relative to frame dimensions

### Advanced Binary Processing

- Time information structures with complex encoding
- Serial number parsing with manufacturing date extraction
- Shot information with exposure and focus data

## Special Patterns

### Leica M9 Lens Encoding

```perl
# Lens codes split into two parts: main ID and lower 2 bits
'42 1' => 'Tri-Elmar-M 28-35-50mm f/4 ASPH. (at 28mm)',
'42 2' => 'Tri-Elmar-M 28-35-50mm f/4 ASPH. (at 35mm)',
'42 3' => 'Tri-Elmar-M 28-35-50mm f/4 ASPH. (at 50mm)',
```

### Binary Data Structures

- Extensive use of `ProcessBinaryData` with format specifications
- Conditional processing based on camera models
- DATAMEMBER usage for cross-referencing between tags

### Video Container Processing

- MP4 atom navigation and TIFF extraction
- Specialized handling for video-specific metadata
- Integration with QuickTime processing framework

## Usage Notes

### Writing Capabilities

- Full write support for most Panasonic tags
- Selective write support for Leica tags
- Binary data preservation during editing operations

### Model-Specific Behavior

- Extensive conditional processing based on camera models
- Different decoding for various firmware versions
- Special handling for discontinued and current product lines

### Lens Database Maintenance

- Comprehensive lens code database requiring updates for new lenses
- Frame selector bit mappings must be maintained in sync
- Model number cross-references need periodic validation

## Debugging Tips

### Lens Identification Issues

- Check both main lens code and frame selector bits
- Verify lens database completeness for new models
- Consider manual coding scenarios for uncoded lenses

### Binary Data Problems

- Use `-v` flag to examine binary data structures
- Check for model-specific format variations
- Validate offset calculations in complex structures

### Video Metadata Extraction

- Ensure MP4 container support is enabled
- Check for PANA atom presence in video files
- Verify TIFF structure within video containers

### Leica Trailer Processing

- Complex trailer validation may fail with corrupted files
- Multiple processing passes may be required
- Check for S2-specific trailer structures
