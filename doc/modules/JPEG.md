# JPEG.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.39
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The JPEG.pm module handles definitions for uncommon JPEG segments. Unlike common JPEG segments which are defined in the main ExifTool.pm module for performance reasons, this module provides comprehensive support for specialized JPEG application segments (APP0-APP15), comment segments, Start of Frame, quantization tables, and various trailer formats.

## Module Structure

### Tag Tables (12 total)

1. **Main** - Primary dispatch table for all JPEG segments
2. **JPS** - JPEG Stereo (3D) image format support
3. **EPPIM** - Toshiba PrintIM in APP6 segments
4. **SPIFF** - Still Picture Interchange File Format (APP8)
5. **MediaJukebox** - Media Jukebox XML metadata (APP9)
6. **HDR** - JPEG-HDR high dynamic range format (APP11)
7. **AdobeCM** - Adobe Color Management (APP13)
8. **Adobe** - Adobe DCT encoding information (APP14)
9. **GraphConv** - GraphicConverter quality information (APP15)
10. **AVI1** - AVI video frame information (APP0)
11. **Ocad** - Photobucket image metadata (APP0)
12. **NITF** - National Imagery Transmission Format (APP6)

### Key Data Structures

- **Main Tag Table**: Complex conditional structure dispatching to specialized handlers based on segment signatures
- **Segment Identification**: Uses pattern matching against binary signatures (`$$valPt =~ /^signature/`)
- **Conditional Processing**: Manufacturer-specific conditions (e.g., DJI thermal data)
- **Binary Data Processing**: Format-specific structures with defined offsets and data types

## Processing Architecture

### 1. Main Processing Flow

The module uses ExifTool's standard segment processing mechanism:

- Segments are identified by APP number and content signature
- Conditional matching routes to appropriate sub-tables
- Each segment type has specialized processing requirements
- Binary data is parsed using ProcessBinaryData where applicable

### 2. Special Processing Functions

- **ProcessOcad()** `lib/Image/ExifTool/JPEG.pm:752`: Parses Photobucket metadata with `$key:value` format
- **ProcessJPEG_HDR()** `lib/Image/ExifTool/JPEG.pm:771`: Extracts HDR metadata and embedded ratio images

## Key Features

### Multi-Format Support

- **Stereo Images**: JPS format with layout detection (side-by-side, over-under, interleaved)
- **HDR Images**: JPEG-HDR with embedded ratio images and calibration data
- **Thermal Imaging**: DJI and InfiRay thermal camera support across multiple APP segments
- **Professional Video**: NITF military imaging standard support

### Manufacturer-Specific Processing

- **DJI Thermal**: Multi-segment thermal data (APP3-APP5) with calibration
- **InfiRay**: Comprehensive thermal imaging support across APP2-APP7
- **Samsung**: Preview images and unique ID handling
- **Canon/Nikon**: Raw format detection and maker notes
- **Adobe**: DCT encoding and color management information

### Trailer Detection

- **Professional Tools**: Canon VRD, FotoStation, PhotoMechanic
- **Mobile Platforms**: Samsung, Vivo, OnePlus, Google
- **Specialized Formats**: MIE, AFCP, Insta360, Sony hidden data

## Special Patterns

### Conditional Segment Recognition

```perl
APP1 => [{
    Name => 'EXIF',
    Condition => '$$valPt =~ /^Exif\0/',
    SubDirectory => { TagTable => 'Image::ExifTool::Exif::Main' },
}, {
    Name => 'XMP',
    Condition => '$$valPt =~ /^http/ or $$valPt =~ /<exif:/',
    SubDirectory => { TagTable => 'Image::ExifTool::XMP::Main' },
}]
```

### Binary Data Extraction

- Format specification for multi-byte values
- Conditional tag processing based on image metadata
- Dynamic offset calculation using hooks
- Manufacturer detection for specialized processing

### Complex Trailer Matching

- Multiple signature patterns for different manufacturers
- Embedded video detection in trailers
- Proprietary format recognition (e.g., Insta360 UUID)

## Usage Notes

### Segment Priority

APP segments are processed in numerical order (APP0-APP15), with multiple handlers per segment type. The first matching condition determines the processing path.

### Performance Considerations

- Common JPEG segments are handled in main ExifTool.pm for speed
- This module focuses on specialized/uncommon formats
- Binary processing used where appropriate for efficiency

### Manufacturer Detection

Several segments require manufacturer identification (`$$self{Make}`) before processing, particularly for thermal imaging applications.

## Debugging Tips

### Segment Identification Issues

- Use `-v` flag to see segment recognition process
- Check binary signature matching with hex dumps
- Verify manufacturer tags are extracted before APP segment processing

### Thermal Imaging Data

- DJI thermal data spans multiple APP segments (3-5)
- InfiRay detection requires HasIJPEG flag from APP2
- Thermal data is often stored as binary blobs

### Trailer Processing

- Trailers are processed after main image data
- Multiple trailer formats may be present
- Some trailers contain embedded images or video

### Common Issues

- **Truncated segments**: Some cameras write incomplete APP data
- **Non-standard locations**: HP writes FPXR in APP4 instead of APP2
- **Manufacturer variations**: Similar formats with different signatures (Samsung variants in APP2)
