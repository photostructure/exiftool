# Olympus.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 2.85
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

Olympus.pm handles Olympus and Epson EXIF maker notes tags. This is one of the most comprehensive camera manufacturer modules, supporting extensive lens databases, image processing parameters, camera settings, and special video/audio formats. It covers both DSLR (Four Thirds) and mirrorless (Micro Four Thirds) systems, plus specialized video formats from Olympus DSS recorders.

## Module Structure

### Tag Tables (29 total)

**Primary Tables:**
1. **Main** - Primary maker notes table (576+ hex tag definitions)
2. **Equipment** - Camera and lens information
3. **CameraSettings** - Shooting parameters and settings
4. **RawDevelopment** / **RawDevelopment2** - RAW processing parameters
5. **ImageProcessing** - Image enhancement settings
6. **FocusInfo** - Autofocus system information
7. **AFInfo** - AF point and sensor data
8. **RawInfo** - RAW file technical information

**Specialized Tables:**
9. **TextInfo** - Text annotations and keywords
10. **UnknownInfo** - Unidentified maker note blocks
11. **FETags** - File extension information
12. **MOV1**, **MOV2**, **MOV3** - QuickTime video metadata variants
13. **MP4** - MP4 video container metadata
14. **OLYM2** - Secondary Olympus video format
15. **prms**, **thmb2**, **scrn2** - Video parameter, thumbnail, and screen data

**Additional specialized tables for various camera generations and formats**

### Key Data Structures

- **%olympusLensTypes**: Comprehensive lens database with 200+ entries (Four Thirds and Micro Four Thirds)
- **%offOn**: Standard on/off value conversion hash
- **Conditional tag processing**: Extensive model-specific tag definitions
- **Multi-format support**: JPEG, ORF (RAW), MOV, AVI, DSS audio

## Processing Architecture

### 1. Main Processing Flow

**Standard EXIF Processing**: Uses TIFF-based maker note structure with write support via `WriteExif` and `CheckExif` procedures.

**Special Format Handlers**:
- **ProcessDSS()**: Processes Olympus Digital Speech Standard audio files
- **ProcessORF()**: Handles Olympus RAW format files
- **Video Processing**: Multiple QuickTime-based handlers for camera video

### 2. Special Processing Functions

- **PrintLensInfo()**: Formats comprehensive lens information strings
- **ExtenderStatus()**: Determines teleconverter attachment status using aperture analysis
- **PrintAFAreas()**: Renders AF point visualization data
- **ProcessDSS()**: Audio file metadata extraction
- **ProcessORF()**: ORF raw file processing

## Key Features

### Comprehensive Lens Database
- **Four Thirds System**: Complete catalog of Zuiko Digital lenses
- **Micro Four Thirds**: All M.Zuiko Digital lenses with version tracking
- **Third-party Support**: Sigma, Tamron, Panasonic, and other manufacturers
- **Teleconverter Detection**: Automatic identification of attached teleconverters
- **Lens ID Validation**: Cross-references lens codes with actual lens specifications

### Advanced Camera Settings
- **Image Quality Modes**: Multiple quality settings including Art Filter modes
- **Image Stabilization**: Multi-mode stabilization system tracking
- **Advanced Scene Modes**: Comprehensive scene and filter mode support
- **High-Resolution Modes**: Tripod and handheld high-resolution shooting
- **Focus Stacking**: Multi-image focus compositing support
- **Live Composite**: Long exposure compositing modes

### Multi-Format Support
- **Still Images**: JPEG, ORF (Olympus RAW Format)
- **Video**: MOV, MP4, AVI containers with Olympus-specific metadata
- **Audio**: DSS (Digital Speech Standard) professional audio format
- **Thumbnails**: Multiple embedded thumbnail formats

### Professional Features
- **Art Filters**: Complete art filter effect catalog
- **Bracketing Modes**: Exposure, white balance, ISO, and focus bracketing
- **HDR Processing**: Multiple HDR mode variants
- **Panorama**: Multi-directional panorama stitching information
- **ND Filter Simulation**: Digital neutral density filter effects

## Special Patterns

### Conditional Tag Processing
```perl
# Model-specific tag variations
{
    Condition => '$$self{Model} =~ /E-M1/',
    Name => 'SpecialE-M1Feature',
    # E-M1 specific processing
},
{
    Condition => '$$self{Model} !~ /E-M1/',
    Name => 'StandardFeature', 
    # Other models
}
```

### Complex Value Conversion
```perl
# Multi-value processing with complex logic
PrintConv => {
    '0 0' => 'No',
    '1 *' => 'Live Composite (* images)',
    '8 8' => 'Tripod high resolution',
    OTHER => sub {
        # Complex conversion logic for special cases
    },
}
```

### Lens Information Formatting
- Combines multiple data sources (lens type, teleconverter, serial numbers)
- Validates lens/body compatibility
- Provides detailed lens specifications including aperture ranges

## Usage Notes

### Supported Models
- **DSLR Systems**: E-1, E-3, E-5, E-30, E-450, E-520, E-620, etc.
- **Mirrorless Systems**: E-P1/P2/P3/P5, E-PL1/PL2/PL3, E-M1/M5/M10, OM-D series
- **Compact Cameras**: XZ-1, SZ series, TG series
- **Professional Audio**: DS-series digital recorders
- **Vintage Support**: Some legacy Olympus digital cameras

### Video Metadata
- QuickTime-based video files with Olympus extensions
- Professional video features (time code, GPS, audio levels)
- Multiple video format variants (MOV1, MOV2, MOV3, MP4)

### Write Capabilities
- Full maker note writing support
- Lens information updates
- Camera settings modifications
- Video metadata editing

## Debugging Tips

### Common Issues
1. **Lens Identification**: Some third-party lenses may not be recognized
2. **Model Variations**: Different firmware versions may use different tag layouts
3. **Video Format Detection**: Multiple similar video formats require careful identification
4. **Teleconverter Detection**: Manual lenses may not report teleconverter status correctly

### Diagnostic Features
- Extensive conditional processing for model-specific variations
- Unknown info table captures unrecognized maker note data
- Detailed lens specification validation
- AF point visualization for troubleshooting focus issues

### Special Considerations
- **Art Filters**: May affect image processing parameter interpretation
- **High-Resolution Modes**: Generate multiple related images requiring special handling
- **Live Composite**: Time-based image accumulation affects timestamp interpretation
- **DSS Audio**: Professional audio features require specialized processing

## Model-Specific Features

### Professional Models (E-1, E-3, E-5)
- Advanced white balance fine-tuning
- Professional-grade image processing options
- Extensive custom function support

### Mirrorless Systems (E-M1, E-M5, E-M10 series)
- In-body image stabilization (IBIS) information
- Electronic viewfinder settings
- Touch screen AF settings
- Art filter processing details

### Compact Cameras
- Simplified settings structure
- Scene-mode focused operation
- Reduced lens information complexity

## References

Multiple sources including Olympus official documentation, reverse engineering efforts, and extensive user testing with various camera models from E-1 through current OM-D series.