# PanasonicRaw.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.29
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The PanasonicRaw.pm module handles RAW/RW2/RWL image formats from Panasonic and Leica cameras. It specializes in parsing proprietary RAW structures, embedded JPEG previews, and RAW-specific metadata that differs significantly from standard TIFF/EXIF formats.

## Module Structure

### Tag Tables (8 total)

**Primary Tables:**

- **Main** (line 70) - Core RAW metadata with sensor and processing parameters
- **WBInfo/WBInfo2** (lines 384, 410) - White balance information structures
- **DistortionInfo** (line 436) - Lens distortion correction parameters
- **CameraIFD** (line 496) - Camera-specific settings subdirectory

**Specialized Tables:**

- **Composite** (line 674) - Calculated image dimensions from sensor borders

### Key Data Structures

**White Balance Processing:**

- **jpgFromRawMap** hash (line 31) - IFD mapping for embedded JPEG processing
- **wbTypeInfo** hash (line 46) - White balance type conversion structure
- **panasonicWhiteBalance** hash (line 51) - White balance mode definitions

**File Format Support:**

- RAW format (original Panasonic RAW)
- RW2 format (RAW version 2)
- RWL format (RAW Large, same as RW2)

## Processing Architecture

### 1. Main Processing Flow

**TIFF-Based Structure:**

- Uses standard EXIF WriteExif/CheckExif procedures
- Maintains TIFF IFD structure with proprietary tags
- Supports full read/write operations with proper validation

**RAW Data Handling:**

- Custom offset patching for non-standard RAW data placement
- Specialized strip offset management for large RAW files
- Support for multiple RAW format variants

### 2. Special Processing Functions

**ProcessDistortionInfo** (line 714):

- Reads lens distortion correction parameters
- Implements checksum validation for data integrity
- Handles 32-byte binary structure with multiple checksums

**WriteDistortionInfo** (line 741):

- Writes distortion parameters with automatic checksum recalculation
- Ensures data integrity through multiple checksum algorithms
- Validates structure size and format consistency

**PatchRawDataOffset** (line 770):

- Handles non-standard RAW data offset schemes
- Supports multiple offset tag combinations (StripOffsets, RawDataOffset)
- Implements GH6 fixed-offset hack for newer models
- Calculates RAW data length from file size

**ProcessJpgFromRaw** (line 872):

- Extracts metadata from embedded JPEG previews
- Maintains separate processing context for JPEG data
- Preserves original RAW context during JPEG processing

**WriteJpgFromRaw** (line 831):

- Writes metadata back to embedded JPEG previews
- Uses specialized IFD mapping to prevent unwanted metadata inclusion
- Maintains preview image integrity during metadata updates

### 3. Checksum Implementation

**Distortion Checksum Algorithm** (line 699):

- Custom checksum: `(73 * csum + byte_value) % 0xffef`
- Multiple checksum validation (4 separate checksums)
- Ensures data integrity for lens correction parameters

## Key Features

### RAW Format Support

- **Format Detection**: Automatic detection of RAW/RW2/RWL variants
- **Compression Handling**: Support for multiple compression schemes (34316, 34826, 34828, 34830)
- **Pixel Shift Support**: Specialized handling for high-resolution pixel shift modes

### Sensor Information

- **Border Definition**: Precise sensor active area definition
- **CFA Pattern**: Color Filter Array pattern identification
- **Raw Format**: Compression and processing format identification

### White Balance Systems

- **Dual WB Structures**: Support for early (WBInfo) and later (WBInfo2) formats
- **Multiple WB Settings**: Up to 7 white balance presets per image
- **RGB/RB Level Support**: Format-dependent color level storage

### Embedded JPEG Processing

- **Full EXIF Support**: Complete metadata extraction from embedded previews
- **Selective Writing**: Controlled metadata writing to prevent bloat
- **Context Preservation**: Maintains RAW processing context during JPEG operations

### Lens Distortion Correction

- **Parameter Storage**: Comprehensive distortion correction coefficients
- **Validation System**: Multiple checksum verification for data integrity
- **Scale Factors**: Distortion scale and parameter normalization

## Special Patterns

### RAW Data Offset Management

```perl
# Multiple offset tag support with fallback handling
if ($rawDataOffset) {
    # Handle via RawDataOffset instead of StripOffsets
    $stripOffsets = $$offsetInfo{0x111} = $rawDataOffset;
    delete $$offsetInfo{0x118};
}
```

### Checksum Validation

```perl
# Four separate checksums for distortion data integrity
my $csum1 = Checksum($dataPt, $start +  4, 12, 1);
my $csum2 = Checksum($dataPt, $start + 16, 12, 1);
my $csum3 = Checksum($dataPt, $start +  2, 14, 2);
my $csum4 = Checksum($dataPt, $start +  3, 14, 2);
```

### Embedded JPEG Context Switching

```perl
# Temporary context modification for JPEG processing
$$et{FILE_TYPE} = $$et{TIFF_TYPE} = 'JPEG';
my $result = $et->ProcessJPEG(\%dirInfo);
$$et{FILE_TYPE} = 'TIFF';
$$et{TIFF_TYPE} = $fileType;
```

## Usage Notes

### File Format Compatibility

- **DNG Conversion**: Handles DNG-converted RW2/RWL files with adjusted offsets
- **RAW Variants**: Automatic format detection and appropriate processing
- **Preview Extraction**: Embedded JPEG can be extracted as separate image

### Writing Limitations

- **Protected Tags**: Many tags marked as Protected to prevent corruption
- **Format Constraints**: Some tags only writable in specific formats
- **Checksum Maintenance**: Automatic checksum recalculation during writes

### Camera Model Support

- **Wide Compatibility**: Supports models from early Digilux to latest Lumix cameras
- **Format Evolution**: Handles changes in RAW format across camera generations
- **Special Cases**: Model-specific handling for unique implementations

## Debugging Tips

### RAW Data Issues

- **Offset Problems**: Check StripOffsets vs RawDataOffset tag usage
- **Size Validation**: Verify RAW data length calculations
- **Format Detection**: Confirm RAW format identification accuracy

### Distortion Data Problems

- **Checksum Failures**: Validate all four checksum calculations
- **Parameter Range**: Check distortion parameter value ranges
- **Data Size**: Ensure 32-byte structure size consistency

### Embedded JPEG Issues

- **Context Problems**: Verify proper context switching during processing
- **Metadata Bloat**: Check for unwanted metadata addition during writes
- **Preview Corruption**: Validate JPEG integrity after metadata updates

### White Balance Anomalies

- **Format Version**: Distinguish between WBInfo and WBInfo2 structures
- **Level Scaling**: Verify RGB vs RB level format handling
- **Entry Count**: Validate NumWBEntries consistency
