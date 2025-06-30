# Red.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 1.01  
**Document Version:** 1.0  
**Last Updated:** 2025-06-30

## Overview

The Red module extracts metadata from Redcode R3D video files created by Red Digital Cinema cameras. It supports both version 1 and version 2 of the Redcode format, handling professional cinema camera metadata including camera settings, lens information, timecode data, and image characteristics. The module is specialized for high-end digital cinema production workflows.

## Module Structure

### Tag Tables (3 total)

**Main Processing:**

- `Main` - Primary tag table for Red directory tags with embedded format codes
- `RED1` - Redcode version 1 header structure (binary data processing)
- `RED2` - Redcode version 2 header structure (binary data processing)

### Key Data Structures

**Format Code System:**

- `%redFormat` - Maps format codes (0-9) to ExifTool data types
- Upper 4 bits of tag ID determine the format type
- Supports: int8u, string, float, int16u, int8s, int32s, int32u, undef

**Constants:**

- `$errTrunc` - Standardized truncation error message
- Format-specific tag ranges (0x1xxx = string, 0x2xxx = float, 0x4xxx = int16u, 0x6xxx = int32s)

### Complex Structures

**Block Structure:**

- Each block starts with int32u size + 4-byte type ("RED1" or "RED2")
- Version 2 files align blocks on 0x1000 byte boundaries
- Directory structure varies between versions

**Directory Processing:**

- Version 1: Directory starts at offset 0x22 in second block
- Version 2: Directory position calculated from "rdi" record count
- Variable-length tag entries with embedded format information

## Processing Architecture

### 1. Main Processing Flow

**File Recognition:**

- Validates header signature: `^\0\0..RED(1|2)`
- Extracts version number from header
- Verifies minimum block size (8 bytes)

**Processing Sequence:**

1. Read and validate file header
2. Extract header metadata (RED1 or RED2 subdirectory)
3. Locate Red directory (version-specific logic)
4. Process variable-length tag entries
5. Handle format-specific data extraction

### 2. Special Processing Functions

**ProcessR3D($$)**

- Primary entry point for R3D file processing
- Handles both version 1 and version 2 file formats
- Implements robust directory location logic
- Features fallback directory detection for unknown file variants

**Key Processing Features:**

- **Block Validation:** Verifies file structure integrity
- **Version Detection:** Automatic format version recognition
- **Directory Location:** Version-specific directory positioning
- **Format Decoding:** Dynamic format assignment from tag IDs
- **Error Recovery:** Fallback directory search for non-standard files

## Key Features

### 1. Professional Cinema Camera Support

**Camera Metadata:**

- Camera model, type, serial number identification
- Lens information (make, model, number, focal length, f-number)
- Focus distance with metric conversion
- Camera operator information

**Technical Parameters:**

- Image dimensions (width, height)
- Frame rate (original and effective)
- ISO settings
- Color temperature
- Crop area information
- RGB curves and color grading data

### 2. Production Workflow Integration

**Timecode Support:**

- Start timecode, reel timecode
- Date/time original with custom parsing
- Multiple date fields for production tracking
- Time created with formatting

**Production Metadata:**

- Reel number and take information
- Original filename preservation
- Storage type and model information
- Storage serial numbers and format dates
- Aspect ratio and video format

### 3. Advanced Format Handling

**Dynamic Format System:**

- Tag ID upper bits determine data format
- Supports 10 different data types
- Automatic format selection during processing
- Mixed-format structure support (format 7)

**Version Flexibility:**

- RED1 and RED2 format support
- Version-specific header processing
- Backward compatibility maintenance
- Different directory location strategies

## Special Patterns

### 1. Format Code Integration

The Red module uses a unique tag identification system:

```
Tag ID: 0xABCD
- Upper 4 bits (A): Format code (0-9)
- Lower 12 bits (BCD): Actual tag identifier
```

**Format Mappings:**

- 0x1xxx: String data (camera model, timecode, dates)
- 0x2xxx: Float data (frame rates, color temperature)
- 0x4xxx: 16-bit unsigned integers (ISO, dimensions)
- 0x6xxx: 32-bit signed integers (focus distance)

### 2. Directory Location Strategy

**Version 1 (RED1):**

- Directory in second block at fixed offset 0x22
- Requires reading additional data beyond header
- Simple offset-based location

**Version 2 (RED2):**

- Directory position calculated from "rdi" record count
- Formula: 0x44 + (record_count \* 0x18)
- More complex but more flexible structure

### 3. Robust Error Handling

**Fallback Directory Detection:**

- Primary method: Use calculated directory position
- Fallback method: Search for tag pattern `\0\x0f\x10\0`
- Graceful degradation for unknown file variants
- Warning system for non-standard files

## Usage Notes

### Camera Support

**Red Digital Cinema Cameras:**

- All models producing R3D files
- Professional cinema camera lineup
- High-end digital cinematography equipment

**File Format Coverage:**

- Redcode version 1 (RED1) - Earlier cameras
- Redcode version 2 (RED2) - Current cameras
- Both formats fully supported with version detection

### Metadata Extraction

**Critical Production Data:**

- Timecode information for post-production
- Camera settings for shot reconstruction
- Lens data for VFX and color work
- Storage information for asset management

**Date/Time Handling:**

- Multiple date format parsing
- Custom ValueConv for date normalization
- Time zone information preservation
- Production-specific time fields

## Debugging Tips

### Common Issues

**File Recognition Problems:**

- Verify R3D header signature
- Check for file truncation
- Ensure proper file transfer (binary mode)

**Directory Location Issues:**

- Version 2 files: Verify "rdi" record count
- Version 1 files: Check second block accessibility
- Use verbose mode (-v3) to see directory detection process

**Format Code Problems:**

- Unknown format warnings indicate new format codes
- Check upper 4 bits of problematic tag IDs
- May indicate new camera firmware or file version

### Advanced Debugging

**Directory Structure Analysis:**

- Enable verbose processing to see directory layout
- Check calculated vs. actual directory positions
- Verify directory size assumptions

**Format Code Debugging:**

- Monitor format assignments during processing
- Unknown formats may indicate file format evolution
- New cameras may introduce additional format codes

**Data Validation:**

- Check for reasonable frame rates and dimensions
- Verify date/time format consistency
- Validate lens and camera parameter ranges

The Red module demonstrates specialized handling of professional video formats with sophisticated format detection and robust error recovery for high-value production data.
