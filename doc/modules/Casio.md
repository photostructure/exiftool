# Casio.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.38
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The Casio.pm module handles EXIF maker notes tags for Casio digital cameras, supporting both older and newer maker note formats. It provides comprehensive metadata extraction for a wide range of Casio camera models from the QV series to modern EX series cameras, including digital cameras, camcorders, and specialized models.

## Module Structure

### Tag Tables (6 total)

1. **Main** - Legacy Casio maker notes format (older QV series cameras)
   - 18 primary tags covering basic camera settings
   - Includes PrintIM support for older models
   - EntryBased offset handling for specific tags

2. **Type2** - Modern Casio maker notes format (EX series and newer)
   - 67+ tags with extensive feature coverage
   - Preview image handling with offset pairs
   - Complex model-specific BestShotMode mappings
   - Video metadata support

3. **FaceInfo1** - Face detection data format for EX-H5 generation
   - Binary data processing for up to 10 faces
   - Coordinate system with frame size reference
   - BigEndian byte order processing

4. **FaceInfo2** - Face detection data format for EX-ZR100/H20G generation
   - Updated binary format with orientation support
   - LittleEndian byte order processing
   - Enhanced face position tracking

5. **QVCI** - Special APP1 segment for QV-7000SX model
   - Quality settings and focal range information
   - Manufacture codes and model type identification
   - Date/time conversion with special formatting

6. **AVI** - Video-specific tags for Casio GV-10 camcorder
   - Software identification for AVI format videos

### Key Data Structures

- **Conditional Tag Arrays**: Extensive use of conditional tag definitions for model-specific behavior
- **BestShotMode Mappings**: Large nested conditional structures for 20+ camera models
- **Face Detection Coordinates**: Binary data structures for face position tracking
- **Preview Image Pairs**: Offset/length pairs for embedded preview images
- **Firmware Date Parsing**: Custom conversion functions for null-terminated date strings

## Processing Architecture

### 1. Main Processing Flow

The module uses standard ExifTool processing with model-specific conditional logic:

1. **Format Detection**: Distinguishes between Main (older) and Type2 (newer) formats
2. **Model Identification**: Uses `$$self{Model}` for conditional tag processing
3. **Tag Processing**: Standard EXIF processing with custom PrintConv functions
4. **Face Detection**: Binary data processing when face info structures are present
5. **Video Handling**: Special processing for AVI and MOV formats

### 2. Special Processing Functions

- **Standard EXIF Procedures**: Uses `WriteExif`, `CheckExif` from ExifTool::Exif
- **Binary Data Processing**: `ProcessBinaryData`, `WriteBinaryData`, `CheckBinaryData` for face detection
- **Custom Print Conversions**: Complex PrintConv functions for firmware dates, coordinates, and mode mappings
- **Conditional Processing**: Extensive use of `Condition` parameters for model-specific behavior

## Key Features

### 1. BestShotMode Model Mapping
- **20+ Camera Models**: Individual BestShotMode mappings for specific camera models
- **File Type Sensitivity**: Different mappings for JPEG vs. MOV/AVI files on same model
- **Future Compatibility**: Generic handler for unknown models

### 2. Face Detection System
- **Dual Format Support**: FaceInfo1 (EX-H5) and FaceInfo2 (EX-ZR100/H20G) formats
- **Multi-Face Tracking**: Support for up to 10 detected faces
- **Coordinate Mapping**: Face positions relative to detection frame size
- **Orientation Handling**: Face orientation relative to image rotation

### 3. Firmware Date Processing
- **Custom Format**: Handles null-terminated date strings with special byte patterns
- **Year Calculation**: Smart year handling for 2-digit years (70+ = 1900s, <70 = 2000s)
- **Bidirectional Conversion**: PrintConv and PrintConvInv for read/write operations

### 4. Preview Image Support
- **Offset Pairs**: Coordinated length/offset pairs for preview image extraction
- **Protection**: Preview images marked as Protected level 2
- **Group Assignment**: Preview images properly grouped under 'Preview'

### 5. Video Metadata
- **Frame Rate Handling**: Complex frame rate conversion for various video modes
- **Quality Mapping**: Video quality settings including HD and Full HD support
- **Format Support**: AVI and MOV video file metadata

## Special Patterns

### 1. Model-Specific Conditional Processing
```perl
Condition => '$$self{Model} eq "EX-FC100"'
Condition => '$$self{Model} eq "EX-Z750" and $$self{FILE_TYPE} eq "JPEG"'
```

### 2. Multi-Format Tag Definitions
Tags that change behavior based on camera model using array references with conditional logic.

### 3. Binary Data with DataMembers
Face detection tables use DataMember pattern to store face count for conditional processing of subsequent tags.

### 4. Complex Print Conversions
Firmware date conversion with regex parsing and bidirectional conversion support.

### 5. EntryBased Offset Handling
PrintIM tag uses EntryBased offset calculation for specific older models.

## Usage Notes

### 1. Model Dependencies
- BestShotMode interpretation requires accurate model identification
- Many tags behave differently across camera generations
- File type (JPEG vs. video) affects tag availability and interpretation

### 2. Face Detection Limitations
- Face position offsets in FaceInfo1 table are marked as "NOT CONFIRMED" for faces 2-10
- Face detection data format varies significantly between camera generations
- Face coordinates are relative to detection frame, not full image

### 3. Firmware Date Handling
- Firmware dates contain embedded null bytes requiring special processing
- Some cameras may have multiple firmware date formats
- Invalid firmware date formats fall back to "Unknown" display

### 4. Preview Image Access
- Preview images are protected and require special handling
- Preview image offsets are relative to maker notes data
- Some models double-reference preview image data

## Debugging Tips

### 1. BestShotMode Issues
- Check exact model string matching - must be precise
- Verify file type for models with different JPEG/video mappings
- Unknown BestShotMode values display as numeric values

### 2. Face Detection Problems
- Use `-v` verbose mode to see binary data structure
- Check FacesDetected count vs. actual face position data
- Face detection may be disabled/unavailable in certain shooting modes

### 3. Firmware Date Parsing
- Invalid firmware dates show as "Unknown (...)" with raw data
- Check for proper null byte patterns in firmware date strings
- Some models may have different firmware date formats

### 4. Model Identification
- Use `exiftool -Model` to verify exact model string
- Model strings must match exactly for conditional processing
- Some models may have regional variations in naming

### 5. Binary Data Analysis
- Use `-htmlDump` to visualize binary maker notes structure
- Face detection data formats are camera-generation specific
- Binary offsets are fixed but may vary between camera models