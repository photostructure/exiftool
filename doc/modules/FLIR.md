# FLIR.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.23
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The FLIR.pm module provides comprehensive metadata extraction from FLIR Systems thermal imaging cameras, supporting multiple file formats including FFF, FPF, AFF, SEQ sequence files, JPEG maker notes, and MP4 videos. It implements advanced thermal imaging science with temperature calculations, environmental corrections, Planck radiation constants, and specialized hardware integration for professional thermal analysis applications.

## Module Structure

### Tag Tables (22 total)

1. **Main** - FLIR maker notes in JPEG images (12 fields)
   - Temperature ranges (max/min with unreliable units)
   - Emissivity settings and environmental data
   - Camera temperature calibration values

2. **FFF** - FLIR FFF format metadata blocks (15 record types)
   - Raw thermal data with multiple image formats
   - Camera calibration and environmental parameters
   - GPS location data and measurement tools
   - Picture-in-Picture overlay information

3. **RawData** - Raw thermal image processing (3 fields)
   - Automatic byte order detection and correction
   - PNG/TIFF format detection and conversion
   - TIFF header generation for raw 16-bit data

4. **CameraInfo** - Comprehensive camera metadata (50+ fields)
   - Complete Planck radiation constants for temperature calculation
   - Environmental correction parameters (atmospheric transmission)
   - Camera/lens/filter identification and serial numbers
   - Temperature ranges, clipping, and saturation limits

5. **GPSInfo** - Complete GPS metadata (18 fields)
   - Position, altitude, speed, and direction
   - GPS validation and coordinate conversion
   - Map datum and dilution of precision

6. **MeasInfo** - Measurement tool data (variable fields)
   - Dynamic tag generation for multiple measurement tools
   - Spot, area, ellipse, line, and alarm measurements
   - Coordinate systems and parameter extraction

7. **MeterLink** - Humidity meter integration (12 fields)
   - Multi-sensor humidity/temperature readings
   - Unit conversion (Kelvin→Celsius, kg/kg→g/kg)
   - Device identification and measurement types

8. **UserData** - MP4 video thermal metadata (8 UUID types)
   - Camera part numbers and serial numbers
   - Environmental parameters and GPS coordinates
   - Software components and thumbnail images

9. **Additional specialized tables** for: PaletteInfo, TextInfo, ParamInfo, PiP, EmbeddedImage, GainDeadData, CoarseData, PaintData, FPF format, AFF sequence format

### Key Data Structures

- **Temperature Conversion**: Planck constants and environmental correction algorithms
- **Record Directory**: FFF file record-based structure with type/offset/length
- **Byte Order Detection**: Automatic endianness detection using known values
- **Multi-Frame Support**: Sequence file processing with ExtractEmbedded
- **UUID Handlers**: Multiple UUID-based metadata extraction for MP4 videos
- **Measurement Coordinates**: Dynamic coordinate parsing for analysis tools

## Processing Architecture

### 1. Main Processing Flow

FLIR processing adapts to multiple format types:

1. **Format Detection**: Identifies FFF, FPF, AFF, SEQ, JPEG, or MP4 format
2. **Byte Order Detection**: Automatic endianness determination using version numbers
3. **Record Processing**: Reads record directory and processes by type
4. **Temperature Calibration**: Applies Planck constants and environmental corrections
5. **Multi-Frame Handling**: Processes sequence files with ExtractEmbedded support
6. **Image Processing**: Handles raw thermal data with format conversion

### 2. Special Processing Functions

- **ProcessFLIR**: Main FFF/AFF/SEQ file processor with record-based architecture
- **ProcessFPF**: FLIR Public Format processor for standardized thermal data
- **ProcessFLIRText**: Unicode text processor with escape sequence handling
- **ProcessMeasInfo**: Measurement tool coordinate and parameter extraction
- **GetImageType**: Image format detection with automatic TIFF header generation
- **UnescapeFLIR**: Unicode escape sequence decoder (\x1234 format)

## Key Features

### 1. Advanced Thermal Science Implementation
- **Planck Radiation Constants**: Complete implementation for temperature calculation
- **Environmental Corrections**: Atmospheric transmission, humidity, distance compensation
- **Temperature Conversion**: Kelvin ↔ Celsius with environmental adjustments
- **Emissivity Processing**: Material property compensation for accurate readings
- **Calibration Parameters**: Camera-specific calibration data extraction

### 2. Multi-Format Thermal File Support
- **FFF Format**: Primary FLIR format with record-based structure
- **FPF Format**: Public thermal format for interoperability
- **AFF Format**: Sequence file format for thermal video
- **SEQ Format**: Multi-frame thermal sequences
- **MP4 Integration**: Thermal metadata in MP4 video containers

### 3. Advanced Image Processing
- **Raw Thermal Data**: 16-bit raw temperature data with TIFF header generation
- **Format Detection**: Automatic PNG/JPEG/TIFF/raw format identification
- **Byte Order Correction**: Automatic endianness detection and correction
- **Multi-Frame Extraction**: Sequence processing with ExtractEmbedded support
- **Paint Data Processing**: FLIR Tools paint overlay data

