# TIFF Processing Module Outline

**ExifTool Version:** 13.26
**Module:** Exif.pm (implements TIFF/EXIF processing)
**Module Version:** 4.56
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

TIFF (Tagged Image File Format) processing in ExifTool is implemented within the **Exif.pm** module, as TIFF and EXIF share the same underlying binary structure. The module handles reading, writing, and validating TIFF/EXIF metadata across all supported image formats that use TIFF-based structures.

**Key Insight**: There is no standalone TIFF.pm module - TIFF functionality is integrated into Exif.pm because EXIF is essentially TIFF with additional standardized tags for digital photography.

## Module Structure

### Tag Tables (4 total)

1. **%Image::ExifTool::Exif::printParameter** (line 317)
   - Handles DNG processing parameters and opcodes
   - Contains specialized print conversion functions

2. **%Image::ExifTool::Exif::Main** (line 387) 
   - Primary TIFF/EXIF tag table with ~1,363 tag definitions
   - Groups: EXIF/IFD0/Image, with dynamic group assignment
   - Covers standard TIFF tags, EXIF tags, and manufacturer extensions

3. **%Image::ExifTool::Exif::Composite** (line 4636)
   - Computed tags derived from multiple source tags
   - Complex calculations and value combinations

4. **%Image::ExifTool::Exif::Unknown** (line 5280)
   - Fallback handling for unrecognized TIFF tags
   - Preserves unknown metadata during read/write operations

### Key Data Structures

- **@formatSize**: Byte sizes for TIFF data formats (1,1,2,4,8,1,1,2,4,8,4,8,4,2,8,8,8,8)
- **@formatName**: Human-readable format names (int8u, string, int16u, int32u, rational64u, etc.)
- **%formatNumber**: Reverse lookup hash for format types by name
- **%subfileType**, **%compression**, **%photometricInterpretation**: Standard TIFF lookup tables
- **%opcodeInfo**: DNG processing opcode definitions
- **Binary data limit**: 10MB maximum for in-memory processing

## Processing Architecture

### 1. Main Processing Flow

**Entry Point**: `ProcessExif($$$)` (line 6169)
- Handles both TIFF and EXIF IFD processing
- Manages base offsets, byte order, and data positioning
- Supports nested IFD structures and maker notes integration

**Core IFD Processing**: `ProcessTiffIFD($$$)`
- Recursive directory parsing
- Tag validation and format checking  
- Offset resolution and data extraction

### 2. Special Processing Functions

- **WriteExif($$$)**: Complete TIFF/EXIF writing with structure preservation
- **CheckExif($$$)**: Tag validation and format verification
- **ValidateIFD($;$)**: IFD structure integrity checking  
- **ValidateImageData($$$;$)**: Image data consistency validation
- **RebuildMakerNotes($$$)**: Maker notes reconstruction during writing
- **AddImageDataHash($$$)**: Image data hash calculation for verification
- **EncodeExifText($$)**: Character encoding handling for text tags

## Key Features

### TIFF Format Support
- **Standard TIFF**: Full IFD0/IFD1 chain processing
- **BigTIFF**: 64-bit extensions via separate BigTIFF.pm module
- **Multi-page TIFF**: Page counting and subfile type detection
- **Compressed TIFF**: All standard compression schemes

### Advanced Capabilities
- **Offset Management**: Complex pointer resolution across nested structures
- **Byte Order Handling**: Automatic little/big endian detection and conversion
- **Format Validation**: Strict TIFF specification compliance checking
- **Data Preservation**: Unknown tag and structure preservation during editing

### Maker Notes Integration
- Seamless integration with manufacturer-specific modules
- Dynamic loading of camera-specific processing functions
- Offset adjustment for relocated maker notes data

## Special Patterns

### TIFF-Specific Patterns

1. **IFD Chain Processing**:
   ```perl
   # Recursive IFD traversal with offset management
   while ($nextIFD) {
       ProcessTiffIFD($et, $dirInfo, $tagTablePtr);
       $nextIFD = GetNextIFD();
   }
   ```

2. **Format Validation**:
   ```perl
   # Strict format checking against TIFF specification
   return unless $format >= 1 and $format <= 18;
   $format = $formatName[$format] or return;
   ```

3. **Offset Resolution**:
   ```perl
   # Complex offset calculation with base adjustments
   my $valuePtr = $base + $dirStart + $valueOffset;
   ```

### Integration Patterns
- **Modular Architecture**: Clean separation between format handling and tag processing
- **Dynamic Dispatch**: Runtime loading of format-specific functions
- **Error Recovery**: Graceful degradation when encountering corrupt TIFF structures

## Usage Notes

### Performance Considerations
- **Memory Management**: 10MB limit for binary data blocks prevents excessive memory usage
- **Lazy Loading**: Tag tables loaded on demand to reduce startup time
- **Offset Caching**: Smart caching of frequently accessed offset calculations

### Compatibility Notes
- **EXIF 2.3/3.0 Support**: Latest EXIF specifications including UTF-8 string support
- **Legacy TIFF**: Maintains compatibility with historical TIFF variations
- **Cross-Platform**: Handles both little-endian and big-endian TIFF files

### Writing Behavior
- **Structure Preservation**: Maintains original IFD ordering and unknown tags
- **Validation**: Automatic format validation prevents corruption
- **Backup Creation**: Safe writing with automatic backup generation

## Debugging Tips

### Common Issues
- **Offset Corruption**: Use `-htmlDump` to visualize IFD structure and offset chains
- **Format Mismatches**: Enable `-validate` to catch TIFF specification violations
- **Maker Notes Problems**: Check offset adjustments with verbose output (`-v3`)

### Diagnostic Commands
```bash
# Dump complete TIFF structure
exiftool -htmlDump image.tif > structure.html

# Validate TIFF compliance  
exiftool -validate -warning image.tif

# Show processing details
exiftool -v3 image.tif
```

### Memory Issues
- Monitor processing of large TIFF files (>10MB binary blocks)
- Use streaming mode for extremely large files
- Check for circular IFD references in corrupted files

## File Relationships

- **BigTIFF.pm**: Handles 64-bit TIFF extensions
- **GeoTiff.pm**: Geographic TIFF tag processing
- **MakerNotes.pm**: Manufacturer-specific TIFF extensions
- **WriteExif.pl**: TIFF writing implementation
- **Validate.pm**: TIFF validation functions

The Exif.pm module serves as the foundation for all TIFF-based metadata processing in ExifTool, making it one of the most critical and complex modules in the entire system.