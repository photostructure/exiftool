# MinoltaRaw.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.20
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

MinoltaRaw.pm handles reading and writing Konica-Minolta RAW (MRW) metadata information. This module is specialized for the MRW file format used by Minolta cameras and some early Sony cameras. It processes structured binary data segments within MRW files and can also handle MRW information embedded in other formats like Sony ARW files.

## Module Structure

### Tag Tables (4 total)

1. **Main** - Primary MRW segment table with 4-character segment identifiers
2. **PRD** - Picture Raw Dimensions (sensor and image size information)
3. **WBG** - White Balance Gains (color correction data)
4. **RIF** - Requested Image Format (image processing parameters)

### Key Data Structures

- **MRW Segment Headers**: 4-character segment identifiers ("\0TTW", "\0PRD", "\0WBG", "\0RIF")
- **Binary Data Structures**: Fixed-format binary data with specific byte layouts
- **Model-Specific Conditionals**: Different processing based on camera model (DiMAGE A200, Sony A100, etc.)

## Processing Architecture

### 1. Main Processing Flow

**ProcessMRW()** - Main entry point that processes MRW format files:

- Reads MRW header ("\0MRM" for big-endian, "\0MRI" for little-endian)
- Iterates through segments with 4-character identifiers and length fields
- Dispatches to appropriate sub-processors based on segment type
- Handles both reading and writing operations
- Supports MRW files embedded in other formats (ARW)

### 2. Special Processing Functions

- **WriteMRW()**: Writes MRW directory structure, used for embedded MRW in ARW files
- **ConvertWBMode()**: Converts white balance mode values with fine-tuning offsets
- **ProcessMRW()**: Main processor handling segment iteration and data extraction

## Key Features

### MRW Format Support

- Handles both standalone MRW files and embedded MRW segments in ARW files
- Automatic endianness detection and handling
- Segment-based processing with proper validation

### Camera-Specific Adaptations

- **DiMAGE A200**: Special white balance level format (GBRG vs RGGB)
- **Sony A100**: Extended white balance settings, special color modes
- **Sony A200/A700**: Alternative color temperature positioning
- **Minolta vs Sony**: Different tag positioning and availability

### Binary Data Processing

- **PRD Segment**: Sensor dimensions, image size, bit depth, Bayer pattern
- **WBG Segment**: White balance scaling factors and color channel levels
- **RIF Segment**: Image processing parameters (saturation, contrast, sharpness, ISO, color mode)

### White Balance Handling

- Multiple white balance presets (Tungsten, Daylight, Cloudy, Flash, etc.)
- Model-specific RB level formats
- Fine-tuning offset support in WB mode conversion

## Special Patterns

### Segment Processing Pattern

```perl
# Main MRW segment loop
while ($pos < $offset) {
    # Read 8-byte header (4-byte tag + 4-byte length)
    my $tag = substr($data, 0, 4);
    my $len = Get32u(\$data, 4);
    # Process segment data based on tag
}
```

### Conditional Tag Processing

```perl
# Model-specific tag definitions
{
    Condition => '$$self{Model} eq "DSLR-A100"',
    # Sony A100 specific processing
},
{
    Condition => '$$self{Make} !~ /^SONY/',
    # Minolta-specific processing
}
```

### Binary Data Layout

- Fixed-offset binary data structures
- Format specifications (int8u, int16u, int32u, string[8])
- FIRST_ENTRY => 0 for zero-based indexing

## Usage Notes

### Supported Formats

- **MRW Files**: Minolta RAW format (standalone files)
- **ARW Files**: Sony ARW files with embedded MRW segments
- **Byte Order**: Automatic detection (big-endian MRM, little-endian MRI)

### Writing Capabilities

- Full write support for all tag tables
- Proper segment reconstruction during writing
- Padding to 4-byte boundaries for compatibility

### Model Compatibility

- **Minolta**: DiMAGE series cameras
- **Sony**: Early DSLR-A series (A100, A200, A700)
- **Backward Compatibility**: Handles legacy Minolta lens information

## Debugging Tips

### Common Issues

1. **Endianness Problems**: Check MRW header format (MRM vs MRI)
2. **Segment Length Errors**: Verify segment length matches actual data
3. **Model Detection**: Ensure correct camera model for conditional processing
4. **Wrong Base Offset**: Module handles offset correction with MRW_WrongBase

### Diagnostic Information

- Use `-v` flag to see MRW segment processing details
- Check for "MRW format error" warnings
- Verify segment offsets and lengths align properly

### File Structure Validation

- Total metadata length should match segment data
- All segments should be properly aligned
- Embedded MRW in ARW may have different base offsets

## References

1. http://www.cybercom.net/~dcoffin/dcraw/ - DCRaw source code
2. http://www.chauveau-central.net/mrw-format/ - MRW format specification
3. Igal Milchtaich private communication (A100 testing)