### 4. Professional Hardware Integration
- **GPS Integration**: Complete GPS metadata with coordinate conversion
- **MeterLink Support**: External humidity meter data integration
- **Measurement Tools**: Analysis tool coordinate and parameter extraction
- **Picture-in-Picture**: Overlay positioning and scaling information
- **Palette Management**: Color mapping and thermal visualization data

### 5. Environmental Compensation System
- **Atmospheric Transmission**: Alpha/beta correction parameters
- **Humidity Compensation**: Relative humidity effects on thermal readings
- **Distance Correction**: Object distance compensation algorithms
- **Window Transmission**: IR window effects and transmission values
- **Reflected Temperature**: Background reflection compensation

### 6. Camera System Identification
- **Hardware Tracking**: Camera, lens, filter part numbers and serial numbers
- **Software Versioning**: Firmware and application version tracking
- **Calibration History**: Camera-specific calibration parameters
- **Optical Properties**: Field of view, focus distance, frame rate

## Special Patterns

### 1. Temperature Conversion with Environment
```perl
ValueConv => '$val - 273.15',  # Kelvin to Celsius
ValueConv => '$$self{TimecodeScale} ? $val * $$self{TimecodeScale} / 1e9 : $val',
```

### 2. Automatic Byte Order Detection
```perl
RawConv => 'ToggleByteOrder() if $val >= 0x0100; undef',
```

### 3. Multi-Format Image Processing
```perl
if ($val =~ /^\x89PNG\r\n\x1a\n/) { $type = 'PNG'; }
elsif ($val =~ /^\xff\xd8\xff/) { $type = 'JPG'; }
```

### 4. UUID-based MP4 Processing
```perl
Condition => '$$valPt=~/^\x43\xc3\x99\x3b\x0f\x94\x42\x4b\x82\x05\x6b\x66\x51\x3f\x48\x5d/s',
```

### 5. Dynamic Tag Generation
```perl
$$tagTablePtr{$tag} or AddTagToTable($tagTablePtr, $tag, { Name => $tag });
```

## Usage Notes

### 1. Temperature Data Interpretation
- Temperature units vary (Celsius, Kelvin, Fahrenheit) with no reliable indicator
- Planck constants enable accurate temperature calculation from raw data
- Environmental parameters required for accurate thermal analysis
- Camera calibration affects temperature accuracy

### 2. Multi-Format Processing
- FFF files typically little-endian, JPEG APP1 typically big-endian
- Automatic byte order detection prevents processing errors
- ExtractEmbedded option enables multi-frame sequence processing
- Some formats support both PNG and raw thermal data

### 3. Image Data Extraction
- Raw thermal data converted to TIFF format for compatibility
- PNG format thermal data may have incorrect byte order
- Image dimensions stored separately from image data
- Multiple image types possible in single file (visible + thermal)

### 4. Environmental Compensation
- Atmospheric correction requires distance, humidity, temperature data
- Emissivity values critical for accurate temperature readings
- IR window effects must be compensated for indoor use
- Reflected temperature affects thermal measurements

## Debugging Tips

### 1. Temperature Calculation Issues
- Verify Planck constants are available for temperature conversion
- Check environmental parameters for atmospheric correction
- Monitor temperature unit assumptions (no reliable unit indicator)
- Validate emissivity values for material properties

### 2. Byte Order Problems
- Check automatic byte order detection in RawDataByteOrder tags
- Verify version number validation for endianness determination
- Monitor ToggleByteOrder calls for proper switching
- Watch for PNG data with incorrect byte order

### 3. Multi-Format Processing
- Use verbose mode to see record types and processing flow
- Check file type detection and format-specific processing
- Monitor ExtractEmbedded behavior for sequence files
- Verify record directory parsing and offset calculations

### 4. Image Processing Issues
- Check image type detection (PNG/JPEG/TIFF/raw)
- Verify image dimensions match data length
- Monitor TIFF header generation for raw data
- Watch for format conversion warnings

### 5. GPS and Environmental Data
- Verify GPSValid flag before using coordinate data
- Check environmental parameter availability and units
- Monitor humidity meter data conversion and units
- Validate measurement tool coordinate extraction

### 6. MP4 Video Processing
- Check UUID pattern matching for metadata blocks
- Verify subdirectory processing for thermal data
- Monitor software component and version extraction
- Watch for thumbnail image extraction

### 7. Text Processing Debugging
- Check Unicode escape sequence handling (\x1234 format)
- Verify string termination and character encoding
- Monitor label and parameter text extraction
- Watch for measurement tool text processing

### 8. Hardware Integration
- Verify camera/lens/filter identification accuracy
- Check serial number and part number extraction
- Monitor MeterLink device data integration
- Validate measurement tool coordinate systems

### 9. Sequence File Processing
- Check multi-frame processing with ExtractEmbedded
- Monitor DOC_NUM handling for embedded documents
- Verify frame extraction and metadata continuity
- Watch for sequence file format variations

### 10. Professional Analysis Features
- Verify measurement tool type identification and coordinates
- Check palette and color mapping data extraction
- Monitor paint data and overlay information
- Validate Picture-in-Picture positioning data