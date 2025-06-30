# Lytro.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 1.04  
**Document Version:** 1.0  
**Last Updated:** 2025-06-30

## Overview

The Lytro module extracts metadata from Lytro Light Field Picture (LFP) files, the proprietary format used by Lytro light field cameras. These revolutionary cameras capture not just a single image but the entire light field, enabling post-capture focus adjustment and depth-of-field control. The module processes JSON-format metadata blocks and embedded JPEG preview images, dynamically creating tags for the extensive metadata available in these files.

## Module Structure

### Tag Tables (1 total)

**Main Processing:**

- `Main` - Primary tag table with predefined tags plus dynamic tag creation

### Key Data Structures

**Core Tags:**

- Camera identification (Type, Make, Model, SerialNumber, FirmwareVersion)
- Optical parameters (FNumber, FocalLength, ExposureTime, ISO)
- Sensor data (PixelPitch, SensorSerial, Temperature readings)
- Accelerometer data (X/Y/Z arrays with timestamps)
- Image processing (ExposureBias, Orientation)

**Dynamic Tag Creation:**

- JSON metadata fields automatically converted to ExifTool tags
- Hierarchical tag names from nested JSON structures
- Automatic group assignment (Device vs Image tags)
- List handling for array-type values

**Special Processing:**

- JSONMetadata: Complete JSON blocks preserved as binary data
- EmbeddedImage: JPEG preview images extracted
- Temperature conversion to Celsius display
- Unit conversions (meters to mm, pixels/inch calculations)

## Processing Architecture

### 1. Main Processing Flow

**File Recognition:**

- Magic number validation: `^\x89LFP\x0d\x0a\x1a\x0a`
- LFP format verification
- Big-endian byte order (Motorola format)

**Segment Processing:**

1. Read 16-byte segment headers with magic `^\x89LF`
2. Extract segment size and validate (max 20MB with MSB check)
3. Skip 80-byte SHA1 hash section
4. Process segment data based on content type
5. Handle 16-byte boundary alignment with padding

### 2. Special Processing Functions

**ProcessLFP($$)**

- Primary entry point for LFP file processing
- Implements segmented file format parsing
- Handles large file seeking for oversized segments
- Provides verbose segment information with ID display
- Manages boundary alignment and padding

**ExtractTags($$$)**

- Recursive JSON metadata extraction
- Dynamic tag name generation from hierarchical keys
- Automatic tag table registration for unknown tags
- Group assignment based on tag prefix patterns
- Array handling for multi-value fields

**Key Processing Features:**

- **Segment Validation:** Size limits and format verification
- **Dynamic Registration:** Automatic tag creation for JSON fields
- **Hierarchical Processing:** Recursive hash traversal
- **Content Detection:** JSON vs JPEG embedded content recognition

## Key Features

### 1. Light Field Camera Technology Support

**Unique Metadata:**

- Light field-specific parameters
- Post-capture processing capabilities
- Depth map information
- Focus plane data

**Camera Parameters:**

- Lens temperature monitoring
- SoC (System on Chip) temperature
- Frame vs pixel exposure timing
- Accelerometer data for stabilization

### 2. Advanced JSON Processing

**Dynamic Tag Creation:**

- Automatic conversion of JSON field names to tag names
- Hierarchical key flattening (DevicesLensFocalLength)
- Special processing for vendor-specific content
- Group classification based on data source

**Metadata Preservation:**

- Complete JSON blocks preserved as binary data
- Individual tag extraction for searchability
- Array handling for multi-sample data
- Nested structure processing

### 3. Embedded Content Extraction

**JPEG Preview Support:**

- Automatic JPEG detection in segments
- Preview image extraction for quick viewing
- Binary data preservation
- Group assignment to Preview category

**Large File Handling:**

- Segment size validation (20MB limit)
- Efficient seeking for oversized segments
- Memory-conscious processing
- Truncation error detection

## Special Patterns

### 1. LFP File Format Structure

```
LFP File Format:
[16-byte header with magic number]
[Segment 1: 16-byte header + 80-byte SHA1 + data + padding]
[Segment 2: 16-byte header + 80-byte SHA1 + data + padding]
...
```

**Segment Types:**

- JSON metadata segments (identified by opening brace)
- JPEG image segments (identified by JPEG SOI marker)
- Binary data segments (unknown format)

### 2. Dynamic Tag Generation

**Tag Name Processing:**

- Hierarchical JSON keys flattened to single tag names
- Special handling for vendor content paths
- Automatic capitalization and formatting
- Invalid character removal and replacement

**Group Assignment:**

- Device-prefixed tags → Camera group
- Other tags → Image group
- Array detection → List flag set
- Automatic name cleanup for readability

### 3. Light Field Specific Features

**Temperature Monitoring:**

- Lens temperature for thermal compensation
- SoC temperature for performance monitoring
- Celsius conversion for display

**Exposure Timing:**

- Frame exposure vs pixel exposure distinction
- Light field-specific timing parameters
- Accurate time display formatting

## Usage Notes

### Camera Support

**Lytro Models:**

- Original Lytro camera
- Lytro Illum (professional model)
- Any device producing LFP files

**Light Field Technology:**

- Post-capture focus adjustment capability
- Depth-of-field modification after shooting
- 3D information capture
- Computational photography features

### Metadata Types

**Technical Parameters:**

- Optical settings (focal length, f-number, ISO)
- Exposure settings (shutter speed, exposure bias)
- Sensor characteristics (pixel pitch, serial number)
- Temperature monitoring for calibration

**Sensor Data:**

- 3-axis accelerometer readings
- Timestamp correlation for motion analysis
- Multi-sample data arrays
- Stabilization reference data

## Debugging Tips

### Common Issues

**File Format Problems:**

- Verify LFP magic number in header
- Check segment magic numbers (LF\*)
- Validate segment size calculations
- Ensure proper 16-byte alignment

**JSON Parsing Issues:**

- Malformed JSON in metadata segments
- Encoding problems with special characters
- Nested structure depth limitations
- Array vs object type mismatches

**Memory Issues:**

- Large segment size handling (>20MB limit)
- JSON parsing memory requirements
- Embedded JPEG size considerations
- Dynamic tag table growth

### Advanced Debugging

**Segment Analysis:**

- Enable verbose mode to see segment details
- Check segment ID strings for content type
- Verify SHA1 hash section skipping
- Monitor padding calculations

**Tag Generation:**

- Track dynamic tag creation process
- Verify hierarchical name flattening
- Check group assignment logic
- Monitor tag table registration

**Content Detection:**

- Validate JSON format recognition
- Check JPEG magic number detection
- Verify binary data handling
- Monitor content type classification

The Lytro module demonstrates specialized handling of innovative light field photography technology with sophisticated JSON metadata processing and dynamic tag generation capabilities.
