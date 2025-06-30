# Exif.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 4.38  
**Document Version:** 1.0  
**Last Updated:** 2025-06-29

## Overview

The Exif module is ExifTool's foundational EXIF/TIFF processing engine, providing core functionality for reading and writing metadata in EXIF, TIFF, and derivative formats. It serves as the architectural base for processing hundreds of camera and image formats.

## Module Structure

### Tag Tables (4 major)

**Primary Tables:**
- **%Exif::Main** - Core EXIF/TIFF tags (850+ entries)
  - Standard EXIF 2.32/3.0 tags
  - TIFF 6.0 tags
  - Manufacturer extension points
  - GPS/Interop subdirectory links

- **%Exif::printParameter** - Print conversion definitions
  - Human-readable value mappings
  - Conditional conversions
  - Complex parameter formatting

- **%Exif::Composite** - Derived/calculated tags
  - Composite calculations
  - Cross-tag dependencies
  - Virtual tag generation

- **%Exif::Unknown** - Unknown tag handling
  - Fallback processing
  - Format detection
  - Value interpretation

### Format System

**EXIF/TIFF Format Types:**
```perl
1  => 'int8u'       # BYTE
2  => 'string'      # ASCII
3  => 'int16u'      # SHORT
4  => 'int32u'      # LONG
5  => 'rational64u' # RATIONAL
6  => 'int8s'       # SBYTE
7  => 'undef'       # UNDEFINED
8  => 'int16s'      # SSHORT
9  => 'int32s'      # SLONG
10 => 'rational64s' # SRATIONAL
11 => 'float'       # FLOAT
12 => 'double'      # DOUBLE
13 => 'ifd'         # IFD
# BigTIFF extensions:
16 => 'int64u'      # LONG8
17 => 'int64s'      # SLONG8
18 => 'ifd64'       # IFD8
```

## Processing Architecture

### 1. IFD Processing Flow
```perl
ProcessExif($$$)
  ├── Validate TIFF header
  ├── Detect byte order
  ├── Process IFD chain
  │   ├── IFD0 (main image)
  │   ├── IFD1 (thumbnail)
  │   └── SubIFDs
  └── Handle maker notes
```

### 2. Tag Processing

**ProcessTiffIFD($$$)**
- Reads IFD entry count
- Processes each tag entry
- Handles value offsets
- Manages subdirectories
- Links to next IFD

**Value Extraction:**
- In-line values (≤4 bytes)
- Offset-based values (>4 bytes)
- Format-specific conversion
- Byte order handling

### 3. Subdirectory System

**Standard Subdirectories:**
```perl
0x8769 => 'ExifIFD'      # Camera settings
0x8825 => 'GPS'          # Location data
0x927c => 'MakerNotes'   # Manufacturer data
0xa005 => 'InteropIFD'   # Interoperability
```

## Key Features

### Offset Management

**Pointer Resolution:**
- Base offset tracking
- Relative vs absolute offsets
- Offset validation
- Circular reference detection

**Image Data Offsets:**
- StripOffsets/StripByteCounts
- TileOffsets/TileByteCounts
- PreviewImageStart/Length
- ThumbnailOffset/Length

### Text Encoding

**Multi-Language Support:**
- ASCII (default)
- Unicode (UCS-2)
- JIS (Japanese)
- UTF-8 (EXIF 3.0)

**Conversion Functions:**
- `ConvertExifText()` - Reading
- `EncodeExifText()` - Writing
- Character set detection
- Encoding validation

### Maker Note Integration

**Detection System:**
- Camera make/model identification
- Format detection
- Offset calculation
- Dispatch to MakerNotes.pm

**Format Types:**
- Standard IFD format
- Proprietary binary
- Offset-based
- Encrypted

### Validation System

**Data Integrity:**
- `CheckExif()` - Structure validation
- `ValidateIFD()` - IFD verification
- Format checking
- Offset boundary validation

## Foundation Architecture

### Core Patterns

**Tag Definition:**
```perl
0x010f => {
    Name => 'Make',
    Groups => { 2 => 'Camera' },
    Writable => 'string',
    WriteGroup => 'IFD0',
},
```

**Subdirectory:**
```perl
0x8769 => {
    Name => 'ExifOffset',
    Groups => { 1 => 'ExifIFD' },
    Flags => 'SubIFD',
    SubDirectory => {
        TagTable => 'Image::ExifTool::Exif::Main',
        DirName => 'ExifIFD',
        Start => '$val',
    },
},
```

### Extensibility

**Format Independence:**
- Used by DNG, CR2, NEF, etc.
- Consistent IFD processing
- Shared tag definitions
- Common validation

**Write System:**
- Preserves unknown tags
- Maintains structure
- Handles relocations
- Updates offsets

## Special Considerations

### BigTIFF Support
- 64-bit offsets
- Large file handling
- Extended format codes
- Backward compatibility

### EXIF 3.0 Features
- UTF-8 text support
- Extended tag space
- New format types
- Enhanced GPS tags

### Performance
- Lazy subdirectory loading
- Efficient offset handling
- Minimal memory footprint
- Fast validation

## Usage Notes

- Foundation for all TIFF-based formats
- Extensive validation capabilities
- Comprehensive text encoding
- Maker note integration point
- Standard compliance focus

## Debugging Tips

- Use `-v2` for IFD structure
- Check `-validate` for integrity
- Examine offsets with `-htmlDump`
- Text encoding visible with `-v`
- Maker note detection in `-v3`